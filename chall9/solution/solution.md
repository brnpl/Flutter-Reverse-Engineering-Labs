# Challenge 9: Release Mode - Hooking Flutter Third-Party Library
*This app was built with the newer Flutter 3.38.7 version, compared to the previous challenges.*

## Overview

Welcome back! In this challenge, we'll dive into how third-party libraries work in Flutter, with a particular focus on anti-root libraries bundled into an application built in release mode. Upon launching the application, you are greeted with a simple welcome screen displaying the message: "Sorry, rooted device! Checked with Rjsniffer".

![alt text](./_images/chall9_1.png)

## Analysis
Searching online for the string shown on screen ("rjsniffer") quickly leads to the `root_jailbreak_sniffer` package on [pub.dev](https://pub.dev/packages/root_jailbreak_sniffer), a Flutter plugin for root (Android) and jailbreak (iOS) detection. Since the plugin is open source, its [Github](https://github.com/romikavinda/root_jailbreak_sniffer) repository can be used to understand exactly how the Dart-facing API maps down to platform code, before even opening the APK.

The public API is intentionally small, three static methods: `amICompromised`, `amIEmulator` and `amIDebugged`.

**File:** `lib/rjsniffer.dart`
```dart
import 'rjsniffer_platform_interface.dart';

class Rjsniffer {
  static Future<bool?> amICompromised() async {
    return await RjsnifferPlatform.instance.amICompromised();
  }

  static Future<bool?> amIEmulator() async {
    return await RjsnifferPlatform.instance.amIEmulator();
  }

  static Future<bool?> amIDebugged() async {
    return await RjsnifferPlatform.instance.amIDebugged();
  }
}
```

Each of these delegates to a platform interface, whose default implementation is a `MethodChannel`.

**File:** `lib/rjsniffer_platform_interface.dart`
```dart
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

import 'rjsniffer_method_channel.dart';

abstract class RjsnifferPlatform extends PlatformInterface {
  /// Constructs a RjsnifferPlatform.
  RjsnifferPlatform() : super(token: _token);

  static final Object _token = Object();

  static RjsnifferPlatform _instance = MethodChannelRjsniffer();

  /// The default instance of [RjsnifferPlatform] to use.
  ///
  /// Defaults to [MethodChannelRjsniffer].
  static RjsnifferPlatform get instance => _instance;

  /// Platform-specific implementations should set this with their own
  /// platform-specific class that extends [RjsnifferPlatform] when
  /// they register themselves.
  static set instance(RjsnifferPlatform instance) {
    PlatformInterface.verifyToken(instance, _token);
    _instance = instance;
  }

  Future<bool?> amICompromised() async {
    throw UnimplementedError('amICompromised() has not been implemented.');
  }

  Future<bool?> amIEmulator() async {
    throw UnimplementedError('amIEmulator() has not been implemented.');
  }

  Future<bool?> amIDebugged() async {
    throw UnimplementedError('amIDebugged() has not been implemented.');
  }
}
```

And here is the concrete `MethodChannel` implementation, this is the piece worth mentioning before moving into the APK, because the string literals passed to `invokeMethod` (`runprog`, `runprog2`, etc) are exactly what we'll be searching for later. So each Dart-facing function is a thin wrapper around the same `MethodChannel`, invoking a differently-named `runprogX` method depending on the check being performed. For instance, `amIEmulator()` invokes `runprog2`, whose actual implementation resides entirely on the native Android/iOS side and is executed according to the platform on which the application is running.


**File:** `lib/rjsniffer_method_channel.dart`
```dart
import 'dart:io';

import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';

import 'rjsniffer_platform_interface.dart';

/// An implementation of [RjsnifferPlatform] that uses method channels.
class MethodChannelRjsniffer extends RjsnifferPlatform {
  /// The method channel used to interact with the native platform.
  @visibleForTesting
  final methodChannel = const MethodChannel('com.emrys.rjsniffer/epic');

  @override
  Future<bool?> amICompromised() async {
    final compromised = await methodChannel.invokeMethod<bool>('runprog');

    if (compromised == false && Platform.isAndroid) {
      final magisk = await methodChannel.invokeMethod<bool>('runprog4');
      return magisk;
    } else {
      return compromised;
    }
  }

  @override
  Future<bool?> amIEmulator() async {
    final emulator = await methodChannel.invokeMethod<bool>('runprog2');
    return emulator;
  }

  @override
  Future<bool?> amIDebugged() async {
    final debugged = await methodChannel.invokeMethod<bool>('runprog3');
    return debugged;
  }
}
```

### Platform code
Before diving into the target APK, let's inspect the plugin's Android implementation and see what happens behind the `runprogX` calls. The relevant `MethodCallHandler` is located in the plugin's `/android` directory.

**File:** `android/src/main/java/com/emrys/rjsniffer/rjsniffer`
```java
package com.emrys.rjsniffer.rjsniffer;
...
import com.scottyab.rootbeer.RootBeer;
...
/** RjsnifferPlugin */
public class RjsnifferPlugin implements FlutterPlugin, MethodCallHandler {
  /// The MethodChannel that will the communication between Flutter and native Android
  ///
  /// This local reference serves to register the plugin with the Flutter Engine and unregister it
  /// when the Flutter Engine is detached from the Activity
  private MethodChannel channel;
  private static final String PLATFORM_CHANNEL = "com.emrys.rjsniffer/epic";
...
  @Override
  public void onMethodCall(@NonNull MethodCall call, @NonNull Result result) {

    try {
      Intent intent = new Intent(context.getApplicationContext(), Sniffer.class);
      context.getApplicationContext().bindService(intent, mIsolatedServiceConnection, BIND_AUTO_CREATE);

      if (call.method.equals("runprog")) {

        boolean detected = false;

        RootBeer rootBeer = new RootBeer(context);

        if (Build.BRAND.contains(ONEPLUS) || Build.BRAND.contains(MOTO) || Build.BRAND.contains(XIAOMI)) {

          if (rootBeer.isRooted()) {
            detected = true;
          }

        } else {
          if (rootBeer.isRootedWithBusyBoxCheck()) {
            detected = true;
          }

        }

        try {

          if (isPathExist("su")
                  || isSUExist()
                  || isTestBuildKey()
                  || isHaveRootHideApps()
                  || isHaveDangerousApps()
                  || isHaveRootManagementApps()
                  || isHaveDangerousProperties()
                  || isHaveReadWritePermission()) {

            detected = true;

          }
        } catch (Exception e) {
          e.printStackTrace();
        }


        try {

          if (checkRootMethod8() || checkRootMethod9()) {

            detected = true;

          }
        } catch (Exception e) {

          e.printStackTrace();
        }


        result.success(detected);

      } else if (call.method.equals("runprog2")) {
        boolean detected = false;

        if (Emulate.isEmulator() || Emulate.isEmulator2()) {
          detected = true;
        }

        result.success(detected);
      } else if (call.method.equals("runprog3")) {
...
```

`runprog` performs the root-detection checks. It first creates a `RootBeer` instance and runs either `isRooted()` or `isRootedWithBusyBoxCheck()`, depending on the device brand. It then performs several additional checks, including looking for `su`, root-management or root-hiding apps, dangerous properties, test-build keys, and writable system paths. Two more root checks are performed afterward, and if any of these checks succeeds, detected is set to `true`. The final value is returned to Flutter through `result.success(detected)`. The other `runprogX` methods follow the same pattern, performing checks for conditions such as emulator detection and debugging, and returning the result to Flutter.

In short, the Android implementation receives a `runprogX` method name from Flutter, runs the corresponding native checks, and returns a boolean indicating whether the condition was detected.

## Two solutions
With this understanding of the plugin, there are two independent paths to bypass the check and recover the flag, mirroring the two techniques introduced in earlier challenges:
- **Solution 1**: hook the platform (Java) code inside the target APK with Frida, same technique used in [chall5](../../chall5/solution/solution.md).
- **Solution 2**: unpack the Dart AOT snapshot with Blutter and hook the native Dart machine code directly, same technique used in [chall2](../../chall2/solution/solution.md), [chall3](../../chall3/solution/solution.md) and [chall4](../../chall4/solution/solution.md).

Both are shown below, and both recover the same flag.

## Solution 1 - Platform Hooking: JADX
Opening the app in `jadx-gui` and searching for the plugin's channel name confirms the application statically links against `rjsniffer`, exactly as expected.
![alt text](./_images/chall9_2.png)

Since the plugin's `MethodCallHandler` dispatches on `runprogX` strings, searching for that literal leads straight to the dispatcher method. The result of each check is delivered as `Boolean.valueOf(z)` through a callback object. Every branch, regardless of which `runprogX` was called, funnels its boolean value through the same sink `c0100l.m540d(Boolean.valueOf(...))`.

**File:** `com.emrys.rjsniffer.rjsniffer.C0191a`
```java
...
    public final void mo451n(C0006a c0006a, C0100l c0100l) {
        byte b;
        byte b2;
        try {
            boolean z = true;
            this.f452c.getApplicationContext().bindService(new Intent(this.f452c.getApplicationContext(), Sniffer.class), this.f455f, 1);
            String str = (String) c0006a.f16c;
            boolean z2 = false;
            int i = 0;
            if (str.equals("runprog")) {
                C0012b c0012b = new C0012b(0, this.f452c);
                String str2 = Build.BRAND;
                boolean contains = str2.contains("oneplus");
                String[] strArr = AbstractC0011a.f25b;
                String[] strArr2 = AbstractC0011a.f24a;
                if (!contains && !str2.contains("moto") && !str2.contains("Xiaomi")) {
                    if (!c0012b.m768v(new ArrayList(Arrays.asList(strArr2)))) {
                        ArrayList arrayList = new ArrayList();
                        arrayList.addAll(Arrays.asList(strArr));
                        if (!c0012b.m768v(arrayList) && !C0012b.m775m("su") && !C0012b.m775m("busybox") && !C0012b.m774o() && !C0012b.m773q()) {
                            String str3 = Build.TAGS;
                            if (str3 != null && str3.contains("test-keys")) {
                                b2 = 1;
                            } else {
                                b2 = 0;
                            }
                            if (b2 == 0) {
                                if (!C0012b.m771s()) {
                                    if (!C0012b.m772r()) {
                                    }
                                }
                            }
                        }
                    }
                    boolean z3 = true;
                    String[] strArr3 = AbstractC0000a.f3d;
                    while (true) {
                        if (i < 22) {
                            break;
                        } else if (new File(strArr3[i], "su").exists()) {
                            break;
                        } else {
                            i++;
                        }
                    }
                    EnumC0005f enumC0005f = EnumC0005f.run_su;
                    new ArrayList();
                    Runtime.getRuntime().exec(enumC0005f.f11b);
                    z3 = z;
                    c0100l.m540d(Boolean.valueOf(z3));
                    return;
                }
                if (!c0012b.m768v(new ArrayList(Arrays.asList(strArr2)))) {
                    ArrayList arrayList2 = new ArrayList();
                    arrayList2.addAll(Arrays.asList(strArr));
                    if (!c0012b.m768v(arrayList2) && !C0012b.m775m("su") && !C0012b.m774o() && !C0012b.m773q()) {
                        String str4 = Build.TAGS;
                        if (str4 != null && str4.contains("test-keys")) {
                            b = 1;
                        } else {
                            b = 0;
                        }
                        if (b == 0) {
                            if (!C0012b.m771s()) {
                                if (!C0012b.m772r()) {
                                }
                            }
                        }
                    }
                }
                boolean z32 = true;
                String[] strArr32 = AbstractC0000a.f3d;
                while (true) {
                    if (i < 22) {
                    }
                    i++;
                }
                EnumC0005f enumC0005f2 = EnumC0005f.run_su;
                new ArrayList();
                Runtime.getRuntime().exec(enumC0005f2.f11b);
                z32 = z;
                c0100l.m540d(Boolean.valueOf(z32));
                return;
            } else if (str.equals("runprog2")) {
                if (!AbstractC0001b.m804a()) {
                    int i2 = 0;
                    while (true) {
                        try {
                            ArrayList arrayList3 = AbstractC0001b.f5a;
                            if (i2 >= arrayList3.size()) {
                                break;
                            } else if (new File((String) arrayList3.get(i2)).exists()) {
                                break;
                            } else {
                                i2++;
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    z = false;
                }
                c0100l.m540d(Boolean.valueOf(z));
                return;
            } else if (str.equals("runprog3")) {
                if (Settings.Secure.getInt(this.f452c.getContentResolver(), "adb_enabled", 0) != 1 && !Native.AntiFridaNativeLoader_checkFridaByPort()) {
                    c0100l.m540d(Boolean.FALSE);
                    return;
                }
                c0100l.m540d(Boolean.TRUE);
                return;
            } else if (str.equals("runprog4")) {
                if (Build.VERSION.SDK_INT >= 28) {
                    try {
                        if (this.f453d) {
                            z2 = this.f454e.mo450a();
                        }
                    } catch (RemoteException e2) {
                        throw new RuntimeException(e2);
                    }
                } else {
                    z2 = m453h();
                }
                c0100l.m540d(Boolean.valueOf(z2));
                return;
            } else {
                return;
            }
        } catch (Exception e3) {
            e3.printStackTrace();
        }
        e3.printStackTrace();
    }
}
...
```

Following the call into `C0100l`, `m540d` is, with high confidence, the success callback of the `MethodChannel.Result`.

**File:** `p010O.C0100l`
```java
package p010O;

import android.util.Log;
import p001C.C0006a;
import p004G.C0024K;
import p006I.C0078f;
import p011P.InterfaceC0116j;

/* renamed from: O.l */
/* loaded from: classes.dex */
public final class C0100l {

    /* renamed from: a */
    public final /* synthetic */ int f300a;

    /* renamed from: b */
    public final /* synthetic */ Object f301b;

    /* renamed from: c */
    public final /* synthetic */ Object f302c;

    public /* synthetic */ C0100l(int i, Object obj, Object obj2) {
        this.f300a = i;
        this.f302c = obj;
        this.f301b = obj2;
    }
...
    /* renamed from: d */
    public final void m540d(Object obj) {
        switch (this.f300a) {
            case 0:
                ((C0101m) this.f302c).f304b = (byte[]) this.f301b;
                return;
            default:
                ((C0078f) this.f301b).mo534a(((InterfaceC0116j) ((C0024K) ((C0006a) this.f302c).f17d).f58c).mo522c(obj));
                return;
        }
    }
...
```

The approach here is identical to the one seen in the previous challenges: hook the callback that carries the final boolean result, and flip it to false whenever it comes back true, before it's handed back to Dart.

Because `m540d`/`O.l.d` is the single common sink for every `runprogX` branch, this one hook transparently covers `amICompromised()`, `amIEmulator()`, `amIDebugged()` and all four boolean results pass through it, so there's no need to enumerate every check individually.

**File:** `bypass.js`
```js
Java.perform(function () {
    console.log("[*] Initializing rjsniffer root detection bypass...");

    const Class = Java.use("O.l");
    const BooleanClass = Java.use("java.lang.Boolean");

    // hook the callback handler responsible for delivering the root check result
    // "Class.d" is the equivalent of "p010O.C0100l.m540d" in JADX 
    Class.d.implementation = function (value) {

        // print argument
        console.log("[*] Callback invoked with value: " + value);

        // call from jadx is "c0100l.m540d(Boolean.valueOf(z32));"
        // 1. check if value is a Boolean
        if (value !== null && value.getClass().getName() === "java.lang.Boolean") {
            
            // 2. cast value
            const isRooted = Java.cast(value, BooleanClass).booleanValue();
            console.log("[*] Root check returned: " + isRooted);

            // 3. if isRooted is true, force to false
            if (isRooted === true) {
                console.log("[!] Root detected; overriding value to false");

                // change method's name accordingly (d)
                return this.d(BooleanClass.valueOf(false));
            }
        }

        // 4. for all other values do nothing
        // change method's name accordingly (d)
        return this.d(value);
    };

    console.log("[+] rjsniffer root detection bypassed successfully");
});
```

Running it:
```shell
$ frida -U -f com.flutter_labs.chall9 -l bypass.js
     ____
    / _  |   Frida 16.7.13 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Pixel 4a (id=)
Spawned `com.flutter_labs.chall9`. Resuming main thread!                
[Pixel 4a::com.flutter_labs.chall9 ]-> [*] Initializing rjsniffer root detection bypass...
[+] rjsniffer root detection bypassed successfully
[*] Callback invoked with value: {}
[*] Callback invoked with value: null
[*] Callback invoked with value: null
[*] Callback invoked with value: null
[*] Callback invoked with value: true
[*] Root check returned: true
[!] Root detected; overriding value to false
[*] Callback invoked with value: null
[*] Callback invoked with value: null
[*] Callback invoked with value: null
[*] Callback invoked with value: true
[*] Root check returned: true
[!] Root detected; overriding value to false
[*] Callback invoked with value: false
[*] Root check returned: false
[*] Callback invoked with value: true
[*] Root check returned: true
[!] Root detected; overriding value to false
```

The screenshot below confirms the flag is revealed:

![alt text](./_images/chall9_3.png)

## Solution 2 - Native Hooking: Blutter
The second approach targets the Dart AOT snapshot directly, without touching the platform (Java) layer at all.

Running Blutter:
```bash
$ docker run --rm -v "$(pwd):/data" blutter /data/chall9.apk /data/output
Cloning into '/app/dartsdk/v3.10.7'...
remote: Enumerating objects: 2743, done.        
remote: Counting objects: 100% (2743/2743), done.        
remote: Compressing objects: 100% (2175/2175), done.        
remote: Total 2743 (delta 52), reused 1582 (delta 40), pack-reused 0 (from 0)        
Receiving objects: 100% (2743/2743), 1.38 MiB | 3.04 MiB/s, done.
Resolving deltas: 100% (52/52), done.
-- Configuring done (0.4s)
-- Generating done (0.0s)
-- Build files have been written to: /app/build/dartvm3.10.7_android_arm64
[1/271] Building CXX object CMakeFiles/dartvm3.10.7_android_arm64.dir/runtime/vm/bytecode_reader.cc.o
[2/271] Building CXX object CMakeFiles/dartvm3.10.7_android_arm64.dir/runtime/vm/base64.cc.o
...
[22/22] Linking CXX executable blutter_dartvm3.10.7_android_arm64
-- Install configuration: "Release"
-- Installing: /app/bin/blutter_dartvm3.10.7_android_arm64
libapp is loaded at 0x7021e357b000
Dart heap at 0x702000000000
Analyzing the application
Dumping Object Pool
Generating application assemblies
Generating Frida script
Dart version: 3.10.7, Snapshot: 1ce86630892e2dca9a8543fdb8ed8e22, Target: android arm64
flags: product no-code_comments no-dwarf_stack_traces_mode dedup_instructions no-tsan no-msan no-shared_data arm64 android compressed-pointers
```

Again, once Blutter's disassembly is available, searching for `runprogX` in the generated assemblies leads straight to Rjsniffer's methods:

![alt text](./_images/chall9_4.png  )

This is where this challenge differs meaningfully from a synchronous native check, such as the ones in [chall4](../../chall4/solution/solution.md) and [chall5](../../chall5/solution/solution.md). Looking at the disassembly, the return type declared at the top of the function is `Future<bool?>`, not `bool?`. That's because every method in this plugin is declared async in Dart:
```dart
static Future<bool?> amICompromised() async { ... }
```

An async Dart function is compiled by the AOT compiler into a state machine. When called, it doesn't run to completion and hand back a boolean, it immediately allocates and returns a `Future` object, then suspends. The actual `MethodChannel` round-trip happens later, off the initial call stack entirely, driven by the platform channel's response arriving asynchronously from the Android/iOS side. When that response arrives, the suspended state machine resumes at the instruction right after the await, with the resolved value sitting in a register.

This means it's not possible to hook `amICompromised()` at its entry and exit points to flip the return value the way we would with a synchronous function. The initial call only produces a `Future` pointer, not the actual boolean.

Disassembling `amICompromised` around the `runprogX` confirms this shape.

**File:** `output/asm/root_jailbreak_sniffer/rjsniffer_method_channel.dart`
```shell
amICompromised:
0x1f1768  InitAsync() -> Future<bool?>     ; sets up the async state machine

; --- call #1: runprog ---
0x1f1794  bl invokeMethod                  ; MethodChannel.invokeMethod('runprog')
0x1f17a0  bl AwaitStub                     ; suspend/resume with result in x1

0x1f17d0  tbnz w0, #4, 0x1f1808            ; if (result == false) -> jump to 0x1f1808

; --- only runs if result != false ---
0x1f17f4  bl invokeMethod                  ; MethodChannel.invokeMethod('runprog4')
0x1f1800  bl AwaitStub                     ; await it
0x1f1804  b ReturnAsyncStub                ; return runprog4's result

; --- short-circuit landing spot ---
0x1f1808  ldur x0, [fp, #-0x10]            ; reload the original 'runprog' result
0x1f180c  b ReturnAsyncStub                ; return it
```

`AwaitStub` is where execution suspends and later resumes with the platform channel's boolean result in `x0`. `ReturnAsyncStub` is the shared trampoline that completes the `Future` with whatever value the `async` function ultimately produces.

The root check in this app, `_checkRoot()` in the Flutter widget's `initState()`, only runs once, at application startup.

**File:** `output/asm/chall9/main.dart`
```shell
...
  _ initState(/* No info */) {
    // ** addr: 0x1f120c, size: 0x30
    // 0x1f120c: EnterFrame
    //     0x1f120c: stp             fp, lr, [SP, #-0x10]!
    //     0x1f1210: mov             fp, SP
    // 0x1f1214: CheckStackOverflow
    //     0x1f1214: ldr             x16, [THR, #0x48]  ; THR::stack_limit
    //     0x1f1218: cmp             SP, x16
    //     0x1f121c: b.ls            #0x1f1234
    // 0x1f1220: r0 = _checkRoot()
    //     0x1f1220: bl              #0x1f123c  ; [package:chall9/main.dart] _HomePageState::_checkRoot
    // 0x1f1224: r0 = Null
    //     0x1f1224: mov             x0, NULL
    // 0x1f1228: LeaveFrame
    //     0x1f1228: mov             SP, fp
    //     0x1f122c: ldp             fp, lr, [SP], #0x10
    // 0x1f1230: ret
    //     0x1f1230: ret             
    // 0x1f1234: r0 = StackOverflowSharedWithoutFPURegs()
    //     0x1f1234: bl              #0x2e23ac  ; StackOverflowSharedWithoutFPURegsStub
    // 0x1f1238: b               #0x1f1220
  }
  ...
```


**First attempt: hooking ReturnAsyncStub**

A first, naive approach is to hook `ReturnAsyncStub` itself, the shared stub every async Dart function funnels through when completing its `Future`, and blindly force its return value to Dart's boxed `false`.

If the Frida script is not attached and its hooks are not installed before that single call occurs, there is no second opportunity and the check executes without being intercepted. This is why this style of hooking can appear inherently non-deterministic: sometimes it works, and sometimes it does not. The outcome depends on a race between two independent timelines: how quickly the polling loop detects that `libapp.so` has been mapped into memory, and how quickly Flutter's startup sequence reaches and executes `_checkRoot()`.

Given this, the polling interval used while waiting for `libapp.so` to load is more significant than it might appear. With a 500 ms interval, which is a common default, the hook can easily lose this race on a fast-booting device. `_checkRoot()` may execute well within the first 500 ms, before the polling loop has an opportunity to detect the module and install the hooks.

Reducing the polling interval to 100 ms substantially narrows this timing window and makes successful attachment considerably more reliable. It is not, however, a complete guarantee, as the outcome still depends on device performance, the exact timing of Flutter's initialization sequence, and Frida's own attachment and hook-installation latency.

**File:** `blutter_frida.js`
```js
function onLibappLoaded() {
    // offset of amICompromised
    const fn_addr = 0x1f1748;

    Interceptor.attach(libapp.add(fn_addr), {
        onEnter: function () {
            init(this.context);
            console.log("[*] amICompromised() called");
        },

        onLeave: function (retval) {
            // for async functions, the return value is a Future, not the bool directly
            // need to hook where the Future completes with the value
            
            // looking at the assembly:
            // 0x1f17a4: mov x1, x0  ; result of first Await (runprog)
            // 0x1f17d0: tbnz w0, #4 ; check if result != false
            // if false (0x30), skip to 0x1f1808 and return it
            // if not false, call runprog4
            
            // intercept the Await return values instead
            console.log("[!] Async function - need to hook Await returns instead");
            console.log("[*] Retval is Future object, not boolean: " + retval);
        }
    });
    
    // hook the ReturnAsyncStub to catch actual return value
    const returnAsyncStub = 0x178ac0;
    Interceptor.attach(libapp.add(returnAsyncStub), {
        onEnter: function() {
            // x0 contains the value being returned from async function
            const returnValue = this.context.x0;
            console.log("[*] ReturnAsync called with value: " + returnValue);
            
            // dart false = NullReg + 0x30
            const dartFalse = this.context[NullReg].add(0x30);
            this.context.x0 = dartFalse;
            console.log("[+] ReturnAsync forced to return false");
        }
    });
}

function tryLoadLibapp() {
    try {
        libapp = Module.findBaseAddress('libapp.so');
    } catch (e) {
        if (e instanceof TypeError && e.message === "not a function") {
            libapp = Process.findModuleByName('libapp.so');
            if (libapp != null) {
                libapp = libapp.base;
            }
        } else {
            throw e;
        }
    }
    if (libapp === null)
        setTimeout(tryLoadLibapp, 100);    
    else
        onLibappLoaded();
}
tryLoadLibapp();
...
```

Running the script:
```shell
$ frida -U -f com.flutter_labs.chall9 -l ./blutter_frida.js
     ____
    / _  |   Frida 16.7.13 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Pixel 4a (id=)
Spawned `com.flutter_labs.chall9`. Resuming main thread!                
[Pixel 4a::com.flutter_labs.chall9 ]-> [*] ReturnAsync called with value: 0x7400583599
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x7400584b79
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x7400008081
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x7400594769
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x7400597c39
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x740059aad9
[+] ReturnAsync forced to return false
[*] amICompromised() called
[!] Async function - need to hook Await returns instead
[*] Retval is Future object, not boolean: 0x74005aa829
[*] ReturnAsync called with value: 0x74005bc599
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x74005dacc9
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x74005daca9
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x74005f9999
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x74005fabe9
[+] ReturnAsync forced to return false
[*] ReturnAsync called with value: 0x7400008081
```

This works, but `ReturnAsyncStub` is a blunt instrument. It is a shared trampoline used by all async Dart functions, not just `amICompromised()`. By forcing `x0` to `false` unconditionally, the hook affects every async function that completes while it is active, including unrelated timers, animations, navigation, and other application logic.

It happens to work in this challenge because little else runs before the flag is displayed, but in a more complex application it could cause crashes or unexpected behavior. The better approach is to identify the async completion path for `amICompromised()` and modify only its result, rather than hooking the global `ReturnAsyncStub`.

**A more targeted approach**


A safer strategy is to hook the specific instruction addresses immediately after each await `Rjsniffer.amIXXX()` resumes inside the application's own `_HomePageState._checkRoot()`, instead of the underlying async execution flow. At each of those addresses, `x0` holds exactly the resolved value of that specific call, and nothing else. 

**File:** `output/asm/chall9/main.dart`
```shell
...
  _ _checkRoot(/* No info */) async {
    // ** addr: 0x1f123c, size: 0x1a0
    // 0x1f123c: EnterFrame
    //     0x1f123c: stp             fp, lr, [SP, #-0x10]!
    //     0x1f1240: mov             fp, SP
    // 0x1f1244: AllocStack(0x80)
    //     0x1f1244: sub             SP, SP, #0x80
    // 0x1f1248: SetupParameters(_HomePageState this /* r1 => r1, fp-0x68 */)
    //     0x1f1248: stur            NULL, [fp, #-8]
    //     0x1f124c: stur            x1, [fp, #-0x68]
    // 0x1f1250: CheckStackOverflow
    //     0x1f1250: ldr             x16, [THR, #0x48]  ; THR::stack_limit
    //     0x1f1254: cmp             SP, x16
    //     0x1f1258: b.ls            #0x1f13d4
    // 0x1f125c: r1 = 5
    //     0x1f125c: mov             x1, #5
    // 0x1f1260: r0 = AllocateContext()
    //     0x1f1260: bl              #0x2e1214  ; AllocateContextStub
    // 0x1f1264: mov             x2, x0
    // 0x1f1268: ldur            x1, [fp, #-0x68]
    // 0x1f126c: stur            x2, [fp, #-0x70]
    // 0x1f1270: StoreField: r2->field_f = r1
    //     0x1f1270: stur            w1, [x2, #0xf]
    // 0x1f1274: InitAsync() -> Future<void?>
    //     0x1f1274: ldr             x0, [PP, #0x2d8]  ; [pp+0x2d8] TypeArguments: <void?>
    //     0x1f1278: bl              #0x157f4c  ; InitAsyncStub
    // 0x1f127c: r0 = amICompromised()
    //     0x1f127c: bl              #0x1f16e0  ; [package:root_jailbreak_sniffer/rjsniffer.dart] Rjsniffer::amICompromised
    // 0x1f1280: mov             x1, x0
    // 0x1f1284: stur            x1, [fp, #-0x78]
    // 0x1f1288: r0 = Await()
    //     0x1f1288: bl              #0x157d0c  ; AwaitStub
HERE
    // 0x1f128c: cmp             w0, NULL
    // 0x1f1290: b.ne            #0x1f129c
    // 0x1f1294: r1 = false
    //     0x1f1294: add             x1, NULL, #0x30  ; false
    // 0x1f1298: b               #0x1f12a0
    // 0x1f129c: mov             x1, x0
    // 0x1f12a0: ldur            x2, [fp, #-0x70]
    // 0x1f12a4: mov             x0, x1
    // 0x1f12a8: stur            x1, [fp, #-0x78]
    // 0x1f12ac: ArrayStore: r2[0] = r0  ; List_4
    //     0x1f12ac: stur            w0, [x2, #0x17]
    //     0x1f12b0: tbz             w0, #0, #0x1f12cc
    //     0x1f12b4: ldurb           w16, [x2, #-1]
    //     0x1f12b8: ldurb           w17, [x0, #-1]
    //     0x1f12bc: and             x16, x17, x16, lsr #2
    //     0x1f12c0: tst             x16, HEAP, lsr #32
    //     0x1f12c4: b.eq            #0x1f12cc
    //     0x1f12c8: bl              #0x2e09b4  ; WriteBarrierWrappersStub
    // 0x1f12cc: r0 = amIEmulator()
    //     0x1f12cc: bl              #0x1f1610  ; [package:root_jailbreak_sniffer/rjsniffer.dart] Rjsniffer::amIEmulator
    // 0x1f12d0: mov             x1, x0
    // 0x1f12d4: stur            x1, [fp, #-0x80]
    // 0x1f12d8: r0 = Await()
    //     0x1f12d8: bl              #0x157d0c  ; AwaitStub
HERE
    // 0x1f12dc: cmp             w0, NULL
    // 0x1f12e0: b.ne            #0x1f12ec
    // 0x1f12e4: r1 = false
    //     0x1f12e4: add             x1, NULL, #0x30  ; false
    // 0x1f12e8: b               #0x1f12f0
    // 0x1f12ec: mov             x1, x0
    // 0x1f12f0: ldur            x2, [fp, #-0x70]
    // 0x1f12f4: mov             x0, x1
    // 0x1f12f8: stur            x1, [fp, #-0x78]
    // 0x1f12fc: StoreField: r2->field_1b = r0
    //     0x1f12fc: stur            w0, [x2, #0x1b]
    //     0x1f1300: tbz             w0, #0, #0x1f131c
    //     0x1f1304: ldurb           w16, [x2, #-1]
    //     0x1f1308: ldurb           w17, [x0, #-1]
    //     0x1f130c: and             x16, x17, x16, lsr #2
    //     0x1f1310: tst             x16, HEAP, lsr #32
    //     0x1f1314: b.eq            #0x1f131c
    //     0x1f1318: bl              #0x2e09b4  ; WriteBarrierWrappersStub
    // 0x1f131c: r0 = amIDebugged()
    //     0x1f131c: bl              #0x1f13dc  ; [package:root_jailbreak_sniffer/rjsniffer.dart] Rjsniffer::amIDebugged
    // 0x1f1320: mov             x1, x0
    // 0x1f1324: stur            x1, [fp, #-0x80]
    // 0x1f1328: r0 = Await()
    //     0x1f1328: bl              #0x157d0c  ; AwaitStub
HERE
    // 0x1f132c: cmp             w0, NULL
    // 0x1f1330: b.ne            #0x1f133c
    // 0x1f1334: r4 = false
    //     0x1f1334: add             x4, NULL, #0x30  ; false
    // 0x1f1338: b               #0x1f1340
    // 0x1f133c: mov             x4, x0
    // 0x1f1340: ldur            x3, [fp, #-0x70]
    // 0x1f1344: mov             x0, x4
    // 0x1f1348: stur            x4, [fp, #-0x78]
    // 0x1f134c: StoreField: r3->field_1f = r0
    //     0x1f134c: stur            w0, [x3, #0x1f]
    //     0x1f1350: tbz             w0, #0, #0x1f136c
    //     0x1f1354: ldurb           w16, [x3, #-1]
    //     0x1f1358: ldurb           w17, [x0, #-1]
    //     0x1f135c: and             x16, x17, x16, lsr #2
    //     0x1f1360: tst             x16, HEAP, lsr #32
    //     0x1f1364: b.eq            #0x1f136c
    //     0x1f1368: bl              #0x2e09d4  ; WriteBarrierWrappersStub
    // 0x1f136c: mov             x2, x3
    // 0x1f1370: r1 = Function '<anonymous closure>':.
    //     0x1f1370: add             x1, PP, #0xa, lsl #12  ; [pp+0xa0d8] AnonymousClosure: (0x1f18b4), in [package:chall9/main.dart] _HomePageState::_checkRoot (0x1f123c)
    //     0x1f1374: ldr             x1, [x1, #0xd8]
    // 0x1f1378: r0 = AllocateClosure()
    //     0x1f1378: bl              #0x2e15d8  ; AllocateClosureStub
    // 0x1f137c: ldur            x1, [fp, #-0x68]
    // 0x1f1380: mov             x2, x0
    // 0x1f1384: r0 = setState()
    //     0x1f1384: bl              #0x1a62b8  ; [package:flutter/src/widgets/framework.dart] State::setState
    // 0x1f1388: b               #0x1f13cc
    // 0x1f138c: sub             SP, fp, #0x80
    // 0x1f1390: ldur            x2, [fp, #-0x70]
    // 0x1f1394: StoreField: r2->field_13 = r0
    //     0x1f1394: stur            w0, [x2, #0x13]
    //     0x1f1398: tbz             w0, #0, #0x1f13b4
    //     0x1f139c: ldurb           w16, [x2, #-1]
    //     0x1f13a0: ldurb           w17, [x0, #-1]
    //     0x1f13a4: and             x16, x17, x16, lsr #2
    //     0x1f13a8: tst             x16, HEAP, lsr #32
    //     0x1f13ac: b.eq            #0x1f13b4
    //     0x1f13b0: bl              #0x2e09b4  ; WriteBarrierWrappersStub
    // 0x1f13b4: r1 = Function '<anonymous closure>':.
    //     0x1f13b4: add             x1, PP, #0xa, lsl #12  ; [pp+0xa0e0] AnonymousClosure: (0x1f1818), in [package:chall9/main.dart] _HomePageState::_checkRoot (0x1f123c)
    //     0x1f13b8: ldr             x1, [x1, #0xe0]
    // 0x1f13bc: r0 = AllocateClosure()
    //     0x1f13bc: bl              #0x2e15d8  ; AllocateClosureStub
    // 0x1f13c0: ldur            x1, [fp, #-0x68]
    // 0x1f13c4: mov             x2, x0
    // 0x1f13c8: r0 = setState()
    //     0x1f13c8: bl              #0x1a62b8  ; [package:flutter/src/widgets/framework.dart] State::setState
    // 0x1f13cc: r0 = Null
    //     0x1f13cc: mov             x0, NULL
    // 0x1f13d0: r0 = ReturnAsyncNotFuture()
    //     0x1f13d0: b               #0x157a58  ; ReturnAsyncNotFutureStub
    // 0x1f13d4: r0 = StackOverflowSharedWithoutFPURegs()
    //     0x1f13d4: bl              #0x2e23ac  ; StackOverflowSharedWithoutFPURegsStub
    // 0x1f13d8: b               #0x1f125c
  }
  ...
```

The three hook addresses (each marked with a `HERE` immediately above the corresponding `Await()` call) correspond exactly to the three call sites in `_checkRoot()` that consume Rjsniffer's results, so only those three values are ever touched, every other async completion in the app, including the plugin's own internal chaining between `runprog` and `runprog4` inside `amICompromised()`, is left completely alone. 

**File:** `blutter_frida.js`
```js
function onLibappLoaded() {
    // _HomePageState._checkRoot — instruction immediately after each
    // `await Rjsniffer.amIXxx()` resumes. x0 holds the resolved value.
    const hooks = [
        { addr: 0x1f128c, name: "amICompromised" },
        { addr: 0x1f12dc, name: "amIEmulator"     },
        { addr: 0x1f132c, name: "amIDebugged"     },
    ];

    hooks.forEach(h => {
        Interceptor.attach(libapp.add(h.addr), {
            onEnter: function () {
                init(this.context);

                const [tptr, cls, value] = getTaggedObjectValue(this.context.x0);
                console.log(`[*] ${h.name}() resolved to ${cls.name} = ${JSON.stringify(value)}`);

                // dart 'false' = NullReg + 0x30
                this.context.x0 = this.context[NullReg].add(0x30);
                console.log(`[+] ${h.name}() forced to false`);
            }
        });
    });
}

function tryLoadLibapp() {
    try {
        libapp = Module.findBaseAddress('libapp.so');
    } catch (e) {
        if (e instanceof TypeError && e.message === "not a function") {
            libapp = Process.findModuleByName('libapp.so');
            if (libapp != null) {
                libapp = libapp.base;
            }
        } else {
            throw e;
        }
    }
    if (libapp === null)
        setTimeout(tryLoadLibapp, 100);    
    else
        onLibappLoaded();
}
tryLoadLibapp();
...
```

All three checks are observed resolving to their real values, one of them `true`, and all three are forced to `false` right at the resume point, before `_checkRoot()` ever gets to branch on them.
```shell
$ frida -U -f com.flutter_labs.chall9 -l ./blutter_frida.js
     ____
    / _  |   Frida 16.7.13 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Pixel 4a (id=)
Spawned `com.flutter_labs.chall9`. Resuming main thread!                
[Pixel 4a::com.flutter_labs.chall9 ]-> [*] amICompromised() resolved to bool = true
[+] amICompromised() forced to false
[*] amIEmulator() resolved to bool = false
[+] amIEmulator() forced to false
[*] amIDebugged() resolved to bool = true
[+] amIDebugged() forced to false
```

## Flag
FLAG{hooking_third_party_libraries}
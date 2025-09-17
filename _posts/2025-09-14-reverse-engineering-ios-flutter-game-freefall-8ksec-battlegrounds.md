---
layout: post
title: "Reverse Engineering an iOS Flutter Game: Bypassing FreeFall's Scoring System"
categories: [iOS, Reverse Engineering]
tags: [ios, flutter, reflutter, reverse-engineering, frida, lldb, security]
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPzRlpflH1tk1a3HnFo5zdj4w_e5i-StVZdHfMVSHDF7lgTrrKkOsPx-1r29T2I-Ywep0Nf6_SynmA72QGr-PDd8C53n1jYQMO3mUEAPIxegmNYc6_kViIBI0JYf3YcJc4OW3ThVanXSZLiuH2i0Akv=w1070-h654-s-no-gm
    alt: FreeFall Game
permalink: /reverse-engineering-ios-flutter-game-freefall-8ksec-battlegrounds/
---

## Introduction

Reverse engineering an iOS application can feel like solving a puzzle, especially when it’s built with a cross-platform framework like Flutter. In this post, I’ll guide you through the process of reverse engineering *FreeFall*, an addictive iOS game from [8ksec iOS Application Exploitation Challenges](https://academy.8ksec.io/course/ios-application-exploitation-challenges), to bypass its scoring system and submit impossibly high scores. As someone passionate about iOS security and reverse engineering, I’m excited to share this journey in a way that’s approachable for beginners while offering enough depth for seasoned engineers.

*FreeFall* is a fast-paced ball game where players navigate obstacles using paddle controls under a tight 60-second time limit. The goal is to earn points by destroying obstacles and advancing through difficulty levels, ultimately climbing the leaderboard. The catch? The game claims to have a “secure, cheat-proof scoring” system. Our task is to bypass this validation to submit arbitrary scores that would be impossible through normal gameplay. To do this, we’ll use powerful tools like **reFlutter**, **IDA**, **LLDB**, and **Frida**, each playing a critical role in unraveling the app’s internals.

![FreeFall Game](https://lh3.googleusercontent.com/pw/AP1GczMvPKFQKIqsBW2bmGum62_Jhqo1_wfg43TxcX_zf2plsMCFADascfZ06mZeGRbTsYzT__EtAG031XlvcofmPsHBFlJ1rv1C6ptLtvKRFNwVMvPCUJbygOnLaWI39AuZ5gWN4mTky8kDmRGxLVtnpXIZ=w322-h654-s-no-gm)
_**Figure: FreeFall Game**_

## Understanding the Challenge

Before we jump into the technical details, let’s break down the objective. *FreeFall* challenges players to achieve high scores by skillfully navigating a ball through obstacles. The game’s scoring system is designed to prevent cheating, ensuring only legitimate scores make it to the leaderboard. However, our goal is to manipulate the score submission process to post an outrageously high score, like **99,999,999**, that no player could realistically achieve. Since *FreeFall* is built with Flutter, a cross-platform framework, we’ll need to adapt traditional iOS reverse engineering techniques to account for Flutter’s unique architecture. Let’s start by examining the app’s structure.

## Identifying the App as a Flutter Application

To begin, we need to understand the app’s composition. An iOS app is distributed as an **IPA file**, which is essentially a zipped archive containing the application’s binary and resources. For those new to iOS, think of an IPA file as a container that holds everything the app needs to run on your device. Unzipping the *FreeFall* IPA reveals the following structure:

```bash
$ cd 11-FreeFall/Payload/Runner.app 
$ ls -1
_CodeSignature
AppFrameworkInfo.plist
AppIcon60x60@2x.png
AppIcon76x76@2x~ipad.png
Assets.car
Base.lproj
embedded.mobileprovision
Frameworks
Info.plist
PkgInfo
Runner

$ ls -1 Frameworks
App.framework
flutter_secure_storage.framework
Flutter.framework
libswiftCore.dylib
libswiftCoreAudio.dylib
libswiftCoreFoundation.dylib
libswiftCoreGraphics.dylib
libswiftCoreImage.dylib
libswiftCoreMedia.dylib
libswiftDarwin.dylib
libswiftDispatch.dylib
libswiftFoundation.dylib
libswiftMetal.dylib
libswiftObjectiveC.dylib
libswiftos.dylib
libswiftQuartzCore.dylib
libswiftsimd.dylib
libswiftUIKit.dylib
path_provider_foundation.framework
shared_preferences_foundation.framework
sqflite_darwin.framework
```

Notice the `Runner` binary and frameworks like `App.framework` and `Flutter.framework`. These are signs that *FreeFall* is built with **Flutter**, Google’s open-source SDK for creating apps across mobile, web, and desktop from a single codebase. For newcomers, Flutter allows developers to write code in **Dart**, a programming language, which is then compiled into native machine code for iOS using **Ahead-of-Time (AOT) compilation**. This process, known as AOT compilation, converts Dart code into platform-specific machine code before the app runs, making it fast but challenging to analyze statically. The Dart code runs inside a **Dart Virtual Machine (VM)**, provided by the Flutter Engine, which we’ll explore next.

## Understanding Flutter’s Architecture

To reverse engineer *FreeFall*, we need to grasp Flutter’s architecture, as shown below:

![Flutter Architecture](https://lh3.googleusercontent.com/pw/AP1GczMU0gPL6P2449-nNJ2G9Ko8WhL53TLNBf39eq0ozw-cr44Hha5j7mUFGqnn7mU2tOHNRXisHqqN3e6K7A1usalEJQBoP082BPHBLxdtxdG9CBlyC-QzclRUs6ki_K2-Dms3yKqlJxH1kxG4wt0PJ29D=w806-h654-s-no-gm)
_**Figure: Flutter Architecture**_

Flutter apps consist of three main components:
- **Framework**: Written in Dart, this contains the app’s logic and user interface, stored in `App.framework`.
- **Engine**: A platform-specific runtime that hosts the Dart VM, found in `Flutter.framework`. It translates Dart code into native instructions.
- **Embedder**: A thin layer that integrates the **Engine** with the target platform (iOS, in this case).

When a Flutter app is compiled, the **Engine** uses AOT compilation to produce an **AppSnapshot**, a precompiled bundle of machine code that includes both the **Framework** and the developer’s Dart code. This makes static analysis—examining the binary without running it—tricky, as the compiled code is obfuscated and lacks clear symbols. To overcome this, we’ll use **reFlutter**, a specialized tool for Flutter reverse engineering.

## Using reFlutter for Dynamic Analysis

Static analysis of Flutter binaries is challenging, so we turn to **reFlutter**, a powerful tool designed to simplify Flutter app reverse engineering. **reFlutter** uses a patched version of the Flutter library to modify the snapshot deserialization process, enabling dynamic analysis by dumping runtime information. Let’s see how it works:

```bash
$ reflutter -p 1-FreeFall.ipa
[*] Processing...

SnapshotHash: d91c0e6f35f0eb2e44124e8f42aa44a7
The resulting ipa file: ./release.RE.ipa
Please sign & install the ipa file.

Configure Potatso (iOS) to use your Burp Suite proxy server.
```

Running the `reflutter` command processes the IPA and generates a modified version, `release.RE.ipa`. We then resign and install this modified app on a jailbroken iOS device using `ios-deploy`:

```bash
$ ios-deploy --bundle Payload/Runner.app
[....] Waiting for iOS device to be connected
...
[ 52%] CreatingStagingDirectory
[ 57%] ExtractingPackage
[ 60%] InspectingPackage
[ 65%] PreflightingApplication
[ 70%] VerifyingApplication
[ 75%] CreatingContainer
[ 80%] InstallingApplication
[ 85%] PostflightingApplication
[ 90%] SandboxingApplication
[ 95%] GeneratingApplicationMap
[100%] InstallComplete
[100%] Installed package Payload/Runner.app
```

Once installed, we run the app and check the device logs using macOS’s `Console.app`. The patched app creates a `dump.dart` file in the app’s sandbox directory:

```bash
/private/var/mobile/Containers/Data/Application/DFFD772E-29E4-4127-BC29-9168828313DE/Documents/dump.dart
```

This file is a goldmine for reverse engineers, containing runtime information about the app’s structure.

![reFlutter dump.dart](https://lh3.googleusercontent.com/pw/AP1GczPzRlpflH1tk1a3HnFo5zdj4w_e5i-StVZdHfMVSHDF7lgTrrKkOsPx-1r29T2I-Ywep0Nf6_SynmA72QGr-PDd8C53n1jYQMO3mUEAPIxegmNYc6_kViIBI0JYf3YcJc4OW3ThVanXSZLiuH2i0Akv=w1070-h654-s-no-gm)
_**Figure: reFlutter dump.dart**_

## Analyzing `dump.dart`

To access `dump.dart`, we copy it from the device using `scp`:

```bash
$ scp -P 2222 ip8:/private/var/mobile/Containers/Data/Application/DFFD772E-29E4-4127-BC29-9168828313DE/Documents/dump.dart ./
root@localhost's password:
dump.dart                                                              100% 1762KB  31.5MB/s   00:00
```

Opening `dump.dart` in a text editor reveals JSON-like objects, such as:

```json
{"method_name":"_showGameOverDialog","offset":"0x00000000001711ac","library_url":"package:freefallgame\/screens\/game_screen.dart","class_name":"_GameScreenState"}
```

These objects contain valuable details like method names, offsets, class names, and library URLs, which help us understand the app’s structure. However, the file isn’t valid JSON due to missing separators. To make it usable, we need to sanitize it by adding commas between objects and wrapping it in square brackets. This sanitized data can then be used to rename symbols in **IDA**, a popular disassembler, though this step is optional for our challenge.

## Renaming Symbols in IDA (Optional)

While not required to solve the challenge, renaming symbols in IDA can make static analysis easier. The Flutter app’s logic resides in `App.framework/App`, but loading this binary into IDA reveals generic function names like `sub_XXXXXX`, making it hard to identify key functions:

![App before symbols renaming](https://lh3.googleusercontent.com/pw/AP1GczMV9JKLxSJ9OeW9Rjvu4L1D3W1ffcmEe7QUchoSyafnSRlvstSGReX1ijYNbjxmVXZ219N7RfigwFrvixTXSoAIVZafxZn2X7_7RAENIxC3kU-tde95mxOju0_9NPfc_kBUOoTDYEWLMooVqbbMLVOr=w1506-h654-s-no-gm)
_**Figure: App before symbols renaming**_

Using `dump.dart`, we can write an IDA Python script to rename functions based on their offsets relative to the exported symbol `_kDartIsolateSnapshotInstructions` (located at `0x000000000000E8C0` in this case):

![Exported _kDartIsolateSnapshotInstructions address](https://lh3.googleusercontent.com/pw/AP1GczNJq5MGJO2MGq9WiitbPjC_9Ko-ZykBBaDfqbbBsmOXFvbk93Tvv6wJhhNbcoOaDGi3fTHXqBQML0BOHyVu-pDCEXsk3zTd01TOD5tLCOKKdHI2AOjtIjpismUZVlMx-fVH1xEJxtGtfO7bwh1Ekb7c=w2164-h200-s-no-gm)
_**Figure: Exported _kDartIsolateSnapshotInstructions address**_

Here’s the script used to rename symbols:

```python
# IDA_rename_symbols.py
import idaapi
import idautils
import idc
import json
import os

def modify_and_load_json(file_path):
    # Read the file content
    with open(file_path, 'r') as file:
        content = file.read()
    
    # Replace '}{' with '},{'
    modified_content = content.replace('}{', '},{')
    
    # Append '[' at start and ']' at end
    modified_content = f'[{modified_content}]'
    
    # Load as JSON
    json_data = json.loads(modified_content)
    
    return json_data

def get_exported_symbol_address(symbol_name):
    """Retrieve the address of an exported symbol."""
    for entry in idautils.Entries():
        if entry[3] == symbol_name:
            return entry[1]
    return None

def rename_functions_from_json(json_data, snapshot_addr):
    """Read JSON array and rename functions based on provided data."""
    try:                
        total_function_renamed = 0
        for item in json_data:
            method_name = item.get('method_name')
            offset = item.get('offset')
            class_name = item.get('class_name')
            
            # Skip invalid entries
            if not method_name or not offset or not class_name:
                continue
                
            # Skip optimized out methods and anonymous closures
            if method_name in ['<optimized out>', '<anonymous closure>']:
                continue
                
            # Replace :: with DoubleColon for class_name
            if class_name == '::':
                class_name = 'DoubleColon'
                
            # Replace special characters in method_name
            if '=' in method_name:
                method_name = method_name.replace('=', 'EqualSign')
            if '#' in method_name:
                method_name = method_name.replace('#', 'PoundSign')
            if '*' in method_name:
                method_name = method_name.replace('*', 'StarSign')
            if '~' in method_name:
                method_name = method_name.replace('~', 'TildeSign')
            if '/' in method_name:
                method_name = method_name.replace('/', 'SlashSign')
            if '<' in method_name:
                method_name = method_name.replace('<', 'LessSign')
            if '>' in method_name:
                method_name = method_name.replace('>', 'GreatSign')
                
            try:
                # Convert offset from hex string to integer
                offset_int = int(offset, 16)
                # Calculate function address
                function_addr = snapshot_addr + offset_int
                
                # Get current function name
                current_name = idc.get_func_name(function_addr)
                if not current_name:
                    current_name = f"sub_{function_addr:X}"
                
                # Create new function name
                new_name = f"{class_name}__{method_name}__{current_name}"
                
                # Rename the function
                if idc.set_name(function_addr, new_name, idc.SN_CHECK):
                    print(f"Renamed function at 0x{function_addr:X} from {current_name} to {new_name}")
                    total_function_renamed = total_function_renamed + 1
                else:
                    print(f"Failed to rename function at 0x{function_addr:X}")
                    
            except ValueError as e:
                print(f"Error processing offset {offset}: {str(e)}")

        
        print(f">>>> Total functions renamed: {total_function_renamed}\n\n\n")
                
    except json.JSONDecodeError as e:
        print(f"Error decoding JSON file: {str(e)}")
    except Exception as e:
        print(f"Unexpected error: {str(e)}")

def main():
    """Main function to execute the renaming process."""
    # Get the address of _kDartIsolateSnapshotInstructions
    snapshotInstructionsAddr = get_exported_symbol_address('_kDartIsolateSnapshotInstructions')
    
    if snapshotInstructionsAddr is None:
        print("Error: Could not find _kDartIsolateSnapshotInstructions export symbol")
        return
    
    print(f"Found _kDartIsolateSnapshotInstructions at 0x{snapshotInstructionsAddr:X}")
    
    # Path to dump.dart file    
    dump_dart_file_path = "/Users/xyz/Downloads/8ksec/11-FreeFall/dump.dart"
    json_data = modify_and_load_json(dump_dart_file_path)
    
    # Process the JSON data and rename functions
    rename_functions_from_json(json_data, snapshotInstructionsAddr)

if __name__ == '__main__':
    main()
```

This script successfully renames **10,734** out of **14,518** functions, making the binary much easier to navigate:

![App after symbols renaming](https://lh3.googleusercontent.com/pw/AP1GczM14Ufezwi4jilH4OVNEpGRncKwGWdByHtrN_zXRGVx8tQuNp5ibDsUB9N4aOrNtEeVe5aP6FGgbc6u-sTBmz0OSLvJTyV-oPFt36wSASt4ovoPSkHdDlA8DqwTs-UBN5sKXRVbDmwCARdKrBpx-keW=w1308-h654-s-no-gm)
_**Figure: App after symbols renaming**_

With clearer function names, we can now focus on finding the logic responsible for score submission.

## Locating the Score Submission Logic

Our goal is to manipulate the score submission process, so we search IDA’s **Functions** tab for the keyword **"score"** to identify relevant methods:

```assembly
DatabaseHelper__getTopScores__sub_1586F0	__text	00000000001586F0
DatabaseHelper__insertScore__sub_180058	__text	0000000000180058
GameEngine__submitScore__sub_1807D0	__text	00000000001807D0
GameEngine___increaseScore__sub_1813D8	__text	00000000001813D8
```

The `GameEngine__submitScore` method stands out, as it likely handles the final score submission. By cross-referencing (XREF) this method in **IDA**, we find the following assembly code where `submitScore` is called:

```assembly
__text:000000000017FF28                 LDUR            X2, [X1,#0x2F]
__text:000000000017FF2C                 LDUR            X1, [X2,#0x27]
__text:000000000017FF30                 MOV             X16, X1
__text:000000000017FF34                 MOV             X1, X2
__text:000000000017FF38                 MOV             X2, X16
__text:000000000017FF3C                 BL              GameEngine__submitScore__sub_1807D0
```

The registers `X1` and `X2` are set just before the call, suggesting they may hold the score. To confirm, we’ll use **LLDB**, a debugger, to inspect these registers dynamically.

## Patching the Score with LLDB

To debug the app, we need a jailbroken iOS device and a debug server. First, we SSH into the device and start `debugserver`:

```bash
$ ssh ip8
root@localhost's password:
iPhone $ debugserver 0.0.0.0:1234 -w Runner
debugserver-@(#)PROGRAM:LLDB  PROJECT:lldb-1403.2.3.13
 for arm64.
Waiting to attach to process Runner...
Listening to port 1234 for a connection from 0.0.0.0...
```

Next, on our local machine, we use **LLDB** to attach to the running *FreeFall* app:

```bash
$ lldb
  Platform: remote-ios
 Connected: no
  SDK Path: "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/16.4.1 (20E252) arm64e"
 SDK Roots: [ 0] "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/iPhone10,4 16.7.11 (20H360)"
Process 1783 stopped
* thread #1, stop reason = signal SIGSTOP
...
Target 0: (Runner) stopped.
(lldb) image list -o -f App
[  0] 0x0000000103adc000 /private/var/containers/Bundle/Application/1E500AEE-EC1D-4809-BC61-C732349A21E4/Runner.app/Frameworks/App.framework/App(0x0000000103adc000)
(lldb) br s -a 0x0000000103adc000+0x17FF3C
Breakpoint 1: where = App`___lldb_unnamed_symbol7158 + 112, address = 0x0000000103c5bf3c
(lldb) continue
Process 1783 resuming
(lldb)
```

We set a breakpoint at the address where `GameEngine__submitScore` is called (`0x103c5bf3c`). Playing the game until the **"Game Over"** screen (showing a score of **563**) triggers the breakpoint:

![Submit score](https://lh3.googleusercontent.com/pw/AP1GczOcxH-a_8bvtLZXpB64i8H_fcCgWKy7NPfwqRY_vbti8GK9lCHYfvmY6FhDTQn3byoScP29BMiXw63YJJPUfNcUpnkbDQLZ5IlD6tvXPLDBRDc3ZjdNQMRAxgx9ULaosTI8SikgknQ46qJ8sUrVqtRa=w400-h654-s-no-gm)
_**Figure: Submit score**_

Examining the registers at this point:

```bash
(lldb)
Process 1783 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000103c5bf3c App`___lldb_unnamed_symbol7158 + 112
App`___lldb_unnamed_symbol7158:
->  0x103c5bf3c <+112>: bl     0x103c5c7d0    ; ___lldb_unnamed_symbol7162
    0x103c5bf40 <+116>: mov    x1, x0
    0x103c5bf44 <+120>: stur   x1, [x29, #-0x20]
    0x103c5bf48 <+124>: bl     0x103b0d4f8    ; ___lldb_unnamed_symbol828
Target 0: (Runner) stopped.
(lldb) register read
General Purpose Registers:
        x0 = 0x000000033e67b861
        x1 = 0x000000033541b8a1
        x2 = 0x0000000000000233
        x3 = 0x000000033e65d8b1
        x4 = 0x0000000000000014
        x5 = 0x000000034bdddf61
        x6 = 0x0000000333f88081
        x7 = 0x0000000333f880a1
        x8 = 0x0000000334db4251
        x9 = 0x903bf274fd6000c6
       x10 = 0x0000000000000000
       x11 = 0x0000000000000002
       x12 = 0x0000000000000002
       x13 = 0x0000000000000000
       x14 = 0x00000000000007fb
       x15 = 0x000000016d9a7900
       x16 = 0x0000000000000233
       x17 = 0x0000000000000072
       x18 = 0x0000000000000000
       x19 = 0x000000016d8cf000
       x20 = 0x0000000102cda848  Flutter`tonic::FfiDispatcher<void, void (*)(_Dart_Handle*), &flutter::DartRuntimeHooks::ScheduleMicrotask(_Dart_Handle*)>::Call(_Dart_Handle*)
       x21 = 0x0000000526108000
       x22 = 0x0000000333f88081
       x23 = 0x0000000523099e00
       x24 = 0x0000000333f88081
       x25 = 0x000000016d8cf000
       x26 = 0x0000000523099e00
       x27 = 0x0000000335380080
       x28 = 0x0000000800000000
        fp = 0x000000016d9a7940
        lr = 0x0000000103c5bf04  App`___lldb_unnamed_symbol7158 + 56
        sp = 0x000000016d8cf000
        pc = 0x0000000103c5bf3c  App`___lldb_unnamed_symbol7158 + 112
      cpsr = 0x20000000

(lldb) po $x2
563
```

The `X2` register holds the score (**563**). To patch it, we modify `X2` and resume execution:

```bash
(lldb) register write x2 99999999
(lldb) continue
Process 1783 resuming
(lldb)
```

The leaderboard now reflects our hacked score of **99,999,999**:

![Hack score leaderboard using lldb](https://lh3.googleusercontent.com/pw/AP1GczMmD4oztDoY_71YxWFHmw8wAUyfMxD8jzSoqqsnLgddiLrzkgrcyKoaLuVVavGAJvX-0VZExyPYk_zVyCd6x8cAEd_mv8_Q4jrbLyZreLoO7JWvZ_6UZHRg_6okJ3yhMMfEKeYDG9dTGFtgoQBP5e2G=w400-h654-s-no-gm)
_**Figure: Hack score leaderboard using LLDB**_

Success! We’ve bypassed the scoring validation using LLDB. But there’s another approach using **Frida** for dynamic instrumentation.

## Patching the Score with Frida

**Frida** is a dynamic instrumentation toolkit that allows us to hook into running processes and modify their behavior. We use a modified Frida script to intercept the `FlutterMethodCall` class’s `methodCallWithMethodName:arguments:` method, which handles communication between Flutter’s Dart code and native iOS code via a **Method Channel**. A Method Channel is a mechanism that enables Flutter to call native functions, such as database operations. Here’s the script:

```bash
/**
 * run the script to a running app: frida -U -F -l flutter_ios.js
 */
// https://codeshare.frida.re/@denizaydemir/ios-proxy-check-bypass/
// https://codeshare.frida.re/@dki/ios-app-info/
// https://gist.github.com/AICDEV/630feed7583561ec9f9421976e836f90
// #############################################
// HELPER SECTION START
var colors = {
    "resetColor": "\x1b[0m",
    "green": "\x1b[32m",
    "yellow": "\x1b[33m",
    "red": "\x1b[31m"
}

function logSection(message) {
    console.log(colors.green, "#################################################", colors.resetColor);
    console.log(colors.green, message, colors.resetColor);
    console.log(colors.green, "#################################################", colors.resetColor);
}

function logMessage(message) {
    console.log(colors.yellow, "---> " + message, colors.resetColor);
}

function logError(message) {
    console.log(colors.red, "---> ERRROR: " + message, colors.resetColor);
}

function getAllClasses() {
    var classes = [];
    for (var cl in ObjC.classes) {
        classes.push(cl);
    }
    return classes;
}

function filterFlutterClass() {
    var matchClasses = [];
    var classes = getAllClasses();

    for (var i = 0; i < classes.length; i++) {
        if (classes[i].toString().toLowerCase().includes('flu')) {
            matchClasses.push(classes[i]);
        }
    }

    return matchClasses;
}


function getAllMethodsFromClass(cl) {
    return ObjC.classes[cl].$ownMethods;
}

function listAllMethodsFromClasses(classes) {
    for (var i = 0; i < classes.length; i++) {
        var methods = getAllMethodsFromClass(classes[i]);
        for (var a = 0; a < methods.length; a++) {
            logMessage("class: " + classes[i] + " --> method: " + methods[a]);
        }
    }
}

function blindCallDetection(classes) {
    for (var i = 0; i < classes.length; i++) {
        var methods = getAllMethodsFromClass(classes[i]);
        for (var a = 0; a < methods.length; a++) {
            var hook = ObjC.classes[classes[i]][methods[a]];
            try {
                Interceptor.attach(hook.implementation, {
                    onEnter: function (args) {
                        this.className = ObjC.Object(args[0]).toString();
                        this.methodName = ObjC.selectorAsString(args[1]);
                        logMessage("detect call to: " + this.className + ":" + this.methodName);
                    }
                })
            } catch (err) {
                logError("error in trace blindCallDetection");
                logError(err);
            }
        }
    }
}

function singleBlindTracer(className, methodName) {
    try {
        var hook = ObjC.classes[className][methodName];
        Interceptor.attach(hook.implementation, {
            onEnter: function (args) {
                this.className = ObjC.Object(args[0]).toString();
                this.methodName = ObjC.selectorAsString(args[1]);
                logMessage("detect call to: " + this.className + ":" + this.methodName);
            }
        })
    } catch (err) {
        logError("error in trace singleBlindTracer");
        logError(err);
    }
}

// #############################################
//HELPER SECTION END
// #############################################
// BEGIN FLUTTER SECTION
// #############################################
function listAllFlutterClassesAndMethods() {
    var flutterClasses = filterFlutterClass();
    for (var i = 0; i < flutterClasses.length; i++) {
        var methods = getAllMethodsFromClass(flutterClasses[i]);
        for (var a = 0; a < methods.length; a++) {
            logMessage("class: " + flutterClasses[i] + " --> method: " + methods[a]);
        }
    }
}
// https://main-api.flutter.dev/ios-embedder/interface_flutter_method_call.html#ad5ec921ce8616518137964e054753ff7
/*
After enable this tracing, once submit the score we can find the log as below:
    ---> FlutterMethodCall:methodCallWithMethodName:arguments:
    ---> method: insert
    ---> args: {
        arguments =     (
            hjj,
            945,
            1751972579382,
            9dd5b3f5ff4d2a9d70c0f09055f62a7dd11ae38eb43b418e147b58e68e3830f0
        );
        id = 2;
        sql = "INSERT INTO leaderboard (name, score, timestamp, token) VALUES (?, ?, ?, ?)";
    }

As we can see, the args is the dictionary that contains the key "arguments" with an array of value where second item 
in the array is the score.
What we can achieve is that we need to modify this 2nd item to a new score before insert to the database to achieve the goal
*/
const NSString = ObjC.classes.NSString;
const NSNumber = ObjC.classes.NSNumber;
const NSArray = ObjC.classes.NSArray;                
const NSMutableArray = ObjC.classes.NSMutableArray;
function traceFlutterMethodCall() {
    var className = "FlutterMethodCall"
    var methodName = "+ methodCallWithMethodName:arguments:"
    var hook = ObjC.classes[className][methodName];

    try {

        Interceptor.attach(hook.implementation, {
            onEnter: function (args) {

                this.className = ObjC.Object(args[0]).toString();
                this.methodName = ObjC.selectorAsString(args[1]);
                logMessage(this.className + ":" + this.methodName);
                const method = ObjC.Object(args[2]).toString()
                logMessage("method: " + method);
                var argsDict =  ObjC.Object(args[3])
                logMessage("args: " + argsDict.toString());
                logMessage("args type: " + argsDict.class().toString());
                                
                var argsKeys = argsDict.allKeys();
                                                
                var isInsertMethod = false

                // this array will contain modified score (2nd item)
                var modifiedSqlInsertItemsArray = NSMutableArray.array();
                for (var i = 0; i < argsKeys.count(); i++) {
                    var key = argsKeys.objectAtIndex_(i);
                    var value = argsDict.objectForKey_(key);
                    logMessage(key + ":" + value)                    
                    
                    // only handle if this is the call to insert score to database
                    if (method == "insert" && key == "arguments") {
                        isInsertMethod = true
                        
                        // as we observed after printing key "arguments" it is an array, hence we can interate to find the 2nd item in the array to modify
                        const count = value.count().valueOf();
                        for (let i = 0; i !== count; i++) {
                            const element = value.objectAtIndex_(i);
                            // found score item at 2nd index (based-zero)
                            if (i == 1) {
                                // modify score to impossible number
                                const modifiedScoreObj = NSNumber.numberWithInt_(88888888)
                                modifiedSqlInsertItemsArray.addObject_(modifiedScoreObj)
                            } else {
                                modifiedSqlInsertItemsArray.addObject_(element)
                            }                            
                        }
                                                
                        // console.log("value of arguments type: ", value.class().toString())
                    }                    
                }

                // Now we check if it's insert method, then modify the args
                if (isInsertMethod) {
                    // copy existing args to mutable dictionary
                    var mutableDict = ObjC.Object(args[3]).mutableCopy()
                    
                    // update "arguments" item to modified array which include modified new score
                    mutableDict.setObject_forKey_(modifiedSqlInsertItemsArray, "arguments");
                    
                    // update original args
                    args[3] = mutableDict

                    // Logging new modified args
                    console.log("Modified args: ", mutableDict.toString())                    
                }

            }
        })
    } catch (err) {
        logError("error in trace FlutterMethodCall");
        logError(err);
    }
}

// https://api.flutter.dev/objcdoc/Classes/FlutterMethodChannel.html#/c:objc(cs)FlutterMethodChannel(im)invokeMethod:arguments:
function traceFlutterMethodChannel() {
    var className = "FlutterMethodChannel"
    var methodName = "- setMethodCallHandler:"
    var hook = ObjC.classes[className][methodName];

    try {
        Interceptor.attach(hook.implementation, {
            onEnter: function (args) {
                this.className = ObjC.Object(args[0]).toString();
                this.methodName = ObjC.selectorAsString(args[1]);
                logMessage(this.className + ":" + this.methodName);
                logMessage("method: " + ObjC.Object(args[2]).toString());
            }
        })
    } catch (err) {
        logError("error in trace FlutterMethodChannel");
        logError(err);
    }
}

// enum function from defined classes
function inspectInteresingFlutterClasses(classes) {
    logSection("START BLIND TRACE FOR SPECIFIED METHODS");
    for (var i = 0; i < classes.length; i++) {
        logMessage("inspect all methods from: " + classes[i]);
        var methods = getAllMethodsFromClass(classes[i]);
        for (var a = 0; a < methods.length; a++) {
            logMessage("method --> " + methods[a]);
            blindTraceWithPayload(classes[i], methods[a]);
        }
    }
}

function blindTraceWithPayload(className, methodName) {
    try {
        var hook = ObjC.classes[className][methodName];
        Interceptor.attach(hook.implementation, {
            onEnter: function (args) {
                this.className = ObjC.Object(args[0]).toString();
                this.methodName = ObjC.selectorAsString(args[1]);
                logMessage(this.className + ":" + this.methodName);
                logMessage("payload: " + ObjC.Object(args[2]).toString());
            },
        })
    } catch (err) {
        logError("error in blind trace");
        logError(err);
    }
}

// #############################################
// END FLUTTER SECTION
// #############################################
/**
 * check if a method in the specified class get called
 */
logSection("BLIND TRACE NATIVE FUNCTION");
var blindCallClasses = [
    "FlutterStringCodec",
]
blindCallDetection(blindCallClasses);

/**
 * List found flutter classes and there methods
 */
logSection("SEARCH ALL FLUTTER CLASSES AND METHODS");
listAllFlutterClassesAndMethods();


/**
 * define custom class for further investigation. be careful: it calls blindTraceWithPayload logMessage("payload: " + ObjC.Object(args[2]).toString());
 * If you are not sure if the arg[2] is present read the function docs or do some try catch
 */
var interestingFlutterClasses = [
    //https://api.flutter.dev/objcdoc/Protocols/FlutterMessageCodec.html#/c:objc(pl)FlutterMessageCodec(im)encode:
    "FlutterJSONMessageCodec",
    //https://api.flutter.dev/objcdoc/Protocols/FlutterMethodCodec.html
    "FlutterJSONMethodCodec",
    //"FlutterStandardReader",
    //https://api.flutter.dev/objcdoc/Classes/FlutterEventChannel.html
    "FlutterEventChannel",
    //https://api.flutter.dev/objcdoc/Classes/FlutterViewController.html
    //"FlutterViewController",
    //https://api.flutter.dev/objcdoc/Classes/FlutterBasicMessageChannel.html
    "FlutterBasicMessageChannel",
]

// inspectInteresingFlutterClasses(interestingFlutterClasses)

/**
 * trace implementation for
 * https://api.flutter.dev/objcdoc/Classes/FlutterMethodCall.html
 * https://api.flutter.dev/objcdoc/Classes/FlutterMethodChannel.html
 */
logSection("TRACING FLUTTER BEHAVIOUR");
traceFlutterMethodCall();
// traceFlutterMethodChannel();


// logSection("SINGLE BLIND TRACING");
// singleBlindTracer("FlutterObservatoryPublisher","- url")
```

Running the script with Frida:

```bash
$ frida -U -F -l modified_flutter_ios.js
     ____
    / _  |   Frida 17.2.11 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to iPhone (id=xxx)
 ---> FlutterMethodCall:methodCallWithMethodName:arguments:
 ---> method: insert
 ---> args: {
    arguments =     (
        "frida score",
        677,
        1758016244782,
        48d8816dca7093011c8941506976ca1e2af4e6c75a84b6c5c9d17aae193adeb7
    );
    id = 2;
    sql = "INSERT INTO leaderboard (name, score, timestamp, token) VALUES (?, ?, ?, ?)";
}
 ---> args type: __NSDictionaryM
 ---> sql:INSERT INTO leaderboard (name, score, timestamp, token) VALUES (?, ?, ?, ?)
 ---> arguments:(
    "frida score",
    677,
    1758016244782,
    48d8816dca7093011c8941506976ca1e2af4e6c75a84b6c5c9d17aae193adeb7
)
 ---> id:2
Modified args:  {
    arguments =     (
        "frida score",
        88888888,
        1758016244782,
        48d8816dca7093011c8941506976ca1e2af4e6c75a84b6c5c9d17aae193adeb7
    );
    id = 2;
    sql = "INSERT INTO leaderboard (name, score, timestamp, token) VALUES (?, ?, ?, ?)";
}  
```

The script intercepts the `insert` method call, modifies the score in the `arguments` array to **88,888,888** and updates the leaderboard:

![Hack score leaderboard using Frida](https://lh3.googleusercontent.com/pw/AP1GczMaMoVhKynDJ-8WaAklMvfj1P0zZSr9zbSTUwPYJ2QUEN0ZS_s1LJ19GzuYkeJRSTzYYOe4waskqkbR-6qYKQZJ-m37H-jQ7MadOGAA_NYwM25GoQhevnwnd4NIqVKDZu0XkUjWb3sWRO0ZAGUCv1ci=w1120-h654-s-no-gm)
_**Figure: Hack score leaderboard using Frida**_

## Conclusion

Reverse engineering *FreeFall* was a rewarding challenge that showcased the power of combining tools like reFlutter, IDA, LLDB, and Frida. By leveraging **reFlutter’s** dynamic analysis capabilities, we extracted critical runtime information. LLDB allowed us to pinpoint and patch the score in memory, while Frida offered a dynamic way to intercept and modify method calls. This approach avoided deep dives into Flutter’s Dart VM internals, making it accessible yet effective. I hope this guide inspires you to explore iOS reverse engineering with curiosity and confidence!

![8ksec challenge feedback](https://lh3.googleusercontent.com/pw/AP1GczMkQOu5P7hyWvrqhNEHbeh06vMzksZGtKJ2MxzZ6Y6oJfOVUNuI_B1ZJZpnsOdyApD9_rahQN2yFWylqg-L8QPb2bNHlFOZOCJL4I9aX5GGXQwA4YK2D7pjQm91cJEp7zXLuZ9dxq5SRER6LRkQ5g3N=w972-h654-s-no-gm)
_**Figure: 8ksec challenge feedback**_

## Further Reading

- [reFlutter](https://github.com/Impact-I/reFlutter)
- [The Current State & Future of Reverse Engineering Flutter™ Apps](https://www.guardsquare.com/blog/current-state-and-future-of-reversing-flutter-apps)
- [Reversing Flutter Application by using Dart Runtime](https://conference.hitb.org/hitbsecconf2023hkt/materials/D2%20COMMSEC%20-%20B(l)utter%20%E2%80%93%20Reversing%20Flutter%20Applications%20by%20using%20Dart%20Runtime%20-%20Worawit%20Wangwarunyoo.pdf)
- [FlutterMethodCall](https://main-api.flutter.dev/ios-embedder/interface_flutter_method_call.html#ad5ec921ce8616518137964e054753ff7)
- [8ksec iOS Application Exploitation Challenges](https://academy.8ksec.io/course/ios-application-exploitation-challenges)
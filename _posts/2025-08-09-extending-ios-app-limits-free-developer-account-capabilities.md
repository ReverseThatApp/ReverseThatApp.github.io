---
layout: post
title: "Exploring iOS App Limits: Extending Free Developer Account Capabilities"
categories: [iOS, Security]
tags: [ios, installd, security, xcode, jailbreak, debugging, lldb, frida]
permalink: /posts/extending-ios-app-limits-free-developer-account-capabilities/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczOyB-pzUU4w-e15aXpC8KDDwsmMgjUnwfXKZ1zi9BYkXwrEtI0pcu9YQ1PhdNd_nK6nVoJuMwUebF73AGrsKQMbnD3aIot_OT1-LD_hX-v3rrinp7OklTkPDoXbKx91P_q3-UGQEPY7uJazdGxdXk9I=w2126-h1262-s-no-gm?authuser=2
    alt: Bypassing Apple's 3-App Limit
---

Apple’s free developer accounts offer a convenient way to test iOS apps using Xcode, but they impose a firm restriction: you’re limited to installing only three apps on your device at a time. If you attempt to add a fourth, Xcode will display the error message: `"The maximum number of apps for free development profiles has been reached."` This can be quite limiting for experimentation. In this beginner-friendly guide, we’ll explore how to extend this constraint on a jailbroken iOS device by modifying the `installd` process, providing step-by-step explanations to make the process approachable even for those new to iOS modifications.

**Important**: This guide is intended solely for educational and research purposes on a jailbroken test device not used for personal or commercial purposes. Modifying iOS or extending Apple’s restrictions, including the 3-app limit, may violate the Apple Developer Program License Agreement (ADPLA), void warranties, and risk account suspension or device security issues. Proceed at your own risk and ensure compliance with Apple’s terms.

## What You’ll Need

To prepare for the steps ahead, let’s outline the essential tools and requirements that will help you successfully navigate this guide:

- **Jailbroken iOS Device**: You'll need an iPhone or iPad that's been jailbroken, running a version like iOS 16.7.11, with jailbreaks such as `palera1n` or `unc0ver` providing the necessary access.
- **Tools**:
  - `ipsw`: This utility allows you to download and examine iOS firmware files, which is crucial for locating system components.
  - IDA: A disassembler tool for analyzing the `installd` binary to understand its internal logic.
  - LLDB or Frida: Debugging tools for dynamically patching and modifying running processes like `installd`.
  - `debugserver`: Enables remote debugging directly on the device, facilitating live interactions.
  - `ldid`: Used for signing modified binaries to ensure they can run on the jailbroken environment.
- **SSH Access**: Establishing a secure shell connection to your device enables remote command execution, which is key for many of the debugging steps.
- **Basic Skills**: While some experience with terminal commands will make things smoother, we'll provide detailed explanations to guide you through even if you're just starting out.

## Step 1: Encounter the 3-App Limit

To fully grasp the restriction we're addressing, it's helpful to first experience it firsthand by triggering the limit in Xcode:

1. Launch Xcode and create a new iOS app project, then configure it to sign with your free Apple developer account in the `Signing & Capabilities` section.
2. Connect your jailbroken iPhone to your computer via USB and run the app, observing as it successfully installs and appears on the device's home screen.
3. Modify the `Bundle Identifier` (for example, changing it from `com.example.app` to `com.example.app2`), allow Xcode to create a new provisioning profile, and run the app again to see it install under the updated identifier.
4. Repeat this modification and installation process two additional times with different bundle identifiers. On your fourth attempt, Xcode will present the error:

![Xcode 3-App Limit Error](https://lh3.googleusercontent.com/pw/AP1GczOyB-pzUU4w-e15aXpC8KDDwsmMgjUnwfXKZ1zi9BYkXwrEtI0pcu9YQ1PhdNd_nK6nVoJuMwUebF73AGrsKQMbnD3aIot_OT1-LD_hX-v3rrinp7OklTkPDoXbKx91P_q3-UGQEPY7uJazdGxdXk9I=w2126-h1262-s-no-gm?authuser=2)
*Figure: Xcode Error for Exceeding 3-App Limit*

This demonstration highlights Apple's enforcement of the three-app cap for free accounts, setting the stage for our extending strategy.

## Step 2: Track Down the Restriction

Building on the error message we've encountered, we'll now investigate its source by examining the iOS firmware to identify where the limit is implemented.

### Get the iOS Firmware

Begin by determining your device's exact iOS version, such as iOS 16.7.11 on an iPhone10,4. You can refer to my related [Beginner's Guide to Disabling ASLR in iOS Apps](/posts/disable-aslr-ios-apps) for guidance on identifying and downloading the appropriate iOS firmware if needed.

Once obtained, mount the firmware’s file system to enable a thorough search of its contents:

```bash
$ ipsw mount fs iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
   • Mounted fs DMG 087-86684-034.dmg
      • Press Ctrl+C to unmount '/tmp/087-86684-034.dmg.mount' ...
```

### Search for the Error

With the file system mounted, conduct a search for the key phrase `"maximum number"` to uncover relevant components:

```bash
$ cd /tmp/087-86684-034.dmg.mount
$ grep -r -i "maximum number" .
Binary file ./usr/bin/sysdiagnose matches
Binary file ./usr/libexec/misd matches
Binary file ./usr/libexec/dasd matches
Binary file ./usr/libexec/locationd matches
Binary file ./usr/libexec/fmfd matches
Binary file ./usr/libexec/usermanagerd matches
Binary file ./usr/libexec/findmydeviced matches
Binary file ./usr/libexec/seserviced matches
Binary file ./usr/libexec/biomesyncd matches
Binary file ./usr/libexec/sensorkitd matches
Binary file ./usr/libexec/installd matches
...
```

Among these results, the entry for `./usr/libexec/installd` stands out as particularly relevant, given that `installd` plays a central role in managing app installations on iOS.

### What Is `installd`?

`installd` serves as a background process, or daemon, in iOS that oversees the installation of applications, handling tasks from App Store downloads to Xcode deployments and even sideloaded apps. It ensures proper signature validation, profile management, and adherence to rules such as the 3-app limit for free developer accounts. Understanding `installd` allows us to target modifications that can alter how it enforces these constraints, potentially enabling the installation of additional apps.

### Reverse Engineer `installd`

Proceed by loading the `installd` binary, located at `/usr/libexec/installd`, into IDA for analysis. Navigate to the `Strings` tab and search for `"maximum number"` to locate the error string:

![IDA Strings Search](https://lh3.googleusercontent.com/pw/AP1GczO1yxj0XI3lPRTys4enlLCSP1TWvZBWZlqpz_AqbdxBgk5L1zs4s8ExXqXlD_RgjCS33Uvk1zDhSGCKvfl1XO1he8uoc9fyZaNuE-UZtY01MoKHVVbUu8-nFUwzwSjyFz4pz4SgQM11ru7wLN_UDf8P=w2126-h1176-s-no-gm)
*Figure: Finding the Error String in IDA*

Trace the cross-reference (XREF) from this string to the method `-[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:]`, which is responsible for tracking and validating apps under free developer profiles.

### Understand the Logic

The `MIFreeProfileValidatedAppTracker` class maintains a set of app identifiers associated with free profiles. Here's the decompiled pseudocode from IDA, presented in Objective-C form to illustrate the method's structure:

```objective-c
bool __cdecl -[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:](
        MIFreeProfileValidatedAppTracker *self,
        SEL a2,
        id a3_applicationIdentifier,
        id a4_bundle,
        id *a5_error)
{
  id v8_applicationIdentifier; // x19
  id v9_bundle; // x20
  NSObject *v10; // x21
  void *v11_MIFileManager; // x21
  void *v12_refs; // x0
  id v13; // x23
  bool v14; // w24
  unsigned __int8 v16; // w25
  void *v17_appTracker_refs; // x25
  void *v18; // x26
  unsigned __int8 v19; // w27
  __int64 v20; // x26
  void *v21; // x25
  void *v22; // x24
  __int64 v23; // x0
  __int64 v24; // x26
  void *v25; // x25
  unsigned __int8 v26; // w27
  id v27; // x26
  void *v28; // x22
  __int64 v29; // x23
  __int64 v30; // x0
  id v31; // [xsp+18h] [xbp-88h] BYREF
  id v32; // [xsp+20h] [xbp-80h] BYREF
  _QWORD v33[2]; // [xsp+28h] [xbp-78h] BYREF
  _QWORD v34[2]; // [xsp+38h] [xbp-68h] BYREF

  v8_applicationIdentifier = objc_retain(a3_applicationIdentifier);
  v9_bundle = objc_retain(a4_bundle);
  v10 = (NSObject *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker q](self, "q"));
  dispatch_assert_queue_V2(v10);
  objc_release(v10);
  v11_MIFileManager = (void *)objc_claimAutoreleasedReturnValue(+[MIFileManager defaultManager](&OBJC_CLASS___MIFileManager, "defaultManager"));
  if ( ((unsigned int)objc_msgSend(v9_bundle, "isPlaceholder") & 1) != 0
    || (unsigned int)objc_msgSend(v9_bundle, "bundleType") != 4 )
  {
    v13 = 0LL;
    v14 = 1;
    goto RETURN_LABEL_6;
  }
  v12_refs = (void *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker refs](self, "refs"));
  if ( v12_refs )
  {
    objc_release(v12_refs);
    v13 = 0LL;
  }
  else
  {
    v32 = 0LL;
    v16 = -[MIFreeProfileValidatedAppTracker _onQueue_discoverReferencesWithError:](
            self,
            "_onQueue_discoverReferencesWithError:",
            &v32);
    v13 = objc_retain(v32);
    if ( (v16 & 1) == 0 )
      goto LABEL_16;
  }
  v17_appTracker_refs = (void *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker refs](self, "refs"));
  if ( (unsigned __int64)objc_msgSend(v17_appTracker_refs, "count") <= 2 )
  {
    objc_release(v17_appTracker_refs);
    goto LESS_THAN_3_FLOW;
  }
  v18 = (void *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker refs](self, "refs"));
  v19 = (unsigned __int8)objc_msgSend(v18, "containsObject:", v8_applicationIdentifier);
  objc_release(v18);
  objc_release(v17_appTracker_refs);
  if ( (v19 & 1) != 0 )
  {
LESS_THAN_3_FLOW:
    v25 = (void *)objc_claimAutoreleasedReturnValue(objc_msgSend(v9_bundle, "bundleURL"));
    v31 = v13;
    v26 = (unsigned __int8)objc_msgSend(
                             v11_MIFileManager,
                             "setValidatedByFreeProfileAppIdentifier:insecurelyCachedOnBundle:error:",
                             v8_applicationIdentifier,
                             v25,
                             &v31);
    v27 = objc_retain(v31);
    objc_release(v13);
    objc_release(v25);
    if ( (v26 & 1) != 0 )
    {
      v28 = (void *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker refs](self, "refs"));
      objc_msgSend(v28, "addObject:", v8_applicationIdentifier);
      objc_release(v28);
      v14 = 1;
      v13 = v27;
      goto RETURN_LABEL_6;
    }
    v29 = MIInstallerErrorDomain;
    v21 = (void *)objc_claimAutoreleasedReturnValue(objc_msgSend(v9_bundle, "bundleURL"));
    v22 = (void *)objc_claimAutoreleasedReturnValue(objc_msgSend(v21, "path"));
    v30 = log_error_sub_10000D854(
            "-[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:]",
            184LL,
            v29,
            4LL,
            v27,
            0LL,
            CFSTR("Failed to set app identifier (%@) for %@"));
    v13 = (id)objc_claimAutoreleasedReturnValue(v30);
    objc_release(v27);
    goto LABEL_15;
  }
  v20 = MIInstallerErrorDomain;
  v33[0] = CFSTR("LegacyErrorString");
  v33[1] = MILibMISErrorNumberKey;
  v34[0] = CFSTR("ApplicationVerificationFailed");
  v34[1] = &off_100078EF8;
  v21 = (void *)objc_claimAutoreleasedReturnValue(
                  +[NSDictionary dictionaryWithObjects:forKeys:count:](
                    &OBJC_CLASS___NSDictionary,
                    "dictionaryWithObjects:forKeys:count:",
                    v34,
                    v33,
                    2LL));
  v22 = (void *)objc_claimAutoreleasedReturnValue(-[MIFreeProfileValidatedAppTracker refs](self, "refs"));
  v23 = log_error_sub_10000D854(
          "-[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:]",
          179LL,
          v20,
          13LL,
          0LL,
          v21,
          CFSTR("This device has reached the maximum number of installed apps using a free developer profile: %@"));
  v24 = objc_claimAutoreleasedReturnValue(v23);
  objc_release(v13);
  v13 = (id)v24;
LABEL_15:
  objc_release(v22);
  objc_release(v21);
LABEL_16:
  if ( a5_error )
  {
    v13 = objc_retainAutorelease(v13);
    v14 = 0;
    *a5_error = v13;
  }
  else
  {
    v14 = 0;
  }
RETURN_LABEL_6:
  objc_release(v11_MIFileManager);
  objc_release(v13);
  objc_release(v9_bundle);
  objc_release(v8_applicationIdentifier);
  return v14;
}
```

And this is a simplified version in Objective-C with meaningful variable names and a focus on brevity which eliminates the need for explicit `retain` and `release` calls:

```objective-c
- (BOOL)_onQueue_addReferenceForApplicationIdentifier:(NSString *)appIdentifier
                                              bundle:(NSBundle *)bundle
                                               error:(NSError **)error {
    // Verify dispatch queue
    dispatch_assert_queue([self queue]);

    // Skip invalid bundles
    if (bundle.isPlaceholder || bundle.bundleType != 4) {
        return YES;
    }

    // Get file manager
    MIFileManager *fileManager = [MIFileManager defaultManager];

    // Load or discover app references
    NSMutableSet *refs = [self refs];
    if (!refs) {
        NSError *discoveryError;
        if (![self _onQueue_discoverReferencesWithError:&discoveryError]) {
            if (error) *error = discoveryError;
            return NO;
        }
        refs = [self refs];
    }

    // Check for 3-app limit
    if (refs.count > 3 && ![refs containsObject:appIdentifier]) {
        NSError *maxAppsError = [[NSError alloc] initWithDomain:@"MIInstallerErrorDomain"
                                                          code:13
                                                      userInfo:@{@"LegacyErrorString": @"ApplicationVerificationFailed"}];
        [self logErrorWithMessage:@"This device has reached the maximum number of installed apps using a free developer profile"];
        if (error) *error = maxAppsError;
        return NO;
    }

    // Add new app identifier
    NSError *setError;
    if ([fileManager setValidatedByFreeProfileAppIdentifier:appIdentifier
                                   insecurelyCachedOnBundle:bundle.bundleURL
                                                      error:&setError]) {
        [refs addObject:appIdentifier];
        return YES;
    }

    // Log error if addition fails
    [self logErrorWithMessage:[NSString stringWithFormat:@"Failed to set app identifier (%@)", appIdentifier]];
    if (error) *error = setError;
    return NO;
}
```

**What’s Happening**:
1. **Queue Check**: Ensures the method runs on the correct dispatch queue.
2. **Bundle Check**: Ignores non-app bundles (e.g., placeholders).
3. **Reference Set**: Loads a set (`refs`) of installed app identifiers. If empty, it fetches existing apps.
4. **Limit Check**: If `refs.count > 3` (more than three apps) and the new `appIdentifier` isn’t in `refs`, it triggers the error.
5. **Add App**: If the limit isn’t hit, it links the `appIdentifier` to the bundle’s URL and adds it to `refs`.

## Step 3: Patch `installd` with LLDB

We’ll use LLDB to dynamically patch `installd` on your jailbroken device, making it think fewer than three apps are installed.

### Prepare Debugging

Setting up debugging requires `debugserver` running on your device to facilitate remote connections. Locate `debugserver` in the mounted firmware at `/usr/libexec/debugserver`, then follow [this guide](https://felipejfc.medium.com/the-ultimate-guide-for-live-debugging-apps-on-jailbroken-ios-12-4c5b48adf2fb) to sign it with proper entitlements (e.g., `com.apple.security.get-task-allow`) and copy it to your device. Once ready, initiate the debugging session:

1. **Simplify SSH**: Configure your `~/.ssh/config` file to streamline connections to the device, reducing the need for repetitive command-line arguments:
   ```bash
   Host ip8
       HostName localhost
       User root
       Port 2222
       ServerAliveInterval 30
       ServerAliveCountMax 1200
   ```
   

2. **Forward Ports**: On your host machine:
   
   Forward `debugserver` port to enable remote debugging:
   ```bash
   $ iproxy 1234 1234
   Creating listening port 1234 for device port 1234
   ```

   
   Forward SSH port for secure shell access:
   ```bash
   $ iproxy 2222 44
   ```

3. **Configure LLDB**: 

    Update your `~/.lldbinit` file with settings to automatically select the remote iOS platform and connect to the forwarded port:
   ```bash
   platform select remote-ios
   process connect connect://localhost:1234
   ```

4. **Run debugserver**: SSH into your device and launch `debugserver` to attach to the `installd` process, listening on the specified port for incoming connections:
   ```bash
   $ ssh ip8
   root@localhost's password:
   iPhone $ debugserver 0.0.0.0:1234 -a installd
   debugserver-@(#)PROGRAM:LLDB  PROJECT:lldb-1403.2.3.13
   for arm64.
   Attaching to process installd...
   Listening to port 1234...
   ```

5. **Start LLDB**: 

    On your host machine, open LLDB to establish the connection and begin the debugging session:
    ```bash
    $ lldb
    Platform: remote-ios
    Connected: no
    SDK Path: "/Users/rta/Library/Developer/Xcode/iOS DeviceSupport/16.4.1 (20E252) arm64e"
    SDK Roots: [ 0] "/Users/rta/Library/Developer/Xcode/iOS DeviceSupport/iPhone10,4 16.7.11 (20H360)"
    Process 364 stopped
    * thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
        frame #0: 0x0000000216cd9030 libsystem_kernel.dylib`mach_msg2_trap + 8
    libsystem_kernel.dylib`mach_msg2_trap:
    ->  0x216cd9030 <+8>: ret

    libsystem_kernel.dylib`macx_swapon:
        0x216cd9034 <+0>: mov    x16, #-0x30 ; =-48
        0x216cd9038 <+4>: svc    #0x80
        0x216cd903c <+8>: ret
    Target 0: (installd) stopped.
    (lldb)
    ```

### Locate the ASLR Slide

Since `installd` employs Address Space Layout Randomization (ASLR) to vary its memory layout, determining its base address is necessary for accurate breakpoint placement. In LLDB, retrieve this information with:
```bash
(lldb) image list -o -f installd
[  0] 0x0000000004ce0000 /usr/libexec/installd(0x0000000104ce0000)
```

The ASLR slide, shown here as `0x0000000004ce0000`, will differ in your setup but is essential for adjusting static addresses.

### Set a Breakpoint

Within IDA, the critical `refs.count` check appears at this assembly snippet, where the comparison enforces the app limit:
```assembly
__text:000000010002E5A8 loc_10002E5A8                           ; CODE XREF: -[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:]+AC↑j
__text:000000010002E5A8                 MOV             X0, X24
__text:000000010002E5AC                 BL              _objc_msgSend$refs ; -[MIFreeProfileValidatedAppTracker refs] ...
__text:000000010002E5B0                 BL              _objc_claimAutoreleasedReturnValue
__text:000000010002E5B4                 MOV             X25, X0
__text:000000010002E5B8                 BL              _objc_msgSend$count
__text:000000010002E5BC                 CMP             X0, #2
__text:000000010002E5C0                 B.LS            loc_10002E694
```

Combine the ASLR slide with this static address to set a precise breakpoint in LLDB:
```bash
(lldb) breakpoint set --address 0x000000010002E5BC+0x0000000004ce0000
Breakpoint 1: where = installd`___lldb_unnamed_symbol952 + 316, address = 0x0000000104d0e5bc
(lldb) continue
Process 364 resuming
(lldb)
```

### Apply the Patch

Proceed by attempting to install a fourth app from Xcode, which will pause execution at the breakpoint for intervention:
```bash
Process 364 stopped
* thread #4, queue = 'com.apple.installd.MIFreeProfileValidatedAppTracker', stop reason = breakpoint 1.1
    frame #0: 0x0000000104d0e5bc installd`___lldb_unnamed_symbol952 + 316
installd`___lldb_unnamed_symbol952:
->  0x104d0e5bc <+316>: cmp    x0, #0x2
    0x104d0e5c0 <+320>: b.ls   0x104d0e694    ; <+532>
    0x104d0e5c4 <+324>: mov    x0, x24
    0x104d0e5c8 <+328>: bl     0x104d2d380
Target 0: (installd) stopped.
(lldb)
```

Inspect the `X0` register, which holds the current app count value:
```bash
(lldb) register read x0
      x0 = 0x0000000000000003
```

Since `X0` reflects `3`, it activates the limit; adjust it to `2` to deceive the check and allow the installation to proceed:
```bash
(lldb) register write x0 2
(lldb) continue
Process 364 resuming
```

Upon success, the app installs without issue—test with another bundle identifier to verify the extension is effective.

## Step 4: Patch with Frida (Alternative)

For those preferring a script-driven method over manual debugging, Frida offers a flexible alternative to hook into `installd` and automate the modification of the register during the count check.

Begin by obtaining the ASLR slide for `installd` using Frida's module enumeration, which helps identify the runtime base address for accurate hooking:

```bash
$ frida -p -U -n installd -e 'Process.enumerateModules().forEach(m => { if (m.name === "installd") console.log(m.name + " @ " + m.base); })'
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
   . . . .   Connected to iPhone (id=5b20ddd0c26f437d20524aa84c5088a48af5c539)

[iPhone::installd ]-> Process.enumerateModules().forEach(m => { if (m.name === "installd") console.log(m.name + " @ " + m.base); })
installd @ 0x104ce0000
```

With the base address (e.g., `0x104ce0000`), calculate the runtime address for the refs.count check (static offset `0x000000010002E5BC`) and create a Frida script to intercept the instruction in the `onEnter` handler, where you can directly modify `this.context.x0` to alter the count value:
```javascript
const filteredInstalldModules = Process.enumerateModules().filter(m => m.name == "installd")
if (filteredInstalldModules.length > 0) {
    const installdModule = filteredInstalldModules[0]
    var baseAddress = ptr(installdModule.base); // Replace with your installd base address
    var targetAddress = baseAddress.add(0x0002E5BC); // Static offset for CMP X0, #2

    Interceptor.attach(targetAddress, {
        onEnter: function (args) {
            // Log current X0 value for verification
            console.log("Current refs.count (X0): " + this.context.x0.toInt32());
            
            // Patch X0 to 2 to extend the limit check
            this.context.x0 = ptr(2);
            
            // Log the patched value
            console.log("Patched refs.count (X0): " + this.context.x0.toInt32());
        }
    });

    console.log("Frida script loaded. Ready to patch installd.");
} else {
    console.log("installd has not been launched yet!!!");
}
```

Run the script on your host machine, targeting the `installd` process on the device, to enable the hook:
```bash
$ frida -U -n installd -l frida_patch_installd.js
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
   . . . .   Connected to iPhone (id=5b20ddd0c26f437d20524aa84c5088a48af5c539)
Attaching...
Frida script loaded. Ready to patch installd.
[iPhone::installd ]-> Current refs.count (X0): 3
Patched refs.count (X0): 2
```

With the script active, attempt app installations from Xcode—the hook will automatically adjust the count, extending the limit without manual intervention each time.

## Step 5: Static Patching (Optional)

If you seek a more enduring solution that persists across restarts, consider statically patching the `installd` binary to raise the comparison threshold, such as changing `CMP X0, #2` to `CMP X0, #0x20` to support up to 32 apps (actually you cant create more than 10 App IDs every 7 days):

1. Open the binary in IDA or a hex editor to locate and modify the comparison instruction, effectively increasing the allowed app count.
2. Re-sign the altered binary with `ldid`, applying the original entitlements to maintain compatibility.
3. Install the `AppSync Unified` tweak via your package manager to disable codesign verification, preventing rejection of the modified binary.
4. Replace the original `/usr/libexec/installd` on the device with your patched version, then restart the daemon to apply the changes.

**Caution:** Static patches carry risks, as they may become incompatible following iOS updates or when the daemon restarts unexpectedly.

## Results

With the extension in place, you've successfully expanded beyond the three-app boundary. As an illustration, here's a device running eight apps installed under a single free account, demonstrating the effectiveness of the modifications:

![8 Apps Installed](https://lh3.googleusercontent.com/pw/AP1GczMhPGu172Juqs9_xvD1l43jmcSoKWSeoGoxUGwP5Hawwi44diN8rybcFJcQ1aT93yt4j7WHYjcHuoNfqrHo74ZL53d_1v-oyEbzlFzYlFywo691BDZ5ByVzMirvCW37DkPVPfdZysnWNb7V1C8GAi-6=w958-h1624-s-no-gm)
*Figure: Eight Apps Installed with Free Account*

## Bonus: Inspecting MobileInstallation Logs

As a supplementary insight, let’s peek behind the curtain to see exactly what happens when the 3‑app limit is reached at the system level. The `installd` daemon keeps detailed records in the MobileInstallation logs, stored at `/private/var/installd/Library/Logs/MobileInstallation`

Here’s an example log entry from a failed fourth app installation attempt:

```
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) -[MIClientConnection _uninstallIdentities:withOptions:completion:]: Uninstall requested by installcoordinationd (pid 117 (501/501)) for identity [com.example.flutterHelloWorld1xz782/PersonalPersonaPlaceholderString] with options: {
    WaitForStorageDeletion = 0;
}
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) -[MIUninstaller performUninstallationByRevokingTemporaryReference:error:]: Taking termination assertion on com.example.flutterHelloWorld1xz782
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) -[MIUninstaller _uninstallBundleWithIdentity:linkedToChildren:waitForDeletion:uninstallReason:temporaryReference:wasLastReference:error:]: Uninstalling identifier com.example.flutterHelloWorld1xz782
Mon Aug 11 14:09:16 2025 [8567] <err> (0x17002b000) -[MIUninstaller _uninstallBundleWithIdentity:linkedToChildren:waitForDeletion:uninstallReason:temporaryReference:wasLastReference:error:]: Destroying container com.example.flutterHelloWorld1xz782 with persona (null) at /private/var/containers/Bundle/Application/0E17174C-14FA-43BE-84F9-73F1A1786879
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) MIUninstallProfilesForAppIdentifier: Uninstalling profiles for **********.com.example.flutterHelloWorld1xz782
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) MIUninstallProfilesForAppIdentifier_block_invoke: Removing matching profile for **********.com.example.flutterHelloWorld1xz782: 09c78fdb-3b1d-41aa-ae1a-e3d061ec7c12
Mon Aug 11 14:09:16 2025 [8567] <err> (0x17002b000) -[MIUninstaller _uninstallBundleWithIdentity:linkedToChildren:waitForDeletion:uninstallReason:temporaryReference:wasLastReference:error:]: Destroying container com.example.flutterHelloWorld1xz782 with persona 24571DA7-E66C-4D48-AC31-32C3F9F38336 at /private/var/mobile/Containers/Data/Application/C278AD12-A1FF-4179-8B62-6730A52CEA90
Mon Aug 11 14:09:16 2025 [8567] <notice> (0x17002b000) -[MILaunchServicesOperationManager _onQueue_addPendingLaunchServicesOperation:error:]: Added pending LS operation <MILaunchServicesUnregisterOperation: EE61C5E9-46AA-44E0-ADE9-1F91C9C73A47:3 com.example.flutterHelloWorld1xz782/MIInstallationDomainSystemShared>
Mon Aug 11 14:09:17 2025 [8567] <notice> (0x17002b000) -[MILaunchServicesOperationManager _onQueue_removePendingLaunchServicesOperationForUUID:dueToLSSave:]: Removing operation after LS save: <MILaunchServicesUnregisterOperation: EE61C5E9-46AA-44E0-ADE9-1F91C9C73A47:3 com.example.flutterHelloWorld1xz782/MIInstallationDomainSystemShared>
Mon Aug 11 14:09:37 2025 [8567] <notice> (0x17002b000) -[MIClientConnection _installURL:identity:targetingDomain:options:completion:]: Running installation as QOS_CLASS_USER_INITIATED
Mon Aug 11 14:09:37 2025 [8567] <notice> (0x17002b000) -[MIClientConnection _doInstallationForURL:identity:domain:options:completion:]: Install of "/private/var/containers/Shared/SystemGroup/systemgroup.com.apple.installcoordinationd/Library/InstallCoordination/PromiseStaging/46E2CDAE-A56A-4A18-BDBD-1A7E8ABC295E/Flutter Hello World.app" type Placeholder (LSInstallType = 1, Domain: MIInstallationDomainDefault) requested by installcoordinationd (pid 117 (501/501))
Mon Aug 11 14:09:37 2025 [8567] <notice> (0x17002b000) -[MIInstaller _installInstallable:containingSymlink:error:]: Installing <MIInstallableBundle ID=com.example.flutterHelloWorld1xz782; Version=1, ShortVersion=(null)>
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17002b000) -[MIContainer makeContainerLiveReplacingContainer:reason:waitForDeletion:withError:]: Made container live for com.example.flutterHelloWorld1xz782 at /private/var/mobile/Containers/Data/Application/632659FC-08F0-488F-84FD-D81F377D1786
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17002b000) -[MIContainer makeContainerLiveReplacingContainer:reason:waitForDeletion:withError:]: Made container live for com.example.flutterHelloWorld1xz782 at /private/var/containers/Bundle/Application/30CD4A97-B604-4305-95AA-7BE43D72DBD4
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17002b000) -[MILaunchServicesOperationManager _onQueue_addPendingLaunchServicesOperation:error:]: Added pending LS operation <MILaunchServicesRegisterOperation: DF6E175B-4C9A-4BBF-BF8B-032F7362BE0F:4 com.example.flutterHelloWorld1xz782/MIInstallationDomainSystemShared personas:[PersonalPersonaPlaceholderString]>
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17002b000) -[MIInstaller performInstallationWithError:]: Install Successful for (Placeholder:com.example.flutterHelloWorld1xz782); Staging: 0.01s; Waiting: 0.00s; Preflight/Patch: 0.00s, Verifying: 0.03s; Overall: 0.25s
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17002b000) -[MIClientConnection updatePlaceholderMetadataForApp:installType:failureReason:underlyingError:failureSource:completion:]: Update placeholder metadata requested by client installcoordinationd (pid 117 (501/501)) for app com.example.flutterHelloWorld1xz782 installType = 1 failureReason = 0 underlyingError = (null) failureSource = 0
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17025b000) -[MIClientConnection _installURL:identity:targetingDomain:options:completion:]: Running installation as QOS_CLASS_USER_INITIATED
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17025b000) -[MIClientConnection _doInstallationForURL:identity:domain:options:completion:]: Install of "/private/var/containers/Shared/SystemGroup/systemgroup.com.apple.installcoordinationd/Library/InstallCoordination/PromiseStaging/D6D8B698-0238-42CE-80ED-641F1E513487/Runner.app" type Developer (LSInstallType = (null), Domain: MIInstallationDomainDefault) requested by installcoordinationd (pid 117 (501/501))
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17025b000) -[MIInstaller _installInstallable:containingSymlink:error:]: Installing <MIInstallableBundle ID=com.example.flutterHelloWorld1xz782; Version=1, ShortVersion=1.0.0>
Mon Aug 11 14:09:38 2025 [8567] <notice> (0x17025b000) MIUninstallProfilesForAppIdentifier: Uninstalling profiles for **********.com.example.flutterHelloWorld1xz782
Mon Aug 11 14:09:39 2025 [8567] <notice> (0x17002b000) -[MILaunchServicesOperationManager _onQueue_removePendingLaunchServicesOperationForUUID:dueToLSSave:]: Removing operation after LS save: <MILaunchServicesRegisterOperation: DF6E175B-4C9A-4BBF-BF8B-032F7362BE0F:4 com.example.flutterHelloWorld1xz782/MIInstallationDomainSystemShared personas:[PersonalPersonaPlaceholderString]>
Mon Aug 11 14:09:39 2025 [8567] <err> (0x17025b000) -[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:]: 179: This device has reached the maximum number of installed apps using a free developer profile: {(
    "**********.com.example.flutterHelloWorld1xz",    
    "**********.com.example.flutterHelloWorld1xz785",
    "**********.com.example.flutterHelloWorld1xz788",
    "**********.com.example.flutterHelloWorld1xz783",
    "**********.com.example.flutterHelloWorld1xz786",
    "**********.com.example.flutterHelloWorld1xz784",
    "**********.com.example.flutterHelloWorld1xz787"
)}
Mon Aug 11 14:09:39 2025 [8567] <err> (0x17025b000) -[MIInstaller _installInstallable:containingSymlink:error:]: Finalize stage failed
Mon Aug 11 14:09:39 2025 [8567] <notice> (0x17002b000) -[MIClientConnection _uninstallIdentities:withOptions:completion:]: Uninstall requested by installcoordinationd (pid 117 (501/501)) for identity [com.example.flutterHelloWorld1xz782/PersonalPersonaPlaceholderString] with options: {
    UninstallParallelPlaceholder = 1;
}
Mon Aug 11 14:09:39 2025 [8567] <notice> (0x17002b000) -[MIUninstaller performUninstallationByRevokingTemporaryReference:error:]: Taking termination assertion on com.example.flutterHelloWorld1xz782
Mon Aug 11 14:09:39 2025 [8567] <notice> (0x1702e7000) -[MIClientConnection updatePlaceholderMetadataForApp:installType:failureReason:underlyingError:failureSource:completion:]: Update placeholder metadata requested by client installcoordinationd (pid 117 (501/501)) for app com.example.flutterHelloWorld1xz782 installType = 7 failureReason = 14 underlyingError = Error Domain=MIInstallerErrorDomain Code=13 "This device has reached the maximum number of installed apps using a free developer profile: {(
    "**********.com.example.flutterHelloWorld1xz",
    "**********.com.example.flutterHelloWorld1xz785",
    "**********.com.example.flutterHelloWorld1xz788",
    "**********.com.example.flutterHelloWorld1xz783",
    "**********.com.example.flutterHelloWorld1xz786",
    "**********.com.example.flutterHelloWorld1xz784",
    "**********.com.example.flutterHelloWorld1xz787"
)}" UserInfo={LibMISErrorNumber=-402620383, LegacyErrorString=ApplicationVerificationFailed, SourceFileLine=179, FunctionName=-[MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:], NSLocalizedDescription=This device has reached the maximum number of installed apps using a free developer profile: {(
    "**********.com.example.flutterHelloWorld1xz",
    "**********.com.example.flutterHelloWorld1xz785",
    "**********.com.example.flutterHelloWorld1xz788",
    "**********.com.example.flutterHelloWorld1xz783",
    "**********.com.example.flutterHelloWorld1xz786",
    "**********.com.example.flutterHelloWorld1xz784",
    "**********.com.example.flutterHelloWorld1xz787"
)}} failureSource = 17
```

**What This Reveals**:

* The rejection comes from `MIFreeProfileValidatedAppTracker _onQueue_addReferenceForApplicationIdentifier:bundle:error:`.
* `Code=13` and `LegacyErrorString=ApplicationVerificationFailed` confirm the internal count exceeded the limit.
* The log enumerates **all currently installed bundle identifiers** under the free developer profile.
* `failureSource = 17` points to the specific internal check that failed.

Seeing these logs in action helps link the familiar XCode error to the precise internal enforcement mechanism—an invaluable clue for reverse engineering or troubleshooting on jailbroken test devices.


## Conclusion

Through this process of patching `installd` using LLDB or Frida, you've effectively sidestepped Apple’s 3-app limit, enabling the installation of additional apps for enhanced testing flexibility. This guide has equipped you with the knowledge to locate restriction logic, configure debugging environments, and implement targeted patches. These foundational skills pave the way for deeper iOS explorations, but it's vital to apply them ethically on test devices, respecting Apple’s guidelines and prioritizing device security. Enjoy your expanded development capabilities!

## Further Reading

- [The Missing Guide to Debug Third-Party Apps on iOS](https://felipejfc.medium.com/the-ultimate-guide-for-live-debugging-apps-on-jailbroken-ios-12-4c5b48adf2fb)
- [Frida Documentation](https://frida.re/docs/home/)
- [ipsw Tool](https://github.com/blacktop/ipsw)
- [AppSync Unified](https://cydia.akemi.ai/?page/net.angelxwind.appsyncunified)
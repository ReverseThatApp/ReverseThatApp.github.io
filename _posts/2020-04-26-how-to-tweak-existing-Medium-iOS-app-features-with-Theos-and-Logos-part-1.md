---
layout: post
title: How to tweak existing Medium iOS app features with Theos & Logos - part 1
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPFEkRa2MipjryjGgs8ixdAKm7ulXmxkcAydCyBJ751trrCcZs7Oh8XOVipsDeCUlf2UAGE0ZDjATybZEJN9Bvo81F5cpg7oMSFbEnZq_AWWYKAUDbfVyzXG4ZPuD69PJdm_bnG5wPA-voVdBBdLkMR=w2048-h1536-s-no-gm?authuser=2
    alt: How to create your own Tweak
tags: [ios, tweak]
---

Today we will reverse engineering **Medium** iOS app and modify existing claps feature. We will also learn about how to create your own Cydia tweak using `theos` platform, hook into a method, and change its behavior as we want to. Let's do it!!!!
![How to create Tweak](https://lh3.googleusercontent.com/pw/AP1GczPFEkRa2MipjryjGgs8ixdAKm7ulXmxkcAydCyBJ751trrCcZs7Oh8XOVipsDeCUlf2UAGE0ZDjATybZEJN9Bvo81F5cpg7oMSFbEnZq_AWWYKAUDbfVyzXG4ZPuD69PJdm_bnG5wPA-voVdBBdLkMR=w2048-h1536-s-no-gm?authuser=2)
_**Figure: How to create your own Tweak**_

## Disclaimer
This post is for educational purposes only. How you use this information is your responsibility. I will not be held accountable for any illegal activities, so please use it at your discretion and contact the app's author if you find issues. We will inspect the [Medium app](https://apps.apple.com/us/app/medium/id828256236){:target="_blank"} installed from AppStore.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [FLEX Loader Cydia package](http://cydia.saurik.com/package/com.joeyio.flexloader/){:target="_blank"}
- [Theos development setup](https://github.com/theos/theos/wiki){:target="_blank"}
- [Install Medium iOS app](https://apps.apple.com/us/app/medium/id828256236){:target="_blank"}


## Overview
Medium is an online publishing platform that allows users to read and publish stories/posts with ease. The topics are varied and I'm sure you would hear about this once. It supports multiple platforms such as Web, Android, iOS. As an iOS user, I found it useful for me to read and share topics that I'm interested in to my friends over the world. They developed the Claps feature to allow readers to show their support for a Medium post. If you find a post interesting, just click the clap 👏 button on the same post page and it will notify the author that you like it. A total of the claps of all readers will be displayed on that post. 

But do you know that you can clap for a post more than one time? Recently I've just discovered that Medium allows to clap 50 times per post? You might wonder why they do that? From their website it's the way you can show the author how much you liked that post. It makes sense if you want to support your "idol" above others, looks interesting. You can tap on the 👏 button one by one to increase your claps or tap and hold the 👏 button until you reach the number you want (of course limit to maximum 50 👏 per post).

Medium implemented very nice animation to attract readers to clap for an author, which includes me. The only thing that if you are a crazy fan and want to support your "idol" 30 or 40 or maximum 50 claps for any posts, clap one by one seems to be a waste of time. What if you can just one clap to make it 35 or 50 claps? Yeah, it might be a good idea (or bad one, you named it 😢) but what you can do if you don't have the source code to modify the app. Worry not, if you're using a jailbroken device, you can tweak any app you like to enhance it and make it convenient for you, write a Tweak for Medium app, which we will walk through next sections. For ones that don't have a jailbroken device, you can use the same idea to patch the app and repack if you really love this feature, but I can say it's not straight forward like the way we use Tweak, so do it at your own risk 😊
![How to create Tweak](https://images.unsplash.com/photo-1566796215362-7f6bd862ea82?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2800&q=80)
_**Claps** (source: unsplash by [@Daniel Lincoln](https://unsplash.com/photos/iK8t56RS4g8){:target="_blank"})_

## Theos development setup
### What is Theos
*Theos is a cross-platform suite of tools for building and deploying software for iOS and other platforms. It is a flexible Make-based build system primarily for jailbreak software development, but also with complete support for building for other supported platforms. Theos runs on, and can build projects for, macOS, iOS, Linux, and Windows (under Cygwin or Windows Subsystem for Linux) - [Wiki](https://github.com/theos/theos/wiki){:target="_blank"}* 

### Installation
You can refer [iphonedevwiki.net page](https://iphonedevwiki.net/index.php/Theos/Setup){:target="_blank"} for how to install Theos for local development. The setup would be straight forward, my case is to install Theos on MacOS, develop Tweak then install it into the jailbroken device.

### Testing setup
To check if your Theos setup successfully, let run this command in your Terminal: `$THEOS/bin/nic.pl`. If you can see Theos New Instance Creator template spits out, then it's time to move on developing your first Tweak.

## Medium Tweak development
### Create Tweak project
From Terminal, run: `$THEOS/bin/nic.pl` then select `[10.] iphone/tweak`
```bash
MBP# $THEOS/bin/nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/activator_event
  [2.] iphone/application_modern
  [3.] iphone/application_swift
  [4.] iphone/flipswitch_switch
  [5.] iphone/framework
  [6.] iphone/library
  [7.] iphone/preference_bundle_modern
  [8.] iphone/tool
  [9.] iphone/tool_swift
  [10.] iphone/tweak
  [11.] iphone/xpc_service
Choose a Template (required): 10
Project Name (required): medium-enhance-claps-tweak
Package Name [com.yourcompany.medium-enhance-claps-tweak]: com.myfirsttweak
Author/Maintainer Name [ReverseThatApp]: ReverseThatApp
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.medium.reader
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: SpringBoard
Instantiating iphone/tweak in mediumenhanceclapstweak/...
Done.
```

This will require you to key some info, let go one by one:
```bashscript
- Project Name(required): name of your tweak
- Package Name: your tweak bundle id (optional)
- Author/Maintainer Name [ReverseThatApp]: Optional
- [iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: This is bundle id of the process you want to hook, for Medium app bundle id is `com.medium.reader`
- [iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: Apps you want to terminate upon tweak installation finish, for Medium it's `hangtag`.
```
After Tweak creation process is done, a new folder `mediumenhanceclapstweak` will be created with some files generated and ready for you to develop:
```bashscript
- control: Tweak information
- Makefile: for automated compilation
- mediumenhanceclapstweak.plist: Define which bundle id that this Tweak can hook
- Tweak.xm: Your tweak functionality will be implemented here
```

### Which classes and methods to hook?
Yah we talked many things about hooking and changing the way of existing clapping function, but where are exactly classes and methods to hook into? This is the common question when you start to reverse engineering an app, so let me walk through how to find it. 

There are many ways to reverse engineering an app such as static analysis or dynamic analysis... and each approach will have many tools and techniques that support the reverse process. In this post, we will dynamic analysis Medium app to find out the entry point of hooking, and the tool that is very powerful to use is `FLEXLoader`

### FLEXLoader 
*[FLEXLoader](http://cydia.saurik.com/package/com.joeyio.flexloader/){:target="_blank"} is a iOS developer tool, it dynamically loads FLEX into iOS apps on jailbroken devices. [FLEX (Flipboard Explorer)](https://github.com/Flipboard/FLEX){:target="_blank"} is a set of in-app debugging and exploration tools for iOS development. When presented, FLEX shows a toolbar that lives in a window above your application. From this toolbar, you can view and modify nearly every piece of state in your running application. (source: Cydia)*
![FLEX](https://user-images.githubusercontent.com/8371943/70185687-e842c800-16af-11ea-8ef9-9e071380a462.gif)
_**Figure 1: FLEX (Flipboard Explorer)** *(source: FLEX)*_

After installing **FLEXLoader** from Cydia, you can configure which app you want to load FLEX via `FLEXLoader` menu in `Settings.app`. Let switch on **Medium** item in `FLEXLoader` menu to load FLEX whenever you launch **Medium** app.

![FLEXLoader]({{ site.baseurl }}/images/20200426/FlexLoader-hook-medium.jpeg)
_**Figure 2: Configure loading FLEXLoader into Medium app**_

### Dynamic analysis
Now launch the **Medium** app, and tap on any posts that you can clap, you will see that **FLEX** is loaded and shown as a toolbar on top of the app.
![FLEX is loaded]({{ site.baseurl }}/images/20200426/FLEX-loaded-in-medium.jpeg)
_**Figure 3: FLEX is loaded in Medium app**_

You can move FLEX toolbar position around the screen or hide that toolbar if you feel annoy, for the next steps let move FLEX toolbar out of **Medium** footer action bar as we need to further inspect this area.

We know that when tapping on the Clap icon at the bottom-left screen it will perform an action from an unknown class which we need to find out, so let trace from the Clap icon.

From the FLEX toolbar, let toggle item `select` to activate select mode. This mode allows us to inspect any views we tap on the screen. We will know which kind of Clap icon soon.
![Inspect clap icon]({{ site.baseurl }}/images/20200426/inspect-clap-icon.jpeg)
_**Figure 4: Inspect Clap icon**_

As you can guess and see, it's an `UIImageView`, so how to trace which class contains this Clap image view? It's easy, though. From FLEX toolbar, tap on `views` item, it will open new screen showing view hierarchy of this screen which selected `UIImageView` highlighted as below:
![Inspect clap icon]({{ site.baseurl }}/images/20200426/clap-views-hierarchy.jpeg)
_**Figure 5: View hierarchy**_

From here you can see that the Clap image view belongs to `ClapButton` class which belongs to `StickyFullPostFooterActionBar` class. Let tap on ℹ️ icon of `ClapButton` row to inspect `ClapButton` class, it's an instance of `ClapButton` in memory, it will navigate to a new screen that lists all of the properties, ivars, methods, base classes of `ClapButton` class. Are you 😲 now? 

Not even list down everything belongs to this class, it also allows us to invoke methods, changing the value of properties and ivars with ease. I was speechless when the first time used this tool, I think you are sharing the same feeling.

Let filter the keyword we are looking for, type `clap` into **Filter search bar**, luckily you can see some matching results:

![Clap button info]({{ site.baseurl }}/images/20200426/clapbutton-filter-clap.jpeg)
_**Figure 6: ClapButton info**_

If you're an iOS developer, I'm sure you know how this clap working behind the scene base on filtered out results. If not, let's guess what's this class doing.

First, we see there is a `<ClapDelegate> *clapDelegate` which is conformed by `StickyFullPostFooterActionBar` class. As the delegate name and above view hierarchy from **Figure 5**, we can guess that when we tap on Clap icon, the `ClapButton` will listen to tap event and use `*clapDelegate` to ask `StickyFullPostFooterActionBar` class handle on behalf, or some sorts like that.

If this guess still does not convince you yet, look into **METHOD** section of FLEX, you can see that there are 2 methods I hope it can clear your mind now: `-(void)clapButtonPressed:(id)` and `-(void)clapButtonReleased:(id)`. The developers named the method so well, thanks to that we can guess what are they doing from the first look. If you remember we mentioned above that **Medium** allows us to single clap or press and hold to clap multiple times, does it ring the bell? 

Let do some testings by invoking these methods to make sure if it's working first. FLEX allows us to invoke the method of an instance from UI, so convenient!! It's very helpful and saves a lot of effort when reversing any apps. Let tap on `-(void)clapButtonPressed:(id)` method, it will navigate to a new screen with `Call` button on the top-right screen that allows you to invoke this method, just tap on `Call` button and see what happen. We assume that it will show the clap animation on UI like we manually tap the Clap button of a post. Let see what happens:

![Invoke method clapButtonPressed](https://thumbs.gfycat.com/NearOpulentGossamerwingedbutterfly-size_restricted.gif)
_**Figure 7: Invoke method clapButtonPressed:**_

HURRAYYYY!!!!! The clap function is working as expected and very nice animation (confetti??), anyway I 😍 this animation. The claps keep counting until `+50` but never stop, because we are calling `clapButtonPressed` method which is simulating press and hold the Clap button. So to make it stop, we need to invoke the second method `-(void)clapButtonReleased:(id)` same way as we invoked `-(void)clapButtonPressed:(id)`, and no need to say it STOPPED the clapping!!!

So we just identified which class (`ClapButton`) and methods (`-(void)clapButtonPressed:(id), -(void)clapButtonReleased:(id)`) to hook, it's time to hands-on and learning how to write a Cydia Tweak using Theos and Logos component.

### Write Medium Tweak
We will use Logos's syntax to develop this tweak. *Logos is a component of the Theos development suite that allows method hooking code to be written easily and clearly, using a set of special preprocessor directives. (source: [http://iphonedevwiki.net/](http://iphonedevwiki.net/index.php/Logos){:target="_blank"})*

Open `Tweak.xm` file and be ready to write our tweak. To hook into a class, we need to use directive `%hook Classname`, in this case, it will be like this:
```objectivec
%hook ClapButton

%end
```

Next we need to hook into method `-(void)clapButtonPressed:(id)`, just put it inside `%hook ClapButton` block:
```objectivec
%hook ClapButton

-(void)clapButtonPressed:(id)arg2 {
	%log; // log method name and argument, you will find this log in `Console` app
	%orig; // invoke original method
}

%end
```

What is the above hooking method doing? It just added a log statement at the beginning to log method name and arguments (`%log`) to the `Console` app then invoke the original method (`%orig;`). Is that very simple?

Before doing anything else, let try to install this Tweak and verify if our hooking method working or not. Let open `Makefile` file and make sure the content will look like this:
```objectivec
# Replace with your jailbroken phone IP address
export THEOS_DEVICE_IP=192.168.1.128 THEOS_DEVICE_PORT=22

# which architectures it supports, remove ARCHS if you not sure about this
ARCHS = arm64

# This is the shortcut to kill the process after installing, this case it will restart SpringBoard
# If you know Medium process, put it here (hint: hangtag)
INSTALL_TARGET_PROCESSES = SpringBoard

# $(THEOS) is environment variable, it's the path of your downloaded theos, my case it's ~/theos
include $(THEOS)/makefiles/common.mk

TWEAK_NAME = mediumenhanceclapstweak

mediumenhanceclapstweak_FILES = Tweak.xm
mediumenhanceclapstweak_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/tweak.mk
```

From Terminal, change directory to the tweak root folder and run this command: `make package install` (if you are developing tweak often, you can create an alias for this command as a shortcut, I used to make it `alias mpi='make package install'`)
```bash
 MBP# make package install
> Making all for tweak mediumenhanceclapstweak…
==> Preprocessing Tweak.xm…
==> Compiling Tweak.xm (arm64)…
==> Linking tweak mediumenhanceclapstweak (arm64)…
==> Generating debug symbols for mediumenhanceclapstweak…
==> Merging tweak mediumenhanceclapstweak…
==> Signing mediumenhanceclapstweak…
> Making stage for tweak mediumenhanceclapstweak…
dm.pl: building package `com.yourcompany.medium-enhance-claps-tweak:iphoneos-arm' in `./packages/com.yourcompany.medium-enhance-claps-tweak_0.0.1-1+debug_iphoneos-arm.deb'
==> Installing…
Selecting previously unselected package com.yourcompany.medium-enhance-claps-tweak.
(Reading database ... 7828 files and directories currently installed.)
Preparing to unpack /tmp/_theos_install.deb ...
Unpacking com.yourcompany.medium-enhance-claps-tweak (0.0.1-1+debug) ...
Setting up com.yourcompany.medium-enhance-claps-tweak (0.0.1-1+debug) ...
```

If you can see above Terminal logs, it means your tweak is complied and installed successfully on the device. To double confirm, open `Cydia` app, tap on `Installed` tab and select `Recent` segment, you will see your tweak `medium-enhance-claps-tweak` installed. The first step achieved, let check if our hooking method is called when clapping a post.

Plug your jailbroken device into laptop using cable and open Console app (assume you're using Macbook), select your device on the left panel and key in "ClapButton" in search text field then clap a post, you will see new log record appears, like this:
```
[1;36m[medium-enhance-claps-tweak] [m[0;36mTweak.xm:79[m [0;30;46mDEBUG:[m -[<ClapButton: 0x102857de0> clapButtonPressed:<ClapButton: 0x102857de0; baseClass = UIButton; frame = (52 -3; 26 50); opaque = NO; tintColor = UIExtendedSRGBColorSpace 0 0 0 0.64; layer = <CALayer: 0x1c042de80>>]
```

Here you go, the hooking function is working with an extra log. We're now confident to tweak this method. Let say when user tap on Clap button, we want default +4 Claps instead of +1, we can do like this:
```objectivec
/* You can move this interface declaration to a header file, i.e Tweak.h
 * and include it in Tweak.xm using #include "Tweak.h"
 */
@interface ClapButton : NSObject
-(void)clapButtonReleased:(id)arg2; 
@end

%hook ClapButton

int numberOfClaps = 4;

-(void)clapButtonPressed:(id)arg2 {
  %log; // log method name and argument, you will find this log in `Console` app
  for (int i = 0; i < numberOfClaps; i++) {      
      %orig; // invoke original method
      [self clapButtonReleased:arg2]; // simulate release click     
  }  
}

-(void)clapButtonReleased:(id)arg2 {
  %log; // log method name and argument, you will find this log in `Console` app  
  %orig; // invoke original method
}

%end
```

You're might aware why we need to declare `@interface ClapButton : NSObject` with method `-(void)clapButtonReleased:(id)arg2;`. The reason is that we're calling `[self clapButtonReleased:arg2];` in method `clapButtonPressed`. If not declare `clapButtonReleased` in that interface, you will get this kind of error when compiling the tweak: 
```bash
Tweak.xm:89:7: error: receiver type 'ClapButton' for instance message is a forward declaration
[self clapButtonReleased:arg2];
```

Now let compile and install it again. IT’S JUST WORKED 😂😂😂!!! But we manage to call `clapButtonReleased` manually looks like not a good way, sometimes calling a method without knowing what it’s doing is dangerous and might cause side effects. Are you curious about how they handle single tap and press action? We will reverse engineering these methods in the next post to see what did they do and find other methods to hook instead.

I see the post is too long so we will cover other things in my next post. There is a problem, however, if we want to change the number of claps to other numbers, modify the code and install again is quite cumbersome, we need to find a way to allow users to configure the number as they want to without modifying the source code and we will talk about how to deploy and share our tweak to others, all will be covered in the next post, so stay tuned!!! 😍😍😍


## Final thoughts
- This method hooking is a way to analyze the app at runtime and understand how the app working, however, it only can hook into Objective-C method, not for Swift.
- This method hooking not only hooks into 3rd apps but also can hook into system apps and processes, like SpringBoard... So if you want to dive deeper, you can write tweak to change your own jailbroken look and feel, it's possible.
- With great power comes great responsibility, so do at your own risk :)

![](https://images.unsplash.com/photo-1521714161819-15534968fc5f?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2850&q=80)
_(source: unsplash by [@Road Trip with Raj](https://unsplash.com/photos/o4c2zoVhjSw){:target="_blank"})_


## Further readings
- [Theos](http://iphonedevwiki.net/index.php/Theos){:target="_blank"}
- [Logos syntax](http://iphonedevwiki.net/index.php/Logos){:target="_blank"}
- [Compile issue](https://github.com/theos/theos/issues/324){:target="_blank"}
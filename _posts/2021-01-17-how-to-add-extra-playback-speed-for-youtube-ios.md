---
layout: post
title: How to add extra playback speed for YouTube iOS?
---

In this post, we will reverse the Youtube app to know how to add custom playback speed and understand why it does not work straight forward. 
[![Playback Speed 3x]({{ site.baseurl }}/images/20210117/youtube-playback-speed-3x.jpeg)]({{ site.baseurl }}/images/20210117/youtube-playback-speed-3x.jpeg){:target="_blank"} <br/>**Figure: Playback Speed 3x** <br/><br/>

## Disclaimer
This post is for educational purposes only. How you use this information is your responsibility. I will not be held accountable for any illegal activities, so please use it at your discretion and contact the app's author if you find issues.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [FLEX Loader Cydia package](http://cydia.saurik.com/package/com.joeyio.flexloader/){:target="_blank"}
- [Hopper Disassembler](https://www.hopperapp.com/download.html){:target="_blank"}
- A bit knowledge of assembly arm64, please read my previous [post]({{ site.baseurl }}/by-pass-ssl-pinning-iOS-with-lldb/){:target="_blank"}

## Overview
Last year there was a Reddit user posted a new topic seeking a Youtube tweak that allows him to add custom playback speeds rather than default ones. He was asking to add extra speeds such as 2.5x, 3x, 3.5x... Never knew why he wanted that fast speeds, but it triggered my curiosity to have a look and surprised me that there were some client-side validations that restricted speeds that out of predefined boundary. Today's post will go through how to add custom playback speeds and trace where is hidden speed client-side validation for these ones.

## Analysis
### Get .ipa file
With the help of [Frida iOS Dump](https://github.com/AloneMonkey/frida-ios-dump){:target="_blank"} or [CrackerXI](https://forum.iphonecake.com/index.php?/topic/363020-crackerxi-gui-app-decryption-tool-for-ios-11-12-13/){:target="_blank"}, we can easily pull out **.ipa** file of Youtube app on a jailbroken device, unzip **.ipa** and navigate to `Payload/YouTube.app` folder. All of the app resources and binary files are inside this folder.

It looks like only 3 executable files you need to check: `YouTube` (14.9MB), and the other 2 in **Frameworks** folders: `Module_Framework` (105.5MB), `widevine_cdm_secured_ios` (3.6MB). By the size of each executable one, `Module_Framework` is the very big one compared to the others, so it means most of the logic might be in this framework. Let take note of this.

### Where playback speeds are setup?
It's a bit tricky to figure out where speed is used, but if you select the **Playback speed** option from playing video, you will see there are **8** predefined ones.
```bashscript
    0.25x
    0.5x
    0.75x
✔️  1x 
    1.25x
    1.5x
    1.75x
    2x
```

By searching these values, we hope it will be appeared in the binary and trace from there. Running this `strings` command for `Module_Framework` file from terminal to search for value `0.25x`, we can see there is one result as below:
```bashscript
$ strings Frameworks/Module_Framework.framework/Module_Framework | grep "0\.25x"
varispeed.0.25x
```

Let load the `Module_Framework` file into **Hopper Disassembler** to find down where it was used. By searching `varispeed.0.25x` in the `Str` (String) tab, you will see it's used in `-[YTVarispeedSwitchController init]`.
```bashscript
cfstring_varispeed_0_25x:
000000000484a750         dq         0x00000000065a4360, 0x00000000000007c8, 0x0000000003d0f9bc, 0x000000000000000f ; "varispeed.0.25x", DATA XREF=-[YTVarispeedSwitchController init]+104
```

Double click on that method, it will navigate to the method implementation:
```bashscript
-[YTVarispeedSwitchController init]:
...
add        x0, x0, #0x750 ; 0x484a750@PAGEOFF, @"varispeed.0.25x"
bl         _GetPlayerString ; _GetPlayerString
mov        x29, x29
bl         imp___stubs__objc_retainAutoreleasedReturnValue ; objc_retainAutoreleasedReturnValue
mov        x2, x0
str        x0, [sp, #0xf0 + var_B8]
adrp       x8, #0x5716000 ; &@selector(playbackRelativeSecondsPrefetchCondition)
ldr        x26, [x8, #0xfe8] ; @selector(initWithTitle:rate:)
fmov       s0, #0x3fd0000000000000 ; #0.25 (float)
mov        x0, x20 ; super.init()
mov        x1, x26 ; @selector(initWithTitle:rate:)
bl         imp___stubs__objc_msgSend ; objc_msgSend
str        x0, [sp, #0xf0 + var_C0]
...
ldr        x0, [x8, #0xad0] ; objc_cls_ref_NSArray,_OBJC_CLASS_$_NSArray
adrp       x8, #0x56bf000
ldr        x1, [x8, #0x9f0] ; "arrayWithObjects:count:",@selector(arrayWithObjects:count:)
add        x2, sp, #0x50 ; options
movz       w3, #0x8     ; 8 elements
bl         imp___stubs__objc_msgSend ; objc_msgSend
mov        x29, x29
bl         imp___stubs__objc_retainAutoreleasedReturnValue ; objc_retainAutoreleasedReturnValue
ldr        x8, [x19, #0x8]
str        x0, [x19, #0x8] ; objc_ivar_offset_YTVarispeedSwitchController__options
```

This is the constructor of `YTVarispeedSwitchController` class. It is trying to get the real speed text to display using `_GetPlayerString` method and construct the option for each rate using `@selector(initWithTitle:rate:)` selector. Each option will be eventually added into an array with the size of **8** elements, and then stored into `_ivar YTVarispeedSwitchController._options`
```bashscript
0000000004f40bc8         struct __objc_ivars { ; DATA XREF=__objc_class_YTVarispeedSwitchController_data
                             32,                                  // entsize
                             3                                    // count
                         }
0000000004f40bd0         struct __objc_ivar { ; "_options","@\\\"NSArray\\\""
                             objc_ivar_offset_YTVarispeedSwitchController__options, // offset pointer
                             aOptions,                            // name
                             aNsarray_40ae529,                    // type
                             0x3,
                             0x8                                  // size
                         }
```

So this is the method we can add our extra speeds. Let write a tweak to add some more speeds here.

### Add extra playback speeds
By using [Theos](https://github.com/theos/theos/wiki){:target="_blank"} and hook into `YTVarispeedSwitchController.init` method, we can add some extra speeds such as 0.1, 2.25, 2.5, 3.0, 3.5, 4.0 and store into `_options ivar` (refer my previous posts for how to write a tweak using Theos).

```bashscript
%hook YTVarispeedSwitchController

-(void *)init {
    %log;
    NSLog(@"====");
    void *ori = (void *)%orig;

    float speeds[] = {	
        0.1,  /* new speed */		
        0.25, 
        0.5, 
        0.75, 
        1.0, 
        1.25, 
        1.5, 
        1.75, 
        2.0, 
        2.25, /* new speed */
        2.5,  /* new speed */			
        3.0,  /* new speed */
        3.5,  /* new speed */
        4.0   /* new speed */
    };

    NSMutableArray *options = [NSMutableArray new];
    for (NSUInteger i = 0, count = sizeof(speeds)/sizeof(speeds[0]); i < count; i++) {
        NSString *optionTitle = [NSString stringWithFormat:@"%gx", speeds[i]];
        [options addObject:[[NSClassFromString(@"YTVarispeedSwitchControllerOption") alloc] initWithTitle:optionTitle rate:speeds[i]]];
    }
    MSHookIvar<NSArray *>(self, "_options") = [options copy];

    return ori;
}

%end
```

Build and install this tweak into the jailbroken phone and relaunch the YouTube app, select the playback speed option you will see the extra speeds reflected!!
[![Extra playback speeds]({{ site.baseurl }}/images/20210117/youtube-playback-speed-extra-options.jpeg)]({{ site.baseurl }}/images/20210117/youtube-playback-speed-extra-options.jpeg){:target="_blank"} <br/>**Figure 1: Extra playback speeds** <br/><br/>

However, tapping on any extra speeds does not make any difference. Any selected options greater than 2x will be reset selected as 2x speed and any selected options less than 0.25x will be reset and selected as 0.25x. There might be some speed checking somewhere waiting for us to reveal :P

### Show me the way to Playback speed boundary check
We need to figure out the proper way to trace the flow instead of looking for every method in the huge binary file. When we select a speed option, it should be internally called the setter methods, so start searching with `setPlayback*` in **Hopper Disassembler** we can see that there are multiple selector matching `setPlaybackRate:` with the input is `float` type. These methods are the ones we can hook or put a breakpoint to debug to understand the flow. However, we can do a quick check by hooking into those methods and log the input value to see when the playback speed is reset, like below:
```bashscript
%hook YTLocalPlaybackController
-(void)setPlaybackRate:(float)arg2 {	
	%log;
	%orig(arg2);
}
%end

%hook YTSingleVideoController
-(void)setPlaybackRate:(float)arg2 {	
	%log;
	%orig(arg2);
}

-(void)playerRateDidChange:(float)arg2 {	
	%log;
	%orig;
}
%end
...
```

Install the tweak again and restart YouTube app, open Mac `Console` app to filtering the YouTube logs, once select new speed option `3.5x` we can see the logs as below:
[![2: Playback speed trace]({{ site.baseurl }}/images/20210117/theos-set-playback-speed-trace.png)]({{ site.baseurl }}/images/20210117/theos-set-playback-speed-trace.png){:target="_blank"} <br/>**Figure 2: Playback speed trace** <br/><br/>

As you can see all speed rate values are logged as `3.500000` which is the one we selected, however, it is reset to `2.000000` after `-[<YTSingleVideoController: 0x1c43c2fd0> setPlaybackRate:3.500000]` and before `-[<YTSingleVideoController: 0x1c43c2fd0> playerRateDidChange:2.000000]` - the highlighted ones. It's time to stop dynamic analysis for a while and get our brain dirty to see what's going on inside `-[YTSingleVideoController setPlaybackRate:]` method.

```bashscript
/* @class YTSingleVideoController */
-(void)setPlaybackRate:(float)arg2 {
    r2 = arg2;
    r0 = self;
    var_20 = d9;
    stack[-40] = d8;
    r31 = r31 + 0xffffffffffffffd0;
    var_10 = r20;
    stack[-24] = r19;
    saved_fp = r29;
    stack[-8] = r30;
    if (*(r0 + 0xa0) == 0x2) { // objc_ivar_offset_YTSingleVideoController__lifecycleState:
            v8 = v0;
            r19 = r0;
            if ([r0 supportsChangingPlaybackRate] != 0x0) {
                    [*(r19 + 0x8) setRate:r2]; // objc_ivar_offset_YTSingleVideoController__player
            }
    }
    return;
}
```

`*(r0 + 0xa0)` is accessing to `ivar` of `YTSingleVideoController` with an offset `0xa0`:
```bashscript
    objc_ivar_offset_YTSingleVideoController__lifecycleState:
000000000578de10         db  0xa0 ; '.'  ; DATA XREF=0x4f150e8
000000000578de11         db  0x00 ; '.'
000000000578de12         db  0x00 ; '.'
000000000578de13         db  0x00 ; '.'
```

The idea of this method is to check if `_lifecycleState` equals to `2` and `self.supportsChangingPlaybackRate` is `true` then it will invoke `self._player.setRate(3.5)`. This method does not change the value of speed, so the place could change it might be inside the `setRate:` method.

`*(r19 + 0x8)` is `MLPlayer` type:
```bashscript
    objc_ivar_offset_YTSingleVideoController__player:
000000000578ddbc         db  0x08 ; '.'  ; DATA XREF=0x4f14e48
000000000578ddbd         db  0x00 ; '.'
000000000578ddbe         db  0x00 ; '.'
000000000578ddbf         db  0x00 ; '.'
```
```bashscript
0000000004f14e48         struct __objc_ivar { ; "_player","@\\\"<MLPlayer>\\\""
                             objc_ivar_offset_YTSingleVideoController__player, // offset pointer
                             aPlayer,                             // name
                             aMlplayer_4128e6f,                   // type
                             0x3,                                 // alignment (// minimum memory size to store a variable (in bytes and a multiple of 8))
                             0x8                                  // size
                         }
```

Double click on the `setRate:` selector, many classes are implementing this, such as `MLHAMPlayer, MLHAMQueuePlayer, MLAVPlayer, MLAVAssetPlayer...`. To quickly identify which is the one, in this case, let use `LLDB` to attach into running YouTube process set a breakpoint to this `setRate:` method call.

Toggle back to Assembly mode, we need to set the breakpoint at address `0x100a768 + ASLR` of `Module_Framework` binary.
```bashscript
    -[YTSingleVideoController setPlaybackRate:]:
...
0100a74c	ldr		x0, [x19, #0x8]
0100a750	adrp	x8, #0x5714000      ; &@selector(updateScreenSizeToSize:)
0100a754	ldr		x1, [x8, #0x198]    ; "setRate:",@selector(setRate:)
0100a758	mov		v0, v8
0100a75c	ldp		x29, x30, [sp, #0x20]
0100a760	ldp		x20, x19, [sp, #0x10]
0100a764	ldp		d9, d8, [sp], #0x30
0100a768	b		imp___stubs__objc_msgSend 
...
```
Launch `LLDB`, attach and set a breakpoint, in this case, `ASLR` value of `Module_Framework` is `0x0000000101cc8000`:
```bashscript
Target 0: (YouTube) stopped.
(lldb) image list -o -f Module_Framework
[  0] 0x0000000101cc8000 /private/var/containers/Bundle/Application/33FF9AB6-2F88-4306-86FD-642DFD943773/YouTube.app/Frameworks/Module_Framework.framework/Module_Framework(0x0000000101cc8000)
(lldb) br s -a 0x100a768+0x0000000101cc8000
Breakpoint 1: where = Module_Framework`___lldb_unnamed_symbol89579$$Module_Framework + 80, address = 0x0000000102cd2768
(lldb) c
Process 5781 resuming
(lldb)
```

We've just managed to successfully set a new breakpoint at address `0x102cd2768` (at the instruction to execute the `setRate:` method). Now it's time to trigger set playback speed again with value `3.5x` from playing YouTube video.
```bashscript
Process 5781 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000102cd2768 Module_Framework` ___lldb_unnamed_symbol89579$$Module_Framework  + 80
Module_Framework`___lldb_unnamed_symbol89579$$Module_Framework:
->  0x102cd2768 <+80>: b      0x10551a2ac               ; symbol stub for: objc_msgSend
    0x102cd276c <+84>: ldp    x29, x30, [sp, #0x20]
    0x102cd2770 <+88>: ldp    x20, x19, [sp, #0x10]
    0x102cd2774 <+92>: ldp    d9, d8, [sp], #0x30
    0x102cd2778 <+96>: ret
```

It stopped at our breakpoint, which's a good sign that we're on the right track. To know which instance trigger `setRate:`, we only need to examine register `x0` using the `po` command:
```bashscript
(lldb) po $x0
<MLHAMQueuePlayer: 0x10f0904b0>
(lldb)
```

It's clear now that `-[MLHAMQueuePlayer setRate:]` is the method we need to look at in this case, thanks to `LLDB` and YouTube (for no anti-debugging mechanism).

### Playback speed boundary check in the `-[MLHAMQueuePlayer setRate:]`
```bashscript
-[MLHAMQueuePlayer setRate:]:
stp        d9, d8, [sp, #-0x40]! ; DATA XREF=0x4f67720
stp        x22, x21, [sp, #0x10]
stp        x20, x19, [sp, #0x20]
stp        x29, x30, [sp, #0x30]
add        x29, sp, #0x30
fmov       s1, #0x4000000000000000 ; #2.0 in float
fminnm     s0, s0, s1
fmov       s1, #0x3fd0000000000000 ; #0.25 in float
fmaxnm     s8, s0, s1
ldr        s0, [x0, #0x16c] ; objc_ivar_offset_MLHAMQueuePlayer__rate
fcmp       s0, s8
b.ne       loc_10a4930
```

Do you see `fminnm` and `fmaxnm` instructions? These are Floating-point Minimum/Maximum Number (scalar) instructions that compare the first and second source register values and write the smaller/larger of the two floating-point values to the destination register, in our case `s0` register is holding the value of speed rate `3.500000`.
```bashscript
fmov       s1, #0x4000000000000000 ; #2.0 in float
fminnm     s0, s0, s1 ; s0 = min (3.5, 2.0) = 2.0
fmov       s1, #0x3fd0000000000000 ; #0.25 in float
fmaxnm     s8, s0, s1 ; s8 = max (2.0, 0.25) = 2.0
ldr        s0, [x0, #0x16c] ; objc_ivar_offset_MLHAMQueuePlayer__rate
fcmp       s0, s8 ; self._rate == 2.0
```

With the above inline comments, you can see that custom speed needs to be in the range of `[0.25, 2.0]`, otherwise it will be reset to the nearest boundary. We are passing the value of speed rate is `3.5` into this `setRate:` method, with these `fminnm` and `fmaxnm` instructions it resets the value to become `2.0` which is reflecting on the YouTube app, so here is the client-side speed validation we are looking for. If you're not convinced yet, let look a bit further instructions inside this method to see what it's doing after this validation:

```bashscript
/* @class MLHAMQueuePlayer */
-(void)setRate:(float)arg2 {
    r0 = self;    
    s1 = #0x4000000000000000; // 2.0
    asm { fminnm     s0, s0, s1 };
    s1 = #0x3fd0000000000000; // 0.25
    asm { fmaxnm     s8, s0, s1 };
    if (*(int32_t *)(r0 + 0x16c) != s8) { // self._rate != max(min(3.5, 2.0), 0.25)
        r19 = r0;            
        r0 = [r0 playerItem];            
        r20 = r0;
        *(int32_t *)(r19 + 0x120) = s8; // self._preferredRate = s8 (keep old value)
        r0 = [r0 config];            
        r22 = [r0 varispeedAllowed];            
        if (r22 != 0x0) { // if true
            if (s8 > 0x3ff0000000000000) { // s8 > 0.25
                if ([r20 peggedToLive] != 0x0) { // if playerItem.peggedToLive is true
                    asm { fcsel      s8, s9, s8, ne };
                }
            }
            *(int32_t *)(r19 + 0x16c) = s8; // self._rate = s8
            [r19 internalSetRate];
        }
    }
    return;
}
```

The whole idea of this method is it will check the current speed rate `self._rate` with new value `s8` (after boundary check), if it's the same then it will do nothing, otherwise, it will check the configuration if this kind of video is supported multiple speed (`varispeedAllowed`) then will update current speed rate `self._rate` to the new value and invoke `internalSetRate` (you can't change the playback speed for live stream video, that would make more sense for this checking condition). Wonder what does `internalSetRate` method do? Here is your answer:
```bashscript
/* @class MLHAMQueuePlayer */
-(void)internalSetRate {
    dispatch_async(*(self + 0x18), &var_38); // [self._player setRate:self._rate] (HAMPlayerInternal)
    [*(self + 0x40) setRate:r2]; // [self._stickySettings setRate:self._rate]
    [*(self + 0x180) broadcastRateChange:r2]; // [self._playerEventCenter broadcastRateChange:self._rate]
    r0 = objc_loadWeakRetained(self + 0x128);
    [r0 playerRateDidChange:r2];    
    return;
}
```

As the method name, it updates the new speed rate internally and broadcast the rate change via `[self._playerEventCenter broadcastRateChange:self._rate]`. We won't go deeper as we found the boundary check and understand why new speed rates do not take effect due to boundary checks. So shall we stop here or thinking how to disable the boundary check? It's up to you, if you're still interested in how to disable this boundary check, it will be revealed below section.

### Disable speed boundary check
As we've already known `-[MLHAMQueuePlayer setRate:]` is the one doing speed validation, we have multiple options to disable it, such as binary patching (replace `fminnm` and `fmaxnm` instructions with `fmov s8, s0` to accept any inputs), hooking (use `theos` tweak). This time, we will go with the hooking approach, which we are trying to rewrite this method again without min-max conditions. We will try to convert the above assembly and pseudo code of `-[MLHAMQueuePlayer setRate:]` into a new tweak method using Objective-C, it will look like this:

```bashscript
%hook MLHAMQueuePlayer

-(void)setRate:(float)rate {	
	float _rate = MSHookIvar<float>(self, "_rate");
	if (_rate == rate) {
		return;
	}

	MLHAMPlayerItemSegment *_currentSegment = MSHookIvar<MLHAMPlayerItemSegment *>(self, "_currentSegment");
	MLHAMPlayerItem *playerItem = (MLHAMPlayerItem *) [_currentSegment playerItem];
	MLInnerTubePlayerConfig *config = (MLInnerTubePlayerConfig *) [playerItem config];
	bool speedAllowed = [config varispeedAllowed];
	if (!speedAllowed) {
		return;
	}

	float _preferredRate = MSHookIvar<float>(self, "_preferredRate");
	if (_preferredRate > 1.0) {
		bool isPeggedToLive = [playerItem peggedToLive];
		if (isPeggedToLive) {
			rate = 1.0;
		}
	}

	MSHookIvar<float>(self, "_rate") = rate;
	[self internalSetRate];
}

%end
```

By building this again, you will get compiler error like this:
```bashscript
error: unknown type name 'MLHAMPlayerItemSegment'
MLHAMPlayerItemSegment *_currentSegment = MSHookIvar<MLHAMPlayerItemSegment *>(self, "_currentSegment");
...
```

I will leave this as a small exercise so you can get your hands dirty if you're keen on it. Hints: You need to add the forward declarations of the above classes so that the compiler will get it successfully.

## Further readings
- [Reading iOS app binary files](https://blog.smartdec.net/reading-ios-app-binary-files-2c9e63a381ad){:target="_blank"}
- [Data processing - floating point](https://developer.arm.com/architectures/learn-the-architecture/aarch64-instruction-set-architecture/data-processing-floating-point){:target="_blank"}
- [Single Precision](https://www.geeksforgeeks.org/difference-between-single-precision-and-double-precision/){:target="_blank"}
- [Floating-point Minimum Number](https://developer.arm.com/documentation/dui0801/g/A64-Floating-point-Instructions/FMINNM--scalar-)
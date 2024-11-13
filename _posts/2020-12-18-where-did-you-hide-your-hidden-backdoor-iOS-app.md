---
layout: post
title: Reverse engineering and find out hidden backdoor in iOS app
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczNa0y6NgJy86Wa1LXMwBg_Ra5xru2mwcU7TfZEhSFESUoBuBkdmCzjg20-b3AOrXl8ePImfHFk9eZFKrULpwy2YupOUKGE__deMQydFJhTYBq0KKCASBIVWoJUnnQOmmVtv4Fmh0Mc4wb3HJOPQ-ZEe=w1058-h1004-s-no-gm?authuser=2
    alt: Hidden backdoor
tags: [ios]
---

In this post, we will reverse an app using the StoreKit framework and bypass In-app purchases (IAP) features. This case's quite special as the app will force the user to purchase the app as a free trial before allowing to use. It will auto-renew the monthly price plan if you forget to cancel before the renewal date. There will be no option on the screen to opt-out for IAP.
![Hidden backdoor](https://lh3.googleusercontent.com/pw/AP1GczNOR3UTk1DvHPnlmzPWCb9qjmFANZQHSgYN20_prGeEBc4TktZR9MIyUWBvTVX0yj7ZwoXDfNLXwat6Xgogsk5m8RrGwQiYkKUsy9AMIOJx3dEd3a9JXJFfVa0SkgzWNVL1joK4Gl_cGgd_0a64TD1c=w640-h1102-s-no-gm?authuser=2)
_**Figure: Hidden backdoor**_

## Disclaimer
This post is for educational purposes only. How you use this information is your responsibility. I will not be held accountable for any illegal activities, so please use it at your discretion and contact the app's author if you find issues. We will inspect an app name REDACTED. The figures during the post just for demonstrations, might not relevant to REDACTED app.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [FLEX Loader Cydia package](http://cydia.saurik.com/package/com.joeyio.flexloader/){:target="_blank"}
- [Hopper Disassembler](https://www.hopperapp.com/download.html){:target="_blank"}
- A bit knowledge of assembly arm64, please read my previous [post]({{ site.baseurl }}/by-pass-ssl-pinning-iOS-with-lldb/){:target="_blank"}

## Overview
Have you ever installed an IAP app that requires to purchase before using it? Even it provides a free trial option, you still need to purchase the trial first before handing on any app features, and it will auto-renew the monthly charge option (or yearly) after the free trial expires. I found REDACTED app has the same IAP behavior, which has more than hundres of thousands ratings in the App Store. When I launch REDACTED app for the first time, it shows the paywall immediately with the only option to purchase to use the rest of the app.

![Paywall]({{ site.baseurl }}/images/20201218/paywall.jpeg)
_**Figure 1: Paywall screen**_

Tap on **Continue** button will lead to the iOS built-in IAP popup:
![IAP popup]({{ site.baseurl }}/images/20201218/apple-purchase-popup.jpeg)
_**Figure 2: IAP popup**_

There is no skip option on the screen unless you make a purchase for a free trial to use the app, needless to mention you might forget to cancel the subscription if you do not like the app for some tries and it will cost you an extra dollar once auto-renewal is triggered. 

This is a cumbersome process for the end-user to explore your app, mostly they only want to explore app features first before making a decision. When I was about to purchase IAP for a free trial, it was thinking how the app developer could submit it to Apple for review? Did the Apple team need to make IAP first to review app features? If not, how could they proceed with the review process with ease? Those questions came to my mind, and I was right that there should be a way for the Apple team to review the app without making any IAP. The app developer might be compromised somewhere to let the Apple team skip this IAP, and he might need to tell the Apple team how to skip it when submitted for review. Is it really a backdoor for the Apple team to review REDACTED app? Let's try to figure it out!

## Analysis
### What is the paywall screen name?
Configure **FLEXLoader** tweak to inject in to `REDACTED` app in `Settings` app then launch `REDACTED` app again ([refer this on how to configure FLEXLoader]({{ site.baseurl }}/how-to-tweak-existing-Medium-iOS-app-features-with-Theos-and-Logos-part-1){:target="_blank"}). Toggle `view` menu and select any view on that paywall screen, I can know that this paywall is `IntroSubscriptionWhiteBaselineViewController`.

### Findout suspicious methods
Once I know the paywall view controller name, just load `REDACTED` app into `Hopper Disassembler` for static analysis and one method caught my eye:
```nasm
-[IntroSubscriptionWhiteBaselineViewController showHiddenFreeSubscription:]:
adrp    x8, #0x100e91000
ldr     x1, [x8, #0x1a0]; @selector(confirmHiddenAppBypassMode)
b       imp___stubs__objc_msgSend; objc_msgSend
```

Base on the method name and method body it's easy to tell that once this one is triggered, it will invoke `confirmHiddenAppBypassMode` method, and again thanks to the meaningful method name it might do something relates to bypass the app. Let dive into `confirmHiddenAppBypassMode` by double click on it.

### Please insert app bypass password to continue
```c
/* @class SubscriptionBaseViewController */
-(void)confirmHiddenAppBypassMode {
    r0 = [UIAlertController alertControllerWithTitle:@"App Bypass" message:@"App bypass is for Apple reviewers and press.\n\nPlease insert app bypass password to continue." preferredStyle:0x1];
    [r0 addTextFieldWithConfigurationHandler:0x100cb8ed8];
    r21 = @class(UIAlertAction);
    r20 = [r0 retain];
    r21 = [[r21 actionWithTitle:@"OK" style:0x0 handler:&var_60] retain];
    [r20 addAction:r21];
    r22 = [[UIAlertAction actionWithTitle:@"Cancel" style:0x1 handler:0x100cb8ef8] retain];
    [r20 addAction:r22];
    [self presentViewController:r20 animated:0x1 completion:0x0];    
    return;
}
```

It turns out that `confirmHiddenAppBypassMode` belongs to `SubscriptionBaseViewController` class which is parent class of `IntroSubscriptionWhiteBaselineViewController`. The method will show a confirm alert with a text field to enter bypass password and 2 buttons **OK** **Cancel**. Now we have a clearer picture, the app have a secret password to bypass IAP. We need to enter the password and trigger **OK** button to proceed. Let's pray the password validation is handled in the binary rather than on server.

### Where is the PASSWORD?
Let toggle to `ASM mode` to see where is the handler closure of **OK** button
```nasm
ldr     x1, [x8, #0xf8] ; "addTextFieldWithConfigurationHandler:",@selector(addTextFieldWithConfigurationHandler:)
adrp    x2, #0x100cb8000 ; 0x100cb8ed8@PAGE
add     x2, x2, #0xed8 ; 0x100cb8ed8@PAGEOFF, 0x100cb8ed8
bl      imp___stubs__objc_msgSend ; objc_msgSend
adrp    x24, #0x100e9b000 ; &@selector(initWithParentView:)
ldr     x21, [x24, #0xad0] ; objc_cls_ref_UIAlertAction
adrp    x8, #_$s10Foundation10URLRequestVMn_100cb0000 ; 0x100cb0c88@PAGE
ldr     x8, [x8, #0xc88] ; 0x100cb0c88@PAGEOFF, __NSConcreteStackBlock_100cb0c88,__NSConcreteStackBlock
str     x8, [sp, #0x60 + var_60]
adrp    x8, #0x100ab8000
ldr     d0, [x8, #0xfd0] ; double_value_1_60807E_minus_314
str     d0, [sp, #0x60 + var_58]
adr     x8, #0x10015ba28; THIS IS THE ADDRESS OF HANDLER CLOSURE
```

For closure handler, you can find out above as a typical stack setup where it will load handler address `#0x10015ba28` into register `x8`. Just double click on that address it will lead you to the real implementation.

```c
int sub_10015ba28(int arg0) {    
    r19 = arg0;
    r0 = *(arg0 + 0x20);
    r0 = [r0 textFields];    
    r21 = r0;
    r0 = [r0 objectAtIndexedSubscript:0x0];
    r20 = [[r0 text] retain];    
    r21 = [[r20 lowercaseString] retain];
    r22 = objc_msgSend(@"siri", @selector(isEqualToString:));
    [r21 release];
    if (r22 != 0x0) {
            [*(r19 + 0x28) presentAppBypassSelector];
    }
    else {
            [SVProgressHUD showErrorWithStatus:@"Incorrect Password"];
    }
    return r0;
}
```

Do you see what is happening here? It gets text value from alert textfield, converts to lowercase, and compare with HARDCODING value `"siri"`. Once it matches, `presentAppBypassSelector` method will be triggered otherwise showing `"Incorrect Password"` alert. So if you enter `siri` into the alert textfield and tap on **OK** button, `presentAppBypassSelector` will be triggered.

```c
/* @class SubscriptionBaseViewController */
-(void)presentAppBypassSelector {
    r20 = [[self analyticsService] retain];
    r21 = [[self paywallIdentifier] retain];
    r22 = [[self callingLocation] retain];
    r23 = [[self pageName] retain];
    [r20 paywallBypassSelectorEnabled:r21 freemiumLocation:r22 pageName:r23];    
    r20 = [[BAAlertController alertControllerWithTitle:@"Select App Mode" message:@"Subscription Bypass Successful"] retain];
    r0 = @class(BAAlertAction);
    r22 = [[r0 actionWithTitle:@"Original" style:0x0 handler:&var_78] retain];
    [r20 addAction:r22];
    r0 = @class(BAAlertAction);
    r0 = [r0 actionWithTitle:@"Demo Mode" style:0x0 handler:&var_A0];
    r21 = [r0 retain];
    [r20 addAction:r21];
    [self presentViewController:r20 animated:0x1 completion:0x0];
}
```

As you can see `presentAppBypassSelector` method will do some analytics service and showing another alert with 2 options `ORIGINAL` and `DEMO MODE`. Trying to check what will happen for these 2 options, the former will trigger `becomeByPassUser` method while the latter will trigger `confirmDemoMode`.

```c
/* @class SubscriptionBaseViewController */
-(void)performBypassOperations {
    ...
    r19 = self;
    r0 = [self subscriptionService];    
    [r0 becomeByPassUser];
    ...
}
```

```c
/* @class SubscriptionBaseViewController */
-(void)confirmDemoMode {
    r0 = [UIAlertController alertControllerWithTitle:@"Demo" message:@"Demo mode is for Apple reviewers and press.\n\nPlease insert demo mode password to continue." preferredStyle:0x1];    
    [r0 addTextFieldWithConfigurationHandler:0x100cb8f18];
    r21 = @class(UIAlertAction);
    r20 = [r0 retain];
    r21 = [[r21 actionWithTitle:@"OK" style:0x0 handler:&var_60] retain];
    [r20 addAction:r21];
    r22 = [[UIAlertAction actionWithTitle:@"Cancel" style:0x1 handler:0x100cb8f38] retain];
    [r20 addAction:r22];
    [self presentViewController:r20 animated:0x1 completion:0x0];    
}
```

Option `ORIGINAL` will treat you as a special bypass user, I think this is a means for internal testing with ease. `DEMO MODE` supposed to be used by the Apple review team which requires another password to key-in, in short, it will compare input password with hardcoded string `"demo"` to proceed with setup steps for demo purpose.

```c
int sub_10015bcfc(int arg0) {    
    r0 = *(arg0 + 0x20);
    r0 = [r0 textFields];
    r21 = r0;
    r0 = [r0 objectAtIndexedSubscript:0x0];
    r19 = [[r0 text] retain];    
    r0 = objc_retainBlock(&var_78);
    r20 = r0;
    r0 = [r19 lowercaseString];
    r29 = &saved_fp;
    r23 = [r0 retain];
    r24 = objc_msgSend(@"demo", @selector(isEqualToString:));
    if (r24 != 0x0) {
        r21 = [[DemoHelper sharedHelper] retain];
        var_80 = [r20 retain];
        [r21 installDemoImages:0x0 completion:&var_A0];
        [r21 release];
    }
    ...
}
```

At this stage, we are quite clear there is a hidden backdoor in the app, but the question still remains is where is the entry to the backdoor?

### How end-user can trigger this backdoor in app?
It's fairly easy!!! We found suspicious method above steps `IntroSubscriptionWhiteBaselineViewController showHiddenFreeSubscription:]:`, in `Hopper Disassembler` you can right click to method name and select **References to selector showHiddenFreeSubscription:** it will navigate to this instruction `ldr  x3, [x8, #0x150] ; "showHiddenFreeSubscription:",@selector(showHiddenFreeSubscription:)` inside of `IntroSubscriptionWhiteBaselineViewController.viewDidLoad()` method.

```nasm
...
ldr		x0, [x8, #0xdb0] ; objc_cls_ref_UILongPressGestureRecognizer
bl		imp___stubs__objc_alloc ; objc_alloc
adrp	x8, #0x100e91000
ldr		x3, [x8, #0x150] ; "showHiddenFreeSubscription:",@selector(showHiddenFreeSubscription:)
adrp	x8, #0x100e8a000
ldr		x1, [x8, #0x250] ; "initWithTarget:action:",@selector(initWithTarget:action:)
mov		x2, x20
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x19, x0
adrp	x8, #0x100e8f000
ldr		x1, [x8, #0x78] ; "setMinimumPressDuration:",@selector(setMinimumPressDuration:)
fmov	d0, #0x4000000000000000 ; #2.0 in Float
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x0, x20
ldr		x23, [sp, #0x440 + var_398]
mov		x1, x23
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x29, x29
bl		imp___stubs__objc_retainAutoreleasedReturnValue ; objc_retainAutoreleasedReturnValue
mov		x21, x0
ldr		x1, [sp, #0x440 + var_3B0]
movz	w2, #0x1     ; argument "instance" for method imp___stubs__objc_msgSend
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x0, x21
bl		imp___stubs__objc_release ; objc_release
mov		x0, x20
mov		x1, x23
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x29, x29
bl		imp___stubs__objc_retainAutoreleasedReturnValue ; objc_retainAutoreleasedReturnValue
mov		x20, x0
adrp	x8, #0x100e8a000 ; &@selector(loadImageWithURL:options:progress:completed:)
ldr		x1, [x8, #0x258] ; "addGestureRecognizer:",@selector(addGestureRecognizer:)
mov		x2, x19
bl		imp___stubs__objc_msgSend ; objc_msgSend
mov		x0, x20
...
```

This block is initializing `UILongPressGestureRecognizer` instance with action is `IntroSubscriptionWhiteBaselineViewController showHiddenFreeSubscription:]:` with minimum press duration is 2 seconds to activate. To know where it is added, you can trace back or toggle into pseudo-code mode as below.

```c
...
r0 = objc_alloc();
r19 = [r0 initWithTarget:r20 action:@selector(showHiddenFreeSubscription:)];
[r19 setMinimumPressDuration:r20]; // 2.0
r0 = objc_msgSend(r20, @selector(titleLabel));
[r0 addGestureRecognizer:r19];
...
```

As you can see it is added into `titleLabel`, in the `IntroSubscriptionWhiteBaselineViewController` screen it will be **START 3 DAY TRIAL** label. You even can double confirm by attach **LLDB** debugger into running process and set breakpoint on this instruction `ldr		x1, [x8, #0x258] ; "addGestureRecognizer:",@selector(addGestureRecognizer:)` to examine value of register `x0`, which will be the `titleLabel` in this case.

Now it's time to confirm all of the above static analysis in the running app.
![Tap and hold START 3 DAY TRIAL for 2 seconds]({{ site.baseurl }}/images/20201218/app-bypass-popup.jpeg)
_**Figure 3: Tap and hold START 3 DAY TRIAL for 2 seconds**_

Enter `siri` text and tap on **OK** button it will show another app mode selection popup.
![Select App Mode popup]({{ site.baseurl }}/images/20201218/bypass-app-mode-popup.jpeg)
_**Figure 4: Select App Mode popup**_

Now you can select `ORIGINAL` mode to become a bypass user and enjoy all features for free or select `DEMO MODE` option to experience pre-setup feature for demo purpose only. Below is the popup when you select `DEMO MODE` option.

![Demo bypass popup]({{ site.baseurl }}/images/20201218/demo-bypass-mode-popup.jpeg)
_**Figure 5: Demo bypass popup**_

Congrats!!! The hidden backdoor is revealed and opened!!

### What else can we find?
By searching `UILongPressGestureRecognizer` in `Labels` tab in `Hopper Disasembler`, I can see it's used in a lot of places: 
```nasm
_OBJC_CLASS_$_UILongPressGestureRecognizer  
    ; DATA XREF=-[IntroSplashViewController viewDidLoad]+2820, 
    -[ReorderViewController viewDidLoad]+684, 
    -[SlideshowReorderViewController viewDidLoad]+592, 
    -[IntroSubscriptionWhiteBaselineViewController viewDidLoad]+8056, 
    sub_10025ba3c+1632, 
    sub_1002892c8+60, 
    sub_1004db4c8+368, 
    sub_100518ae8+5944, 
    sub_10078d0b0+112, 
    sub_10078e850+72, 
    sub_100793a98+128
```

Follow one of the above references I can toggle another interesting app variant debug popup.
![App variant debug popup]({{ site.baseurl }}/images/20201218/app-variant-debug-popup.jpeg)
_**Figure 6: App variant debug popup**_

## Final thoughts
- Hardcoding password or secret key in-app binary should not be an option, we better store it on the server-side and do the validation there to protect premium feature, REDACTED app has hundreds of thousands of rating and users, so it should consider it seriously.
- For static analysis automation, you and add this small script to quickly scan app binary:
```bash
$ nm REDACTED | grep -i UILongPressGestureRecognizer
    U _OBJC_CLASS_$_UILongPressGestureRecognizer
    U _OBJC_METACLASS_$_UILongPressGestureRecognizer
```
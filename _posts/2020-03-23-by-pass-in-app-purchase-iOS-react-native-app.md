---
layout: post
title: Bypass In-app purchases of react native iOS app
---

Today we will analysis an react native iOS app and by pass in-app purchase to use locked features. Below is an example of purchase screen whenever tap on premium content.
[![purchase screen]({{ site.baseurl }}/images/ugp-ios/purchase-screen.png)]({{ site.baseurl }}/images/ugp-ios/purchase-screen.png){:target="_blank"} <br/>**BEFORE: In-app purchases locked contents**<br/><br/>

[![unlocked screen]({{ site.baseurl }}/images/ugp-ios/chord-library-screen.png)]({{ site.baseurl }}/images/ugp-ios/chord-library-screen.png){:target="_blank"} <br/>**AFTER: In-app purchases unlocked contents**<br/><br/>

## Disclaimer
This post is for educational purposes only, please use it at your discretion and contact app's author if you find issues. For the demo purpose, I picked up and installed an app on the app store which has In-app purchases feature, please allow me to redact the app name as REDACTED.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [Beautify Code](https://beautifier.io/)
- [JS Nice](http://jsnice.org/)

## Overview
After installing and launching REDACTED app on the device, it shows me the main screen which some navigation items on the tab-bar menu. Select one of the items, let say *Tools* lead me to a new screen which allows me to select tools app supports. Select either tool will navigate to a new screen that allows us to see that tool for a second followed by a purchase screen.

![locked chord library tool](https://thumbs.gfycat.com/UniqueAmazingGrison-size_restricted.gif) <br/>**Figure 1: Chord library tool require In-app purchases**<br/><br/>

The only option we have is to subscribe and do In-app purchases, otherwise, click **X** button to cancel purchase will pop us back to *Tools* screen again. We nearly can use that tool but it's blocked in front of our eyes. However, this seems to be client-side validation, if we can find out where is logic to show purchase screen and disable that, we can get rid of it and use the tool for free. Let's do it ^_^

## Static Analysis
First of all, let's start with UI. As usual, let trace from the text we can see on the Purchase screen, something like *"other great features with Pro access"*.
So let do some static analysis by extracting the app **.ipa** file and looking for that screen.
With the help of [Frida iOS Dump](https://github.com/AloneMonkey/frida-ios-dump) or [CrackerXI](https://forum.iphonecake.com/index.php?/topic/363020-crackerxi-gui-app-decryption-tool-for-ios-11-12-13/), we can easily pull out **.ipa** file of REDACTED app on a jailbroken device, unzip **.ipa** and navigate to Payload/REDACTED.app folder. All of the app resources and binary files are inside this folder. Normally app strings are stored in some below places:
- Localization files (*.lproj/*.strings)
- Property List *.plist
- Hard-coded in Mach-O file (binary file)
- json/text files...
- ...

With simple search command `grep -iRl "other great features with Pro access" ./Payload/` luckily we found this text is contained in `main.jsbundle` file.
[![Find files containing specific text]({{ site.baseurl }}/images/ugp-ios/search-text-in-folder.png)]({{ site.baseurl }}/images/ugp-ios/search-text-in-folder.png){:target="_blank"}<br/>**Figure 2: Find files containing specific text**<br/><br/>


### React Native
If you are a mobile app developer, you can confirm that this app was developed (fully/partially) using [React Native framework](https://reactnative.dev/). React Native is an open-source mobile application framework created by Facebook. It is used to develop applications for iOS, Android, Web by enabling developers to use React along with native platform capabilities. And the main logic of the app remains inside main.jsbundle which used to be minified javascript file.

[![main.jsbundle]({{ site.baseurl }}/images/ugp-ios/main-jsbundle.png)]({{ site.baseurl }}/images/ugp-ios/main-jsbundle.png){:target="_blank"}<br/>**Figure 3: main.jsbundle content**<br/><br/>

This file size is around 5MB, it's huge and looks like the whole application logic is inside this single file, but scare not!! [Beautifier](https://beautifier.io/) comes to rescue ^_^

### Reverse main.jsbundle
Copy `main.jsbundle` content and paste on [Beautifier](https://beautifier.io/) site we will get formatted one. It won't decrypt the file to original source code but it formats syntax for readability, which we can use to reverse the app logic.

[![formatted main.jsbundle]({{ site.baseurl }}/images/ugp-ios/formatted-main-jsbundle.png)]({{ site.baseurl }}/images/ugp-ios/formatted-main-jsbundle.png){:target="_blank"}<br/>**Figure 4: Formatted main.jsbundle**<br/><br/>

Let's open original main.jsbundle and paste formatted content into any text editor, in my case I'm using *Sublime* and search for the text _"other great features with Pro access"_

Now searching for something like _"purchase"_, we will see that we can find some results, one of them likes below is worth to pay attention and dive deep.
[![onPurchaseFinish]({{ site.baseurl }}/images/ugp-ios/on-purchase-finish-callback.png)]({{ site.baseurl }}/images/ugp-ios/on-purchase-finish-callback.png){:target="_blank"}<br/>**Figure 5: Purchase search leads us to onPurchaseFinish callback**<br/><br/>

Do you see what I see? `hasProAccount` rings the bell? By searching _"purchase"_ we found a new flag `hasProAccount` which is a boolean type and might be used to decide if the user has a free account or pro account. Let's note down this variable and searching all of its references to see how it's used.
But before that, let's try to understand what this method is doing. It will be hard to read minified javascript code as it uses comma as the end of statements and some if/else syntax not clearly to observe.
To make our life easier, I found this [JS Nice](http://jsnice.org/) tools helpful. It's statistical rename, type inference and deobfuscation, perfectly fit for our case so let give it a try
[![js nice]({{ site.baseurl }}/images/ugp-ios/jsnice-onfeatureaction.png)]({{ site.baseurl }}/images/ugp-ios/jsnice-onfeatureaction.png){:target="_blank"}<br/>**Figure 6: JS Nice comes to rescue**<br/><br/>

The deobfuscated result is awesome. We can see there is a bunch of if-else logic and it's much easier to read and understand the function now.
This method will be triggered when we tap on any features in Tools tabs (CHROMATIC_TUNER_KEY, METRONOME_KEY, BRAIN_TUNER_KEY, CHORD_LIBRARY_KEY). It will do nothing if `hasProAccount` is `true`. But if `hasProAccount` is `false`, tap on any above features will do 2 things:
1. Log an event with the name `BannerUpgradeView`. The name is self-explained this would be the purchase view we see in **Figure 1**
2. Show some kinds of marketing screen (which probably the purchase screen)

If `hasProAccount` is `false` and tab in features that do not belong to the above list, it will trigger that event as normal.
So from here we can make sure `hasProAccount` will be the key to unlock In-app purchases. Let's search `hasProAccount =` to find out where this value is set, the results are only 3 places.
[![hasProAccount =]({{ site.baseurl }}/images/ugp-ios/search-has-pro-account-set.png)]({{ site.baseurl }}/images/ugp-ios/search-has-pro-account-set.png){:target="_blank"}<br/>**Figure 7: Search where hasProAccount is set**<br/><br/>


### Patch the app
Now the job is trivial by replacing `hasProAccount = true` for all 3 places:
1. `g.hasProAccount = s`  => `g.hasProAccount = true`
2. `g.hasProAccount = u`  => `g.hasProAccount = true`
3. `g.hasProAccount = !1` => `g.hasProAccount = true`

Then copy modified main.jsbundle to overwrite the one in the jailbroken device by below command:
`scp main.jsbundle root@iphone_ip:/private/var/containers/Bundle/Application/REDACTED_UUID/REDACTED.app/main.jsbundle`
Relaunch the app, BOOM!!! No more in-app purchases screen.

![unlocked chord library tool](https://thumbs.gfycat.com/RawBronzeAracari-size_restricted.gif) <br/>**Figure 8: Chord library unlocked**<br/><br/>

## Final thought
- The client-side should show a purchase screen first instead of a content screen followed by. This would hide the feeling that premium content already been on the client but blocked by another purchase screen, try to get rid of purchase screen means can use the app.
- React native is fast to develop applications for cross-platforms, but as we can see client-side logic is in plain text and with some steps required we can reverse and unlock client-side logic.
- [File Integrity Checks](https://github.com/OWASP/owasp-mstg/blob/master/Document/0x06j-Testing-Resiliency-Against-Reverse-Engineering.md#file-integrity-checks-mstg-resilience-3-and-mstg-resilience-11) can be used to mitigate main.jsbundle modification
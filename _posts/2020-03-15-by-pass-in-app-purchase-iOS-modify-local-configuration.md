---
layout: post
title: Bypass in-app purchase content in iOS apps by modifying local configuration!
---

In this post, we will reverse engineering and unlock in-app purchase contents by modifying local configuration on device. Below is an example of locked features and unlocked ones.
[![locked content]({{ site.baseurl }}/images/wp-ios/in-app-purchase-locked-contents.png)]({{ site.baseurl }}/images/wp-ios/in-app-purchase-locked-contents.png){:target="_blank"} <br/>**Figure 1: Sample In-app purchase locked contents**<br/><br/>

## Disclaimer
This post is for educational purposes only, please use it at your discretion and contact app's author if you find issues. We will inspect an app has In-app purchases feature, name PATCHABLE. The figures during the post just for demonstrations, might not relevant to PATCHABLE app.

## Prerequisites
Below tools are used during this post:
- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload)
- [Installing Burp's CA Certificate in an iOS Device](https://support.portswigger.net/customer/portal/articles/1841109-Mobile%20Set-up_iOS%20Device%20-%20Installing%20CA%20Certificate.html)
- [Configuring an iOS Device to Work With Burp](https://support.portswigger.net/customer/portal/articles/1841108-configuring-an-ios-device-to-work-with-burp)
- [Hopper Disassembler](https://www.hopperapp.com/download.html)
- A jailbroken device. This is needed to modify app sandbox data after installation.
- [SourceTree](https://www.sourcetreeapp.com/)

## Overview
After installing and launching PATCHABLE app on jailbreak device, it shows me a screen with many selectable cards that can be differentiated on UI between free content cards and in-app purchase ones (with/without top-left lock icon). For the free cards, the selection will open content and ready to use, however, locked cards will show a dialog asking for In-app purchase to unlock full bundle.

What we need to do is find out if we can disable lock icon on UI, then it might lead us to unlock in-app purchase contents without purchasing. Let's do it ^_^

## Analysis
### Static Analysis
First of all, let's start with UI. What we can see here is locked contents will have top-left lock icon. Let think about how we will implement this UI as a developer.
We just need a card background image and a lock icon on top, right? So let do some static analysis by extracting app **.ipa** file and looking for such kind of image.

With the help of [Frida iOS Dump](https://github.com/AloneMonkey/frida-ios-dump) or [CrackerXI](https://forum.iphonecake.com/index.php?/topic/363020-crackerxi-gui-app-decryption-tool-for-ios-11-12-13/), we can easily pull out **.ipa** file of PATCHABLE app on jailbroken device, unzip **.ipa** and navigate to Payload/PATCHABLE folder and look for image with lock icon, luckily we can find one inside `res-ipadhd` folder.

Why I say luckily, because sometime developer put images inside Assets catalog instead, so after compiling images will be bundled inside **Assets.car** file. Actually there is a tool to extract **Assets.car** file but we will skip for now as we found icon outside.

As of now, with that icon name **sub_lock.png**, let find out where it is used in code.
Let open **Hopper Disassembler** and drag PATCHABLE Mach-O file into it, wait a while until disassembling process completed, switch to **Strings** tab and search for **sub_lock** keyword. Unfortunately nothing is found for this keyword, which mean somehow the code does not reference to this string directly.

[![lock icon on Hopper]({{ site.baseurl }}/images/wp-ios/sub-lock-hopper-search.png)]({{ site.baseurl }}/images/wp-ios/sub-lock-hopper-search.png){:target="_blank"} <br/>**Figure 2: Search sub_lock references icon in Hopper Disassembler**<br/><br/>

### Dynamic Analysic
Without luck, let's change strategy to dynamic analysis.
This time we will launch Burp Suite with proxy configuration setup for device ready.
Relaunch the app and monitor HTTP requests in Burp Suite **HTTP history** tab, we can see many requests go through with request & response body.
[![App requests via Burp Suite]({{ site.baseurl }}/images/wp-ios/burp-app-configuration-response-body.png)]({{ site.baseurl }}/images/wp-ios/burp-app-configuration-response-body.png){:target="_blank"} <br/>**Figure 3: Burp Suite app HTTP requests**<br/><br/>


Go through response's body of each request, we can see as above there is one response's body contains some JSON fields related to our app content:
```bashscript
  "listdata": {
    "url": "http://REDACTED/appdata/REDACTED/booklist/book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756.json", 
    "checksum": "f79e2bfd42967bddc6089cd9a556c756", 
    "localpath": "book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756.json"
  }
```

With given url **http://REDACTED/appdata/REDACTED/booklist/book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756.json**, open it on browser we can see some kind of this sample JSON format:
```bashscript
{
  "data": {
    "album_list": [
      {
        "album_item_id": 1, 
        "data": {
          "checksum": "a780fe18b060c30927f0950e1e87eae8", 
          "size": 12087581, 
          "url": "http://cdn.REDACTED/ipad/en_song01new_a780fe18b060c30927f0950e1e87eae8.npk"
        }, 
        "lang": "EN"
      }, 
      {
        "album_item_id": 2, 
        "data": {
          "checksum": "6c096b05b5aa93a4cc7e1f21035d6c18", 
          "size": 10749633, 
          "url": "http://cdn.REDACTED/movie/ipad/en_song02new_6c096b05b5aa93a4cc7e1f21035d6c18.npk"
        }, 
        "lang": "EN"
      },      
      ... 
      ...
      {
        "charge_type": "charge", 
        "inapp_id": "REDACTED.fullpack", 
        "include_album_item_ids": [
          1, 
          2, 
          ...
          26
          27
        ], 
        "name": "REDACTED Pack", 
        "price": "", 
        "price_desc": "", 
        "price_state": 1, 
        "store_item_id": 1001
      }, 
      {
        "charge_type": "free", 
        "inapp_id": "", 
        "include_album_item_ids": [
          1, 
          2,            
          20
        ], 
        "name": "Free Items", 
        "price": "0", 
        "price_desc": "$0.00", 
        "price_state": 4, 
        "store_item_id": 1002
      }
    ]
  }, 
  "version": 1
}
```

Now guess what? This kind of server configuration (remote feature flags) will be used to applying in PATCHABLE app. The way it works is that there are bundles (Free and paid) that include cards content for each bundle (card ids). Needless to say what we can do is figure out how to modify `"include_album_item_ids"` array to contains all `"album_item_id"` to enjoy all premium contents as a free user.
Let double confirm again those values like `"include_album_item_ids"`, `"charge_type"` or `"price_state"` ... will be used in code, search those string in Hopper Disassembler we can see its references.
[![Search json fields in Hopper]({{ site.baseurl }}/images/wp-ios/hopper-search-charge-type.png)]({{ site.baseurl }}/images/wp-ios/hopper-search-charge-type.png){:target="_blank"} <br/>**Figure 4: Response fields are being used in app**<br/><br/>

Look back to the JSON response field `"localpath": "book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756.json"`, it's saying that this configuration file will be stored in local device (localpath) with the name **book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756** in app sandbox.
Let `ssh` to device, then navigate to `/var/mobile/Containers/Data/Application/` folder. This is the place where all installed app sandboxes locate. Due to those are named as random UUIDs, there are some ways to find out which one is PATCHABLE sandbox folder:
1. `ls -lat`: This command will list all of child directories/files of current directory and sorted by last modify date. If you just installed PATCHABLE app, this command will display PATCHABLE sandbox folder as the top one.
[![List command]({{ site.baseurl }}/images/wp-ios/list-command.png)]({{ site.baseurl }}/images/wp-ios/list-command.png){:target="_blank"} <br/>**Figure 5: List and sort directories**<br/><br/>
2. `find . -name "app-bundle-id.plist"`: This command will search plist file inside PATCHABLE sandbox (remember to replace `app-bundle-id.plist` with the one you found in `Info.plist` with key `Bundle identifier`)
[![Find plist command]({{ site.baseurl }}/images/wp-ios/find-plist-command.png)]({{ site.baseurl }}/images/wp-ios/find-plist-command.png){:target="_blank"} <br/>**Figure 6: Find plist location**<br/><br/>

Using one of above command, we can identify that **FBD05B9A-9B57-4637-B8D4-13CFFC51A19A** is the PATHCHABLE sandbox directory. Navigate to this directory and search json file using `find . -name "book_list_ios_appstore_tablet_f79e2bfd42967bddc6089cd9a556c756.json"` we can see as below this file locates inside Document directory. BRAVO!!!
[![Find json file]({{ site.baseurl }}/images/wp-ios/find-json-file.png)]({{ site.baseurl }}/images/wp-ios/find-json-file.png){:target="_blank"} <br/>**Figure 7: Find json file path**<br/><br/>

It's very straight forward now, let open this file and modify item ids of Free bundle to include all ids and save the file. I will be using `nano` command to edit and save the file
[![Modify json on local device]({{ site.baseurl }}/images/wp-ios/modify-json-nano-command.png)]({{ site.baseurl }}/images/wp-ios/modify-json-nano-command.png){:target="_blank"} <br/>**Figure 8: Modify json file on local device**<br/><br/>


After modifying and saving the file, let kill the app and relaunch (keep Burp Suite open to monitor requests), we will see nothing happen, locked cards still there. What went WRONG!!!!!

Have a look on Burp Suite HTTP history tab, we notice that there is a new request to download json configuration file, hmm.....
[![App redownload configuration file if it's modified]({{ site.baseurl }}/images/wp-ios/relaunch-app-redownload-configuration.png)]({{ site.baseurl }}/images/wp-ios/relaunch-app-redownload-configuration.png){:target="_blank"} <br/>**Figure 9: App re-downloads configuration file if it's modified**<br/><br/>

No doubt that this re-download file will overwrite the one we modified on previous step, so no wonder app did not take any effects of modified one. What we can do to prevent this re-download process whenever we modify local file???? Let's try to switch off mobile internet connection after modifying local file and relaunch the app, there will be no download file trigger due to no internet connection.

Guess what? Card unlocked, yeah!!!
Wake up buddy, it's still locked. So what's wrong here, even local file modified as expected and no file overwrite at all. Let try another approach, modify response on the fly and see if any differences...

### Intercept response with Burp Suite 
With Burp Suite **Intercept Server Responses** feature, we can setup to intercept and modify configuration json file before reaching device, which mean original response is modified. Make sure to modify local configuration file before re-launching the app to trigger download configuration file request, otherwise app will use existing one without downloading new one.
[![Intercept response with Burp Suite]({{ site.baseurl }}/images/wp-ios/burp-suite-intercept-response.png)]({{ site.baseurl }}/images/wp-ios/burp-suite-intercept-response.png){:target="_blank"} <br/>**Figure 10: Intercept and modify configuration json response**<br/><br/>

### Thanks God it's unlocked
After modified response reach device, we can see that all locked card now unlocked and can be used as normal. We are done here???
[![Intercept response with Burp Suite](https://images.unsplash.com/photo-1523050854058-8df90110c9f1?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=800&q=60)](https://images.unsplash.com/photo-1523050854058-8df90110c9f1?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=800&q=60){:target="_blank"} <br/>**Figure 11: It's over** _(source: unsplash by @Vasily Koloda)_<br/><br/>

After digging around, I found out that there is another way to modify response body to unlock content also.
Instead of modify item ids of free bundle, we can modify below values of premium bundle using Burp Suite **Match and Replace** feature, you can refer this [post]({{ site.baseurl }}/by-pass-locked-feature-iOS-apps-with-burp/){:target="_blank"} for how to use this feature.
- Replace `"price_state": 1` to `"price_state": 4` (1 is charge state while 4 is free state)
- Replace `"charge_type": "charge"` to `"charge_type": "free"`

### Shall we stop here or try to find out what happened behind the scene???
We managed how to unlock the card, but we still dont know what is going here, why modify local file did not work but modify on the fly.
Let compare app sandbox folders and files before and after it's unlocked.
One small tips is that using git to compare files change, so here is what we can do:
- First, let copy all app sandbox folders from device to laptop, then using git as version control to commit all files inside locally.
- Next, after content unlocked, copy all app sandbox again and paste on same location as step one, the popup will come out as files existed, so just REPLACE all files.
- Using SourceTree as GUI to see what's new changes
[![Compare app sandbox changes]({{ site.baseurl }}/images/wp-ios/source-tree-comparation.png)]({{ site.baseurl }}/images/wp-ios/source-tree-comparation.png){:target="_blank"} <br/>**Figure 12: sandbox files changed**<br/><br/>

As we can see file **ssapp_property** has some new lines with key and value defined which bundles were purchased and there are some changes in same file to configure checksum of downloaded contents also. The file name and file content changes make more senses to what just happened to unlock process.
To double confirm if new added lines `<key>is_purchased[StoreData.id=...]` are factors to lock premium contents, let remove those keys of this file which locates on device app sandbox `Document/ssapp_property` and relaunch the app, we will see that locked cards back.

Add those keys back will unlock contents, so are we happy now? ^_^

## Final thought
- Remote feature flags is convenient and easy to change without releasing new app, but sometime it's a flaw for someone to tweak it and enable hidden features
- Store configuration file on device as plain text makes it easy to understand app logic and sometime not secure
- Sometime compare app sandbox between each state help us to figure out how app works.

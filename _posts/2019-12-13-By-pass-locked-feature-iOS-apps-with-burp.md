---
layout: post
title: Bypass locked features on iOS apps with Burp Suite!
image:
    path: /images/ewa-ios/original-response.png
    alt: Sample In-app purchase locked contents
tags: [ios, iap, bypass, burp-suite]
---

In this post I will share some steps to enable blocked or hidden features on iOS app using Burp Suite tool. I will use EWA, a language learning app on AppStore, to illustrate the whole process.

## Disclaimer
This post is for educational purposes only. How you use this information is your responsibility. I will not be held accountable for any illegal activities, so please use it at your discretion and contact the app's author if you find issues.

## Prerequisites
Below tools are used during this post:
- [Download Burp Suite Community Edition](https://portswigger.net/burp/communitydownload){:target="_blank"}
- [Install EWA: Learn English & Spanish](https://apps.apple.com/us/app/ewa-learn-english-spanish/id1200778841){:target="_blank"}
- [Installing Burp's CA Certificate in an iOS Device](https://support.portswigger.net/customer/portal/articles/1841109-Mobile%20Set-up_iOS%20Device%20-%20Installing%20CA%20Certificate.html){:target="_blank"}
- [Configuring an iOS Device to Work With Burp](https://support.portswigger.net/customer/portal/articles/1841108-configuring-an-ios-device-to-work-with-burp){:target="_blank"}

## Overview
Install and launch the app from App Store and landing to Courses screen then tap on _350 Spanish words_ card, we can see this app provides some free lessons (with star icon) and premium lessons (with lock icon).

![Lessons screen]({{ site.baseurl }}/images/ewa-ios/lessons-screen.PNG)
_**Figure 2: Lessons screen**_

When we tap on free lesson, let say _`5. In the hospital`_ one, we will see below screen prompted with words ready to learn :)

![Free lesson popup]({{ site.baseurl }}/images/ewa-ios/free-lesson.PNG)
_**Figure 3: Free lesson popup**_


But when we tap on locked lesson, let say _6. Body parts_ one, purchase screen will be shown instead and will ask us to subscribe before proceeding premium content :(
![Purchase screen]({{ site.baseurl }}/images/ewa-ios/purchase-screen.PNG)
_**Figure 4: Purchase screen**_


- We need to find the way to by pass client validation whether lesson is free or locked. Let's do it!!!

## Analysis
Assume we already setup and configure iOS device to using laptop proxy as step in *Prerequisites* section. Install and launch the app from App Store and landing to Courses screen then tap on _350 Spanish words_ card, we can see this app provide some free lessons (with star icon) and premium lessons (with lock icon).

Check Burp Proxy - HTTP History tab, you might notice there is request to server to fetch lessons details for tapped course (_350 Spanish words_) **`https://api.asia.appewa.com/api/v9/courses/ae3a9691-092d-4c16-9e72-f3972e15984a`**

![Burp all lessons response body]({{ site.baseurl }}/images/ewa-ios/original-response.png)
_**Figure 5: Burp all lessons response body**_


Look at Response tab, we can see response is in JSON format, take notes these 2 boolean attributes **isFree** & **isLocked**. These might be ones that tell client to show lock icon and purchase screen. Try to search **"isLocked":true** or **"isFree":false** we can see there are 26 matches which is matching with 26 lessons blocking on UI.

To confirm if these 2 attributes used in client validation logic or not, we can use Burp to modify the these attributes value through proxy before reaching EWA iOS app. Thanks to Burp Match & Replace rules function, it provides the ability to find (or match) and replace certain parts of requests and responses. 
- [Follow this great article how to use replace & match rules](https://matthewsetter.com/write-burp-suite-match-and-replace-rules/), in this case we need to replace our _Response body_ for **"isLocked":true -> "isLocked":false** and **"isFree":false -> "isFree":true**

![Add new match & replace rule]({{ site.baseurl }}/images/ewa-ios/replace-isLocked.png)
_**Figure 6: Add new match & replace rule**_

![Apply new match & replace rules]({{ site.baseurl }}/images/ewa-ios/replace-response-body-rules.png)
_**Figure 7: Apply new match & replace rules**_


All setup is done, let try to send request again if response body is replaced or not. Please note EWA is caching request on server so if we try to make same request again, it wont return body data but response 304 not modified instead. So let reinstall the app and verify that request again. As you can see below, there will be new tab _Auto-modified response_, which mean response body is modified. Let further check by by searching **"isLocked":true** we can see 0 matching results, double check on UI all lessons are **UNLOCKED!!!!!!!**

![Auto modified response]({{ site.baseurl }}/images/ewa-ios/auto-modified_response.png)
_**Figure 8: Auto modified response**_


Let take step further by tapping on one locked lesson, we can see that it's not loading words like free lesson. Checking response in Burp it's showing **"Access forbidden"** message with error code 403, which make sense now as this premium content is not only protected at client but also at server.

![Access permission for premium content]({{ site.baseurl }}/images/ewa-ios/access-permission-response.png)
_**Figure 9: Access permission for premium content**_


![Premium lesson popup without any words]({{ site.baseurl }}/images/ewa-ios/lesson-start-error.PNG# thumbnail)
_**Figure 10: Premium lesson popup without any words**_

![Premium lesson details without content]({{ site.baseurl }}/images/ewa-ios/lesson-details-error.PNG# thumbnail)
_**Figure 11: Premium lesson details without content**_


## Summary
- Match & Replace rule is very useful to quickly identify which fields are used in app validation, then you might do further steps like write tweak for jailbreak or binary patching for permanent patch on non-jailbreak device.
- From developer side, client side validation is good but not enough. For apps with premium content it should be validated on server side also to mitigate app tampering.
- SSL Pinning can be employed on iOS app to prevent sniff and modify request/response payload, there are many tools and techniques to by pass SSL Pinning though.


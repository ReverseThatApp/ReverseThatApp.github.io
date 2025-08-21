---
layout: post
title: "Revealing Hidden iOS Apps: Exploring System Applications on Jailbroken Devices"
categories: [iOS, Security]
tags: [ios, springboard, security, jailbreak, debugging, frida, theos]
permalink: /posts/revealing-hidden-system-ios-apps/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPJ7Ao6h4NzVcJHjBE-7PCgYebJxh27aY8b1MKu-UUGU3HcD17SvqrtPrTAqC51DcWZZtXDRkDQXsvCgUg_ss1em3HYoyXV87oyf9R2agBeeAjARjbjqMdF4rAfZT37_RSOZpzrXziG3sHPCkwNADPo=w1176-h1180-s-no-gm
    alt: Exploring Hidden iOS System Apps
---

Apple’s iOS ecosystem is renowned for its sleek user interface and curated app experience, prominently displaying stock apps like Safari, Photos, and Calendar on the Home screen. However, beneath this polished surface lies a treasure trove of internal system apps that remain invisible to users, absent from both the Home screen and Spotlight search. As an iOS security and reverse engineering enthusiast. In this post, we’ll uncover these hidden apps, explore how the `SBAppTags->hidden` tag in the `Info.plist` file governs their visibility, and demonstrate how to reveal them on a jailbroken device using tools like Frida and a custom `theos` tweak. Whether you’re a seasoned developer or new to iOS tinkering, this journey into iOS internals will equip you with practical skills and insights.

> **Important**: This guide is strictly for educational and research purposes on a jailbroken test device not used for personal or commercial purposes. Modifying iOS to reveal hidden apps may violate the Apple Developer Program License Agreement (ADPLA), void warranties, and risk account suspension or device security issues. Proceed at your own risk and ensure compliance with Apple’s terms.
{: .prompt-info}

## What You’ll Need

To embark on this exploration of iOS’s hidden apps, you’ll need a specific set of tools and a foundational setup. Don’t worry if you’re new to this—we’ll break down each requirement clearly to ensure you’re ready to dive in:

- **Jailbroken iOS Device**: A jailbroken iPhone or iPad is essential, this post is for iOS 16.7.11 . Jailbreaks such as `palera1n` or `unc0ver` grant the elevated privileges needed to access and modify system components. Ensure your device is a test unit, as jailbreaking voids Apple’s warranty.
- **Tools**:
  - `ipsw`: A command-line utility for downloading and analyzing iOS firmware, critical for identifying system apps within the firmware’s file system.
  - IDA: A disassembler for examining iOS binaries like `SpringBoard`, helping us understand how app visibility is controlled.
  - Frida: A dynamic instrumentation toolkit for runtime manipulation, allowing us to interact with `SpringBoard` and modify app behavior live.
  - `theos`: A development framework for creating iOS tweaks, enabling us to build custom modifications to reveal hidden apps.
- **SSH Access**: A secure shell connection to your device facilitates remote command execution and file transfers, streamlining the debugging and tweaking process.
- **Basic Skills**: Familiarity with terminal commands is helpful, but we’ll provide detailed explanations to make this accessible for beginners dipping their toes into iOS reverse engineering.

## Discovering Hidden Apps in iOS Firmware

Our journey begins by uncovering the full scope of apps bundled within iOS. While your iPhone’s Home screen displays familiar apps like Maps and Notes, the firmware contains many more that Apple keeps hidden. Let’s explore this hidden landscape by diving into the iOS firmware.

To start, identify your device’s iOS version (e.g., iOS 16.7.11 on iPhone10,4) via `Settings > General > About`. Then, use the `ipsw` tool to download the corresponding firmware file. You can refer to my related [Beginner's Guide to Disabling ASLR in iOS Apps](/posts/disable-aslr-ios-apps/#step-3-get-the-ios-firmware) post for `ipsw` usage. Mount the firmware’s file system to access its contents:

```bash
$ ipsw mount fs iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
   • Mounted fs DMG 087-86684-034.dmg
      • Press Ctrl+C to unmount '/tmp/087-86684-034.dmg.mount' ...
```

Next, search for directories ending in `.app`, which represent iOS applications. This reveals a surprising number of apps far beyond what’s visible on your device:

```bash
$ cd /tmp/087-86684-034.dmg.mount
$ find . -type d -name "*.app" | wc -l
     229
$ find . -type d -name "*.app"
./System/Library/CoreServices/BluetoothUIService.app
./System/Library/CoreServices/OverlayUI.app
./System/Library/CoreServices/SpringBoard.app
./System/Library/CoreServices/AccessibilityUIServer.app
./System/Library/CoreServices/DeviceOMatic.app
./System/Library/CoreServices/AssistiveTouch.app
./System/Library/CoreServices/ScreenSharingServer.app
./System/Library/CoreServices/CommandAndControl.app
./System/Library/CoreServices/ClarityBoard.app
./System/Library/CoreServices/AegirProxyApp.app
./System/Library/CoreServices/LiveTranscriptionUI.app
./System/Library/CoreServices/EscrowSecurityAlert.app
./System/Library/CoreServices/FullKeyboardAccess.app
./System/Library/CoreServices/CMPluginApp.app
./System/Library/CoreServices/CarPlay.app
./System/Library/CoreServices/VoiceOverTouch.app
./System/Library/CoreServices/CarPlayTemplateUIHost.app
./System/Library/AppPlaceholders/Health.app
./System/Library/AppPlaceholders/MobileStore.app
./System/Library/AppPlaceholders/News.app
./System/Library/AppPlaceholders/Shortcuts.app
./System/Library/AppPlaceholders/Stocks.app
./System/Library/AppPlaceholders/Podcasts.app
./System/Library/AppPlaceholders/Reminders.app
./System/Library/AppPlaceholders/Measure.app
./System/Library/AppPlaceholders/Fitness.app
./System/Library/AppPlaceholders/Compass.app
./System/Library/AppPlaceholders/Home.app
./System/Library/AppPlaceholders/Bridge.app
./System/Library/AppPlaceholders/Magnifier.app
./System/Library/AppPlaceholders/Freeform.app
./System/Library/AppPlaceholders/Calculator.app
./System/Library/AppPlaceholders/AppleTV.app
./System/Library/AppPlaceholders/Passbook.app
./System/Library/AppPlaceholders/MobileNotes.app
./System/Library/AppPlaceholders/Tips.app
./System/Library/AppPlaceholders/VoiceMemos.app
./System/Library/AppPlaceholders/MobileTimer.app
./System/Library/AppPlaceholders/SequoiaTranslator.app
./System/Library/AppPlaceholders/FaceTime.app
./System/Library/AppPlaceholders/FindMy.app
./System/Library/AppPlaceholders/Maps.app
./System/Library/AppPlaceholders/Contacts.app
./System/Library/AppPlaceholders/Files.app
./System/Library/AppPlaceholders/Books.app
./System/Library/AppPlaceholders/MobileCal.app
./System/Library/AppPlaceholders/Music.app
./System/Library/AppPlaceholders/Weather.app
./System/Library/AppPlaceholders/MobileMail.app
./System/Library/PrivateFrameworks/SocialLayer.framework/sociallayerd.app
./System/Library/PrivateFrameworks/IDSFoundation.framework/IDSCredentialsAgent.app
./System/Library/PrivateFrameworks/IDSFoundation.framework/IDSRemoteURLConnectionAgent.app
./System/Library/PrivateFrameworks/IDS.framework/identityservicesd.app
./System/Library/PrivateFrameworks/IMCore.framework/imagent.app
./System/Library/PrivateFrameworks/IMTransferServices.framework/IMTransferAgent.app
./System/Library/PrivateFrameworks/IMDPersistence.framework/IMAutomaticHistoryDeletionAgent.app
./System/Library/PrivateFrameworks/AutomationMode.framework/AutomationModeUI.app
./private/var/staged_system_apps/Health.app
./private/var/staged_system_apps/MobileStore.app
./private/var/staged_system_apps/News.app
./private/var/staged_system_apps/Shortcuts.app
./private/var/staged_system_apps/Stocks.app
./private/var/staged_system_apps/Podcasts.app
./private/var/staged_system_apps/Reminders.app
./private/var/staged_system_apps/Measure.app
./private/var/staged_system_apps/Fitness.app
./private/var/staged_system_apps/Compass.app
./private/var/staged_system_apps/Home.app
./private/var/staged_system_apps/Bridge.app
./private/var/staged_system_apps/Magnifier.app
./private/var/staged_system_apps/Freeform.app
./private/var/staged_system_apps/Calculator.app
./private/var/staged_system_apps/AppleTV.app
./private/var/staged_system_apps/Passbook.app
./private/var/staged_system_apps/MobileNotes.app
./private/var/staged_system_apps/Tips.app
./private/var/staged_system_apps/VoiceMemos.app
./private/var/staged_system_apps/MobileTimer.app
./private/var/staged_system_apps/SequoiaTranslator.app
./private/var/staged_system_apps/FaceTime.app
./private/var/staged_system_apps/FindMy.app
./private/var/staged_system_apps/Maps.app
./private/var/staged_system_apps/Contacts.app
./private/var/staged_system_apps/Files.app
./private/var/staged_system_apps/Books.app
./private/var/staged_system_apps/MobileCal.app
./private/var/staged_system_apps/Music.app
./private/var/staged_system_apps/Weather.app
./private/var/staged_system_apps/MobileMail.app
./Applications/ClarityCamera.app
./Applications/Coverage Details.app
./Applications/PaperBoard.app
./Applications/ClarityPhotos.app
./Applications/AAUIViewService.app
./Applications/DemoApp.app
./Applications/PDUIApp.app
./Applications/MobileSMS.app
./Applications/EventViewService.app
./Applications/AccountAuthenticationDialog.app
./Applications/CTKUIService.app
./Applications/Screen Time.app
./Applications/CheckerBoardRemoteSetup.app
./Applications/MusicRecognition.app
./Applications/FontInstallViewService.app
./Applications/TVSetupUIService.app
./Applications/AnimojiStickers.app
./Applications/Setup.app
./Applications/AppSSOUIService.app
./Applications/HearingApp.app
./Applications/Diagnostics.app
./Applications/FMDMagSafeSetupRemoteUI.app
./Applications/PhotosUIService.app
./Applications/InputUI.app
./Applications/CarPlayWallpaper.app
./Applications/CTNotifyUIService.app
./Applications/Spotlight.app
./Applications/Xcode Previews.app
./Applications/HashtagImages.app
./Applications/FunCameraShapes.app
./Applications/Batteries.app
./Applications/iCloud.app
./Applications/SystemPaperViewService.app
./Applications/DisplayCal.app
./Applications/Sidecar.app
./Applications/PosterBoard.app
./Applications/DataActivation.app
./Applications/DDActionsService.app
./Applications/FieldTest.app
./Applications/Family.app
./Applications/iMessageAppsViewService.app
./Applications/BusinessExtensionsWrapper.app
./Applications/TrustMe.app
./Applications/Apple TV Remote.app
./Applications/HomeCaptiveViewService.app
./Applications/DiagnosticsService.app
./Applications/Preferences.app
./Applications/CompanionAuthViewService.app
./Applications/CheckerBoard.app
./Applications/CarPlaySettings.app
./Applications/HomeUIService.app
./Applications/FamilyControlsAuthenticationUI.app
./Applications/FindMyExtensionContainer.app
./Applications/HealthENLauncher.app
./Applications/CoreAuthUI.app
./Applications/Print Center.app
./Applications/MediaRemoteUI.app
./Applications/GameCenterWidgets.app
./Applications/BarcodeScanner.app
./Applications/BacklinkIndicator.app
./Applications/SafariViewService.app
./Applications/AMSEngagementViewService.app
./Applications/ContactlessReaderUIService.app
./Applications/FunCameraEmojiStickers.app
./Applications/Siri.app
./Applications/MBHelperApp.app
./Applications/ExposureNotificationRemoteViewService.app
./Applications/FTMInternal-4.app
./Applications/MusicUIService.app
./Applications/ShortcutsUI.app
./Applications/SleepLockScreen.app
./Applications/DiagnosticsReporter.app
./Applications/CompassCalibrationViewService.app
./Applications/StoreKitUIService.app
./Applications/PeopleViewService.app
./Applications/AirDropUI.app
./Applications/PassbookUISceneService.app
./Applications/CarPlaySplashScreen.app
./Applications/ReplayKitAngel.app
./Applications/AppStore.app
./Applications/SubcredentialUIService.app
./Applications/SLYahooAuth.app
./Applications/SharedWebCredentialViewService.app
./Applications/FunCameraText.app
./Applications/ScreenTimeUnlock.app
./Applications/SoftwareUpdateUIService.app
./Applications/TVAccessViewService.app
./Applications/SOSBuddy.app
./Applications/AXRemoteViewService.app
./Applications/SleepWidgetContainer.app
./Applications/MessagesViewService.app
./Applications/PeopleMessageService.app
./Applications/MTLReplayer.app
./Applications/ShortcutsViewService.app
./Applications/GameCenterUIService.app
./Applications/GameCenterRemoteAlert.app
./Applications/MobilePhone.app
./Applications/SharingViewService.app
./Applications/AirPlayReceiver.app
./Applications/iCloud+.app
./Applications/AXUIViewService.app
./Applications/RemoteiCloudQuotaUI.app
./Applications/PassbookUIService.app
./Applications/MobileSafari.app
./Applications/WebContentAnalysisUI.app
./Applications/WebSheet.app
./Applications/VideoSubscriberAccountViewService.app
./Applications/PassbookSecureUIService.app
./Applications/AuthKitUIService.app
./Applications/StoreDemoViewService.app
./Applications/SMS Filter.app
./Applications/HDSViewService.app
./Applications/MailCompositionService.app
./Applications/CTCarrierSpaceAuth.app
./Applications/ScreenshotServicesService.app
./Applications/SIMSetupUIService.app
./Applications/InCallService.app
./Applications/Feedback Assistant iOS.app
./Applications/ProximityReaderUIService.app
./Applications/CredentialSharingUIViewService.app
./Applications/HomeControlService.app
./Applications/SpringBoardEducation.app
./Applications/ClockAngel.app
./Applications/AskPermissionUI.app
./Applications/RecoverDeviceUI.app
./Applications/ScreenSharingViewService.app
./Applications/MobileSlideShow.app
./Applications/PCViewService.app
./Applications/HealthPrivacyService.app
./Applications/ActivityMessagesApp.app
./Applications/ClipViewService.app
./Applications/FaceTimeLinkTrampoline.app
./Applications/HealthENBuddy.app
./Applications/BusinessChatViewService.app
./Applications/RemotePaymentPassActionsService.app
./Applications/Camera.app
./Applications/Web.app
./Applications/PreBoard.app
./Applications/FindMyRemoteUIService.app
./Applications/AuthenticationServicesUI.app
```

The output reveals 229 `.app` directories, a stark contrast to the handful of apps visible on your Home screen. These hidden apps, often residing in `/System/Library/CoreServices` or `/Applications`, include system utilities like `ScreenSharingViewService` and `Diagnostics` that perform specialized tasks but are not meant for direct user interaction. This discovery sets the stage for understanding why these apps are concealed and how we can reveal them.

Not all of these directories contain fully functional apps. For instance, `/System/Library/AppPlaceholders/` serves as a system directory that stores placeholder images and icons used during initial device setup or when an app is not yet fully installed or available.

The `/private/var/staged_system_apps/` directory acts as a temporary staging area for preinstalled or "stageable" system apps in iOS. In earlier versions, these apps resided here before being relocated. Starting with iOS 11, they are moved to `/private/var/containers/Bundle/Application/` within a **GUID-based** subfolder structure, preparing them for full installation and user accessibility.

To test launching a few of these apps on a jailbroken device, you can use the following commands, though not all apps may open successfully:

```bash
iPhone $ uiopen --path /Applications/ClarityCamera.app
iPhone $ uiopen --path /Applications/Web.app
```

## Understanding App Visibility with SpringBoard

The key to an app’s visibility lies in `SpringBoard`, iOS’s Home screen manager. `SpringBoard` determines which apps appear on the Home screen or in Spotlight search by processing their metadata, specifically the `Info.plist` file within each `.app` bundle. Let’s investigate why some apps remain hidden.

### Identifying App Visibility Configurations

How does iOS determine which apps to display or conceal? Typically, such settings are stored in configuration files. In iOS, the `Info.plist` file within each app bundle holds this information. By comparing the `Info.plist` files of visible and hidden apps, we discovered that hidden apps include an additional `SBAppTags` key, which dictates their visibility status.

![SBAppTags hidden](https://lh3.googleusercontent.com/pw/AP1GczPfiaRIBMlRoRjQqbnkyqQanGYDI_VeOQjoEgaWMRN5fjGqmCOLuJmuJT1BkjbGx-5hfne_zxt9YmrhYPMLlWNicrDKgIEXQPLYxJMlceUb1ybaB5XQB4EKWi8S52SaRio612WHpalonUkzcri8O7fa=w2126-h1290-s-no-gm)
*Figure: SBAppTags Configuration for Hidden Apps*

Upon further inspection of the iOS 16.7.11 firmware, approximately 144 were found to contain the `SBAppTags` key:

```bash
$ grep -r --include \Info.plist  SBAppTags ./ | wc -l
     144
$ grep -r --include \Info.plist  SBAppTags ./
Binary file ./System/Library/CoreServices/OverlayUI.app/Info.plist matches
Binary file ./System/Library/CoreServices/AccessibilityUIServer.app/Info.plist matches
Binary file ./System/Library/CoreServices/ClarityBoard.app/Info.plist matches
Binary file ./System/Library/CoreServices/CarPlayTemplateUIHost.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/MobileStore.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/Measure.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/Compass.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/Bridge.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/FaceTime.app/Info.plist matches
Binary file ./System/Library/AppPlaceholders/Contacts.app/Info.plist matches
Binary file ./System/Library/PrivateFrameworks/ContactlessReaderUI.framework/Info.plist matches
Binary file ./System/Applications/Family/InviteMessageBubbleExtension.appex/Info.plist matches
Binary file ./private/var/staged_system_apps/MobileStore.app/Info.plist matches
Binary file ./private/var/staged_system_apps/Measure.app/Info.plist matches
Binary file ./private/var/staged_system_apps/Compass.app/Info.plist matches
Binary file ./private/var/staged_system_apps/Bridge.app/Info.plist matches
Binary file ./private/var/staged_system_apps/FaceTime.app/Info.plist matches
Binary file ./private/var/staged_system_apps/Contacts.app/Info.plist matches
Binary file ./Applications/ClarityCamera.app/Info.plist matches
Binary file ./Applications/Coverage Details.app/Info.plist matches
Binary file ./Applications/PaperBoard.app/Info.plist matches
Binary file ./Applications/ClarityPhotos.app/Info.plist matches
Binary file ./Applications/AAUIViewService.app/Info.plist matches
Binary file ./Applications/DemoApp.app/Info.plist matches
Binary file ./Applications/PDUIApp.app/Info.plist matches
Binary file ./Applications/MobileSMS.app/Info.plist matches
Binary file ./Applications/EventViewService.app/Info.plist matches
Binary file ./Applications/AccountAuthenticationDialog.app/Info.plist matches
Binary file ./Applications/CTKUIService.app/Info.plist matches
Binary file ./Applications/Screen Time.app/Info.plist matches
Binary file ./Applications/CheckerBoardRemoteSetup.app/Info.plist matches
Binary file ./Applications/MusicRecognition.app/Info.plist matches
Binary file ./Applications/FontInstallViewService.app/Info.plist matches
Binary file ./Applications/TVSetupUIService.app/Info.plist matches
Binary file ./Applications/Setup.app/Info.plist matches
Binary file ./Applications/AppSSOUIService.app/Info.plist matches
Binary file ./Applications/HearingApp.app/Info.plist matches
Binary file ./Applications/Diagnostics.app/Info.plist matches
Binary file ./Applications/FMDMagSafeSetupRemoteUI.app/Info.plist matches
Binary file ./Applications/PhotosUIService.app/Info.plist matches
Binary file ./Applications/InputUI.app/Info.plist matches
Binary file ./Applications/CarPlayWallpaper.app/Info.plist matches
Binary file ./Applications/CTNotifyUIService.app/Info.plist matches
Binary file ./Applications/Spotlight.app/Info.plist matches
Binary file ./Applications/Batteries.app/Info.plist matches
Binary file ./Applications/iCloud.app/Info.plist matches
Binary file ./Applications/SystemPaperViewService.app/Info.plist matches
Binary file ./Applications/PosterBoard.app/Info.plist matches
Binary file ./Applications/DataActivation.app/Info.plist matches
Binary file ./Applications/DDActionsService.app/Info.plist matches
Binary file ./Applications/FieldTest.app/Info.plist matches
Binary file ./Applications/Family.app/PlugIns/InviteMessageBubbleExtension.appex/Info.plist matches
Binary file ./Applications/Family.app/Info.plist matches
Binary file ./Applications/iMessageAppsViewService.app/Info.plist matches
Binary file ./Applications/TrustMe.app/Info.plist matches
Binary file ./Applications/Apple TV Remote.app/Info.plist matches
Binary file ./Applications/HomeCaptiveViewService.app/Info.plist matches
Binary file ./Applications/DiagnosticsService.app/Info.plist matches
Binary file ./Applications/CompanionAuthViewService.app/Info.plist matches
Binary file ./Applications/CheckerBoard.app/Info.plist matches
Binary file ./Applications/CarPlaySettings.app/Info.plist matches
Binary file ./Applications/HomeUIService.app/Info.plist matches
Binary file ./Applications/FamilyControlsAuthenticationUI.app/Info.plist matches
Binary file ./Applications/FindMyExtensionContainer.app/Info.plist matches
Binary file ./Applications/HealthENLauncher.app/Info.plist matches
Binary file ./Applications/CoreAuthUI.app/Info.plist matches
Binary file ./Applications/Print Center.app/Info.plist matches
Binary file ./Applications/MediaRemoteUI.app/Info.plist matches
Binary file ./Applications/GameCenterWidgets.app/Info.plist matches
Binary file ./Applications/BarcodeScanner.app/Info.plist matches
Binary file ./Applications/BacklinkIndicator.app/Info.plist matches
Binary file ./Applications/SafariViewService.app/Info.plist matches
Binary file ./Applications/AMSEngagementViewService.app/Info.plist matches
Binary file ./Applications/ContactlessReaderUIService.app/Info.plist matches
Binary file ./Applications/Siri.app/Info.plist matches
Binary file ./Applications/MBHelperApp.app/Info.plist matches
Binary file ./Applications/ExposureNotificationRemoteViewService.app/Info.plist matches
Binary file ./Applications/FTMInternal-4.app/Info.plist matches
Binary file ./Applications/MusicUIService.app/Info.plist matches
Binary file ./Applications/ShortcutsUI.app/Info.plist matches
Binary file ./Applications/SleepLockScreen.app/Info.plist matches
Binary file ./Applications/DiagnosticsReporter.app/Info.plist matches
Binary file ./Applications/CompassCalibrationViewService.app/Info.plist matches
Binary file ./Applications/StoreKitUIService.app/Info.plist matches
Binary file ./Applications/PeopleViewService.app/Info.plist matches
Binary file ./Applications/AirDropUI.app/Info.plist matches
Binary file ./Applications/PassbookUISceneService.app/Info.plist matches
Binary file ./Applications/CarPlaySplashScreen.app/Info.plist matches
Binary file ./Applications/ReplayKitAngel.app/Info.plist matches
Binary file ./Applications/SubcredentialUIService.app/Info.plist matches
Binary file ./Applications/SLYahooAuth.app/Info.plist matches
Binary file ./Applications/SharedWebCredentialViewService.app/Info.plist matches
Binary file ./Applications/ScreenTimeUnlock.app/Info.plist matches
Binary file ./Applications/SoftwareUpdateUIService.app/Info.plist matches
Binary file ./Applications/TVAccessViewService.app/Info.plist matches
Binary file ./Applications/SOSBuddy.app/Info.plist matches
Binary file ./Applications/AXRemoteViewService.app/Info.plist matches
Binary file ./Applications/SleepWidgetContainer.app/Info.plist matches
Binary file ./Applications/MessagesViewService.app/Info.plist matches
Binary file ./Applications/PeopleMessageService.app/PlugIns/PeopleMessagesScreenTime.appex/Info.plist matches
Binary file ./Applications/PeopleMessageService.app/PlugIns/PeopleMessagesAskToBuy.appex/Info.plist matches
Binary file ./Applications/PeopleMessageService.app/Info.plist matches
Binary file ./Applications/MTLReplayer.app/Info.plist matches
Binary file ./Applications/ShortcutsViewService.app/Info.plist matches
Binary file ./Applications/GameCenterUIService.app/Info.plist matches
Binary file ./Applications/GameCenterRemoteAlert.app/Info.plist matches
Binary file ./Applications/MobilePhone.app/Info.plist matches
Binary file ./Applications/SharingViewService.app/Info.plist matches
Binary file ./Applications/AirPlayReceiver.app/Info.plist matches
Binary file ./Applications/iCloud+.app/Info.plist matches
Binary file ./Applications/AXUIViewService.app/Info.plist matches
Binary file ./Applications/RemoteiCloudQuotaUI.app/Info.plist matches
Binary file ./Applications/PassbookUIService.app/Info.plist matches
Binary file ./Applications/WebContentAnalysisUI.app/Info.plist matches
Binary file ./Applications/WebSheet.app/Info.plist matches
Binary file ./Applications/VideoSubscriberAccountViewService.app/Info.plist matches
Binary file ./Applications/PassbookSecureUIService.app/Info.plist matches
Binary file ./Applications/AuthKitUIService.app/Info.plist matches
Binary file ./Applications/StoreDemoViewService.app/Info.plist matches
Binary file ./Applications/HDSViewService.app/Info.plist matches
Binary file ./Applications/MailCompositionService.app/Info.plist matches
Binary file ./Applications/CTCarrierSpaceAuth.app/Info.plist matches
Binary file ./Applications/ScreenshotServicesService.app/Info.plist matches
Binary file ./Applications/SIMSetupUIService.app/Info.plist matches
Binary file ./Applications/InCallService.app/Info.plist matches
Binary file ./Applications/ProximityReaderUIService.app/Info.plist matches
Binary file ./Applications/CredentialSharingUIViewService.app/Info.plist matches
Binary file ./Applications/HomeControlService.app/Info.plist matches
Binary file ./Applications/SpringBoardEducation.app/Info.plist matches
Binary file ./Applications/ClockAngel.app/Info.plist matches
Binary file ./Applications/AskPermissionUI.app/Info.plist matches
Binary file ./Applications/RecoverDeviceUI.app/Info.plist matches
Binary file ./Applications/ScreenSharingViewService.app/Info.plist matches
Binary file ./Applications/PCViewService.app/Info.plist matches
Binary file ./Applications/HealthPrivacyService.app/Info.plist matches
Binary file ./Applications/ClipViewService.app/Info.plist matches
Binary file ./Applications/FaceTimeLinkTrampoline.app/Info.plist matches
Binary file ./Applications/HealthENBuddy.app/Info.plist matches
Binary file ./Applications/BusinessChatViewService.app/Info.plist matches
Binary file ./Applications/RemotePaymentPassActionsService.app/Info.plist matches
Binary file ./Applications/Camera.app/Info.plist matches
Binary file ./Applications/PreBoard.app/Info.plist matches
Binary file ./Applications/FindMyRemoteUIService.app/Info.plist matches
Binary file ./Applications/AuthenticationServicesUI.app/Info.plist matches
```

### Examining `Info.plist` and `SBAppTags`

Each iOS app bundle contains an `Info.plist` file that defines its properties, such as the bundle identifier and display name. For system apps, this file may include an `SBAppTags` key, which `SpringBoard` uses to categorize apps. The `hidden` tag within `SBAppTags` is particularly significant, as it instructs `SpringBoard` to exclude the app from the Home screen and Spotlight search.

To illustrate, navigate to an app like `/Applications/Screen Time.app` in the mounted firmware and inspect its `Info.plist`. You might find:

```xml
<key>SBAppTags</key>
<array>
    <string>hidden</string>
    <string>SBNonDefaultSystemAppTag</string>
</array>
```

The `hidden` tag signals `SpringBoard` to suppress the app’s icon, while `SBNonDefaultSystemAppTag` further indicates it’s a non-standard system app. In contrast, visible apps like `MobileCal.app` (Calendar) typically lack these tags, allowing `SpringBoard` to display them. This mechanism explains why only a subset of the 229 apps appears on your device, and it provides a target for our modification efforts.

### Locating Key Entry Points

To pinpoint where the `SBAppTags` configuration is processed within the iOS firmware, we can use the `ipsw dyld str` command to search for the `SBAppTags` string in the shared cache. This approach helps us identify relevant files and modules among the numerous components in the firmware, streamlining our starting point for analysis.

```bash
$ ipsw dyld str 20H360__iPhone10,1_4/dyld_shared_cache_arm64 "SBAppTags"
   • Searching for strings: SBAppTags
0x1a3904a5c: "SBAppTags"	image=InstallCoordination
0x19db924cb: "SBAppTags"	image=UIAccessibility
0x1f240cd01: "SBAppTags"	image=Widgets
0x181d35555: "SBAppTags"	image=CoreServices
0x1abe4aa6e: "SBAppTags"	image=MobileInstallation
```

## Analysis
Let’s begin our exploration by diving into the `CoreServices` framework - highly likely the one - to understand how `SBAppTags` influences app visibility.

### Extracting and Loading CoreServices in IDA

Our journey starts with loading the `dyld_shared_cache_arm64` into IDA, a disassembler that lets us inspect binary code.

To optimize performance, we’ll use IDA’s selective modules mode instead of loading the entire cache, which can be resource-intensive. Our prior search for `SBAppTags` pointed to the `CoreServices` framework, so we’ll focus on loading this module first.

```bash
$ ipsw extract --dyld 20H360__iPhone10,1_4/iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
```

![Load dyld_shared_cache_arm64 in IDA](https://lh3.googleusercontent.com/pw/AP1GczMqTQJWqE84F8du8Xqa0tpd-FlY4opJqqePsNNn1ex6GoYOCAqbwqKic4P-hS48ZU5eDod3_HeD-rxQTRTwfOHOqSGsiwE2jdytdsm6Jj-wmtwUbcPdCfgogycVBDVf0I6fMGrEd5E85GSjGb1mhbpf=w1684-h1170-s-no-gm)
*Figure: Loading dyld_shared_cache_arm64 in IDA*

After loading, filter for the `CoreServices` module in IDA to begin our analysis.

![Load CoreServices](https://lh3.googleusercontent.com/pw/AP1GczMh5Jcgbif6ItdfD6O6ReJi5Z6RKkPnNyHKHjOxl36TVOurgFiCIXonG-_EO9fjqHmyqeq9dbPL7CWUSCbAujrDZm8lLfzguenWdCFkHb6HAJzRv7eJTI_lGufTqXM1Jp17zLP6JX9k7W89PufyAbEP=w2126-h1270-s-no-gm)
*Figure: Loading CoreServices*

> Loading all modules in `dyld_shared_cache_arm64` at once can significantly slow down IDA. Instead, load modules on demand by navigating to `File -> Load file -> DYLD Shared Cache Utils -> Load module...` and selecting the desired module.
{: .prompt-info}

### Resolving Missing Modules

With `CoreServices` loaded, let’s search for the `SBAppTags` string in IDA’s `Strings` tab to locate its cross-references (XREFs). This leads us to the `CoreServices:__cfstring` section. However, we notice that adjacent addresses appear in **red**, indicating unresolved references.

![Unknown addresses in Red](https://lh3.googleusercontent.com/pw/AP1GczPJKcxo7O-O4kcwcZ7Brm3P4pfTgIufUc47ESaUG_VVVf9wsbCHLkdl-P8GNIDg_zyurAd1gKQkljaLp7GPy82vywZ0azq07-WdYQYoitHcqNDNQhg-8zsr7XhK8I_p-jOgZmDEQkHrMtbRUX8oDMMp=w2126-h1130-s-no-gm)
*Figure: Unresolved addresses in red*

These red addresses indicate missing modules, as iOS system libraries are prelinked in the `dyld_shared_cache_arm64`. Since we only loaded `CoreServices`, dependencies like `CoreFoundation` remain unresolved. To fix this, right-click the red addresses in IDA and select the suggested module, in this case, `Load CoreFoundation`.

![Load missing module](https://lh3.googleusercontent.com/pw/AP1GczMwX6D2a902n-yxmAGRegxGwFheNjiB47IA89pOGd-bWuNiEkOFwoTgaSQjIeEliSXjLNwJ-3ZaDBnmtFbljrIZR_zlutGpiFTgPYhW8vQZhT7mRX6FViydwK8SU5RQx8YbiVa6EZ5GnJRCZ-33gScF=w2126-h1158-s-no-gm)
*Figure: Loading missing module*

Once `CoreFoundation` is loaded, the addresses resolve, providing a clearer view of the code.

![Resolved missing CoreFoundation](https://lh3.googleusercontent.com/pw/AP1GczPJ7K5PI6-ZOiNe5hWecw_gF4vZJ05hqzPGj-tMScN2UQPKh-YJKN1olusuVBWXsb14Lew_50W_6gJbb8m9a5qVa-m-AgfZv7mnu6viQnm6XtEjFAsuDvZ0HcXp5mZAgLXT8AetrvfpnYolM7oEZZ7d=w2126-h1136-s-no-gm)
*Figure: Resolved CoreFoundation module*

> The `dyld_shared_cache_arm64` is a consolidated file containing prelinked iOS system libraries, optimized for performance. Loading only specific modules, like `CoreServices` or `CoreFoundation`, reduces memory usage and speeds up analysis in IDA.
{: .prompt-info}

This on-demand module loading technique is a cornerstone of efficient static analysis, and we’ll revisit it as we dig deeper.

### Analyzing `SBAppTags` in Pseudocode Mode

Switching to IDA’s pseudocode mode, we can see how `SBAppTags` is implemented. Initially, unresolved addresses appear as `MEMORY[XXXX]` patterns, indicating missing dependencies.

![Unknown addresses in pseudocode mode](https://lh3.googleusercontent.com/pw/AP1GczPZF4Y-kWtd_KIhxbdqYl2g0wL0cKcZFl-ZwelzqNds4VeEj39AZXVkQm7vDTg_IYl1XmDS8MmjMVnhJBbwipcP4gkbniAgiyXnOI7_tbDiS2dAB93qMh63gczcs1WwSkEr2aReVEDWhEytHTM4z8Tf=w2126-h662-s-no-gm)
*Figure: Unresolved addresses in pseudocode mode*

To resolve these, right-click the unresolved addresses and load the suggested module, such as `dyld_shared_cache_arm64.02:__stubs`. After loading, press `F5` to re-decompile the pseudocode, reflecting the resolved references.

![Resolved addresses in pseudocode mode](https://lh3.googleusercontent.com/pw/AP1GczMlRL-2XvD3cijINZPWdZC6AMDpIt-KSJVJXbEOY4luk9kK0ehWaJJwUzR1pxShTNAkLsN8WoyWXg5EKWYuOmIiAbCJVrUIppJRq4xaKlavjiR3wH_lByFrBgU0nD3eAqE2-SkT055JLaIbyLRPq6pz=w2126-h712-s-no-gm)
*Figure: Resolved addresses in pseudocode mode*

This step ensures our analysis is accurate, paving the way for a deeper understanding of `SBAppTags`.

### Understanding `SBAppTags` with `LSApplicationRecord`

The `SBAppTags` key is central to controlling app visibility, and its implementation resides in the `LSApplicationRecord` class. Let’s examine the key method responsible for retrieving `SBAppTags`:

```c
id __cdecl -[LSApplicationRecord appTagsWithContext:tableID:unitID:unitBytes:](
        LSApplicationRecord *self,
        SEL a2,
        LSContext *a3_context,
        unsigned int a4_tableID,
        unsigned int a5_unitID,
        const LSBundleData *a6_bundleData)
{
  void *v6_dict; // x19
  __int64 v7_class; // x20
  void *v8_appTagsArray; // x0
  void *v9_returnArray; // x20

  if ( (a6_bundleData->_bundleFlags & 0x80000000000LL) == 0
    || (v6_dict = (void *)objc_claimAutoreleasedReturnValue_146(
                            -[LSBundleRecord infoDictionary](
                              self,
                              "infoDictionary",
                              a3_context,
                              *(_QWORD *)&a4_tableID,
                              *(_QWORD *)&a5_unitID)),
        v7_class = objc_opt_class_146(&OBJC_CLASS___NSArray),
        v8_appTagsArray = objc_msgSend(
                            v6_dict,
                            "objectForKey:ofClass:valuesOfClass:",
                            CFSTR("SBAppTags"),
                            v7_class,
                            objc_opt_class_146(&OBJC_CLASS___NSString)),
        v9_returnArray = (void *)objc_claimAutoreleasedReturnValue_146(v8_appTagsArray),
        objc_release(v6_dict),
        !v9_returnArray) )
  {
    v9_returnArray = __NSArray0__struct_ptr;
  }
  return objc_autoreleaseReturnValue(v9_returnArray);
}
```

This method fetches the `SBAppTags` array from the `LaunchServices` database for the app represented by the `LSApplicationRecord` instance. If no `SBAppTags` are defined, it returns an empty `NSArray`.

### Decoding the Info Dictionary

The `Info.plist` data is accessed through the `-[LSBundleRecord infoDictionary]` method, which relies on `_LSLazyPropertyList` to efficiently load metadata from the `LaunchServices` database (`lsContext->db`) rather than directly from the app’s file system.

```c
LSPropertyList *__cdecl -[LSBundleRecord infoDictionary](LSBundleRecord *self, SEL a2)
{
  return (LSPropertyList *)__LSRECORD_GETTER__<objc_object * {__strong}>(
                             self,
                             a2,
                             "infoDictionaryWithContext:tableID:unitID:unitBytes:");
}

id __cdecl -[LSBundleRecord infoDictionaryWithContext:tableID:unitID:unitBytes:](
        LSBundleRecord *self,
        SEL a2,
        LSContext *a3,
        unsigned int a4,
        unsigned int a5,
        const LSBundleBaseData *a6)
{
  return +[_LSLazyPropertyList lazyPropertyListWithContext:unit:](
           &OBJC_CLASS____LSLazyPropertyList,
           "lazyPropertyListWithContext:unit:",
           a3,
           a6->infoDictionary,
           *(_QWORD *)&a5);
}

_LSLazyPropertyList *__cdecl __noreturn +[_LSLazyPropertyList lazyPropertyListWithContext:unit:](
        id self,
        SEL selector,
        LSContext *lsContext,
        unsigned int unit)
{
  id v5; // x0
  void *v6; // x0
  __int64 v7; // x1

  if ( lsContext )
  {
    if ( unit )
    {
      v5 = _LSPlistGet((__int64)lsContext->db, *(__int64 *)&unit);
      v6 = (void *)objc_claimAutoreleasedReturnValue_146(v5);
      if ( v6 )
        +[_LSLazyPropertyList lazyPropertyListWithPropertyListData:]((__int64)self, v7, v6);
    }
  }
  return (_LSLazyPropertyList *)objc_autoreleaseReturnValue((id)objc_claimAutoreleasedReturnValue_146(
                                                                  +[_LSEmptyPropertyList sharedInstance](
                                                                    &OBJC_CLASS____LSEmptyPropertyList,
                                                                    "sharedInstance")));
}

struct _LSDatabase // sizeof=0x688
{
    unsigned __int8 superclass_opaque[8];
    __CSStore *store;
    LSSchema schema;
    FSNode *node;
    LSSessionKey sessionKey;
    OS_dispatch_queue *accessQueue;
    __int32 needsUpdate : 1;
    __int32 isForcedForXCTesting : 1;
    __int32 isForcedForRemoteUpdates : 1;
}
```

> The `_LSLazyPropertyList` is a performance optimization in iOS that loads metadata only when needed, reducing memory usage. The `lsContext->db` refers to the `LSDatabase`, a binary store containing metadata for all apps on the device, managed by the `LaunchServices` daemon (`lsd`).
{: .prompt-info}

### Exploring the LaunchServices Database with Frida

To deepen our understanding of the `LSDatabase`, let’s use Frida to inspect it at runtime. Attach Frida to the `SpringBoard` process on your jailbroken device:

```bash
$ frida -U -n SpringBoard
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

[iPhone::SpringBoard ]-> console.log(ObjC.chooseSync(ObjC.classes._LSDatabase)[0])
<LSDatabase 0x80102ce00> { userID = 501, path = '/private/var/mobile/Containers/Data/InternalDaemon/61CB26B5-A0E9-41D7-9FFE-ECD8A28370A1/Library/Caches/com.apple.LaunchServices-4035-v2.csstore' }
```

The output reveals the `LSDatabase` instance, pointing to the `com.apple.LaunchServices-4035-v2.csstore` file, a binary CoreServices Store (`csstore`) that holds serialized metadata for all registered apps.

> The `csstore` file is a proprietary binary format, not a standard plist, containing essential app metadata like bundle identifiers, executable paths, and display names. It’s managed by the `lsd` daemon to provide fast access to app information.
{: .prompt-info}

To inspect the `csstore` file, run the `lsdiagnose` command on your jailbroken device:

```bash
iPhone $ /usr/bin/lsdiagnose
Checking data integrity...
...done.
Database is seeded.
Status:                     Preferences are loaded.
Seeded System Version:      16.7.11 (20H360)
Seeded Cryptex Version:     ? ()
Seeded Model Code:          D201AP/iPhone10,4
CacheGUID:                  7683A294-6EB1-483E-BC01-0E6F049FA9EB
CacheSequenceNum:           1916
Date Initialized:           2025-07-04 23:00 (POSIX 1751641203, 𝛥 5wks 2days 15hr 22min 51sec)
Path:                       /private/var/mobile/Containers/Data/InternalDaemon/61CB26B5-A0E9-41D7-9FFE-ECD8A28370A1/Library/Caches/com.apple.LaunchServices-4035-v2.csstore
DB Object:                  <LSDatabase 0xc5f008200> { userID = 0, path = '/private/var/mobile/Containers/Data/InternalDaemon/61CB26B5-A0E9-41D7-9FFE-ECD8A28370A1/Library/Caches/com.apple.LaunchServices-4035-v2.csstore' }
DB Bundle:                  /System/Library/Frameworks/CoreServices.framework (v 839)
Store Object:               <_CSStore 0xc5e309080> { p = 0xc5f800000, gen = 257384, length = 6848512/6848176/6333120 }
Store Bundle:               /System/Library/PrivateFrameworks/CoreServicesStore.framework (v 1141.1)
0x00000000 00000000:	6264736c 02003ccb 68ed0300 b07e6800    bdsl··<·h····~h·
+++++++++++++++++++++++++++++++ Structure Sizes ++++++++++++++++++++++++++++++++
sizeof(Data):                      100 ( 100 bytes) -----
sizeof(Table):                      80 (  80 bytes) -----
sizeof(Unit):                        8 (   8 bytes) -----
sizeof(IdentifierCache):             4 (   4 bytes) -----
+++++++++++++++++++++++++++++++ Memory by Table ++++++++++++++++++++++++++++++++
<array>:                        653234 (    653 KB) 13225 units (≅49 bytes/unit)
<string>:                      1150925 (    1.2 MB) 33952 units (≅33 bytes/unit)
ActivityTypeBinding:             16336 (     16 KB) 1 units
Alias:                          149429 (    149 KB) 1680 units (≅88 bytes/unit)
BindableKeyMap:                  32656 (     33 KB) 1 units
BindingList:                     83656 (     84 KB) 3007 units (≅27 bytes/unit)
BluetoothVendorProductIDBinding:       4096 (      4 KB) 1 units
Bundle:                         132616 (    133 KB) 242 units (≅548 bytes/unit)
BundleIDBinding:                  4096 (      4 KB) 1 units
BundleNameBinding:                8176 (      8 KB) 1 units
CanonicalString:                   528 ( 528 bytes) 22 units (≅24 bytes/unit)
Claim:                           16720 (     17 KB) 209 units (≅80 bytes/unit)
Container:                         112 ( 112 bytes) 4 units (≅28 bytes/unit)
DB Header:                           0 (   Zero KB) 0 units
DeviceModelCodeBinding:          16336 (     16 KB) 1 units
ExtensionBinding:                16336 (     16 KB) 1 units
ExtensionPoint:                  10564 (     11 KB) 139 units (≅76 bytes/unit)
ExtensionPointIDBinding:          4096 (      4 KB) 1 units
HandlerPref:                         0 (   Zero KB) 0 units
LinkedParentBundleIDBinding:       4096 (      4 KB) 1 units
LocalizedString:                 61420 (     61 KB) 3071 units (≅20 bytes/unit)
MIMEBinding:                      8176 (      8 KB) 1 units
Plugin:                         123840 (    124 KB) 516 units (≅240 bytes/unit)
PluginBundleIDBinding:           16336 (     16 KB) 1 units
PluginProtocolBinding:            4096 (      4 KB) 1 units
PluginUUIDBinding:               16336 (     16 KB) 1 units
PropertyList:                  2105252 (    2.1 MB) 3273 units (≅643 bytes/unit)
Type:                           165300 (    165 KB) 1653 units (≅100 bytes/unit)
URLSchemeBinding:                 4096 (      4 KB) 1 units
UTIBinding:                      32656 (     33 KB) 1 units
++++++++++++++++++++++++++++++++ Memory Summary ++++++++++++++++++++++++++++++++
Tables:                           3036 (      3 KB) -----
Identifier caches:             1000024 (      1 MB) -----
String caches:                  416884 (    417 KB) -----
All units:                     4841516 (    4.8 MB) 61008 units (≅79 bytes/unit)
Collectable:                    515056 (    515 KB) -----
Total bytes used:              6333120 (    6.3 MB) -----

--------------------------------------------------------------------------------
bundle id:                  ClarityCamera (0x8)
class:                      kLSBundleClassApplication (0x2)
container:                  / (0x4)
mount state:                mounted
Mach-O UUIDs:               A52AAE54-A915-381E-83A7-7CA088B1A517
Device Family:              1, 2
sequenceNum:                8
dataContainer:              /private/var/mobile/Containers/Data/Application/90795FBB-99D3-4AAE-BF70-D90BA619C1EB/ (0x23c)
path:                       /Applications/ClarityCamera.app/ (0x238)
directory:                  /Applications
name:                       ClarityCamera
displayName:                Camera
teamID:                     0000000000
identifier:                 com.apple.ClarityCamera
canonical id:               com.apple.claritycamera
type:                       System
version:                    1.0 ({length = 32, bytes = 0x01000000 00000000 00000000 00000000 ... 00000000 00000000 })
versionString:              1
displayVersion:             1.0
codeInfoID:                 com.apple.ClarityCamera
mod date:                   2025-03-20 14:47 (POSIX 1742453245, 𝛥 4mths 3wks 5days 23hr 35min 33sec)
reg date:                   2025-07-04 23:00 (POSIX 1751641209, 𝛥 5wks 2days 15hr 22min 49sec)
rec mod date:               2025-07-04 23:00 (POSIX 1751641209, 𝛥 5wks 2days 15hr 22min 49sec)
bundle flags:               has-display-name  requires-iphone-os  shows-sec-prompts  is-ad-hoc-signed  is-containerized  always-available-app (0000044004001002)
more flags:                 is-secured-system-content
plist flags:                has-sbapptags  has-required-device-capabilities (0000000000002800)
icon flags:                 supports-asset-catalog (0000000000000004)
slices:                     arm64 (0000000000000080)
item flags:                 package  application  container  native-app  extension-hidden (000000000010008e)
base flags:                 apple-internal
platform:                   native
iconName:                   AppIcon
iconDict:                   1 values (2204 (0x89c))
                            {
                                CFBundlePrimaryIcon =     {
                                    CFBundleIconName = AppIcon;
                                };
                            }
executable:                 ClarityCamera
min version:                16.7 ({length = 32, bytes = 0x10000000 00000000 07000000 00000000 ... 00000000 00000000 })
min version platform:       native
execSDK ver:                16.7.3 ({length = 32, bytes = 0x10000000 00000000 07000000 00000000 ... 00000000 00000000 })
infoDictionary:             21 values (2208 (0x8a0))
                            {
                                ...
                                CFBundleVersion = 1;
                                SBAppTags =     (
                                    hidden
                                );
                                SBIconVisibilityDefaultVisible = 0;
                                UIApplicationSceneManifest =     {
                                    UIApplicationSupportsMultipleScenes = 1;
                                };
                                UIApplicationSupportsIndirectInputEvents = 1;
                                UIDeviceFamily =     (
                                    1,
                                    2
                                );
                                UILaunchScreen =     {
                                    UILaunchScreen =         {
                                    };
                                };
                                UIRequiredDeviceCapabilities =     (
                                    arm64
                                );
                                UISupportedInterfaceOrientations =     (
                                    UIInterfaceOrientationPortrait,
                                    UIInterfaceOrientationLandscapeLeft,
                                    UIInterfaceOrientationLandscapeRight
                                );
                                UISupportsClarityUI = 1;
                            }
                            ...
code signature version:     132096
entitlements:               2 values (2216 (0x8a8))
                            {
                                "com.apple.private.tcc.allow" =     (
                                    kTCCServiceCamera,
                                    kTCCServicePhotos,
                                    kTCCServicePhotosAdd
                                );
                                "com.apple.security.exception.shared-preference.read-only" =     (
                                    "com.apple.ClarityUI.Camera"
                                );
                            }
...
...
```

The `lsdiagnose` output reveals that the `ClarityCamera` app has an `SBAppTags` value of `hidden` in its `infoDictionary`, explaining why it’s not visible on the Home screen.

### Tracing the Callers of `SBAppTags`

To understand how `SBAppTags` affects visibility, we need to trace its callers. The `-[LSApplicationRecord appTagsWithContext:tableID:unitID:unitBytes:]` method is invoked by `-[LSApplicationRecord appTags]`, found in the `libobjc.A` module.

```assembly
; search result
CoreServices:__objc_methname:0000000181D8EFFB	0000002D	C	appTagsWithContext:tableID:unitID:unitBytes:
libobjc.A:__OBJC_RO:00000001832E08A7	0000002D	C	appTagsWithContext:tableID:unitID:unitBytes:
...
...
; XREF
libobjc.A:__OBJC_RO:00000001832E08A7 sel_appTagsWithContext_tableID_unitID_unitBytes_ DCB "appTagsWithContext:tableID:unitID:unitBytes:",0
libobjc.A:__OBJC_RO:00000001832E08A7                                         ; DATA XREF: -[LSApplicationRecord appTags]↑o
libobjc.A:__OBJC_RO:00000001832E08A7                                         ; -[LSApplicationRecord appTags]+4↑o ...
```

```c
NSArray *__cdecl -[LSApplicationRecord appTags](LSApplicationRecord *self, SEL a2)
{
  return (NSArray *)__LSRECORD_GETTER__<objc_object * {__strong}>(
                      self,
                      a2,
                      "appTagsWithContext:tableID:unitID:unitBytes:");
}
```

Continuing our trace, we find that `-[LSApplicationRecord appTags]` is called by `-[FBSApplicationInfo _initWithApplicationProxy:record:appIdentity:processIdentity:overrideURL:]` in the `FrontBoardServices` framework, which handles app metadata initialization.

```c
id objc_msgSend_appTags(void *a1, const char *a2, ...)
{
  return objc_msgSend_ptr(a1, "appTags");
}
```

### Analyzing FBSApplicationInfo

The `-[FBSApplicationInfo _initWithApplicationProxy:record:appIdentity:processIdentity:overrideURL:]` method populates metadata for an app, including its tags:

```c
v47_appTags = (void *)objc_claimAutoreleasedReturnValue_1(objc_msgSend(v12_bundleProxy, "appTags"));
if ( objc_msgSend(v47_appTags, "count") )
    appTags = (void *)objc_claimAutoreleasedReturnValue_1(objc_msgSend(v12_bundleProxy, "appTags"));
else
    appTags = 0LL;

objc_release(v47_appTags);
applicationType = (void *)objc_claimAutoreleasedReturnValue_1(objc_msgSend(v12_bundleProxy, "applicationType"));
isApplicationTypeHidden = (unsigned int)objc_msgSend(applicationType, "isEqual:", CFSTR("Hidden"));
objc_release(applicationType);
v104 = v15_processIdentity;

if ( isApplicationTypeHidden )
{
    if ( appTags )
    v51_appTagsWithHiddenItem = (_UNKNOWN **)objc_claimAutoreleasedReturnValue_1(objc_msgSend(appTags, "arrayByAddingObject:", CFSTR("hidden")));
    else
    v51_appTagsWithHiddenItem = &off_1D7179E80;// _OBJC_CLASS_$_NSConstantArray
    objc_release(appTags);
    appTags = v51_appTagsWithHiddenItem;
}
j__objc_storeStrong_1((id *)&v21_fbsBundleInfo->_tags, appTags);
```

This code retrieves the app’s tags, checks if the app type is `"Hidden"`, and ensures a `"hidden"` tag is included, either by appending it or creating a new tag list. The tags are stored in the `_tags` property of the `FBSApplicationInfo` object.

![FBSApplicationInfo tags](https://lh3.googleusercontent.com/pw/AP1GczMEO5uSt_liQpKlhC56u3nyFbSq8Z1MtDASFzFERRjLp-z2jaiJj3Ny97Q8WGMNY50BxWLAX3UrB86oSXA1f83_9QpPu3QbDXJLqF_VQPKkX0QwmrpHa5ZtlhglRLdER_m40QGdZgc02_4daWJoczZM=w2126-h948-s-no-gm)
*Figure: FBSApplicationInfo_tags*

### Exploring the Tags Getter

The `-[FBSApplicationInfo tags]` method serves as the getter for the `_tags` property. Searching for `$tags` in IDA’s `Functions` tab reveals multiple synthetic thunk functions created by IDA for readability:

```assembly
_objc_msgSend$tagSpecification	  CoreServices:__objc_stubs
_objc_msgSend$tags	              FrontBoardServices:__objc_stubs
_objc_msgSend$tags_0	          FrontBoard:__objc_stubs
_objc_msgSend$tags_1	          SpringBoardHome:__objc_stubs
_objc_msgSend$tagsForIcon_	      SpringBoardHome:__objc_stubs
_objc_msgSend$tags_2	          SpringBoard:__objc_stubs
```

These `_objc_msgSend$tags_X` functions are not actual binary functions but IDA’s way of labeling calls to `objc_msgSend` with specific selectors. The suffixes (`_1`, `_2`) ensure unique symbol names. By filtering for `tags]` in the `Functions` tab, we can identify the relevant methods

```assembly
-[FBSApplicationInfo tags]	      FrontBoardServices:__text
-[FBSDisplayConfiguration tags]	  FrontBoardServices:__text
-[SBLeafIcon tags]	              SpringBoardHome:__text
-[SBIcon tags]	                  SpringBoardHome:__text
-[SBHSimpleApplication tags]	  SpringBoardHome:__text
-[SBApplication tags]	          SpringBoard:__text
```

After tracing the XREFs, we confirm that `_objc_msgSend$tags_1` and `_objc_msgSend$tags_2` are the correct calls to `-[FBSApplicationInfo tags]`.

![_objc_msgSend$tags_1 XREF](https://lh3.googleusercontent.com/pw/AP1GczPyXM4Ywq8AhPkNZm7-iLvazw4_PpXYu1y4s0vtLXPRALjcfZJ5izTtqSFxGi2kSKsLpRus3PitelUzFsfUaqF7u_c-Sc3G75vewnDbj4yMAXqcviHU89DkD-B5rTnEJCf9RyJYH6Ku5srpBWeOGGk8=w2126-h784-s-no-gm)
*Figure: _objc_msgSend$tags_1 XREF*

![_objc_msgSend$tags_2 XREF](https://lh3.googleusercontent.com/pw/AP1GczPb8gLrDERwo3W_EU_D0iQBoHoIANSgc7GMNrJ946Nh1VkKFuLcAEv_fbnZRI-elS2hWwhT9SipAyb-tQhxnDpHvLB1QzmPw3GY9pokUmxzDT641eHa7iH4oHqHtvb2Pu_lqJIsRtYnLS0sD5ANEDDH=w2126-h786-s-no-gm)
*Figure: _objc_msgSend$tags_2 XREF*


Further exploration of the cross-references (XREFs) reveals that `SBAppTags` plays a critical role in the `SBIconModel` class, specifically within methods like `-[SBIconModel shouldAvoidCreatingIconForApplication:]` and `-[SBHIconModel isIconVisible:]`. These methods act as supporting functions for `SBIconController`, which is tasked with rendering all application icons on the iOS Home screen, ensuring that only the appropriate icons are displayed based on the app’s metadata.

For those interested in delving deeper into iOS Home screen mechanics, several methods are worth investigating. These include `-[SBIconVisibilityService _visibleIdentifiersChanged:]`, which handles changes in visible app identifiers, `-[SBIconController _mutateIconListsForInstalledAppsDidChangeWithController:added:modified:removed:]`, which manages updates to installed apps, `-[SBApplicationRestrictionController _postRestrictionStateToObservers:]`, which notifies observers of restriction changes, and `-[SBHIconManager updateVisibleIconsToShowLeafIcons:hideLeafIcons:forceRelayout:]`, which controls the visibility and layout of icons. These methods offer valuable insights into how iOS manages app presentation.

```c
bool __cdecl -[SBIconModel shouldAvoidCreatingIconForApplication:](SBIconModel *self, SEL a2, id a3)
{
  SBApplication *sbApplication; // x19
  unsigned __int8 v5; // w22
  __int64 v6; // x0
  void *v7; // x23
  SBApplicationInfo *sbApplicationInfo; // x20
  unsigned __int8 has_hiddenTag; // w21
  void *tags; // x21
  unsigned __int64 isVisibilityOverride; // x22
  objc_super v13; // [xsp+0h] [xbp-40h] BYREF

  sbApplication = (SBApplication *)objc_retain(a3);
  v13.receiver = self;
  v13.super_class = (Class)&OBJC_CLASS___SBIconModel;
  v5 = -[SBHIconModel shouldAvoidCreatingIconForApplication:](
         &v13,
         "shouldAvoidCreatingIconForApplication:",
         sbApplication);
  v6 = objc_opt_self_129(&OBJC_CLASS___SBApplication);
  v7 = (void *)objc_claimAutoreleasedReturnValue_248(v6);
  if ( (objc_opt_isKindOfClass_232(sbApplication, v7) & 1) != 0 )
    sbApplicationInfo = (SBApplicationInfo *)objc_claimAutoreleasedReturnValue_248(-[SBApplication info](sbApplication, "info"));
  else
    sbApplicationInfo = 0LL;
  objc_release(v7);
  if ( (v5 & 1) != 0 )
    goto LABEL_5;
  if ( self->_createsIconsForInternalApps )
    goto CHECK_HIDDEN_TAG_LABEL;
  tags = (void *)objc_claimAutoreleasedReturnValue_248(-[SBApplicationInfo tags](sbApplicationInfo, "tags"));
  if ( ((unsigned int)objc_msgSend(tags, "containsObject:", CFSTR("SBInternalAppTag")) & 1) == 0 )
  {
    objc_release(tags);
    goto CHECK_HIDDEN_TAG_LABEL;
  }
  isVisibilityOverride = -[SBApplicationInfo visibilityOverride](sbApplicationInfo, "visibilityOverride");
  objc_release(tags);
  if ( isVisibilityOverride )
  {
CHECK_HIDDEN_TAG_LABEL:
    has_hiddenTag = -[SBApplicationInfo hasHiddenTag](sbApplicationInfo, "hasHiddenTag");
    goto RETURN_LABEL;
  }
LABEL_5:
  has_hiddenTag = 1;
RETURN_LABEL:
  objc_release(sbApplicationInfo);
  objc_release(sbApplication);
  return has_hiddenTag;
}
```

> The `SBIconController` serves as the central component within `SpringBoard`, orchestrating the layout and presentation of the Home screen. It oversees the arrangement of app icons, folders, and widgets, ensuring a cohesive user interface. By utilizing a collection view-like structure through the `SBIconListView` class, it organizes icons into individual pages or grids. Additionally, `SBIconController` manages user interactions—such as tapping, dragging, or rearranging icons—and collaborates with other system components to update icon states, including badges, notifications, and folder contents, maintaining a dynamic and responsive Home screen experience.
{: .prompt-info}

To provide context before discussing any modification-related topics, it’s helpful to understand the Home screen’s view hierarchy. This structure outlines the relationship between key components:  
```bash
SBHomeScreenView (SBHomeScreenViewController)
    |- SBIconContentView (SBIconController)
        |- SBFolderContainerView (SBRootFolderController)
            |- SBRootFolderView
                |- SBIconScrollView
                    |- SBIconListView
                        |- SBIconView
                            |- SBIconImageView or SBFolderIconImageView
```                            

## Using Frida to Explore and Manipulate SpringBoard

Frida, a powerful dynamic instrumentation toolkit, allows us to interact with `SpringBoard` at runtime, inspect its behavior, and modify app visibility. This step is ideal for us, as Frida’s JavaScript-based interface simplifies complex debugging tasks. Let’s set up Frida to explore `SpringBoard` and reveal hidden apps.

### Setting Up Frida

First, ensure Frida is installed on your jailbroken device (available via Cydia or Sileo), on your host machine (`pip install frida-tools`) and connect your device via USB.

Next, create a helper script (`frida-helper.js`) to simplify Objective-C interactions with Frida, as provided in the original document:

```javascript
// print object
function po(obj) {
    console.log(obj)
}

// wrapper for ObjC.chooseSync
function cs(className) {
    const instances = ObjC.chooseSync(ObjC.classes[className])
    const totalInstances = instances.length
    if (totalInstances == 1) {
        const instance = instances[0]        
        return instance
    }

    if (totalInstances == 0) {
        console.log("No instances found!!!")
    } else {
        console.log(`${totalInstances} instances found!!!`)        
    }

    return instances
}
```

Run Frida against `SpringBoard` to interact with its runtime environment:

```bash
$ frida -U -n SpringBoard -l frida-helper.js
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
   . . . .   Connected to iPhone (id=63xxx)
[iPhone::SpringBoard ]->
```

### Exploring `SBIconController`

Using the helper script, locate the `SBIconController` instance, which manages app icons on the Home screen:

```bash
# Find living SBIconController instance(s)
[iPhone::SpringBoard ]-> var ic = cs('SBIconController')

# Examine class name
[iPhone::SpringBoard ]-> ic.$className
"SBIconController"

# Access class ivars (refer Local types for the ivar name)
[iPhone::SpringBoard ]-> var im = ic.$ivars._iconModel

[iPhone::SpringBoard ]-> po(ic.$methods.length)
3696

# Invoke ObjC method/selector
[iPhone::SpringBoard ]-> var sbApplications = ic.allApplicationsForIconModel_(im)

# Another way to invoke ObjC method/selector
[iPhone::SpringBoard ]-> sbApplications = ic['- allApplicationsForIconModel:'](im)

# There are 184 apps included hidden system apps
[iPhone::SpringBoard ]-> po(sbApplications.count())
184

# print result with extra debug description
[iPhone::SpringBoard ]-> po(sbApplications)
(
    "<SBApplication: 0x283d03a20; com.apple.BarcodeScanner> {\n}",
    "<SBApplication: 0x283d02a30; com.apple.Magnifier> {\n}",
    "<SBApplication: 0x283d19950; com.tigisoftware.Filza> {\n}",
    "<SBApplication: 0x283d07b10; com.apple.SleepLockScreen> {\n}",
    "<SBApplication: 0x283d1dd10; com.apple.GameCenterRemoteAlert> {\n}",
    "<SBApplication: 0x283d1b1b0; com.apple.FaceTimeLinkTrampoline> {\n}",
    "<SBApplication: 0x283d02b20; com.apple.shortcuts> {\n}",
    "<SBApplication: 0x283d1d4a0; com.apple.PeopleViewService> {\n}",
    "<SBApplication: 0x283d1de00; com.apple.mobilenotes> {\n}",
    ...
    ...
    "<SBApplication: 0x283d07de0; com.apple.AskPermissionUI> {\n}",
    "<SBApplication: 0x283d181e0; com.apple.NewDeviceOutreachApp> {\n}",
    "<SBApplication: 0x283d01c20; com.apple.ProximityReaderUIService> {\n}",
    "<SBApplication: 0x283d1d2c0; com.apple.SharingViewService> {\n}",
    "<SBApplication: 0x283d1c690; com.apple.DataActivation> {\n}",
    "<SBApplication: 0x283d1aa30; com.apple.Music> {\n}",
    "<SBApplication: 0x283d1a0d0; com.apple.Home.HomeUIService> {\n}",
    "<SBApplication: 0x283d060d0; com.apple.TVRemoteUIService> {\n}",
    "<SBApplication: 0x283d182d0; com.apple.Batteries> {\n}",
    "<SBApplication: 0x283d06fd0; com.apple.HealthPrivacyService> {\n}",
    "<SBApplication: 0x283d070c0; com.apple.findmy> {\n}"
)

# Print out apps has tags (system apps)
[iPhone::SpringBoard ]-> sbApplications.forEach(sbApp => {if (sbApp.$ivars._appInfo.tags() != null) console.log(cc.displayName().UTF8String(), sbApp.$ivars._appInfo.tags())})
PaperBoard (
    hidden,
    SBNonDefaultSystemAppTag
)
DemoApp (
    hidden,
    SBNonDefaultSystemAppTag
)
FaceTime (
    "any-telephony",
    venice
)
Screen Time (
    hidden,
    SBNonDefaultSystemAppTag
)
Music Recognition (
    hidden,
    SBNonDefaultSystemAppTag
)
FontInstallViewService (
    hidden,
    SBNonDefaultSystemAppTag
)
AppSSOUIService (
    hidden,
    SBNonDefaultSystemAppTag
)
FMDMagSafeSetupRemoteUI (
    hidden,
    SBNonDefaultSystemAppTag
)
...
Siri (
    SBInternalAppTag,
    SBNonDefaultSystemAppTag
)
...
```

This output confirms that apps like `PaperBoard` and `Screen Time` are marked with the `hidden` tag, explaining their absence from the Home screen. To make these apps visible, we can use Frida to manipulate `SpringBoard`’s behavior, but for a more permanent solution, let’s develop a tweak.

### Experimenting with Home Screen Services

Frida also allows us to interact with `SBSHomeScreenService`, a class that provides methods to manipulate the Home screen layout. Let’s explore some fun and educational experiments:

```bash
[iPhone::SpringBoard ] var hss = cs('SBSHomeScreenService')

# animate delete and restore apps on the home screen
[iPhone::SpringBoard ]-> hss.runRemoveAndRestoreIconTest()

# animate downloading apps on the home screen
[iPhone::SpringBoard ]-> hss.runDownloadingIconTest()

[iPhone::SpringBoard ]-> hss.removeAllWidgets()
[iPhone::SpringBoard ]-> hss.addEmptyPage()

# remove all apps from screen and float dock
[iPhone::SpringBoard ]-> hss.ignoreAllApps()

# reset and display apps
[iPhone::SpringBoard ]-> hss.resetHomeScreenLayoutWithCompletion_(null)
```

These commands trigger animations (e.g., downloading icons), remove widgets, add empty pages, or reset the Home screen layout. The figure below captures the dynamic effects of these manipulations:

![Fun with Frida](https://lh3.googleusercontent.com/pw/AP1GczMbVPPm4Lhqym_U2Z3uketiT2kzb1v4cqhrzyKNtEs87xYHHVDtyCXHje8531_e55_K2LT9U0kKTK2QQGsSWIpimzJN18bHc-g98aQm8FjekskJq-EYsUqiiWBKDetcUTH0OlenGyIeXcgRDRwSxgUi=w1024-h604-s-no-gm)
*Figure: Fun with Frida*

These experiments not only highlight Frida’s power but also deepen your understanding of `SpringBoard`’s role in managing the user interface, paving the way for advanced tweaks.

## Developing a Tweak with Theos

For a persistent solution, we’ll create a `theos` tweak to override the `hidden` tag and display these apps on the Home screen. Additionally, we’ll showcase a fun example with the `iSnake` tweak, which transforms app icons into a playable Snake game.

Install `theos` on your host machine following the official guide ([theos.dev](https://theos.dev)). Create a new tweak project:

```bash
$ $THEOS/bin/nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  ...
  [13.] iphone/tweak
  [14.] iphone/tweak_swift
  ...
Choose a Template (required): 13
Project Name (required): ShowSystemHiddenApps
Package Name [com.yourcompany.showsystemhiddenapps]: com.rta.showsystemhiddenapps
Author/Maintainer Name: RTA
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.apple.springboard
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: SpringBoard
Instantiating iphone/tweak in ShowSystemHiddenApps/...
Done.
```

Navigate to the project directory and edit `Tweak.x` to hook into `SpringBoard`’s logic. Here’s a sample tweak to bypass the `hidden` tag:

```objc
// Tweak.x
#import <Foundation/Foundation.h>

%hook LSApplicationRecord

- (NSArray *)appTags {
    NSArray *result = %orig;

    if ([result isKindOfClass:[NSArray class]]) {
        NSMutableArray *mutableResult = [result mutableCopy];
        if ([mutableResult containsObject:@"hidden"]) {
            NSLog(@"[Tweak] Removing 'hidden' tag in appTags for %@", self);
            [mutableResult removeObject:@"hidden"];
            result = [mutableResult copy];
        }
    }

    return result;
}

%end

%hook SBApplicationInfo

// this is used in -[SBIconModel shouldAvoidCreatingIconForApplication:] logic
- (NSInteger)visibilityOverride {    
  return 1;
}

%end
```

This tweak hooks the `tags` method of `SBApplicationInfo`, checks for the `hidden` tag, and removes it, allowing `SpringBoard` to display the app. 

Then modify `Makefile` file accordingly:
```bash
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = SpringBoard
THEOS_DEVICE_IP=localhost
THEOS_DEVICE_PORT=2222

# Enable this so the can see NSLog() content in the Console.app
# Disable this for release build
DEBUG=1

# target rootful only
ifneq ($(THEOS_PACKAGE_SCHEME),rootless)
TARGET := iphone:clang:18.5:7.0
ARCHS := $(ARCHS)
PREFIX = /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/
endif

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = ShowSystemHiddenApps

$(TWEAK_NAME)_FILES = Tweak.x
$(TWEAK_NAME)_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/tweak.mk
```

Compile and install the tweak:

```bash
$ cd ShowSystemHiddenApps
$ make package install
==> Notice: Build may be slow as Theos isn’t using all available CPU cores on this computer. Consider upgrading GNU Make: https://theos.dev/docs/parallel-building
==> Warning: Building for iOS 7.0, but the current toolchain can’t produce arm64e binaries for iOS earlier than 14.0. More information: https://theos.dev/docs/arm64e-deployment
> Making all for tweak ShowSystemHiddenApps…
==> Preprocessing Tweak.x…
==> Compiling Tweak.x (arm64)…
==> Linking tweak ShowSystemHiddenApps (arm64)…
ld: warning: -multiply_defined is obsolete
==> Generating debug symbols for ShowSystemHiddenApps…
==> Preprocessing Tweak.x…
==> Compiling Tweak.x (arm64e)…
==> Linking tweak ShowSystemHiddenApps (arm64e)…
ld: warning: -multiply_defined is obsolete
==> Generating debug symbols for ShowSystemHiddenApps…
==> Merging tweak ShowSystemHiddenApps…
==> Signing ShowSystemHiddenApps…
> Making stage for tweak ShowSystemHiddenApps…
dm.pl: building package `com.rta.showsystemhiddenapps:iphoneos-arm' in `./packages/com.rta.showsystemhiddenapps_0.0.1-1+debug_iphoneos-arm.deb'
==> Installing…
root@localhost's password:
(Reading database ... 6502 files and directories currently installed.)
Preparing to unpack /tmp/_theos_install.deb ...
Unpacking com.rta.showsystemhiddenapps (0.0.1-1+debug) over (0.0.1-1+debug) ...
Setting up com.rta.showsystemhiddenapps (0.0.1-1+debug) ...
Processing triggers for org.coolstar.sileo (2.5.1-1) ...
Not running in Sileo. Trigger UICache
==> Unloading SpringBoard…
root@localhost's password:
$ 
```

Once the installation is complete, your device will automatically undergo a respring. As a result, previously hidden system apps should now be visible on the Home screen.

![Hidden system app shown](https://lh3.googleusercontent.com/pw/AP1GczNE3y0c_KMRtCMLnguJ47xPnPxFnhYuZApd-6W6BqOGDPBXvTi8eBv-hDnl1jGLb_MNCwid0PfYuTpt4k4kUS875lo5bu5s2K9dGWFkUT--DoJSOUsJe1IQUyMoqeL1BtIWExWvfAeuuljnE_TPYovo=w982-h1624-s-no-gm)
*Figure: Hidden system app shown*

You can locate these log entries in the `Console.app` on a Mac, which will also reflect the removal of the `hidden` tag for system apps.
```bash
default	12:36:00.333869+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.PosterBoard, URL: file:///Applications/PosterBoard.app/ }
default	12:36:00.342245+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.PosterBoard, URL: file:///Applications/PosterBoard.app/ }
default	12:36:00.842434+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.EventViewService, URL: file:///Applications/EventViewService.app/ }
default	12:36:00.850593+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.AccountAuthenticationDialog, URL: file:///Applications/AccountAuthenticationDialog.app/ }
default	12:36:00.924542+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.CloudKit.ShareBear, URL: file:///Applications/iCloud.app/ }
default	12:36:00.929867+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.SystemPaperViewService, URL: file:///Applications/SystemPaperViewService.app/ }
default	12:36:00.933235+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.DataActivation, URL: file:///Applications/DataActivation.app/ }
default	12:36:00.939069+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.family, URL: file:///Applications/Family.app/ }
default	12:36:00.942881+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.TrustMe, URL: file:///Applications/TrustMe.app/ }
default	12:36:01.019471+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.ios.StoreKitUIService, URL: file:///Applications/StoreKitUIService.app/ }
default	12:36:01.027533+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.PassbookUISceneService, URL: file:///Applications/PassbookUISceneService.app/ }
default	12:36:01.038929+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.SharedWebCredentialViewService, URL: file:///Applications/SharedWebCredentialViewService.app/ }
default	12:36:01.110939+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.TVAccessViewService, URL: file:///Applications/TVAccessViewService.app/ }
default	12:36:01.117896+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.ClipViewService, URL: file:///Applications/ClipViewService.app/ }
default	12:36:01.124125+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.HealthENBuddy, URL: file:///Applications/HealthENBuddy.app/ }
default	12:36:01.127914+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.RemotePassUIService, URL: file:///Applications/RemotePaymentPassActionsService.app/ }
default	12:36:01.205996+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.PreBoard, URL: file:///Applications/PreBoard.app/ }
default	12:36:01.211662+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.AAUIViewService, URL: file:///Applications/AAUIViewService.app/ }
default	12:36:01.212016+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.ClarityPhotos, URL: file:///Applications/ClarityPhotos.app/ }
default	12:36:01.214443+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.PDUIApp, URL: file:///Applications/PDUIApp.app/ }
default	12:36:01.215337+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.EventViewService, URL: file:///Applications/EventViewService.app/ }
default	12:36:01.215992+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.ctkui, URL: file:///Applications/CTKUIService.app/ }
default	12:36:01.217426+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.ScreenTimeWidgetApplication, URL: file:///Applications/Screen%20Time.app/ }
default	12:36:01.217513+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.AccountAuthenticationDialog, URL: file:///Applications/AccountAuthenticationDialog.app/ }
default	12:36:01.219208+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.FontInstallViewService, URL: file:///Applications/FontInstallViewService.app/ }
default	12:36:01.219392+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.CBRemoteSetup, URL: file:///Applications/CheckerBoardRemoteSetup.app/ }
default	12:36:01.219615+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.musicrecognition, URL: file:///Applications/MusicRecognition.app/ }
default	12:36:01.220328+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.HearingApp, URL: file:///Applications/HearingApp.app/ }
default	12:36:01.220413+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.TVSetupUIService, URL: file:///Applications/TVSetupUIService.app/ }
default	12:36:01.220935+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.AppSSOUIService, URL: file:///Applications/AppSSOUIService.app/ }
default	12:36:01.221718+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.FMDMagSafeSetupRemoteUI, URL: file:///Applications/FMDMagSafeSetupRemoteUI.app/ }
default	12:36:01.222018+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.purplebuddy, URL: file:///Applications/Setup.app/ }
default	12:36:01.222796+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.Photos.PhotosUIService, URL: file:///Applications/PhotosUIService.app/ }
default	12:36:01.222994+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.Diagnostics, URL: file:///Applications/Diagnostics.app/ }
default	12:36:01.223035+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.CarPlayWallpaper, URL: file:///Applications/CarPlayWallpaper.app/ }
default	12:36:01.356512+0800	SpringBoard	[Tweak] Removing 'hidden' tag in appTags for { bundleID: com.apple.InputUI, URL: file:///Applications/InputUI.app/ }
...
```

## Fun with iSnake Tweak

For a creative twist, I developed the `iSnake` tweak a few years ago, which transforms Home screen app icons into a playable Snake game. Add my repository to Cydia or Sileo to try it out: [https://ReverseThatApp.github.io/cydia](https://ReverseThatApp.github.io/cydia)

This tweak, compatible with rootful jailbreaks (note: it may not work on rootless setups), activates with a double-tap on the Home screen, turning your app icons into a nostalgic Snake game. The figure below showcases this playful modification:

![Fun with iSnake tweak](https://lh3.googleusercontent.com/pw/AP1GczNA5hrFfKKBAuIQ09fniC5ZClDhRWODkX1lPwmmkllXYD3j5AiyfZtu0Tlrcx9ZO8kMoKvt5_TsKL_3T1EYQYxcbP2ia62NX5APel1QaMO5QfIiv8OBmeGrdpQDi-0buxfWvHX2fcXUMwL66QrZ9m_p=w720-h1280-s-no-gm)
*Figure: Playing Snake with App Icons Using iSnake Tweak*

The `iSnake` tweak demonstrates the creative potential of `theos`, combining technical manipulation with user engagement, making your Home screen an interactive playground.

## Conclusion

This exploration into iOS’s hidden apps has unveiled the intricate workings of `SpringBoard` and its `SBAppTags` mechanism, revealing how Apple controls app visibility. By leveraging tools like Frida and `theos`, we’ve not only made hidden system apps visible but also created a playful `iSnake` tweak that transforms the Home screen into a game. These techniques open doors to deeper iOS reverse engineering, offering insights into system internals while fostering creative experimentation. However, always conduct such research ethically on test devices, respecting Apple’s ADPLA to avoid legal or security risks. With these skills, you’re well-equipped to dive further into iOS’s hidden depths—happy tinkering!

## Further Reading

- [Frida Documentation](https://frida.re/docs/home/)
- [Theos Documentation](https://theos.dev)
- [ipsw Tool](https://github.com/blacktop/ipsw)
- [iOS Security Guides](https://www.apple.com/support/security/)
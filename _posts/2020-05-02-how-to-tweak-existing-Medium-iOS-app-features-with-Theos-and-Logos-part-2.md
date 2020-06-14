---
layout: post
title: How to tweak existing Medium iOS app features with Theos & Logos - part 2
---

In this post we will continue to enhance **Medium** tweak. We will learn how to create preference bundle and hook in to Settings.app to configure default claps. If you have not read [part 1]({{ site.baseurl }}/how-to-tweak-existing-Medium-iOS-app-features-with-Theos-and-Logos-part-1){:target="_blank"} yet, I suggest to have a look first before continuing.
[![Medium Tweak Preference]({{ site.baseurl }}/images/20200502/final-medium-tweak.png)]({{ site.baseurl }}/images/20200502/final-medium-tweak.png){:target="_blank"} <br/>**Figure 1: Medium Tweak Preference**<br/><br/>

## Disclaimer
This post is for educational purposes only, please use it at your discretion and contact the app's author if you find issues.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [FLEX Loader Cydia package](http://cydia.saurik.com/package/com.joeyio.flexloader/){:target="_blank"}
- [Theos development setup](https://github.com/theos/theos/wiki){:target="_blank"}
- [Install Medium iOS app](https://apps.apple.com/us/app/medium/id828256236){:target="_blank"}
- [Hopper Disassembler](https://www.hopperapp.com/download.html){:target="_blank"}

## Overview
We will fix problems that encountered in previous post. Let me list down things we will cover today:
- What are `clapButtonPressed:` and `clapButtonReleased:` doing?
- Are there any alternative methods to hook?
- How to create preference bundle that allows changing number of claps instead of hard coding.
- Spoil alert another thing we can tweak Medium

I bet it will be more hands on than previous post, please get our hands dirty!! 👌👌

## Clapping function behind the scene
### Static anlysis using Hopper Disassembler
As promised in previous post, we will reveal what's going on in `clapButtonPressed:` and `clapButtonReleased:` methods. We need to do some analysis on the Medium binary file.

With the help of [Frida iOS Dump](https://github.com/AloneMonkey/frida-ios-dump){:target="_blank"} or [CrackerXI](https://forum.iphonecake.com/index.php?/topic/363020-crackerxi-gui-app-decryption-tool-for-ios-11-12-13/){:target="_blank"}, we can easily pull out **.ipa** file of Medium app on a jailbroken device, unzip **.ipa** and navigate to `Payload/hangtag.app` folder.

Drag and drop `hangtag` binary (MachO) file into Hopper Disassembler and wait for a while for it to disassemble. When it finishes, in the left panel make sure `Labels` tab is selected, let search for `clapButtonPressed` and click on `-[ClapButton clapButtonPressed:]` result you will be navigated to method implementation on the right, assembly instructions again!!! But dont worry, this time we don't need to read every instruction, we only need to understand what method is idoing in general. To do that, switch on `Pseudo-code mode` tab, you will be impressed how great it is:
```assembly
/* @class ClapButton */
-(void)clapButtonPressed:(void *)arg2 {
    [self prepareFeedbackGenerator];
    [self->_longPressClapController startTimer];
    return;
}
```

As you can see, it invoked timer when button is pressed or tapped (`[self->_longPressClapController startTimer]`). Just hover your mouse over `startTimer` and double click on it, it should navigate you to method implementation of this selector. In this case, `startTimer` selector is referenced by multiple classes, so let select `-[LongPressClapController startTimer]` option when popup appears to proceed, and this is implementation:
```assembly
/* @class LongPressClapController */
-(void)startTimer {
    [self tryToPerformClap]; // 1. [self tryToPerformClap
    [*(self + 0x8) invalidate]; // 2. [_clapTimer invalidate]
    r22 = [[WeakProxy proxyWithTarget:self] retain];
    r0 = [NSTimer scheduledTimerWithTimeInterval:r22 
    		target:@selector(tryToPerformClap) 
    		selector:0x0 
    		userInfo:0x1 
    		repeats:r6]; // 3.
    r0 = [r0 retain];
    r8 = *(self + 0x8);
    *(self + 0x8) = r0; // 4. _clapTimer = r0
    [r8 release];
    [r22 release];
    return;
}
```

I put some inline comments, let focus on that and ignore the rest:
1. It invoke method `tryToPerformClap`, by the name we can guest it perform a clap, so it should be count as 1 clap
2. It tries to invoke method `invalidate` from unknown object at address *(self + 0x8). This is common syntax to access ivars, in this case it's _clapTimer ivar (NSTimer). From apple document, invoke invalidate will stop the timer from ever firing again and request its removal from its run loop.
3. It create new NSTimer instance, but it seems decompiler is taking wrong arguments, i.e. r22 suppose to be number but it's `WeakProxy`... We have no choice, and we will find out soon.
4. Store new timer instance to _clapTimer

Switch to assembly mode and focus on these instructions:
```assembly
...
ldr    x1, [x8, #0xce0] ; @selector(scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:)
adrp   x8, #0x10060f000
ldr    d0, [x8, #0xae8] ; 0x10060fae8, timerInterval = 0.2
mov    x0, x21      ; _OBJC_CLASS_$_NSTimer
mov    x2, x22      ; target = self = LongPressClapController
mov    x3, x20      ; "tryToPerformClap",@selector(tryToPerformClap)
movz   x4, #0x0     ; userInfo
movz   w5, #0x1     ; repeats, argument "instance" for method imp___stubs__objc_msgSend
bl     imp___stubs__objc_msgSend ; objc_msgSend
...
```

They are instructions of NSTimer creation and my inline comments for each one. As you can see register `d0` holding address `0x10060fae8`, double click on this address it holding value `0.2` (`000000010060fae8  dq  0.2`), so it should be timer interval (200 milliseconds). All can be rewritten like this: 
```assembly
r0 = [NSTimer scheduledTimerWithTimeInterval:0.2 
    		target:self 
    		selector:@selector(tryToPerformClap) 
    		userInfo:nil 
    		repeats:YES]; // 3.
```

Base on this, we know that timer will be fired every 200 milliseconds to invoke `[self tryToPerformClap]`

Check references to `clapButtonPressed` and `clapButtonReleased:`, it's showing this:
```assembly
/* @class ClapButton */
-(void)addActions {
    [self addTarget:self 
    	action:@selector(clapButtonPressed:) 
    	forControlEvents:0x1];
    [self addTarget:self 
    	action:@selector(clapButtonReleased:) 
    	forControlEvents:0x1c0];
    return;
}
```
`ClapButton` is subclass of `UIButton` so it will inherit `addTarget:action:forControlEvents:` method. This is the place to register actions for events of type `UIControl.Event`. Let figure it out what are events (`0x1` and `0x1c0` are in hexadecimal) of each action .

Let have a look inside `UIControl.Event` declaration (I stripped `public static` modifiers for short) and put inline comments for raw values of each event in decimal and binary.
```objective-c
extension UIControl {
    
    public struct Event : OptionSet {

        public init(rawValue: UInt)
        
        var touchDown: UIControl.Event { get }	       // 1   = 0000 0000 0001
        var touchDownRepeat: UIControl.Event { get }   // 2   = 0000 0000 0010
        var touchDragInside: UIControl.Event { get }   // 4   = 0000 0000 0100
        var touchDragOutside: UIControl.Event { get }  // 8   = 0000 0000 1000
        var touchDragEnter: UIControl.Event { get }    // 16  = 0000 0001 0000
        var touchDragExit: UIControl.Event { get }     // 32  = 0000 0010 0000
        var touchUpInside: UIControl.Event { get }     // 64  = 0000 0100 0000
        var touchUpOutside: UIControl.Event { get }    // 128 = 0000 1000 0000
        var touchCancel: UIControl.Event { get }       // 256 = 0001 0000 0000
        ...
}
```
As you can see the raw values pattern, they are all bitmask constants. With this kind of bitmask represent, multiple events can be represented in one number. Let examine it!!

The control event registered for `clapButtonPressed:` is easy to guess `0x1 = 1 = .touchDown`. But for `clapButtonReleased:` it is a bit tricky. `0x1c0 = 448` does not match any defined events, is Hopper Disassembler decompiler wrong? Convert `0x1c0` to binary will be `0001 1100 0000`. There are 3 bits set, check with above constants it will be this combination: `0001 0000 0000 | 0000 1000 0000 | 0000 0100 0000` or human-readable would be `.touchCancel | .touchUpOutside | .touchUpInside`, so it's revealed:
```assembly
/* @class ClapButton */
-(void)addActions {
    [self addTarget:self 
    	action:@selector(clapButtonPressed:) 
    	forControlEvents:.touchDown];
    [self addTarget:self 
    	action:@selector(clapButtonReleased:) 
    	forControlEvents:.touchCancel | .touchUpOutside | .touchUpInside];
    return;
}
```

Let summarize where we are:
```bashscript
-[ClapButton clapButtonPressed:]
    | -[LongPressClapController startTimer]
        | -[LongPressClapController tryToPerformClap]	

```

I bet we can hook into `-[LongPressClapController tryToPerformClap]` method and handle clapping stuff. You can delve into `tryToPerformClap` to reverse more thing, I will leave it to you (spoil alert: look for method that allows to hook and bypass max 50 claps, it will be only 2 or 3 levels deeper from `tryToPerformClap`, below is the example 😊)

![Hook to bypass max 50 claps per post](https://thumbs.gfycat.com/ReasonableAmusingHomalocephale-size_restricted.gif)<br/>**Figure 4: Hook to bypass max 50 claps per post (client-side working only)**<br/><br/>

### Alternative method to hook
We found out that `-[LongPressClapController tryToPerformClap]` is possible to hook for clapping, let comment out hooking methods of `ClapButton` class and add below new code:
```bashscript
%hook LongPressClapController

int numberOfClaps = 4;

-(void)tryToPerformClap {	
	%log; // log method name and argument, you will find this log in `Console` app
	for (int i = 0; i < numberOfClaps; i++) {
	    %orig; // invoke original method	    
	}
}

%end
```

Compile and install the tweak again, you will you it will work as expected.

## Preference bundle
The only thing we feel not clean is that we are hard coding `numberOfClaps` in the tweak source code, the end users have no option to change it unless they have source code. Theos has an template that allow you to create preference bundle to hook into `Settings.app` and add new setting item as you want. Our plan is using this preference bundle template to create new `Medium Tweak` item with a slider allows users to set number of claps as they want. How do you feel? 🤯🤯

### Create Preference bundle project
From the root folder of your tweak, run Theos command to create new preference bundle template as below:
```bashscript
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
Choose a Template (required): 7
Project Name (required): Medium Tweak Pref
Package Name [com.yourcompany.mediumtweakpref]:com.reversethatapp.mediumtweakpref
Author/Maintainer Name [ReverseThatApp]: ReverseThatApp
[iphone/preference_bundle_modern] Class name prefix (three or more characters unique to this project) [XXX]: RTA
Instantiating iphone/preference_bundle_modern in mediumtweakpref/...
Adding 'Medium Tweak Pref' as an aggregate subproject in Theos makefile 'Makefile'.
Done.
```

When it's done, you will see new folder `mediumtweakpref` created and new lines added into your `Makefile`:
```bashscript
SUBPROJECTS += mediumtweakpref
include $(THEOS_MAKE_PATH)/aggregate.mk
```

Your project structure will look like this:
```bashscript
|- mediumenhanceclapstweak
|  |- mediumtweakpref
|  |  |- Resources
|  |  |  |- Info.plist
|  |  |  |- Root.plist // declare your UI here
|  |  |- entry.plist   // setup entry icon, title
|  |  |- Makefile
|  |  |- RTARootListController.h
|  |  |- RTARootListController.m // handle code logic
|  |- control
|  |- Makefile
|  |- mediumenhanceclapstweak.plist
|  |- Tweak.xm

```

### Design preference layout
Our target is build the preference UI like **Figure 1**. Let make the entry first (left panel on iPad).

#### Entry item
For entry item, we need an icon and change the entry title a bit. Let open `entry.plist` file and make it like this:
```bashscript
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>entry</key>
	<dict>
		<key>bundle</key>
		<string>MediumTweakPref</string>
		<key>cell</key>
		<string>PSLinkCell</string>
		<key>detail</key>
		<string>RTARootListController</string>
		<key>icon</key>
		<string>entry-icon.png</string><!-- change icon name here-->
		<key>isController</key>
		<true/>
		<key>label</key>
		<string>Medium Tweak</string><!-- change entry title here-->
	</dict>
</dict>
</plist>
```
For entry icon, I downloaded from [flaticon.com](https://www.flaticon.com/free-icon/clapping_599538?term=clap&page=1&position=9){:target="_blank"} and put same location as `entry.plist` file, name it `entry-icon.png`(size 32x32) and `entry-icon@2x.png`(size 64x64).

For entry label, I changed value to **Medium Tweak**, it's up to you.

#### Preference UI
For preference UI, I'm using slider cell and static text cell to display slider value (we call it **specifiers**). Let open `Resources/Root.plist` file and modify like this (you might have a look other [preferences specifier plist](https://iphonedevwiki.net/index.php/Preferences_specifier_plist){:target="_blank"} to understand how to use it)
```bashscript
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>title</key>
    <string>Medium Tweak</string>
    <key>items</key>
    <array>
        <dict>
            <key>cell</key>
            <string>PSGroupCell</string>
            <key>label</key>
            <string>Default claps</string>
        </dict>
        <dict>
            <key>cell</key>
            <string>PSSliderCell</string>
            <key>default</key>
            <integer>1</integer> <!-- Default value of the slider -->
            <key>defaults</key>
            <string>com.reversethatapp.mediumtweakpref</string> <!-- entry path no extension -->
            <key>key</key>
            <string>DEFAULT_CLAPS</string> <!-- Your key name here -->
            <key>max</key>
            <real>50</real> <!-- Enter your maximum value here -->
            <key>min</key>
            <real>1</real> <!-- Enter your minimum value here -->
            <key>showValue</key> <!-- Show the value of the slider. true or false. -->
            <false/>
            <key>isSegmented</key>
            <true/>
            <key>segmentCount</key>
            <integer>50</integer>
            <key>leftImage</key>
            <string>clapping-hands-left.png</string>
            <key>rightImage</key>
            <string>clapping-hands-right.png</string>
        </dict>	
    </array>
</dict>
</plist>
```

We are using `PSGroupCell` (you can think it like section) and `PSSliderCell` belongs to `PSGroupCell`. For `PSGroupCell` I will add it programmatically so you can understand how to add specifiers in plist or in code file. 

For `PSSliderCell` we are using `leftImage` and `rightImage` key, you can download those images and put in `Resources` folder. My case I made `leftImage` size smaller than `rightImage` size to illustrate increment order.

Now it's time for you to run and see how preference bundle look like, from Terminal navigate to Tweak root folder (not preference root folder) then compile and install the tweak as normal, it will compile and install our preference together, would be look like this
[![Medium Tweak Preference]({{ site.baseurl }}/images/20200502/preference-ui-without-static-cell.png)]({{ site.baseurl }}/images/20200502/preference-ui-without-static-cell.png){:target="_blank"} <br/>**Figure 5: Medium Tweak Preference draft**<br/><br/>

It looks nice and slider working fine, let add one more specifier programmatically right below using `PSStaticTextCell` to show value of slider. Open `RTARootListController.m` and modify existing `- (NSArray *)specifiers` as below:
```bashscript
// This capture slider value
static NSInteger currentClaps = 1;

- (NSArray *)specifiers {
    if (!_specifiers) {
        _specifiers = [self loadSpecifiersFromPlistName:@"Root" target:self];
    }

    // 1. load persisted default claps to currentClaps
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"identifier == %@", @"DEFAULT_CLAPS"];
    NSArray *filteredArray = [_specifiers filteredArrayUsingPredicate:predicate];
    PSSpecifier* sliderSpec = [filteredArray objectAtIndex:0];

    NSString *path = [NSString stringWithFormat:@"/User/Library/Preferences/%@.plist", sliderSpec.properties[@"defaults"]];
    NSMutableDictionary *settings = [NSMutableDictionary dictionary];
    [settings addEntriesFromDictionary:[NSDictionary dictionaryWithContentsOfFile:path]];
    NSString *key = sliderSpec.properties[@"key"];	
    if (key != nil && [key isEqualToString:@"DEFAULT_CLAPS"]) {
        NSInteger currentClapsNumber = [[settings objectForKey:key] intValue];		
        currentClaps = currentClapsNumber;
    }

    // 2. create new PSStaticTextCell
    PSSpecifier* spec = [PSSpecifier preferenceSpecifierNamed: [self generateClapsText]
                    target:self
                    set:@selector(setPreferenceValue:specifier:)
                    get:@selector(readPreferenceValue:)
                    detail:Nil
                    cell:PSStaticTextCell
                    edit:Nil];	
    // 3. alignment center
    [spec setProperty:@2 forKey:@"alignment"];	
	
    // 4. Insert below slider
    [_specifiers insertObject:spec atIndex:2];

    return _specifiers;
}
```

The idea is that it loaded defined specifiers in `Root.plist` file then add one more `PSStaticTextCell` at index 2 (after slider specifier). 
[![Medium Tweak Preference with static cell]({{ site.baseurl }}/images/20200502/preference-ui-with-static-cell.png)]({{ site.baseurl }}/images/20200502/preference-ui-with-static-cell.png){:target="_blank"} <br/>**Figure 6: Medium Tweak Preference with static cell**<br/><br/>

#### Read/Write preference

We've just added new static cell to display value of segment. Because segment cell value only can display decimal number, we will need another static cell to round up and display as an integer number. Open `RTARootListController.m` and add 2 more methods that read preference value from plist file and store value into plist file:

```bashscript
- (id)readPreferenceValue:(PSSpecifier*)specifier {
    NSString *path = [NSString stringWithFormat:@"/User/Library/Preferences/%@.plist", specifier.properties[@"defaults"]];
    NSMutableDictionary *settings = [NSMutableDictionary dictionary];
    [settings addEntriesFromDictionary:[NSDictionary dictionaryWithContentsOfFile:path]];
    return (settings[specifier.properties[@"key"]]) ?: specifier.properties[@"default"];    
}

- (void)setPreferenceValue:(id)value specifier:(PSSpecifier*)specifier {
    NSString *path = [NSString stringWithFormat:@"/User/Library/Preferences/%@.plist", specifier.properties[@"defaults"]];
    NSMutableDictionary *settings = [NSMutableDictionary dictionary];
    [settings addEntriesFromDictionary:[NSDictionary dictionaryWithContentsOfFile:path]];
    NSString *key = specifier.properties[@"key"];   

    // Handle if this is segment cell
    if ([key isEqualToString:@"DEFAULT_CLAPS"]) {
        // 1. Get current segment decimal value
        CGFloat defaultClapsFloat = [value floatValue];

        // 2. Convert to integer value and store to global variable
        NSInteger defaultClaps = (NSInteger) defaultClapsFloat;
        currentClaps = defaultClaps;
        [settings setObject:[NSNumber numberWithInteger:defaultClaps] forKey:key];        
        cellForRowAtIndexPath:[NSIndexPath indexPathForRow:0 inSection:1]];        

        // 3. Refresh static cell to display currentClaps
        PSTableCell1 *staticCell = (PSTableCell1 *)[super tableView:(UITableView *)self.table cellForRowAtIndexPath:[NSIndexPath indexPathForRow:1 inSection:0]];        
        [staticCell setTitle:[self generateClapsText]];

    } else {
        [settings setObject:value forKey:key];
    }

    // 4. Store preference values to file
    [settings writeToFile:path atomically:YES];
}

- (NSString *)generateClapsText {
    return [NSString stringWithFormat:@"%ld 👏", currentClaps];
}
```

Clean build and install the tweak, you will see number of claps change when drag segment. To double confirm if new segment value is persisted to file, open Filza app and navigate to `/User/Library/Preferences/com.reversethatapp.mediumtweakpref.plist` you will see there is an entry `DEFAULT_CLAPS` reflects new segment value. 

The codes require for preference bundle is complete, now we just need to read this `DEFAULT_CLAPS` value from `Tweak.xm` and simulate number of taps on UI.

### Using Preference 
#### Which method to hook?
Switch back to Hopper Disassembler and navigate to `tryToPerformClap` method and have a look:
```assembly
/* @class LongPressClapController */
-(void)tryToPerformClap {
    r0 = [self delegate];
    r0 = [r0 retain];
    [r0 clapViewWasTapped];
    [r0 release];
    return;
}
```

Just make it short this method will call `[[self delegate] clapViewWasTapped]`, so what's the delegate here? Double click on selector `clapViewWasTapped` you will be prompted this dialog:
[![Methods implementing clapsViewWasTapped]({{ site.baseurl }}/images/20200502/clapViewWasTapped-selector.png)]({{ site.baseurl }}/images/20200502/clapViewWasTapped-selector.png){:target="_blank"} <br/>**Figure 3: Methods implementing selector clapViewWasTapped**<br/><br/>

It has 3 classes implements this selector, the usual way is dynamic analysis with `lldb` to set breakpoint and print out `delegate` instance. But there is another way I want to introduce in this post is using [frida-trace](https://frida.re/docs/ios/){:target="_blank"} (I will have another post how to use it in details soon). From Terminal just run `frida-ps -Ua` to see which is process ID of Medium app:
```bashscript
MBP# frida-ps -Ua
  PID  Name         Identifier
-----  -----------  -----------------------------
...
14389  Medium       com.medium.reader
...
```

Using this PID `14389` for next command to trace how method `clapViewWasTapped` is called:
```bashscript
MBP# frida-trace -U -p 14389 -m "-[* clapViewWasTapped]"
Instrumenting functions...
-[StickyFullPostFooterActionBar clapViewWasTapped]: Auto-generated handler at "/__handlers__/__StickyFullPostFooterActionBar__6b41bd3e.js"
-[NowPlayingViewController clapViewWasTapped]: Auto-generated handler at "/__handlers__/__NowPlayingViewController_clapV_52240ab8.js"
-[hangtag.IcelandPostActionBarCell clapViewWasTapped]: Auto-generated handler at "/__handlers__/__hangtag.IcelandPostActionBarCe_0381b398.js"
-[PostPreviewActionBar clapViewWasTapped]: Auto-generated handler at "/__handlers__/__PostPreviewActionBar_clapViewWasTapped_.js"
Started tracing 4 functions. Press Ctrl+C to stop.           
```

We are tracing all classes that implement selector `clapViewWasTapped` (`-U` is usb, `-p` is process ID). Let clap any post, you will see some logs in Terminal console
```bashscript
        /* TID 0x403 */
  3371 ms  -[StickyFullPostFooterActionBar clapViewWasTapped]
  3372 ms     | -[PostPreviewActionBar clapViewWasTapped]
```

BINGO!! `StickyFullPostFooterActionBar` is the `delegate`, and it also means we can hook this method like this in the `Tweak.xm` file (take note to remove existing `%hook ClapButton` section in `Tweak.xm` file before adding new hook below):

#### Using preference
```bashscript
// Define preference path, it would be /var/mobile/Library/Preferences/prefernceBundleId.plist
#define PLIST_PATH "/var/mobile/Library/Preferences/com.reversethatapp.mediumtweakpref.plist"

%hook StickyFullPostFooterActionBar

- (void)clapViewWasTapped {
    %log;

    // 1. Load info stored to device in `.plist` file into dictionary
    NSDictionary *settings = [[NSMutableDictionary alloc] initWithContentsOfFile:@PLIST_PATH];

    // 2. Load default clap setup in preference 
    CGFloat defaultClapsFloat = [[settings objectForKey:@"DEFAULT_CLAPS"] floatValue];
    NSInteger defaultClaps = (NSInteger) defaultClapsFloat;

    // 3. Simulate clap
    for (int i = 0; i < defaultClaps; i++) {
        NSLog(@"Clap no. %d", i);
        %orig;
    }
}

%end
```

Preference bundle will persist settings in `.plist` file at above location, and from `Tweak.xm` we just need to fetch that file and extract the correct key that defined for each cell in the Preference `Resources/Root.plist` file, in this case `DEFAULT_CLAPS` is the key of clap segment setting.

### Testing
![You made it!!](https://thumbs.gfycat.com/DelectableHarshBellfrog-size_restricted.gif)<br/>**Figure 7: You made it!!**<br/><br/>

### Spoil alert
Next posts we will continue to reverse Medium app and enable unlimited read feature for free user, is that cool? Follow my Twitter to get notified for up comming posts :P
[![Unlimited read]({{ site.baseurl }}/images/20200502/spoil-alert.png)]({{ site.baseurl }}/images/20200502/spoil-alert.png){:target="_blank"} <br/>**Figure 8: Unlimited read**<br/><br/>

## Final thoughts
- Again hooking is a fast way to change existing behaviors of the app, you can find there are so many useful tweaks out there to install and experience yourself.
- From research prospective, hooking to methods can help change behaviors of the app, for example disable jailbreak detection, disable SSL Pinning... which allow you to inspect the app in runtime.

## Further readings
- [Preferences Specifier plist](https://iphonedevwiki.net/index.php/Preferences_specifier_plist){:target="_blank"}
- [Theos](http://iphonedevwiki.net/index.php/Theos){:target="_blank"}
- [Logos syntax](http://iphonedevwiki.net/index.php/Logos){:target="_blank"}
- [Compile issue](https://github.com/theos/theos/issues/324){:target="_blank"}
- [frida-trace](https://frida.re/docs/frida-trace/){:target="_blank"}
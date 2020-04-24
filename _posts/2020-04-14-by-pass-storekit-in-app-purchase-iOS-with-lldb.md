---
layout: post
title: Bypass StoreKit In-app purchases of iOS apps using LLDB
---

StoreKit is an iOS framework that supports in-app purchases and interactions with the App Store (rate and review app...). It leverages iOS developers to implement in-app purchases feature with ease. However, most developers just made it work without knowing the best practices, which created flaws for attackers to inspect and defeat In-app purchases feature easily, which means they can use the premium content of the app for FREE. Let's do some reverse engineering and find out what are flaws and how to exploit them using LLDB.
[![StoreKit In-app Purchases]({{ site.baseurl }}/images/20200414/storekit-flow.png)]({{ site.baseurl }}/images/20200414/storekit-flow.png){:target="_blank"} <br/>**Figure: StoreKit In-app Purchases flow** *(source: Medium)*<br/><br/>

## Disclaimer
This post is for educational purposes only, please use it at your discretion and contact the app's author if you find issues. We will inspect an app name REDACTED. The figures during the post just for demonstrations, might not relevant to REDACTED app.

## Prerequisites
Below tools are used during this post:
- A jailbroken device.
- [Hopper Disassembler](https://www.hopperapp.com/download.html)
- [Setup LLDB environment](https://kov4l3nko.github.io/blog/2016-04-27-debugging-ios-binaries-with-lldb/)
- A bit knowledge of assembly arm64, refer previous [post]({{ site.baseurl }}/by-pass-ssl-pinning-iOS-with-lldb/){:target="_blank"}

## Overview
When you install an app from App Store, if it has In-App Purchases feature it will show beside then Install button. Let install REDACTED app on the jailbroken device first and launch the app. For these kinds of app, it will provide you some contents for Free and some contents require to purchase, which will show Apple purchase popup.
[![In-app purchases features]({{ site.baseurl }}/images/20200414/sample-inapp-purchases-features.png)]({{ site.baseurl }}/images/20200414/sample-inapp-purchases-features.png){:target="_blank"} <br/>**Figure 1: Sample In-app purchases features** *(source: developer.apple.com)*<br/><br/>

Tap on one of the items, you will be asked for purchasing and showing Apple In-app purchases popup, might be like this:

[![In-app purchases popup]({{ site.baseurl }}/images/20200414/native-in-app-purchases-popup.png)]({{ site.baseurl }}/images/20200414/native-in-app-purchases-popup.png){:target="_blank"} <br/>**Figure 2: In-app purchases native popup**<br/><br/>

Let's do some analysis ^_^

## Static Analysis
### Inspect .ipa resources
With the help of [Frida iOS Dump](https://github.com/AloneMonkey/frida-ios-dump){:target="_blank"} or [CrackerXI](https://forum.iphonecake.com/index.php?/topic/363020-crackerxi-gui-app-decryption-tool-for-ios-11-12-13/){:target="_blank"}, we can easily pull out **.ipa** file of REDACTED app on a jailbroken device, unzip **.ipa** and navigate to `Payload/REDACTED.app` folder. All of the app resources and binary files are inside this folder.
[![.ipa resources]({{ site.baseurl }}/images/20200414/ipa-resources.png)]({{ site.baseurl }}/images/20200414/ipa-resources.png){:target="_blank"} <br/>**Figure 3: .ipa resources**<br/><br/>

Besides common resources you will see in other .ipa files, this one has a bit different. Look for `spine-lua`, `CoronaResources.bundle`, `resource.corona-archive`, it's CORONA!!! Don't get goosebumps please, this is not the kind of Corona virus but Corona framework. [Corona](https://coronalabs.com/) is the 2D game engine, a cross-platform framework ideal for rapidly creating apps and games for mobile devices and desktop systems. That means you can create your project once and publish it to multiple types of devices, including Apple iPhone and iPad, Android phones and tablets, Amazon Fire, Mac Desktop, Windows Desktop, and even connected TVs such as Apple TV, Fire TV, and Android TV (from [coronalabs site](ttps://coronalabs.com/)). I will have another post to share the process of reversing Corona files soon, just leave these kinds of stuff for now.

Let's scan through files inside this folder and stop at REDACTED Mach-O file, it's very small in size - **ONLY 2MB**. That means all of the app logic will remain in those Corona files. You might wonder if the In-app purchase there also? It would be, but in the end the purchase process should be in native iOS and the REDACTED file. Let's examine this guy right away.

### Inspect In-app purchases process using Hopper Disassembler
Drag and drop REDACTED binary file into Hopper Disassembler application, this time you won't need to wait so long because the file is small and the disassembling process will take seconds to complete. Where to look for In-app purchases implementation?

Let's take a break and see how the StoreKit work. Look back on the first **Figure** diagram, your app makes a request and the observer is called when processes the payment. In the StoreKit framework, `SKPaymentTransactionObserver` is the protocol that app needs to conform to handle the payment process. Check `SKPaymentTransactionObserver` from [Apple documents](https://developer.apple.com/documentation/storekit/), there is a method handling transaction `func paymentQueue(_ queue: SKPaymentQueue, 
updatedTransactions transactions: [SKPaymentTransaction])`, click on this [method](https://developer.apple.com/documentation/storekit/skpaymenttransactionobserver/1506107-paymentqueue) to see details explanation - this method tells an observer that one or more transactions have been updated. Apple says that:

> The application should process each transaction by examining the transaction’s `transactionState` property. If `transactionState` is `SKPaymentTransactionStatePurchased`, payment was successfully received for the desired functionality. The application should make the functionality available to the user. If `transactionState` is `SKPaymentTransactionStateFailed`, the application can read the transaction’s error property to return a meaningful error to the user.

> Once a transaction is processed, it should be removed from the payment queue by calling the payment queue’s `finishTransaction(_:)` method, passing the transaction as a parameter.

As the above statement, if `transactionState` is `SKPaymentTransactionStatePurchased`, payment was successfully received for the desired functionality and app should make the functionality available to the user, for this case, it might unlock all items. So if we can change the value of `transactionState` into `SKPaymentTransactionStatePurchased`, it's possible to unlock the app. Let's hunt for this.

We know the method name to look for - `func paymentQueue(_ queue: SKPaymentQueue, 
updatedTransactions transactions: [SKPaymentTransaction])`, but if you look for this string in Hopper Disassembler, it won't spit out any matches. The reason is it's not a selector, so we need to search by selector name of this method, with could be `paymentQueue:updatedTransactions:`. Switch to Hopper and search this selector name in **Labels** tab, we can see one result matches `-[AppleStoreManager paymentQueue:updatedTransactions:]:`, which also mean class `AppleStoreManager` is the one conforms to `SKPaymentTransactionObserver` protocol and handle transactions.
[![paymentQueue:updatedTransactions: implementation]({{ site.baseurl }}/images/20200414/search-paymentQueue-updatedTransactions.png)]({{ site.baseurl }}/images/20200414/search-paymentQueue-updatedTransactions.png){:target="_blank"} <br/>**Figure 4: paymentQueue:updatedTransactions: implementation**<br/><br/>

### It's ARM64 again, let's swallow it
Let's take another oppotunity to learn ARM64 this time, I hope you dont quit this post for now. Let toggle CFG (Control Flow Graph) mode to see what this method is doing easier. Don't panic, it would be simple to understand.
[![paymentQueue:updatedTransactions: implementation]({{ site.baseurl }}/images/20200414/paymentQueue-updatedTransactions-CFG.png)]({{ site.baseurl }}/images/20200414/paymentQueue-updatedTransactions-CFG.png){:target="_blank"} <br/>**Figure 5: paymentQueue:updatedTransactions: implementation CFG mode**<br/><br/>

Let read through from top to bottom, we won't need to understand all assembly instructions but focus on the important ones. Let's understand flow from **Block 1** -> **Block 6** will do.

#### Block 1
```assembly
		-[AppleStoreManager paymentQueue:updatedTransactions:]:
sub        sp, sp, #0x140 ; Objective C Implementation defined at 0x100222690 (instance method), DATA XREF=0x100222690
stp        x28, x27, [sp, #0xe0]	; function prologue
stp        x26, x25, [sp, #0xf0]	; function prologue
stp        x24, x23, [sp, #0x100]	; function prologue
stp        x22, x21, [sp, #0x110]	; function prologue
stp        x20, x19, [sp, #0x120]	; function prologue
stp        x29, x30, [sp, #0x130]	; function prologue 
add        x29, sp, #0x130
mov        x19, x3      ; transactions: [SKPaymentTransaction]
mov        x20, x0   	; self: AppleStoreManager
adrp       x8, 
ldr        x8, [x8, #0x238]
ldr        x8, [x8]     ; ___stack_chk_guard
stur       x8, [x29, var_60]
stp        xzr, xzr, [sp, #0x40]
stp        xzr, xzr, [sp, #0x30]
stp        xzr, xzr, [sp, #0x20]
stp        xzr, xzr, [sp, #0x10]
adrp       x8, #0x10022e000
ldr        x1, [x8, #0x5c8] ; @selector(countByEnumeratingWithState:objects:count:)
add        x2, sp, #0x10 ; x2 = state = 0 (beginning of iteration)
add        x3, sp, #0x50 ; x3 = objects: A C array of objects over which the sender is to iterate
orr        w4, wzr, #0x10 ; count = 16 (The maximum number of objects to return in stackbuf)
mov        x0, x19      ; transactions
str        x1, [sp, #0x130 + var_128]
bl         imp___stubs__objc_msgSend ; objc_msgSend(x0, x1, x2, x3, x4)
cbz        x0, loc_10000df1c ; x0 == 0 (transactions.count == 0)?
```

I put some comments at the end of important instructions (after **;**). Let break it down a bit.
- `-[AppleStoreManager paymentQueue:updatedTransactions:]:`
Objective-C method always has 2 hidden arguments: `self` and `_cmd`. `self` argument refers to the method's receiver, `_cmd` is the method's selector. 
For example: 
```objective-c
AppleStoreManager manager = [AppleStoreManager init];
[manager paymentQueue:queue updatedTransactions:transactions];
```
would be converted into:
```C
objc_msgSend(x0, x1, x2, x3, x4)
```
with values:
```assembly
x0 = manager
x1 = @selector(paymentQueue:transactions:)
x2 = queue
x3 = transactions
```
- **Function prologue**: In assembly language programming, the **function prologue** is a few lines of code at the beginning of a function, which prepare the stack and registers for use within the function. Similarly, the **function epilogue** appears at the end of the function, and restores the stack and registers to the state they were in before the function was called. (source: [Wiki](https://en.wikipedia.org/wiki/Function_prologue))

- **What does this block do?** Follow above `objc_msgSend` call convention, this block is trying to call selector `countByEnumeratingWithState:objects:count` of sender `transactions` for `transactions` to be iterated. `bl imp___stubs__objc_msgSend ; objc_msgSend(x0, x1, x2, x3, x4)` is a function call to `objc_msgSend(transactions, @selector(countByEnumeratingWithState:objects:count:), 0, arrayStartAtStackPointerAddressPlus0x50, 16)` method and the return value will be stored in register `x0`. This function call return value is the number of objects returned in `arrayStartAtStackPointerAddressPlus0x50`, for simplicity, you can assume it is `transactions.count` and will store return value to register `x0`, which means: `x0 = transactions.count`. The next instruction `cbz  x0, loc_10000df1c` means it will branch to block `loc_10000df1c` if register `x0 == 0` (or `transactions.count == 0`), otherwise it will continue to execute next instruction in **Block 2**. In this case if method `-[AppleStoreManager paymentQueue:updatedTransactions:]` is triggered, `transactions.count` mostly greater than 0 so it will branch to **Block 2**.

#### Block 2
```assembly
mov        x22, x0      ; x22 = x0 = transactions.count
ldr        x8, [sp, #0x130 + var_110]
ldr        x28, [x8]
```

This block does not have much info, `mov x22, x0` means to move value in register `x0` into register `x22`, or `x22 = x0` for short. We need to put comments at the end of each instruction we know, so later we can traceback to the closest assignment register to know which value it's holding. In Hopper Disassembler, press combination `Shift ;` to put an inline comment. Let move to next execution block - **Block 3**

#### Block 3
```assembly
loc_10000de60:
movz       x21, #0x0    ; CODE XREF=-[AppleStoreManager paymentQueue:updatedTransactions:]+304
adrp       x8, #0x10022e000
ldr        x23, [x8, #0x948] ; "transactionState",@selector(transactionState)
nop
ldr        x24, [x8, #0x950] ; "completeTransaction:",@selector(completeTransaction:)
nop
ldr        x25, [x8, #0x958] ; "failedTransaction:",@selector(failedTransaction:)
nop
ldr        x26, [x8, #0x960] ; "restoreTransaction:",@selector(restoreTransaction:)
```

This block mainly load selectors and store into registers. `ldr` (Load Register) command loads value from memory and writes it to a register (on ARM data need to be moved from memory into registers first so that it can operate on). The address that is used for the load is calculated from a *base register* and an *immediate offset*. As you can see *base register* here is `x8` (`adrp x8, #0x10022e000` can simply understand as `x8 = #0x10022e000`), `ldr  x23, [x8, #0x948]` is `x23 = [x8 + #0x948] = [#0x10022e000 + #0x948] = [#0x10022E948]`. The square `[address]` means load value holding at address. In Hopper Disassembler, press `G` then enter `0x10022E948` will navigate you to below address:

`000000010022e948  dq   aTransactionsta; @selector(transactionState), "transactionState"`

This address holding value of `@selector(transactionState)`. You can apply the same formula for other `ldr` commands in this block. Actually Hopper Disassembler will auto-detect and put the comments for you during the disassembling process. Let move to next execution block - **Block 4**

#### Block 4
```assembly
loc_10000de84:
ldr        x8, [sp, #0x130 + var_110] ; x8 = value at address (sp + #0x130 - #0x110)
ldr        x8, [x8] ; x8 = value at x8
cmp        x8, x28 ; compare x8 with x28, need to traceback what's value of x28
b.eq       loc_10000de9c
```
This block contains logic for branching and you will see the benefit of inline comments to traceback. Quick look this block will compare value of register `x8` and `x28` (`cmp` is compare command) and branch to `loc_10000de9c` if both equals (`b.eq` - branch if equals), otherwise branch to **Block 5**. Let find out what are values of those registers.

##### Register `x8` = ?
```assembly
ldr        x8, [sp, #0x130 + var_110] ;
```
This instruction will load value at address inside square brackets and store into register `x8`. `var_110` is the variable label Hopper Disassembler generated during disassembling process, if you notice you can see these labels at the top of each method, for this method it will be like this:
```assembly
; ================ B E G I N N I N G   O F   P R O C E D U R E ================

        ; Variables:
        ;    saved_fp: 0
        ;    var_10: -16
        ;    var_20: -32
        ;    var_30: -48
        ;    var_40: -64
        ;    var_50: -80
        ;    var_60: int64_t, -96
        ;    var_F0: -240
        ;    var_100: -256
        ;    var_110: int64_t, -272
        ;    var_118: int64_t, -280
        ;    var_120: -288
        ;    var_128: int64_t, -296
```
`var_110: int64_t, -272` is a signed integer type with exactly 64 bits, it's holding value **-272** in decimal, convert to hexadecimal would be minus **0x110**, that why Hopper Disassembly name it `var_110` for readability. So `ldr        x8, [sp, #0x130 + var_110]` = `ldr        x8, [sp, #0x130 - #0x110]` = `ldr        x8, [sp, #0x20]`(`sp` is stack pointer register). This instruction means load value at address `[sp + #0x20]` into register `x8`. Let traceback what's value at address `[sp + #0x20]`, we can see that in **Block 1**, this instruction `stp xzr, xzr, [sp, #0x20]` store zero register or 0 (the zero register is a special register that is hard-wired to the integer value 0. This register is found in many RISC instruction set architectures such as MIPS and RISC-V. On those architectures writing to that register is always discarded and reading its value will always result in a 0 being read.). So `x8` value is 0.

```assembly
ldr        x8, [x8]
```
This instruction will load value at address holding by register `x8` and store into register `x8` again. `x8` would be `0` after this execution. We are done for finding the value of register `x8`, let's continue with register `x28`

##### Register `x28` = ?
Trace back to **Block 2** you can see closest assignment of register `x28` is these 2 instructions:
```assembly
ldr        x8, [sp, #0x130 + var_110]
ldr        x28, [x8]
```
It's exactly the same way it loads value for register `x8` as above steps, so `x28` will hold value of `0`.


Now look back to **Block 4** again, `cmp  x8, x28` will compare value of registers `x8` and `x28` and both value is the same, so the next instruction `b.eq  loc_10000de9c` means branch to `loc_10000de9c` if both registers (`x8` and `x28`) equals, and it does. So next execution block is `loc_10000de9c` - **Block 6**

#### Block 6
```assembly
loc_10000de9c:
ldr        x8, [sp, #0x130 + var_118]  
ldr        x27, [x8, x21, lsl #3]
mov        x0, x27      
mov        x1, x23      ; "transactionState",@selector(transactionState)
bl         imp___stubs__objc_msgSend ; objc_msgSend
cmp        x0, #0x3
b.eq       loc_10000dee0
```
This block contains an `objc_msgSend` method, which means application will invoke a method. We need to find down what are parameters passing to this method. As above explanation in **Block 1**, the first 2 arguments  is `x0` (receiver) and `x1` (selector). We can see `x1` is `@selector(transactionState)` (auto generated by Hopper Disassembler), now we need to find what is receiver - register `x0` in `mov  x0, x27` instruction. This instruction will move value of `x27` into `x0`, we need to find what is value `x27` holding. But for this case, base on the selector `@selector(transactionState)` we can guess receiver is `SKPaymentTransaction` type, so let put inline comments into above instructions to make it clearer:
```assembly
mov        x0, x27      ; transaction: SKPaymentTransaction
mov        x1, x23      ; "transactionState",@selector(transactionState)
bl         imp___stubs__objc_msgSend ; objc_msgSend(transaction, @selector(transactionState))
```

And the result of `objc_msgSend(transaction, @selector(transactionState))` will be always stored in register `x0`, so after executing this instruction, register `x0` will hold value of `transaction.transactionState` or `SKPaymentTransactionState` type. 

Next instruction `cmp  x0, #0x3` will compare `transactionState` with `#0x3` (3 in decimal) to decide which location to branch in next instruction `b.eq  loc_10000dee0`. It would make sense here to compare `SKPaymentTransactionState` type with number `3` because `SKPaymentTransactionState` is `enum` type and will use its raw value to compare with `3`.
[![SKPaymentTransactionState-enum]({{ site.baseurl }}/images/20200414/SKPaymentTransactionState-enum.png)]({{ site.baseurl }}/images/20200414/SKPaymentTransactionState-enum.png){:target="_blank"} <br/>**Figure 6: SKPaymentTransactionState enum**<br/><br/>

From Apple implementation, `SKPaymentTransactionState` does not explicitly assign a raw value for each case, so the implicit value for each case is one more than the previous case. Because first case `purchasing` doesn’t have a value set, its value is 0. It would be same like this:
```Swift
public enum SKPaymentTransactionState : Int {    
    case purchasing = 0
    case purchased = 1
    case failed = 2
    case restored = 3
    case deferred = 4
}
```

Why does it need to bother with transaction state? Let have a look on below actions we usually do for each state:
[![transactionStates]({{ site.baseurl }}/images/20200414/transactionStates.png)]({{ site.baseurl }}/images/20200414/transactionStates.png){:target="_blank"} <br/>**Figure 7: Transaction State action needed** *(source: WWDC)*<br/><br/>

It makes more sense now, as you can see if the transaction state is `.purchased` or `.restored` the app should deliver the content to the user, then call `StoreKit` `finishTransaction` method to finish the transaction. So the app will check the transaction state and take corrective actions. 

We can stop the binary analysis at this point, let think about if we can change the value of transaction state into `.purchase` or `.restored` here, will the app unlock all contents? There are many ways to change the value of this transaction state, for this post we will use `LLDB` to debug and examine state value at run time. The boring part is over, the funny one is coming :P

## Dynamic Analysis
### Start debugserver
Let launch the app on the jailbroken device first, then open Terminal on laptop and SSH into the jailbroken device and start `debugserver` waiting for the app. To know what's app name to attach, you can get it in **Info.plist** file (key `Executable file`).
```bashscript
sh-3.2# ssh root@192.168.1.113
root@192.168.1.113's password:
iPhone-SE:~ root# debugserver 0.0.0.0:1234 -a REDACTED
debugserver-@(#)PROGRAM:LLDB  PROJECT:lldb-900.3.57..2
 for arm64.
Attaching to process REDACTED...
Listening to port 1234 for a connection from 0.0.0.0...
Waiting for debugger instructions for process 0.
```

### Attach LLDB into app process
Open another tab on Terminal app and launch `lldb`, for my case it will auto attach into remote `debugserver` as I put some config in `~/.lldbinit` file. You can read my [previous post]({{ site.baseurl }}/by-pass-ssl-pinning-iOS-with-lldb/#launch-lldb){:target="_blank"} how to do that.
```bashscript
 MBP:~ lldb
  Platform: remote-ios
 Connected: no
  SDK Path: "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/13.3.1 (17D50) arm64e"
 SDK Roots: [ 0] "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/13.3.1 (17D50) arm64e"
 SDK Roots: [ 1] "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/11.3.1 (15E302)"
 SDK Roots: [ 2] "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/11.0 (15A372)"
 SDK Roots: [ 3] "/Users/xyz/Library/Developer/Xcode/iOS DeviceSupport/12.4 (16G77)"
Process 29446 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x000000018111fe5c libsystem_kernel.dylib`semaphore_timedwait_trap + 8
libsystem_kernel.dylib`semaphore_timedwait_trap:
->  0x18111fe5c <+8>: ret

libsystem_kernel.dylib`semaphore_timedwait_signal_trap:
    0x18111fe60 <+0>: mov    x16, #-0x27
    0x18111fe64 <+4>: svc    #0x80
    0x18111fe68 <+8>: ret
Target 0: (REDACTED) stopped.
(lldb)
```
When `lldb` can attach to the app, the process will stop, let first check what's ASLR shift:
```bashscript
(lldb) image list -o REDACTED
[  0] 0x00000000027f4000
```

Now we know `ASLR shift = 0x00000000027f4000`, let take note and switch back to Hopper Disassembler, we need to get the address to set breakpoint. Let select the instruction `bl imp___stubs__objc_msgSend ; objc_msgSend` in **Block 6**, this is the instruction to get the transaction state. When you select this instruction, at the bottom-left of Hopper Disassember, you will see it's showing address something like this `0x10000deac`, this is the one we need. So the final address can be calculated by sum up ASLS Shift value and the instruction address value, which means `0x00000000027f4000 + 0x10000deac = 0x102801EAC`. Back to `lldb` prompt and set breakpoint to this address: `br s -a 0x102801EAC`
```bashscript
(lldb) br s -a 0x102801EAC
Breakpoint 1: where = REDACTED`___lldb_unnamed_symbol223$$REDACTED + 196, address = 0x0000000102801eac
```

The breakpoint is set, let's resume the app by typing: `continue` in `lldb` prompt
```bashscript
(lldb) continue
Process 29446 resuming
```

It's time to trigger In-app purchases transaction by tapping on locked items. When it's about to show the native In-app purchase popup for purchasing, IT HITS BREAKPOINT!! YEAH!!!
```bashscript
Process 29446 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000102801eac REDACTED`___lldb_unnamed_symbol223$$REDACTED + 196
REDACTED`___lldb_unnamed_symbol223$$REDACTED:
->  0x102801eac <+196>: bl     0x10294924c               ; symbol stub for: objc_msgSend
    0x102801eb0 <+200>: cmp    x0, #0x3                  ; =0x3
    0x102801eb4 <+204>: b.eq   0x102801ee0               ; <+248>
    0x102801eb8 <+208>: cmp    x0, #0x2                  ; =0x2
Target 0: (REDACTED) stopped.
```

As explained in **Block 6** section, this instruction will invoke the method to get transaction state, so the first hidden argument is the register `x0` should be the transaction, let print out to double confirm if it's correct:
```bashscript
(lldb) po $x0
<SKPaymentTransaction: 0x1c00170f0>
```

YEAH!! Register `x0` is holding address of object `SKPaymentTransaction`. Let's check what is state of this transaction using `valueForKey:` selector:
```bashscript
(lldb) po [0x1c00170f0 valueForKey:@"transactionState"]
0
```

Raw value `0` means it's `.purchasing` state (raw value is `0`), let modified it to `.purchased`. But we got problem here that `transactionState` property only allows getter without setter. However, `SKPaymentTransaction` extends `NSObject` which provides default implementation of the key-value coding protocol methods, so we can use `setValue:forKey:` selector to set value for `transactionState` property to `.purchased` (raw value is `1`), let's try:
```bashscript
(lldb) po [0x1c00170f0 setValue:[NSNumber numberWithInteger:1] forKey:@"transactionState"]
<SKPaymentTransaction: 0x1c00170f0>

(lldb) po [0x1c00170f0 valueForKey:@"transactionState"]
1
```

We can see new value set successfully. The transaction object at address `0x1c00170f0` now holding state `.purchased`, let step over `obj_msgSend` instruction (type `next` then enter), you can see the result in register `x0` is `1` (`.purchased`)
```bashscript
(lldb) next
Process 29446 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000102801eb0 REDACTED`___lldb_unnamed_symbol223$$REDACTED + 200
REDACTED`___lldb_unnamed_symbol223$$REDACTED:
->  0x102801eb0 <+200>: cmp    x0, #0x3                  ; =0x3
    0x102801eb4 <+204>: b.eq   0x102801ee0               ; <+248>
    0x102801eb8 <+208>: cmp    x0, #0x2                  ; =0x2
    0x102801ebc <+212>: b.eq   0x102801ed4               ; <+236>
Target 0: (REDACTED) stopped.

(lldb) po $x0
1
```

Let resume the app to see if it can unlock all items.
```bashscript
(lldb) continue
```

You might see the app hits the breakpoint again, but just `continue` again to ignore it and resume the app. After this, the app can continue to run as normal flow and the premium features you purchased are unlocked, which also means we bypassed StoreKit In-app purchases feature. MISSION COMPLETED!!

## But WAIT!! Does this mean all In-app purchases apps can be bypassed like this?
The answer is NO! Apple has a mechanism to validate if the user has purchased the item or not using receipts. This is StoreKit best practice and highly recommended for all developers when working with In-app purchases feature. The only problem is that it's required efforts to implement, so be often ignored by developers. However, even with these extra efforts for receipt validation to enhance In-app purchases security, attackers can reverse the app and patch it to bypass again. I will put the reference links for further reading about receipt validation and suggestion to secure reverse process.

## Final thoughts
- Again, LLDB is a powerful tool to debug and understand all states of the app.
- An app without anti-debug is vulnerable.
- StoreKit is fast to integrate with Apple In-app purchase feature but if we implement it incorrectly it will create a flaw for attackers to access your premium content without paying anything, your revenue will be affected (attackers can develop tweaks, patch the binary and release to everyone)
- This bypass technique is the same that most tweaks on Cydia are using.

## Further readings
- [Receipt validation](https://www.objc.io/issues/17-security/receipt-validation/)
- [WWDC](https://developer.apple.com/videos/play/wwdc2018/704/)
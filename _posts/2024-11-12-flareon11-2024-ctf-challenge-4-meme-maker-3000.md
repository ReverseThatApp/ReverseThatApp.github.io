---
layout: post
title: "Cracking the Flare-On 11 CTF 2024: Challenge 4 - Meme Maker 3000"
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczOEMCXdYQXjlO2NTNW6t-yxDSkcGgkwUrMHcW8MsNmxRNLfjpQdTrsgwOadQEKdJT0BH8vTWS8hjY4VqCKz7JkpJxeptjfAX_SXEcQeETSdFJeWwtZkQZL3G4s8CEE1URbihYzofjw-_R-1duWRTtPV=w2444-h1398-s-no-gm?authuser=2
    alt: Challenge 4 - Meme Maker 3000
tags: [flareon, flareon11, flareon-2024, ctf, javascript, obfuscated]
categories: [CTF, Flareon 2024]
---

Given a deceptively "simple" `mememaker3000.html` file weighing in at `2.5MB` and packed with obfuscated JavaScript—don't panic! At first glance, the file might feel like a cosmic joke on developers, but I dove in and found a way to deobfuscate it. Turns out, hunting for a flag has never been this entertaining. Let's get our digital magnifying glass and dive in!

## Challenge description
> 4 - Meme Maker 3000
>
>You've made it very far, I'm proud of you even if noone else is. You've earned yourself a break with some nice HTML and JavaScript before we get into challenges that may require you to be very good at computers.

## The Main `mememaker3000.html` File
Even though this is an `.html` file, it barely contains any HTML—most of it is a dense block of JavaScript logic. The user interface is as minimalist as it gets, with just a dropdown list to generate funny images and texts. It’s almost as if HTML was just invited to the party to watch JavaScript do all the heavy lifting!

![Meme Maker 3000 interface](https://lh3.googleusercontent.com/pw/AP1GczMi8MeGdUpHITKs9tJoqqToz5SEWUqA1i68c_E4GmwEEtGc3egWLAS6qmEPlUajdzZugSNAxgxjphRuWKhMnfCXqMzJXPxNvdLfRTM4m3Ze5om2GW3zbAFecF92-KGwxgGxMumI41erCkq_Pl8GJCr3=w1698-h1290-s-no-gm?authuser=2)
_**Figure: 2 - Meme Maker 3000 interface**_

Nothing fancy or interesting here—so let's hop into the JavaScript code to see what secrets it’s trying to hide.

## Obfuscated JavaScript Logic

Drop `mememaker3000.html` into a text editor, and you’ll spot a massive obfuscated JavaScript block embedded between the `<script></script>` tags at the bottom of the page. Here’s a sneak peek at how it looks:
```js
const a0p=a0b;(function(a,b){const o=a0b,c=a();while(!![]){try{const d=parseInt(o(0xd7ed))/0x1*(parseInt(o(0x381d))/0x2)+-parseInt(o(0x10a7f))/0x3*(-parseInt(o(0x15fd2))/0x4)+parseInt(o(0x128f8))/0x5+-parseInt(o(0x1203c))/0x6+parseInt(o(0xe319))/0x7*(parseInt(o(0xe69f))/0x8)+-parseInt(o(0x17d84))/0x9+parseInt(o(0x6866))/0xa*(-parseInt(o(0x2e3b))/0xb);if(d===b)break;else c['push'](c['shift']());}catch(e){c['push'](c['shift']());}}}(a0a,0x56f9f));const a0c=[a0p(0x14c8f)+a0p(0x114df)+a0p(0x17cca)+a0p(0xcd68)+'verflo'+a0p(0xccba)+'egacy\x20'+a0p(0x7d61),a0p(0x13c3f)+a0p(0x10d3)+a0p(0x17a2),a0p(0x14c8f)+'ou\x20dec'+a0p(0x8440)+a0p(0xd950)+'bfusca'+'ted\x20co'+a0p(0x143ce)+a0p(0x562f)+'kes\x20pe'+a0p(0x17b7c)+a0p(0x10d4a),a0p(0x257)+'er\x20a\x20w'+a0p(0x16235)+a0p(0x168a9)+a0p(0xbbc2)+a0p(0x6e47)+'ng','When\x20y'+a0p(0xd14e)+'compil'+'er\x20cra'+'shes',a0p(0x1525f)+a0p(0x2220)+a0p(0x18635)+a0p(0x12631)+a0p(0xd3c7),'Securi'+'ty\x20\x27Ex'+a0p(0x11370),'AI',a0p(0x12de4)+a0p(0x11202)+',\x20but\x20'+'can\x20yo'+a0p(0x3968)+'\x20it?',a0p(0x14c8f)+a0p(0x1a83)+a0p(0xabcf)+a0p(0x293c)+a0p(0x8e99)+'e\x20firs'+a0p(0xe162),a0p(0x100c0)+a0p(0x8c1b)+a0p(0x3bba)+a0p(0xb640)+a0p(0x1490f),'Readin'+'g\x20some'+a0p(0x16a1a)+a0p(0xa262)+a0p(0x247),a0p(0x15660),'This\x20i'+'s\x20fine',a0p(0x17f91)+'On',a0p(0xee13)+a0p(0xacec)+a0p(0xf9a7),'string'+a0p(0xa56b),a0p(0x9e4)+a0p(0x1734)+a0p(0xe903)+'t.','When\x20y'+a0p(0x114df)+a0p(0x17ec6)+a0p(0xb32e)+a0p(0x17ef4)+a0p(0x131d8)+'oit',a0p(0xb307)+'ty\x20thr'+'ough\x20o'+a0p(0x11fca)+'ty',a0p(0x104a)+'t\x20Coff'+'ee',a0p(0x132af),a0p(0x2c4)+'e','$1,000'+a0p(0x77d2),'IDA\x20Pr'+'o',a0p(0xb307)+a0p(0xf501)+a0p(0x17d36)],a0d={'doge1':[[a0p(0xcdee),a0p(0xbb66)],[a0p(0xcdee),a0p(0xb06c)]],'boy_friend0':[[a0p(0xcdee),a0p(0xbb66)],[a0p(0x16ec5),'60%'],[a0p(0x18399),a0p(0x18399)]],'draw':[['30%',a0p(0x6070)]],'drake':[[a0p(0x7f30),a0p(0xcdee)],[a0p(0x34ae),a0p(0xcdee)]],'two_buttons':
...
...
...
```

At first glance, it looks weird and hard to understand, but with a closer look, a few patterns emerge—like the use of `parseInt()`, the `a0p` function, and some heavy concatenation. The `a0p` function seems to accept an index and return a string for concatenation, making it likely a string decryption function.

We can quickly confirm this by opening the `.html` file in the browser, using browser `Dev Tools` to inspect values. For example, executing `a0p(0x14c8f)+a0p(0x114df)+a0p(0x17cca)+a0p(0xcd68)` will return `'When you find a buffer o'`. 

Since the `a0p` function is called many times for decryption, manually decrypting and replacing each call would be tedious. Instead, let’s see if there’s an existing deobfuscator. Luckily, this JavaScript obfuscation technique isn’t new, and [deobfuscate.relative.im](https://deobfuscate.relative.im/) is an online tool that can handle it for us.

## Deobfuscating JavaScript with [deobfuscate.relative.im](https://deobfuscate.relative.im/)

Simply copy the obfuscated script and paste it into the portal, which will deobfuscate and format it nicely for us. Below is the fully deobfuscated code (with embedded base64 image data removed for readability).

```js
const a0c = [
    'When you find a buffer overflow in legacy code',
    'Reverse Engineer',
    'When you decompile the obfuscated code and it makes perfect sense',
    'Me after a week of reverse engineering',
    'When your decompiler crashes',
    "It's not a bug, it'a a feature",
    "Security 'Expert'",
    'AI',
    "That's great, but can you hack it?",
    'When your code compiles for the first time',
    "If it ain't broke, break it",
    "Reading someone else's code",
    'EDR',
    'This is fine',
    'FLARE On',
    "It's always DNS",
    'strings.exe',
    "Don't click on that.",
    'When you find the perfect 0-day exploit',
    'Security through obscurity',
    'Instant Coffee',
    'H@x0r',
    'Malware',
    '$1,000,000',
    'IDA Pro',
    'Security Expert',
  ],
  a0d = {
    doge1: [
      ['75%', '25%'],
      ['75%', '82%'],
    ],
    boy_friend0: [
      ['75%', '25%'],
      ['40%', '60%'],
      ['70%', '70%'],
    ],
    draw: [['30%', '30%']],
    drake: [
      ['10%', '75%'],
      ['55%', '75%'],
    ],
    two_buttons: [
      ['10%', '15%'],
      ['2%', '60%'],
    ],
    success: [['75%', '50%']],
    disaster: [['5%', '50%']],
    aliens: [['5%', '50%']],
  },
  a0e = {
    'doge1.png':
        'data:image/png;base64, iVBORw0KGgoA...',
    'draw.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'drake.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQAAAQA...',
    'two_buttons.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQ...',
    'fish.jpg':
        'data:binary/red; base64, TVqQAAMAAAAE...',
    'boy_friend0.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'success.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'disaster.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'aliens.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABA...'
    }
function a0f() {
  document.getElementById('caption1').hidden = true
  document.getElementById('caption2').hidden = true
  document.getElementById('caption3').hidden = true
  const a = document.getElementById('meme-template')
  var b = a.value.split('.')[0]
  a0d[b].forEach(function (c, d) {
    var e = document.getElementById('caption' + (d + 1))
    e.hidden = false
    e.style.top = a0d[b][d][0]
    e.style.left = a0d[b][d][1]
    e.textContent = a0c[Math.floor(Math.random() * (a0c.length - 1))]
  })
}
a0f()
const a0g = document.getElementById('meme-image'),
  a0h = document.getElementById('meme-container'),
  a0i = document.getElementById('remake'),
  a0j = document.getElementById('meme-template')
a0g.src = a0e[a0j.value]
a0j.addEventListener('change', () => {
  a0g.src = a0e[a0j.value]
  a0g.alt = a0j.value
  a0f()
})
a0i.addEventListener('click', () => {
  a0f()
})
function a0k() {
  const a = a0g.alt.split('/').pop()
  if (a !== Object.keys(a0e)[5]) {
    return
  }
  const b = a0l.textContent,
    c = a0m.textContent,
    d = a0n.textContent
  if (
    a0c.indexOf(b) == 14 &&
    a0c.indexOf(c) == a0c.length - 1 &&
    a0c.indexOf(d) == 22
  ) {
    var e = new Date().getTime()
    while (new Date().getTime() < e + 3000) {}
    var f =
      d[3] +
      'h' +
      a[10] +
      b[2] +
      a[3] +
      c[5] +
      c[c.length - 1] +
      '5' +
      a[3] +
      '4' +
      a[3] +
      c[2] +
      c[4] +
      c[3] +
      '3' +
      d[2] +
      a[3] +
      'j4' +
      a0c[1][2] +
      d[4] +
      '5' +
      c[2] +
      d[5] +
      '1' +
      c[11] +
      '7' +
      a0c[21][1] +
      b.replace(' ', '-') +
      a[11] +
      a0c[4].substring(12, 15)
    f = f.toLowerCase()
    alert(atob('Q29uZ3JhdHVsYXRpb25zISBIZXJlIHlvdSBnbzog') + f)
  }
}
const a0l = document.getElementById('caption1'),
  a0m = document.getElementById('caption2'),
  a0n = document.getElementById('caption3')
a0l.addEventListener('keyup', () => {
  a0k()
})
a0m.addEventListener('keyup', () => {
  a0k()
})
a0n.addEventListener('keyup', () => {
  a0k()
})
```

The code is now much cleaner and easier to understand, so we can dive right into the reversing process without any issues. But before we do, let’s refactor it a bit to add meaningful variable and function names based on the context. The refactored code will look like this:

```js
const meme_captions_array = [
    'When you find a buffer overflow in legacy code',
    'Reverse Engineer',
    'When you decompile the obfuscated code and it makes perfect sense',
    'Me after a week of reverse engineering',
    'When your decompiler crashes',
    "It's not a bug, it'a a feature",
    "Security 'Expert'",
    'AI',
    "That's great, but can you hack it?",
    'When your code compiles for the first time',
    "If it ain't broke, break it",
    "Reading someone else's code",
    'EDR',
    'This is fine',
    'FLARE On',
    "It's always DNS",
    'strings.exe',
    "Don't click on that.",
    'When you find the perfect 0-day exploit',
    'Security through obscurity',
    'Instant Coffee',
    'H@x0r',
    'Malware',
    '$1,000,000',
    'IDA Pro',
    'Security Expert',
  ],
  meme_image_size_dict = {
    doge1: [
      ['75%', '25%'],
      ['75%', '82%'],
    ],
    boy_friend0: [
      ['75%', '25%'],
      ['40%', '60%'],
      ['70%', '70%'],
    ],
    draw: [['30%', '30%']],
    drake: [
      ['10%', '75%'],
      ['55%', '75%'],
    ],
    two_buttons: [
      ['10%', '15%'],
      ['2%', '60%'],
    ],
    success: [['75%', '50%']],
    disaster: [['5%', '50%']],
    aliens: [['5%', '50%']],
  },
  meme_image_dict = {
    'doge1.png':
        'data:image/png;base64, iVBORw0KGgoA...',
    'draw.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'drake.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQAAAQA...',
    'two_buttons.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQ...',
    'fish.jpg':
        'data:binary/red; base64, TVqQAAMAAAAE...',
    'boy_friend0.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'success.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'disaster.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'aliens.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABA...'
    }
function changeMemeCaption() {
  document.getElementById('caption1').hidden = true
  document.getElementById('caption2').hidden = true
  document.getElementById('caption3').hidden = true
  const meme_template_element = document.getElementById('meme-template')
  var meme_template_name = meme_template_element.value.split('.')[0]
  meme_image_size_dict[meme_template_name].forEach(function (c, index) {
    var caption_element = document.getElementById('caption' + (index + 1))
    caption_element.hidden = false
    caption_element.style.top = meme_image_size_dict[meme_template_name][index][0]
    caption_element.style.left = meme_image_size_dict[meme_template_name][index][1]
    caption_element.textContent = meme_captions_array[Math.floor(Math.random() * (meme_captions_array.length - 1))]
  })
}
changeMemeCaption()
const meme_image_element = document.getElementById('meme-image'),
  meme_container_element = document.getElementById('meme-container'),
  remake_element = document.getElementById('remake'),
  meme_template_element = document.getElementById('meme-template')
meme_image_element.src = meme_image_dict[meme_template_element.value]
meme_template_element.addEventListener('change', () => {
  meme_image_element.src = meme_image_dict[meme_template_element.value]
  meme_image_element.alt = meme_template_element.value
  changeMemeCaption()
})
remake_element.addEventListener('click', () => {
  changeMemeCaption()
})
function validate_conditions_and_show_flag() {
  const meme_image_name = meme_image_element.alt.split('/').pop()
  if (meme_image_name !== Object.keys(meme_image_dict)[5]) {
    return
  }
  const caption1 = caption1_element.textContent,
    caption2 = caption2_element.textContent,
    caption3 = caption3_element.textContent
  if (
    meme_captions_array.indexOf(caption1) == 14 &&
    meme_captions_array.indexOf(caption2) == meme_captions_array.length - 1 &&
    meme_captions_array.indexOf(caption3) == 22
  ) {
    var time = new Date().getTime()
    while (new Date().getTime() < time + 3000) {}
    var flag =
      caption3[3] +
      'h' +
      meme_image_name[10] +
      caption1[2] +
      meme_image_name[3] +
      caption2[5] +
      caption2[caption2.length - 1] +
      '5' +
      meme_image_name[3] +
      '4' +
      meme_image_name[3] +
      caption2[2] +
      caption2[4] +
      caption2[3] +
      '3' +
      caption3[2] +
      meme_image_name[3] +
      'j4' +
      meme_captions_array[1][2] +
      caption3[4] +
      '5' +
      caption2[2] +
      caption3[5] +
      '1' +
      caption2[11] +
      '7' +
      meme_captions_array[21][1] +
      caption1.replace(' ', '-') +
      meme_image_name[11] +
      meme_captions_array[4].substring(12, 15)
    flag = flag.toLowerCase()
    alert(atob('Q29uZ3JhdHVsYXRpb25zISBIZXJlIHlvdSBnbzog') + flag)
  }
}
const caption1_element = document.getElementById('caption1'),
  caption2_element = document.getElementById('caption2'),
  caption3_element = document.getElementById('caption3')
caption1_element.addEventListener('keyup', () => {
  validate_conditions_and_show_flag()
})
caption2_element.addEventListener('keyup', () => {
  validate_conditions_and_show_flag()
})
caption3_element.addEventListener('keyup', () => {
  validate_conditions_and_show_flag()
})
```

## Find the Flag

With the code now easier to understand, we can quickly spot the flag logic in the `validate_conditions_and_show_flag()` function by noting the base64-encoded string `'Q29uZ3JhdHVsYXRpb25zISBIZXJlIHlvdSBnbzog'` which decodes to `'Congratulations! Here you go: '`. Usually, where high-entropy data appears, the flag is nearby—so let’s focus on this `validate_conditions_and_show_flag()` function.

### Demystifying the `validate_conditions_and_show_flag()` Function

This method performs a few validations; if any condition fails, the flag logic won’t be triggered.

First, it checks that the displayed meme image is the `5th` item (0-based) in `meme_image_dict` keys, meaning `meme_image_name` must be `'boy_friend0.jpg'`.

Next, it verifies the following:

- `caption1` must equal `meme_captions_array[14]`, which is `'FLARE On'`.
- `caption2` must equal the last item in `meme_captions_array`, which is `'Security Expert'`.
- `caption3` must equal `meme_captions_array[22]`, which is `'Malware'`.

If all the above conditions are satisfied, it waits for 3 seconds (using a `while` loop) before starting to concatenate the flag by cherry-picking each letter from `meme_image_name`, `caption1`, etc., at specific positions. We don’t need to manually reverse each letter—since we have the JavaScript source code, we can use the browser’s `Dev Tools` to execute the `validate_conditions_and_show_flag()` function right away with the expected conditions and reveal the flag in an alert.

### Triggering the Alert

Let’s update the `validate_conditions_and_show_flag()` function by setting the conditions to be satisfied, removing unnecessary checks and the delay loop. The function will look like this:

```js
function validate_conditions_and_show_flag() {
    const meme_image_name = Object.keys(meme_image_dict)[5]  
    const caption1 = meme_captions_array[14]
    const caption2 = meme_captions_array[meme_captions_array.length - 1]
    const caption3 = meme_captions_array[22]      
    var flag =
        caption3[3] +
      'h' +
      meme_image_name[10] +
      caption1[2] +
      meme_image_name[3] +
      caption2[5] +
      caption2[caption2.length - 1] +
      '5' +
      meme_image_name[3] +
      '4' +
      meme_image_name[3] +
      caption2[2] +
      caption2[4] +
      caption2[3] +
      '3' +
      caption3[2] +
      meme_image_name[3] +
      'j4' +
      meme_captions_array[1][2] +
      caption3[4] +
      '5' +
      caption2[2] +
      caption3[5] +
      '1' +
      caption2[11] +
      '7' +
      meme_captions_array[21][1] +
      caption1.replace(' ', '-') +
      meme_image_name[11] +
      meme_captions_array[4].substring(12, 15)
    flag = flag.toLowerCase()
    alert(atob('Q29uZ3JhdHVsYXRpb25zISBIZXJlIHlvdSBnbzog') + flag)
}
```

Toggle `Dev Tools` on in the browser (I’m using **Chrome** here) and paste this final script into the `Console` tab to reveal the flag:

```js
const meme_captions_array = [
    'When you find a buffer overflow in legacy code',
    'Reverse Engineer',
    'When you decompile the obfuscated code and it makes perfect sense',
    'Me after a week of reverse engineering',
    'When your decompiler crashes',
    "It's not a bug, it'a a feature",
    "Security 'Expert'",
    'AI',
    "That's great, but can you hack it?",
    'When your code compiles for the first time',
    "If it ain't broke, break it",
    "Reading someone else's code",
    'EDR',
    'This is fine',
    'FLARE On',
    "It's always DNS",
    'strings.exe',
    "Don't click on that.",
    'When you find the perfect 0-day exploit',
    'Security through obscurity',
    'Instant Coffee',
    'H@x0r',
    'Malware',
    '$1,000,000',
    'IDA Pro',
    'Security Expert',
  ],
  meme_image_size_dict = {
    doge1: [
      ['75%', '25%'],
      ['75%', '82%'],
    ],
    boy_friend0: [
      ['75%', '25%'],
      ['40%', '60%'],
      ['70%', '70%'],
    ],
    draw: [['30%', '30%']],
    drake: [
      ['10%', '75%'],
      ['55%', '75%'],
    ],
    two_buttons: [
      ['10%', '15%'],
      ['2%', '60%'],
    ],
    success: [['75%', '50%']],
    disaster: [['5%', '50%']],
    aliens: [['5%', '50%']],
  },
  meme_image_dict = {
    'doge1.png':
        'data:image/png;base64, iVBORw0KGgoA...',
    'draw.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'drake.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQAAAQA...',
    'two_buttons.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQ...',
    'fish.jpg':
        'data:binary/red; base64, TVqQAAMAAAAE...',
    'boy_friend0.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABAQA...',
    'success.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'disaster.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgAB...',
    'aliens.jpg':
        'data:image/jpeg;base64, /9j/4AAQSkZJRgABA...'
    }
function validate_conditions_and_show_flag() {
    const meme_image_name = Object.keys(meme_image_dict)[5]  
    const caption1 = meme_captions_array[14]
    const caption2 = meme_captions_array[meme_captions_array.length - 1]
    const caption3 = meme_captions_array[22]      
    var flag =
        caption3[3] +
      'h' +
      meme_image_name[10] +
      caption1[2] +
      meme_image_name[3] +
      caption2[5] +
      caption2[caption2.length - 1] +
      '5' +
      meme_image_name[3] +
      '4' +
      meme_image_name[3] +
      caption2[2] +
      caption2[4] +
      caption2[3] +
      '3' +
      caption3[2] +
      meme_image_name[3] +
      'j4' +
      meme_captions_array[1][2] +
      caption3[4] +
      '5' +
      caption2[2] +
      caption3[5] +
      '1' +
      caption2[11] +
      '7' +
      meme_captions_array[21][1] +
      caption1.replace(' ', '-') +
      meme_image_name[11] +
      meme_captions_array[4].substring(12, 15)
    flag = flag.toLowerCase()
    alert(atob('Q29uZ3JhdHVsYXRpb25zISBIZXJlIHlvdSBnbzog') + flag)
}

validate_conditions_and_show_flag()
```

And with that, the flag is handed over to you!! 🎉
![Alert the flag](https://lh3.googleusercontent.com/pw/AP1GczMbw2IFxLAmj3LbjCFXByMhKsjvhloXgiq90S45QTy8QJuByo73RFDSIzSceqUb0Y48a2puY-vMuwdBR9drguvf0zaNNaoPHXW2BCQSTnjuCCRacC7zPQ6vz9nLSfavNNm86U9WydtC07bhvYua7tdF=w2820-h1612-s-no-gm?authuser=2)
_**Figure: 3 - Alert the flag**_

## Replay with Unmodified Source Code

Some might say we took a shortcut to get the flag — fair enough! Let’s open `mememaker3000.html` in the browser and reproduce the steps manually to keep the critics silent:

- Select `Distracted Boyfriend` from the dropdown list.
- Enter `FLARE On` for the red girl’s caption.
- Enter `Security Expert` for the boyfriend’s caption.
- Enter `Malware` for the other girl’s caption.
- And take a sip of coffee...

![Distracted Boyfriend](https://lh3.googleusercontent.com/pw/AP1GczPn3wfOfkbf2fBhKnD_PAtwd-DimBL71VmUs0LusUo6pC19R_OAGYticxHiKr0isevQNAUHlrGsbsYKQZG7vCz01nHXQM_qKag6jTa7uNMMemebS4AJpooZS-8BkXMLUeIMm02JSXjWLjzG8dhoEtwU=w2156-h1192-s-no-gm?authuser=2)
_**Figure: 4 - Distracted Boyfriend**_


## Conclusion

This challenge felt a bit troublesome at first, but once we removed the obfuscation obstacle, it became a straightforward task to solve. Surprisingly, this felt easier than Challenge 2 — perhaps the author wanted to let us relax a bit before the next challenge. A storm is coming?

## References
- [deobfuscate.relative.im](https://deobfuscate.relative.im/)
- [obfuscator.io](https://obfuscator.io/)
---
layout: post
title: "Cracking the Flare-On 11 CTF 2024: Challenge 2 - Checksum"
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczPuUm_pFiCyEgh6r1Zdbq9r8T07IPkcAVBeWxqFNpeZkUGUcDYEGqoUO77RLIRye4D7wK9TUCcm7_FO0vG7hq9y3TJEYh5soUNjQjfF0pjIdmeSncv-tTEtbaK4OhF5jWJQ1xyPYH9oQkiByKPMvg06=w1954-h1022-s-no-gm?authuser=2
    alt: Challenge 2 - Checksum
tags: [flareon, flareon11, flareon-2024, ctf, windows, golang]
categories: [CTF, Flareon 2024]
---
We recently came across a silly executable that appears benign. It just asks us to do some math... From the strings found in the sample, we suspect there are more to the sample than what we are seeing. Please investigate and let us know what you find!

## Challenge description

>2 - checksum
>
>We recently came across a silly executable that appears benign. It just asks us to do some math... From the strings found in the sample, we suspect there are more to the sample than what we are seeing. Please investigate and let us know what you find!
{: .prompt-info}

## Run the Challenge
Run `checksum.exe`, which will display a console application and prompt us with a series of math questions. If the answers are correct, it will repeat the questions a few times, followed by a **"Checksum"** prompt for input. If any math answers or the checksum input are incorrect, the application terminates without a trace.

## Analysis with Ghidra
For this challenge, we’ll use `Ghidra` for static analysis, as it yields better results than `IDA`. Once we load `checksum.exe` into `Ghidra`, it displays the Ghidra Import Results Summary dialog as shown below:

```
Project File Name:                  checksum.exe
Language ID:                        x86:LE:64:default (4.1)
Compiler ID:                        golang
Processor:                          x86
Endian:                             Little
# of Symbols:                       2368
Golang BuildId:                     xFQuXGyZW2wTW9jhnCpK/61yJviL_t4EUhqtApiAn/8Fno1G69FrZCFysLpoB_/sO-mbcYoNv-6X3MFMSBk
Golang app path:                    flareon/chuong/checksum
Golang dep[   0]:                   golang.org/x/crypto v0.23.0 h1:dIJU/v2J8Mdglj/8rJ6UUOM3Zc9zLZxVZwwxMooUSAI=
Golang dep[   1]:                   golang.org/x/sys v0.20.0 h1:Od9JTbYCk261bKm4M/mw7AklTlFYIa0bIp9BgSm1S8Y=
Golang go version:                  1.22.2
Golang main package path:           flareon/chuong/checksum
Golang main package version:        (devel)
...
```
{: .prompt-info }

As we can see, this console app is written in the `Go` language, version `1.22.2`, and the author of this challenge is `chuong` (more precisely, `chuong.dong`, as you’ll later see in Ghidra with a path like `Golang source: C:/Users/chuong.dong/Exclusions/check_sum/checksum.go:40`).

Once `Ghidra` completes its analysis, let’s hop into the `Strings` window (`Window -> Defined Strings`) and look for where the **`Checksum:`** string is being used.

![Checksum string XREFs](https://lh3.googleusercontent.com/pw/AP1GczN8z9BoVP_ggMybQzah_x63BdJyyZrD9vhtW4p6MCtZVORn1HGAftHRkthM5ikwhudxiQXtdao_YMyIu95h3EtxJu42dbveOHwYKhmeM3okdDgUer8Kh2sq5aAQNwV2X7wMyU4hQ80PBWIbiYbERcXK=w2828-h1234-s-no-gm?authuser=2)
_**Figure: 2 - Checksum string XREFs**_

We can see that the **`Checksum:`** string in the app console is referenced in the `main.main` method. In the **Symbol Tree** on the left panel, alongside the `main.main` function, there are also `main.a` and `main.b` functions. Let’s take a quick look at these as well.

### `main.b` function
```c
void main::main.b(error err,string errorString)
{
  internal/abi.Type *piVar1;
  undefined8 uVar2;
  io.Writer w;
  []interface_{} a;
  error err_spill;
  string errorString_spill;
  internal/abi.Type *local_18;
  void *pvStack_10;
  
  piVar1 = (internal/abi.Type *)0x0;
  uVar2 = 0;
  while (&stack0x00000000 <= CURRENT_G.stackguard0) {
    runtime::runtime.morestack_noctxt();
  }
  if (err.tab != (runtime.itab *)0x0) {
    local_18 = piVar1;
    pvStack_10 = (void *)uVar2;
    pvStack_10 = runtime::runtime.convTstring(errorString);
    local_18 = &string___internal/abi.Type;
    w.data = os.Stdout;
    w.tab = &*os.File__implements__io.Writer___runtime.itab;
    a.len = 1;
    a.array = (interface_{} *)&local_18;
    a.cap = 1;
    fmt::fmt.Fprintln(w,a);
    os::os.Exit(0xdeadbeef);
  }
  return;
}
```

Thanks to the `Ghidra` decompiler, we can clearly see that this is a helper function that checks if the `error err` passed in is not nil. If it’s not nil, it prints the `string errorString` and exits the console app; otherwise, it does nothing. We can rename this function to `main.b_exitIfHasError` for readability.

### `main.a` function
```c
bool main::main.a(string checksum)

{
  bool is_correct_checksum;
  uintptr *puVar2_checksum_str_ptr;
  uint xor_key_index;
  int len;
  int loop_counter;
  string checksum_xored_base64;
  []uint8 checksum_xored_array;
  []uint8 src;
  string checksum_spill;
  
  len = checksum.len;
  puVar2_checksum_str_ptr = (uintptr *)checksum.str;
  
  checksum_xored_array = runtime::runtime.makeslice(&uint8___internal/abi.Type,len,len);
  loop_counter = 0;
  while( true ) {
    if (len <= loop_counter) {
      src.len = len;
      src.array = checksum_xored_array.array;
      src.cap = len;
      checksum_xored_base64 =
           encoding/base64::encoding/base64.(*Encoding).EncodeToString
                     (encoding/base64.StdEncoding,src);
      if (checksum_xored_base64.len == 0x58) {
        is_correct_checksum =
             runtime::runtime.memequal
                       ((undefined (*) [32])checksum_xored_base64.str,
                        (undefined (*) [32])
                        "cQoFRQErX1YAVw1zVQdFUSxfAQNRBXUNAxBSe15QCVRVJ1pQEwd/WFBUAlElCFBFUnlaB1ULByR dBEFdfVtWVA=="
                        ,0x58);
      }
      else {
        is_correct_checksum = false;
      }
      return is_correct_checksum;
    }
    xor_key_index = loop_counter + (loop_counter / 0xb + (loop_counter >> 0x3f)) * -0xb;
    if (10 < xor_key_index) break;
    checksum_xored_array.array[loop_counter] =
         *(byte *)((int)puVar2_checksum_str_ptr + loop_counter) ^
         (&DAT_004c8035_xor_key)[xor_key_index];
    loop_counter = loop_counter + 1;
  }
}
```

This function (with some variables renamed for readability) accepts the `checksum` as a `string`, performs transformations, and validates it against a hardcoded `base64` value to determine if it’s a valid checksum, returning the result as a boolean.

First, it enters a loop and calculates `xor_key_index` based on the formula `xor_key_index = loop_counter + (loop_counter / 0xb + (loop_counter >> 0x3f)) * -0xb;`, where `loop_counter` increments by 1 on each iteration. This can be simplified to `xor_key_index = loop_counter + (loop_counter // 0xb) * -0xb; // loop_counter >> 0x3f will always be zero`, resulting in values `0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 0, 1, 2,` and so on.

Next, it retrieves a byte from the XOR key array `DAT_004c8035_xor_key`. Navigating to this address, we see it’s a hardcoded string: **`FlareOn2024`**. Each byte from the XOR key is used to XOR with each character in checksum input string.

After XOR-ing all characters in the input checksum, it base64-encodes the result and compares it to `cQoFRQErX1YAVw1zVQdFUSxfAQNRBXUNAxBSe15QCVRVJ1pQEwd/WFBUAlElCFBFUnlaB1ULByRdBEFdfVtWVA==` to determine validity.

To simplify this process, we can rewrite this logic in `Python` to find the expected `checksum` string that satisfies the function's requirements.

```python
# decode_checksum.py
import base64

# Constants
target_base64 = "cQoFRQErX1YAVw1zVQdFUSxfAQNRBXUNAxBSe15QCVRVJ1pQEwd/WFBUAlElCFBFUnlaB1ULByRdBEFdfVtWVA=="
xor_key = b"FlareOn2024"  # ASCII values of the string 'FlareOn2024'

def decode_checksum(transformed):
    checksum = bytearray(len(transformed))
    for i, byte in enumerate(transformed):
        # Apply the reverse transformation logic (same index calculation but reverse XOR)
        xor_key_index = i + (i // 0xb + (i >> 0x3f)) * -0xb        
        if xor_key_index >= len(xor_key):
            raise IndexError("Index out of range in XOR key")
        checksum[i] = byte ^ xor_key[xor_key_index]
    return checksum

def recover_checksum():
    # Step 1: Decode the target base64 string
    transformed_bytes = base64.b64decode(target_base64)

    # Step 2: Reverse the transformation to get the original checksum string
    original_checksum_bytes = decode_checksum(transformed_bytes)

    # Step 3: Convert the byte array back to a string
    original_checksum = original_checksum_bytes.decode('utf-8')

    return original_checksum

# Run the function
checksum = recover_checksum()
print(f"Recovered checksum: {checksum}")
```

Now, let’s run the Python script to recover the checksum input string that we need to enter:
```bashscript
$ python3 decode_checksum.py
Recovered checksum: 7fd7dd1d0e959f74c133c13abb740b9faa61ab06bd0ecd177645e93b1e3825dd
```

If we use this input to test the console app again, it still exits after the checksum input without showing any output. It’s time to trace back how `main.a` is being used. Before that, let’s rename this method to `main.a_validate_checksum`.

### `main.main` Function
Let’s trace the XREFs of `main.a_validate_checksum`. It’s being called from the `main.main` function. Below is the snippet where it’s called:

```c
if (local_188 <= iVar2) {
    file_full_path = runtime::runtime.slicebytetostring((runtime.tmpBuf *)local_1e8,ptr,local_168)
    ;
    if (file_full_path.len == local_10->len) {
        is_valid_checksum =
                runtime::runtime.memequal
                        ((undefined (*) [32])file_full_path.str,(undefined (*) [32])local_10->str,
                        local_10->len);
        if (is_valid_checksum) {
            is_valid_checksum = main.a_validate_checksum(*local_10);
        }
        else {
            is_valid_checksum = false;
        }
    }
    else {
        is_valid_checksum = false;
    }
    if (is_valid_checksum == false) {
        local_88 = &string___internal/abi.Type;
        psStack_80 = &gostr_Maybe_it's_time_to_analyze_the_b;
        w_01.data = os.Stdout;
        w_01.tab = &*os.File__implements__io.Writer___runtime.itab;
        a_03.len = 1;
        a_03.array = (interface_{} *)&local_88;
        a_03.cap = 1;
        fmt::fmt.Fprintln(w_01,a_03);
    }

    userCacheDir = os::os.UserCacheDir();
    userCacheDir_len = userCacheDir.~r0.len;
    local_a8 = userCacheDir.~r0;
    errorString_04.len = 0x13;
    errorString_04.str = (uint8 *)"Fail to get path...";
    main.b_exitIfHasError(userCacheDir.~r1,errorString_04);
    flag_file_name.len = 0x16;
    flag_file_name.str = (uint8 *)"\\REAL_FLAREON_FLAG.JPG";
    a0.len = userCacheDir_len;
    a0.str = local_a8;
    file_full_path = runtime::runtime.concatstring2((runtime.tmpBuf *)0x0,a0,flag_file_name);
    file_data.len = local_180;
    file_data.array = (uint8 *)local_b0;
    file_data.cap = local_178;
    eVar8 = os::os.WriteFile(file_full_path,file_data,0x1a4);

    errorString_05.len = 0x15;
    errorString_05.str = (uint8 *)"Fail to write file...";
    main.b_exitIfHasError(eVar8,errorString_05);
    local_98 = &string___internal/abi.Type;
    psStack_90 = &gostr_Noice!!;
    w_02.data = os.Stdout;
    w_02.tab = &*os.File__implements__io.Writer___runtime.itab;
    a_04.len = 1;
    a_04.array = (interface_{} *)&local_98;
    a_04.cap = 1;
    fmt::fmt.Fprintln(w_02,a_04);
    return;
}
```

As we can see, once the checksum is validated, it concatenates the `os::os.UserCacheDir()` path with the file name `REAL_FLAREON_FLAG.JPG` and writes the file at that location. On Windows, you can find it at `C:\Users\[CURRENT_USER]\AppData\Local\REAL_FLAREON_FLAG.JPG`.

Navigate to that location, and you’ll find the newly created image with the FLAGGGGGGGGG: `Th3_M4tH_Do_b3_mAth1ng@flare-on.com`


![The flag](https://lh3.googleusercontent.com/pw/AP1GczMMDqbAZAHs1L3aL_LW1zwtPoz_S4WQo_wDQHFOU4MsZO07LyvF-ON7t5qCXt7HiXmeEdEpoY8SkbcWzUGEcW2n3Tt5g4I3hRDAGg4v6-0kjPgp-WhP5jJgaBfAMllT25D3GGSULGSjHnOgr5a3V90r=w1403-h920-s-no-gm?authuser=2)
_**Figure: 3 - The flag**_

### How about the remaining logic of `main.main` function?
A top-down approach isn’t always ideal; this time, we used a bottom-up approach, tracing back to avoid *unnecessary* logic while still retrieving the flag. If you want to understand the entire app’s logic, I’ll leave that exploration up to you! :P

## Conclusion
This challenge involves a bit more reversing compared to the first one, as we need to dig into the binary without source code. Additionally, the app is written in Golang, but luckily, the symbols aren’t entirely stripped, and `Ghidra` provides excellent decompilation, making the reversing process smooth.

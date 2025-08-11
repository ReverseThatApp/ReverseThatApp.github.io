---
layout: post
title: "Beginner's Guide to Disabling ASLR in iOS Apps"
categories: [Kernel, Security]
tags: [kernel, ios, xnu, security, aslr, launchd, jailbreak]
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczN4VkxKSRr_zttj5XnvFBVEU-xNObJAGtVNNruzN4m7BEHdeqsQlhDTRgXQiMvEHFA0vDzsG13F5upW-cGT3VvSnlRFfT4oUMD-dJT_391xk1pI8AAdBno_6DYHdQG7jLsmxtsBOdi5zLe7t1iuWU_b=w1830-h966-s-no-gm
    alt: Disabling ASLR on iOS
---

Address Space Layout Randomization (ASLR) is a security feature in iOS that randomizes where apps load in memory, making it harder for attackers to exploit vulnerabilities. For security researchers, however, disabling ASLR can simplify analyzing apps by ensuring consistent memory layouts. In this guide, we'll walk through how to disable ASLR for all iOS apps by modifying the `launchd` process, which launches all user apps. This process, known as patching, requires a jailbroken device and some technical know-how, but we'll break it down step-by-step to make it approachable.

**Important**: This is for research purposes only on a jailbroken test device. Disabling ASLR reduces security, so proceed with caution and never do this on a personal device.

## What You'll Need

Before diving in, let’s ensure you have the right tools and setup:

- **Jailbroken iOS Device**: Your device must be jailbroken with `tfp0` (task_for_pid(0)) enabled, allowing kernel access.
- **Tools**:
  - `ipsw`: To download iOS firmware.
  - IDA: For analyzing the iOS kernel.
  - LLDB: For verifying app ASLR only.
  - `clang` and `ldid`: For compiling and signing code.
- **SSH Access**: To interact with your device remotely.
- **Basic Knowledge**: Familiarity with terminal commands and C programming helps, but I’ll explain key concepts.

## Step 1: Find Your Device’s Kernel Version

To start, you need to know your device’s XNU kernel version, as this guides our analysis. First, connect to your jailbroken device via SSH and run:

```bash
iPhone $ uname -a
Darwin iPhone 22.6.0 Darwin Kernel Version 22.6.0: Tue Jul  2 20:47:35 PDT 2024; root:xnu-8796.142.1.703.8~1/RELEASE_ARM64_T8015 iPhone10,4 arm Darwin
```

Here, the XNU version is `xnu-8796.142.1.703.8~1`. Next, visit [Apple’s open-source XNU repository](https://github.com/apple-oss-distributions/xnu/releases) and download the closest matching version, such as `xnu-8796.141.3`. This source code helps us understand how ASLR works in iOS.

![XNU Source Code](https://lh3.googleusercontent.com/pw/AP1GczMvGxjoUrRAheorQXLsmodYBHVyx2ofrvgDgeQrA_iE25-QlpAauybzDDm5o99gVviyVnZ3eIIsjFXEqBd4NbNc6XaB4JiNKb1Wyp6fogm-vX2a5N7c9ZTZunvU0rqCwE0uvpGM9PWpw8O8hm0cxC9K=w2126-h1256-s-no-gm)
*Figure: Browsing XNU Source Code Releases*

## Step 2: Explore the XNU Source Code

Now, let’s dig into the XNU source to understand ASLR. Open the downloaded source in a text editor like Visual Studio Code and search for "ASLR" You’ll find references in `kern_exec.c`, particularly the `P_DISABLE_ASLR` flag, defined in `proc.h` as:

```c
#define P_DISABLE_ASLR  0x00001000  /* Disable address space layout randomization */
```
![XNU P_DISABLE_ASLR](https://lh3.googleusercontent.com/pw/AP1GczPnapSZP4cYybifD24E1vSEYgfQSjiLfhuxmXBHkKdMTSlywZrHN7MaCow-r-BFD0iJSLOVGITSWMSfO_9ywhlH6WWrmqGejMjuWwGotRagDqWdMcrPRbveCl7J6dKoRi6Y7ieMj7p4Ylewaky1cMvt=w2126-h1372-s-no-gm)
*Figure: Locating P_DISABLE_ASLR in XNU Source*

This flag determines whether a process uses ASLR. Since `launchd` (PID 1) launches all user apps, setting this flag in `launchd`’s process structure disables ASLR for all apps it spawns.

### Understanding the Process Structure

The `struct proc` in `proc_internal.h` holds process details, including:

- `p_pid`: The process ID (e.g., 1 for `launchd`).
- `p_name`: The process name (e.g., "launchd").
- `p_flag`: Flags controlling behavior, like `P_DISABLE_ASLR`.

Additionally, XNU defines `kernproc` (the kernel process) and `initproc` (PID 1, `launchd`) as global variables, which we’ll use to locate `launchd` in memory.

```c
// bsd_init.c
SECURITY_READ_ONLY_LATE(proc_t) kernproc;
proc_t XNU_PTRAUTH_SIGNED_PTR("initproc") initproc;

void
bsd_utaskbootstrap(void)
{
	thread_t thread;
	struct uthread *ut;

	/*
	 * Clone the bootstrap process from the kernel process, without
	 * inheriting either task characteristics or memory from the kernel;
	 */
	thread = cloneproc(TASK_NULL, NULL, kernproc, CLONEPROC_FLAGS_MEMSTAT_INTERNAL);

	/* Hold the reference as it will be dropped during shutdown */
	initproc = proc_find(1);
    ...
}

```

## Step 3: Get the iOS Firmware

To analyze the kernel, we need the firmware for your device. For example, if you’re using an iPhone10,4 on iOS 16.7.11, download it with the `ipsw` tool:

```bash
$ ipsw download ipsw --version 16.7.11 --device iPhone10,4
   • Getting IPSW              build=20H360 device=iPhone10,4 signed=true version=16.7.11
	5.62 GiB / 5.62 GiB [==========================================================| ✅  ] 33.17 MiB/s
      • verifying sha1sum...
   • Created: iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
$ ls
checksums.txt.sha1                        
iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
```

This downloads a file like `iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw`. Next, extract the kernel cache:

```bash
$ ipsw extract --kernel iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
   • Extracting kernelcache
      • Created 20H360__iPhone10,1_4/kernelcache.release.iPhone10,1_4
$ ls
20H360__iPhone10,1_4
$ ls 20H360__iPhone10,1_4
kernelcache.release.iPhone10,1_4
```

This creates `kernelcache.release.iPhone10,1_4`, ready for analysis in IDA Pro.

![IDA Loading Kernel Cache](https://lh3.googleusercontent.com/pw/AP1GczNuDEuKx2DbK7kMez10pU2bZjEqcaFkkN_H6pWa2J63ILzgbki3e-vYFSlzdwv8_a3SKcG5kwX_E1oToY_41uXuE6Sw6Thh8o0nF6_CnXfSkuJSm9ZcaTKSSKtwDOEjAmaa96mdsT0e5wXdLftgfNfR=w1564-h1064-s-no-gm)
*Figure: Kernel Cache in IDA Pro*

## Step 4: Analyze the Kernel in IDA

Load the kernel cache into IDA Pro and check the `Exports` tab. You’ll find `kernproc` at `0xFFFFFFF0071997E0` (the kernel process), however you can't find `initproc`

### Finding initproc
- Searching `initproc` in the IDA `Exports` tab, we found this exported symbol `int __cdecl proc_isinitproc(proc_t) at 0xFFFFFFF0075CDF70`
```c
int __cdecl proc_isinitproc(proc_t a1)
{
  return qword_FFFFFFF0078B7CD0 && qword_FFFFFFF0078B7CD0 == (_QWORD)a1;
}
``` 

Compare with the XNU source code `qword_FFFFFFF0078B7CD0` must be `initproc` which is `launchd` proccess.
```c
int
proc_isinitproc(proc_t p)
{
	if (initproc == NULL) {
		return 0;
	}
	return p == initproc;
}
```

### Finding Key Offsets

To modify `launchd`, we need the memory offsets for `p_flag` and `p_name` in `struct proc`. By cross-referencing XNU source and IDA:

#### **p_flag Offset**: 
The function `proc_issetugid` in `kern_prot.c` accesses `p_flag` at offset `1108` bytes (found in IDA as 277 * 4).
By searching `->p_flag` in XNU source code, we found plenty of results, we can filter out by finding the result that is in a simple method and that method can be found in IDA, in this case found this:
```c
/// XNU kern_prot.c
int
proc_issetugid(proc_t p)
{
	return (p->p_flag & P_SUGID) ? 1 : 0;
}

// in IDA:
int __cdecl proc_issetugid(proc_t p)
{
  return (*((_DWORD *)p + 277) >> 8) & 1;
}
```

It's similar, it casts `p` to a pointer to 32‑bit integers, calculate offset `277 * 4 = 1108` bytes from the start of `struct proc` which is the location corresponds to `p_flag` inside `struct proc`, hence we know `p_flag` offset is `1108` bytes from the start of `struct proc`

#### **p_name Offset**: 
The function `proc_best_name` in `kern_proc.c` shows `p_name` at offset `1401` bytes.
```c
// XNU kern_proc.c
char *
proc_best_name(proc_t p)
{
	if (p->p_name[0] != '\0') {
		return &p->p_name[0];
	}
	return &p->p_comm[0];
}

// in IDA
char *__cdecl proc_best_name(proc_t p)
{
  if ( *((_BYTE *)p + 1401) )
    return (char *)p + 1401;
  else
    return (char *)p + 1384;
}
``` 

These offsets let us read and modify `launchd`’s process details.

## Step 5: Patch `launchd` to Disable ASLR

Now, let’s write a program to patch `launchd`. This requires kernel access, which we get using `tfp0`.

### Get Kernel Access (tfp0)

Most jailbreaks enable `task_for_pid(0)` (`tfp0`), allowing us to access the kernel task. Create a C program, `disable_aslr.c`:

```c
#include <mach/mach.h>
#include <stdio.h>

int main() {
    mach_port_t kernel_task;
    kern_return_t ret = task_for_pid(mach_task_self(), 0, &kernel_task);
    if (ret != KERN_SUCCESS) {
        printf("Failed to get tfp0: %d\n", ret);
        return 1;
    }
    printf("Got tfp0: %u\n", kernel_task);
    return 0;
}
```

To run this, sign it with special permissions (entitlements):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>get-task-allow</key>
    <true/>
    <key>run-unsigned-code</key>
    <true/>
    <key>task_for_pid-allow</key>
    <true/>
</dict>
</plist>
```

Compile, sign, and copy to the device:

```bash
$ cc disable_aslr.c -o disable_aslr && ldid -Stfp0.plist disable_aslr && scp -P 2222 disable_aslr root@localhost:/usr/bin
```

Test it via SSH:

```bash
$ ssh -p 2222 root@localhost
root@localhost's password:
iPhone $ disable_aslr
Got tfp0: 6915
```

You should see a `tfp0` port number, confirming kernel access.

### Find the KASLR Slide

The Kernel Address Space Layout Randomization (KASLR) slide shifts kernel addresses. For `palera1n` jailbreaks, check `/cores/jbinit.log` for the slide (e.g., `0xA9BC000`). Alternatively, use code like this [existing code](https://github.com/Cryptiiiic/x8A4/blob/main/Kernel/slide.c#L57) to retrieve it from palera1n ramdisk `/dev/rmd0`.

```c
/**
 * @brief           Get kaslr slide from palera1n ramdisk
 * @return          Kaslr slide
 */
uint64_t palera1n_get_slide(void) {
    int verbose_cached = 1;
  uint64_t slide = 0;
  int rmd0 = open("/dev/rmd0", O_RDONLY, 0);
  if (rmd0 < 0) {
    printf("Could not get paleinfo! (%d:%s:%d:%s)\n", rmd0, strerror(rmd0), errno, strerror(errno));
    return 0;
  }
  uint64_t off = lseek(rmd0, 0, SEEK_SET);
  if (off == -1) {
    printf("Failed to lseek ramdisk to 0\n", "");
    close(rmd0);
    return 0;
  }
  uint32_t pinfo_off;
  ssize_t didRead = read(rmd0, &pinfo_off, sizeof(uint32_t));
  if (didRead != (ssize_t)sizeof(uint32_t)) {
    printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(uint32_t));
    close(rmd0);
    return 0;
  }
  off = lseek(rmd0, pinfo_off, SEEK_SET);
  if (off != pinfo_off) {
    printf("Failed to lseek ramdisk to %u\n", pinfo_off);
    close(rmd0);
    return 0;
  }
  struct paleinfo {
    uint32_t magic; /* 'PLSH' */
    uint32_t version; /* 2 */
    uint64_t kbase; /* kernel base */
    uint64_t kslide; /* kernel slide */
    uint64_t flags; /* unified palera1n flags */
    char rootdev[0x10]; /* ex. disk0s1s8 */
                        /* int8_t loglevel; */
  } __attribute__((packed));
  struct paleinfo_legacy {
    uint32_t magic;   // 'PLSH' / 0x504c5348
    uint32_t version; // 1
    uint32_t flags;
    char rootdev[0x10];
  };
  struct paleinfo *pinfo_p = (struct paleinfo *)calloc(1, sizeof(struct paleinfo));
  struct paleinfo_legacy *pinfo_legacy_p = NULL;
  didRead = read(rmd0, pinfo_p, sizeof(struct paleinfo));
  if (didRead != (ssize_t)sizeof(struct paleinfo)) {
    printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(struct paleinfo));
    close(rmd0);
    free(pinfo_p);
    return 0;
  }
  if (pinfo_p->magic != 'PLSH') {
    close(rmd0);
    pinfo_off += 0x1000;
    pinfo_legacy_p = (struct paleinfo_legacy *)calloc(1, sizeof(struct paleinfo_legacy));
    didRead = read(rmd0, pinfo_legacy_p, sizeof(struct paleinfo_legacy));
    if (didRead != (ssize_t)sizeof(struct paleinfo_legacy)) {
      printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(struct paleinfo_legacy));
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    if(verbose_cached) {
      printf("pinfo_legacy_p->magic: %s\n", (char *)&pinfo_legacy_p->magic);
      printf("pinfo_legacy_p->magic: 0x%X\n", pinfo_legacy_p->magic);
      printf("pinfo_legacy_p->version: 0x%Xd\n", pinfo_legacy_p->version);
      printf("pinfo_legacy_p->flags: 0x%X\n", pinfo_legacy_p->flags);
      printf("pinfo_legacy_p->rootdev: %s\n", pinfo_legacy_p->rootdev);
    }
    if (pinfo_legacy_p->magic != 'PLSH') {
      printf("Detected corrupted paleinfo!\n", "");
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    if (pinfo_legacy_p->version != 1U) {
      printf("Unexpected paleinfo version: %u, expected %u\n", pinfo_legacy_p->version, 1U);
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    lseek(rmd0, pinfo_off - 0x1000, SEEK_SET);
    struct kerninfo {
      uint64_t size;
      uint64_t base;
      uint64_t slide;
      uint32_t flags;
    };
    struct kerninfo *kerninfo_p = malloc(sizeof(struct kerninfo));
    read(rmd0, kerninfo_p, sizeof(struct kerninfo));
    close(rmd0);
    slide = kerninfo_p->slide;
    free(kerninfo_p);
  } else {
    if(verbose_cached) {
      printf("pinfo_p->magic: %s\n", (const char *)&pinfo_p->magic);
      printf("pinfo_p->magic: 0x%X\n", pinfo_p->magic);
      printf("pinfo_p->version: 0x%Xd\n", pinfo_p->version);
      printf("pinfo_p->kbase: 0x%llX\n", pinfo_p->kbase);
      printf("pinfo_p->kslide: 0x%llX\n", pinfo_p->kslide);
      printf("pinfo_p->flags: 0x%llX\n", pinfo_p->flags);
      printf("pinfo_p->rootdev: %s\n", pinfo_p->rootdev);
      slide = pinfo_p->kslide;
    }
  }
  free(pinfo_p);
  free(pinfo_legacy_p);
  return slide;
}

int main(void) {
    ...
    // Step 2: Read KASLR from ramdisk
    uint64_t kASLR = palera1n_get_slide();
    printf("KASLR: 0x%llX\n", kASLR);
}
```

```bash
iPhone $ disable_aslr
Got tfp0: 5891
pinfo_p->magic: HSLP
pinfo_p->magic: 0x504C5348
pinfo_p->version: 0x2d
pinfo_p->kbase: 0xFFFFFFF00CA58000
pinfo_p->kslide: 0x5A54000
pinfo_p->flags: 0xC00081
pinfo_p->rootdev: disk1s8
KASLR: 0x5A54000
```

For other kind of jailbreak, you may use [all_image_info_size](https://github.com/bazad/memctl/blob/master/src/libmemctl/kernel_slide.c#L468) to retrieve the KASLR value.

### Add Kernel Read/Write Helpers

To modify kernel memory, add these functions to `disable_aslr.c`:

```c
kern_return_t kread(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_read_overwrite(kernel_task, addr, size, (vm_address_t)buf, (vm_size_t *)&size);
}

kern_return_t kwrite(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_write(kernel_task, addr, (vm_offset_t)buf, size);
}
```

### Locate launchd

Read the `initproc` address (adjusted by the KASLR slide) to find `launchd`’s `proc` structure:

```c
#define LAUNCHD_PROC_ADDRESS 0xFFFFFFF0078B7CD0 // qword_FFFFFFF0078B7CD0 found in proc_isinitproc(proc_t a1)
int main(void) {
    ...
    // Read launchd address
    uint64_t launchd_proc_ptr_addr = LAUNCHD_PROC_ADDRESS + kASLR;
    uint64_t launchd_proc_addr;
    ret = kread(kernel_task, launchd_proc_ptr_addr, &launchd_proc_addr, sizeof(launchd_proc_addr));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read launchd_proc_ptr_addr at 0x%llx: %s\n", launchd_proc_ptr_addr, mach_error_string(ret));
        return 1;
    }
    printf("launchd proc address: 0x%llX\n", launchd_proc_addr);
}
```

```bash
iPhone $ disable_aslr
Got tfp0: 3843
pinfo_p->magic: HSLP
pinfo_p->magic: 0x504C5348
pinfo_p->version: 0x2d
pinfo_p->kbase: 0xFFFFFFF0119C0000
pinfo_p->kslide: 0xA9BC000
pinfo_p->flags: 0x40C00081
pinfo_p->rootdev: disk1s8
KASLR: 0xA9BC000
launchd proc address: 0xFFFFFFE322219640
```

### Verify It’s `launchd`

Check the process name at offset `1401` to confirm it’s `launchd`:

```c
#define P_NAME_OFFSET 1401 // found in proc_best_name()
int main(void) {
    ...
    // Verify if it's launchd
    uint64_t p_name_addr = launchd_proc_addr + P_NAME_OFFSET;
    char proc_name[50];
    ret = kread(kernel_task, p_name_addr, proc_name, sizeof(proc_name));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_name at 0x%llx: %s\n", p_name_addr, mach_error_string(ret));
        return 1;
    }
    printf("launchd proc name: %s\n", proc_name);
}
```

```bash
iPhone $ disable_aslr
Got tfp0: 3843
pinfo_p->magic: HSLP
pinfo_p->magic: 0x504C5348
pinfo_p->version: 0x2d
pinfo_p->kbase: 0xFFFFFFF0119C0000
pinfo_p->kslide: 0xA9BC000
pinfo_p->flags: 0x40C00081
pinfo_p->rootdev: disk1s8
KASLR: 0xA9BC000
launchd proc address: 0xFFFFFFE322219640
launchd proc name: launchd
```

### Patch the ASLR Flag

Read `p_flag` at offset `1108`, set the `P_DISABLE_ASLR` bit, and write it back:

```c
#define P_FLAG_OFFSET 1108 // found in proc_issetugid()
// ============ P_FLAG HELPER ============
// got these from XNU proc.h
#define P_ADVLOCK       0x00000001      /* Process may hold POSIX adv. lock */
#define P_CONTROLT      0x00000002      /* Has a controlling terminal */
#define P_LP64          0x00000004      /* Process is LP64 */
#define P_NOCLDSTOP     0x00000008      /* No SIGCHLD when children stop */

#define P_PPWAIT        0x00000010      /* Parent waiting for chld exec/exit */
#define P_PROFIL        0x00000020      /* Has started profiling */
#define P_SELECT        0x00000040      /* Selecting; wakeup/waiting danger */
#define P_CONTINUED     0x00000080      /* Process was stopped and continued */

#define P_SUGID         0x00000100      /* Has set privileges since last exec */
#define P_SYSTEM        0x00000200      /* Sys proc: no sigs, stats or swap */
#define P_TIMEOUT       0x00000400      /* Timing out during sleep */
#define P_TRACED        0x00000800      /* Debugged process being traced */

#define P_DISABLE_ASLR  0x00001000      /* Disable address space layout randomization */
#define P_WEXIT         0x00002000      /* Working on exiting */
#define P_EXEC          0x00004000      /* Process called exec. */

struct flag_def {
    uint32_t value;
    const char *name;
};

struct flag_def flags[] = {
    {P_ADVLOCK,       "P_ADVLOCK"},
    {P_CONTROLT,      "P_CONTROLT"},
    {P_LP64,          "P_LP64"},
    {P_NOCLDSTOP,     "P_NOCLDSTOP"},
    {P_PPWAIT,        "P_PPWAIT"},
    {P_PROFIL,        "P_PROFIL"},
    {P_SELECT,        "P_SELECT"},
    {P_CONTINUED,     "P_CONTINUED"},
    {P_SUGID,         "P_SUGID"},
    {P_SYSTEM,        "P_SYSTEM"},
    {P_TIMEOUT,       "P_TIMEOUT"},
    {P_TRACED,        "P_TRACED"},
    {P_DISABLE_ASLR,  "P_DISABLE_ASLR"},
    {P_WEXIT,         "P_WEXIT"},
    {P_EXEC,          "P_EXEC"},
};

void print_p_flag_set(uint32_t p_flag) {
    printf("p_flag = 0x%08X\n", p_flag);

    printf("Set flags:\n");
    for (size_t i = 0; i < sizeof(flags)/sizeof(flags[0]); i++) {
        if (p_flag & flags[i].value) {
            printf(" - %s\n", flags[i].name);
        }
    }
}
// ============ END P_FLAG HELPER ============

int main(void) {
    ...
    // Read current launchd p_flag
    uint64_t p_flag_addr = launchd_proc_addr + P_FLAG_OFFSET;
    uint32_t current_p_flag = 0;    
    ret = kread(kernel_task, p_flag_addr, &current_p_flag, sizeof(current_p_flag));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_flag at 0x%llx: %s\n", p_flag_addr, mach_error_string(ret));
        return 1;
    }
    printf("[+] Current p_flag: 0x%x\n", current_p_flag);
    print_p_flag_set(current_p_flag);

    // Patch launchd with P_DISABLE_ASLR
    uint32_t disable_aslr_p_flag = current_p_flag | P_DISABLE_ASLR;
    // uint32_t enable_aslr_p_flag = current_p_flag & ~P_DISABLE_ASLR; // use this if you want to enable ASLR again
    kwrite(kernel_task, p_flag_addr, &disable_aslr_p_flag, sizeof(disable_aslr_p_flag));

    // print out new p_flag
    uint32_t new_p_flag = 0;    
    ret = kread(kernel_task, p_flag_addr, &new_p_flag, sizeof(new_p_flag));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_flag at 0x%llx: %s\n", p_flag_addr, mach_error_string(ret));
        return 1;
    }
    printf("[+] Patched p_flag: 0x%x\n", new_p_flag);
}
```

```bash
iPhone $ disable_aslr
Got tfp0: 6659
pinfo_p->magic: HSLP
pinfo_p->magic: 0x504C5348
pinfo_p->version: 0x2d
pinfo_p->kbase: 0xFFFFFFF0119C0000
pinfo_p->kslide: 0xA9BC000
pinfo_p->flags: 0x40C00081
pinfo_p->rootdev: disk1s8
KASLR: 0xA9BC000
launchd proc address: 0xFFFFFFE322219640
launchd proc name: launchd
[+] Current p_flag: 0x4004
p_flag = 0x00004004
Set flags:
 - P_LP64
 - P_EXEC
[+] Patched p_flag: 0x5004
p_flag = 0x00005004
Set flags:
 - P_LP64
 - P_DISABLE_ASLR
 - P_EXEC
```

## Step 6: Confirm ASLR Is Disabled

Launch any app and attach LLDB. Run:

```bash
(lldb) image list -o -f
```

If ASLR is disabled, the app’s memory slide will be `0x0000000000000000`. If ASLR is enabled, you’ll see a non-zero value, like `0x0000000000460000`.

![ASLR Disabled](https://lh3.googleusercontent.com/pw/AP1GczOLdxWqFBcvY1_cq3i34NhR1URbiExtuc4yCveOXCdn_N1jE33wq7iLZxXmKnrQaL83WV-4XoMCLvSLcx-9vk3B4245L4H7MZfscPHABp87dggl9fTF0j-w9a2q0bJJnzMMErKtwf_iLQKUSapPBaGm=w2126-h1366-s-no-gm)
*Figure: ASLR Disabled in LLDB*

![ASLR Enabled](https://lh3.googleusercontent.com/pw/AP1GczM9cxRRaAD4RZ4VI6snx31OGFjZzSAHVD5OvRw68X-JoxvxnJotY-bkA2HImhavCqW2rkcF7f9TyO9X-MZbDgJD0B5FEkNrjha5E8ZOX6S1VSPztrWZk8jTuAGeYpndV_TF9RmdiWrESggs2MSfCYHN=w2126-h1374-s-no-gm)
*Figure: ASLR Enabled in LLDB*

## How About Lauching An Executable From The Shell?
The `P_DISABLE_ASLR` flag is inherited by child processes spawned by `launchd` (PID 1). However, when you run an executable from a shell (e.g., `bash` or `zsh`), it’s typically launched by the shell process, not directly by `launchd`. The shell itself is a child of `launchd`, but if the shell was running before you patched `launchd`, it won’t have the `P_DISABLE_ASLR` flag set in its `proc` structure. Consequently, executables launched from the shell inherit the shell’s flags, which may still enable ASLR.

### Why It Happens
- `launchd` only passes P_DISABLE_ASLR to new child processes spawned after the patch.
- If the shell (e.g., `bash`) was started before the patch, its `p_flag` lacks `P_DISABLE_ASLR`.
- When you run `./my_executable` in the shell, the shell forks and execs the executable, passing its own `p_flag` (with ASLR enabled) to the new process.

### How to Verify
Compile below test program `test_aslr.c` and copy to the device for testing:
```c
// test_aslr.c
#include <stdio.h>
#include <stdlib.h>

int global_var = 123; // Global variable in data segment

int main(void) {
    printf("Address of code  : %p\n", (void *)main);
    printf("Address of global: %p\n", (void *)&global_var);
    return 0;
}
```

```bash
$ cc test_aslr.c -o test_aslr && scp -P 2222 test_aslr root@localhost:/usr/bin

# testing
iPhone $ test_aslr
Address of code  : 0x1049784f8
Address of global: 0x104980000
iPhone $ test_aslr
Address of code  : 0x102a904f8
Address of global: 0x102a98000
iPhone $ test_aslr
Address of code  : 0x104b8c4f8
Address of global: 0x104b94000
```

After running a few times, you can observe the addresses changed for each execution, which means ASLR is still enabled.

### Disable ASLR For The Shell
Restart the shell to make it a new child of the patched `launchd`. After patching `launchd`, close your SSH session and reconnect, or kill and restart the shell:
```bash
iPhone $ killall zsh
iPhone $ zsh

# testing again
iPhone $ test_aslr
Address of code  : 0x1000004f8
Address of global: 0x100008000
iPhone $ test_aslr
Address of code  : 0x1000004f8
Address of global: 0x100008000
iPhone $ test_aslr
Address of code  : 0x1000004f8
Address of global: 0x100008000
```

It worked as expected!!

## Wrapping Up

Congratulations! By patching `launchd` to set `P_DISABLE_ASLR`, you’ve disabled ASLR for all iOS apps, making reverse engineering easier. However, this reduces your device’s security, so use this technique only on a test device. With practice, you’ll get comfortable navigating XNU source, analyzing kernel caches, and applying kernel patches.

You can find the complete `disable_aslr.c` codes as below:
```c
// disable_aslr.c
#include <mach/mach.h>
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#include <unistd.h>
#include <ctype.h>

#define P_PID_OFFSET  0x68
#define P_FLAG_OFFSET 1108 // found in proc_issetugid()
#define P_NAME_OFFSET 1401 // found in proc_best_name()
#define LAUNCHD_PROC_ADDRESS 0xFFFFFFF0078B7CD0 // qword_FFFFFFF0078B7CD0 found in proc_isinitproc(proc_t a1)
#define KERNPROC_ADDRESS 0xFFFFFFF0071997E0

// ============ P_FLAG HELPER ============
// got these from XNU proc.h
#define P_ADVLOCK       0x00000001
#define P_CONTROLT      0x00000002
#define P_LP64          0x00000004
#define P_NOCLDSTOP     0x00000008
#define P_PPWAIT        0x00000010
#define P_PROFIL        0x00000020
#define P_SELECT        0x00000040
#define P_CONTINUED     0x00000080
#define P_SUGID         0x00000100
#define P_SYSTEM        0x00000200
#define P_TIMEOUT       0x00000400
#define P_TRACED        0x00000800
#define P_DISABLE_ASLR  0x00001000
#define P_WEXIT         0x00002000
#define P_EXEC          0x00004000

struct flag_def {
    uint32_t value;
    const char *name;
};

struct flag_def flags[] = {
    {P_ADVLOCK,       "P_ADVLOCK"},
    {P_CONTROLT,      "P_CONTROLT"},
    {P_LP64,          "P_LP64"},
    {P_NOCLDSTOP,     "P_NOCLDSTOP"},
    {P_PPWAIT,        "P_PPWAIT"},
    {P_PROFIL,        "P_PROFIL"},
    {P_SELECT,        "P_SELECT"},
    {P_CONTINUED,     "P_CONTINUED"},
    {P_SUGID,         "P_SUGID"},
    {P_SYSTEM,        "P_SYSTEM"},
    {P_TIMEOUT,       "P_TIMEOUT"},
    {P_TRACED,        "P_TRACED"},
    {P_DISABLE_ASLR,  "P_DISABLE_ASLR"},
    {P_WEXIT,         "P_WEXIT"},
    {P_EXEC,          "P_EXEC"},
};

void print_p_flag_set(uint32_t p_flag) {
    printf("p_flag = 0x%08X\n", p_flag);

    printf("Set flags:\n");
    for (size_t i = 0; i < sizeof(flags)/sizeof(flags[0]); i++) {
        if (p_flag & flags[i].value) {
            printf(" - %s\n", flags[i].name);
        }
    }
}
// ============ END P_FLAG HELPER ============

// Kernel memory read/write
kern_return_t kread(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_read_overwrite(kernel_task, addr, size, (vm_address_t)buf, (vm_size_t *)&size);
}

kern_return_t kwrite(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_write(kernel_task, addr, (vm_offset_t)buf, size);
}

// Helper method to read kernel memory and print in hex dump format
void hex_dump(mach_port_t kernel_task, uint64_t addr, size_t size) {
    // Allocate buffer for reading
    uint8_t *buffer = (uint8_t *)malloc(size);
    if (!buffer) {
        printf("[!] Failed to allocate buffer for hex dump\n");
        return;
    }

    // Read kernel memory
    kern_return_t kr = kread(kernel_task, addr, buffer, size);
    if (kr != KERN_SUCCESS) {
        printf("[!] vm_read_overwrite failed at 0x%llx: %s\n", addr, mach_error_string(kr));
        free(buffer);
        return;
    }

    // Print hex dump
    printf("[+] Hex dump at 0x%llx (%zu bytes):\n", addr, size);
    for (size_t i = 0; i < size; i += 16) {
        // Print address
        printf("%016llx  ", addr + i);

        // Print hex bytes
        for (size_t j = 0; j < 16; j++) {
            if (i + j < size) {
                printf("%02x ", buffer[i + j]);
            } else {
                printf("   ");
            }
            if (j == 7) printf(" ");
        }

        // Print ASCII representation
        printf(" |");
        for (size_t j = 0; j < 16 && i + j < size; j++) {
            uint8_t c = buffer[i + j];
            printf("%c", isprint(c) ? c : '.');
        }
        printf("|\n");
    }

    free(buffer);
}

// https://github.com/Cryptiiiic/x8A4/blob/main/Kernel/slide.c#L57
/**
 * @brief           Get kaslr slide from palera1n ramdisk
 * @return          Kaslr slide
 */
uint64_t palera1n_get_slide(void) {
    int verbose_cached = 1;
  uint64_t slide = 0;
  int rmd0 = open("/dev/rmd0", O_RDONLY, 0);
  if (rmd0 < 0) {
    printf("Could not get paleinfo! (%d:%s:%d:%s)\n", rmd0, strerror(rmd0), errno, strerror(errno));
    return 0;
  }
  uint64_t off = lseek(rmd0, 0, SEEK_SET);
  if (off == -1) {
    printf("Failed to lseek ramdisk to 0\n", "");
    close(rmd0);
    return 0;
  }
  uint32_t pinfo_off;
  ssize_t didRead = read(rmd0, &pinfo_off, sizeof(uint32_t));
  if (didRead != (ssize_t)sizeof(uint32_t)) {
    printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(uint32_t));
    close(rmd0);
    return 0;
  }
  off = lseek(rmd0, pinfo_off, SEEK_SET);
  if (off != pinfo_off) {
    printf("Failed to lseek ramdisk to %u\n", pinfo_off);
    close(rmd0);
    return 0;
  }
  struct paleinfo {
    uint32_t magic; /* 'PLSH' */
    uint32_t version; /* 2 */
    uint64_t kbase; /* kernel base */
    uint64_t kslide; /* kernel slide */
    uint64_t flags; /* unified palera1n flags */
    char rootdev[0x10]; /* ex. disk0s1s8 */
                        /* int8_t loglevel; */
  } __attribute__((packed));
  struct paleinfo_legacy {
    uint32_t magic;   // 'PLSH' / 0x504c5348
    uint32_t version; // 1
    uint32_t flags;
    char rootdev[0x10];
  };
  struct paleinfo *pinfo_p = (struct paleinfo *)calloc(1, sizeof(struct paleinfo));
  struct paleinfo_legacy *pinfo_legacy_p = NULL;
  didRead = read(rmd0, pinfo_p, sizeof(struct paleinfo));
  if (didRead != (ssize_t)sizeof(struct paleinfo)) {
    printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(struct paleinfo));
    close(rmd0);
    free(pinfo_p);
    return 0;
  }
  if (pinfo_p->magic != 'PLSH') {
    close(rmd0);
    pinfo_off += 0x1000;
    pinfo_legacy_p = (struct paleinfo_legacy *)calloc(1, sizeof(struct paleinfo_legacy));
    didRead = read(rmd0, pinfo_legacy_p, sizeof(struct paleinfo_legacy));
    if (didRead != (ssize_t)sizeof(struct paleinfo_legacy)) {
      printf("Read %ld bytes does not match expected %lu bytes\n", didRead, sizeof(struct paleinfo_legacy));
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    if(verbose_cached) {
      printf("pinfo_legacy_p->magic: %s\n", (char *)&pinfo_legacy_p->magic);
      printf("pinfo_legacy_p->magic: 0x%X\n", pinfo_legacy_p->magic);
      printf("pinfo_legacy_p->version: 0x%Xd\n", pinfo_legacy_p->version);
      printf("pinfo_legacy_p->flags: 0x%X\n", pinfo_legacy_p->flags);
      printf("pinfo_legacy_p->rootdev: %s\n", pinfo_legacy_p->rootdev);
    }
    if (pinfo_legacy_p->magic != 'PLSH') {
      printf("Detected corrupted paleinfo!\n", "");
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    if (pinfo_legacy_p->version != 1U) {
      printf("Unexpected paleinfo version: %u, expected %u\n", pinfo_legacy_p->version, 1U);
      close(rmd0);
      free(pinfo_p);
      free(pinfo_legacy_p);
      return 0;
    }
    lseek(rmd0, pinfo_off - 0x1000, SEEK_SET);
    struct kerninfo {
      uint64_t size;
      uint64_t base;
      uint64_t slide;
      uint32_t flags;
    };
    struct kerninfo *kerninfo_p = malloc(sizeof(struct kerninfo));
    read(rmd0, kerninfo_p, sizeof(struct kerninfo));
    close(rmd0);
    slide = kerninfo_p->slide;
    free(kerninfo_p);
  } else {
    if(verbose_cached) {
      printf("pinfo_p->magic: %s\n", (const char *)&pinfo_p->magic);
      printf("pinfo_p->magic: 0x%X\n", pinfo_p->magic);
      printf("pinfo_p->version: 0x%Xd\n", pinfo_p->version);
      printf("pinfo_p->kbase: 0x%llX\n", pinfo_p->kbase);
      printf("pinfo_p->kslide: 0x%llX\n", pinfo_p->kslide);
      printf("pinfo_p->flags: 0x%llX\n", pinfo_p->flags);
      printf("pinfo_p->rootdev: %s\n", pinfo_p->rootdev);
      slide = pinfo_p->kslide;
    }
  }
  free(pinfo_p);
  free(pinfo_legacy_p);
  return slide;
}

int main(void) {
    mach_port_t kernel_task = 0;
    kern_return_t ret;

    // Step 1: Get tfp0
    ret = task_for_pid(mach_task_self(), 0, &kernel_task);
    if (ret != KERN_SUCCESS) {
        printf("Failed to get tfp0: %d\n", ret);
        return 1;
    }
    printf("Got tfp0: %u\n", kernel_task);

    // Step 2: Read KASLR from ramdisk
    uint64_t kASLR = palera1n_get_slide();
    printf("KASLR: 0x%llX\n", kASLR);

    // Step 3: Read launchd address
    uint64_t launchd_proc_ptr_addr = LAUNCHD_PROC_ADDRESS + kASLR;
    uint64_t launchd_proc_addr;
    ret = kread(kernel_task, launchd_proc_ptr_addr, &launchd_proc_addr, sizeof(launchd_proc_addr));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read launchd_proc_ptr_addr at 0x%llx: %s\n", launchd_proc_ptr_addr, mach_error_string(ret));
        return 1;
    }
    printf("launchd proc address: 0x%llX\n", launchd_proc_addr);    

    // Step 4: Verify if it's launchd
    uint64_t p_name_addr = launchd_proc_addr + P_NAME_OFFSET;
    char proc_name[50];
    ret = kread(kernel_task, p_name_addr, proc_name, sizeof(proc_name));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_name at 0x%llx: %s\n", p_name_addr, mach_error_string(ret));
        return 1;
    }
    printf("launchd proc name: %s\n", proc_name);

    // Step 5: Read current launchd p_flag
    uint64_t p_flag_addr = launchd_proc_addr + P_FLAG_OFFSET;
    uint32_t current_p_flag = 0;    
    ret = kread(kernel_task, p_flag_addr, &current_p_flag, sizeof(current_p_flag));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_flag at 0x%llx: %s\n", p_flag_addr, mach_error_string(ret));
        return 1;
    }
    printf("[+] Current p_flag: 0x%x\n", current_p_flag);
    print_p_flag_set(current_p_flag);

    // Step 6: Patch launchd with P_DISABLE_ASLR to disable ASLR
    // uint32_t disable_aslr_p_flag = current_p_flag | P_DISABLE_ASLR;    
    // kwrite(kernel_task, p_flag_addr, &disable_aslr_p_flag, sizeof(disable_aslr_p_flag));
    uint32_t enable_aslr_p_flag = current_p_flag & ~P_DISABLE_ASLR; // use this if you want to enable ASLR again
    kwrite(kernel_task, p_flag_addr, &enable_aslr_p_flag, sizeof(enable_aslr_p_flag));

    // print out new p_flag
    uint32_t new_p_flag = 0;    
    ret = kread(kernel_task, p_flag_addr, &new_p_flag, sizeof(new_p_flag));
    if (ret != KERN_SUCCESS) {
        printf("[!] Failed to read p_flag at 0x%llx: %s\n", p_flag_addr, mach_error_string(ret));
        return 1;
    }
    printf("[+] Patched p_flag: 0x%x\n", new_p_flag);
    print_p_flag_set(new_p_flag);

    hex_dump(kernel_task, launchd_proc_addr, P_NAME_OFFSET + 50);
}
```

## Further Reading

- [ASLR & the iOS Kernel](https://bellis1000.medium.com/aslr-the-ios-kernel-how-virtual-address-spaces-are-randomised-d76d14dc7ebb)
- [Apple OSS XNU Source](https://github.com/apple-oss-distributions/xnu)
- [ipsw Tool](https://github.com/blacktop/ipsw)
- [tfp0 Patch](https://theapplewiki.com/wiki/Tfp0_patch)
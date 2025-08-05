---
layout: post
title: "Disable ASLR iOS apps"
categories: [Kernel]
tags: [kernel,ios,xnu,security,aslr,launchd]
permalink: /disclosures/unauthenticated-api-testing-interface-hardcoded-production-credentials/
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczN4VkxKSRr_zttj5XnvFBVEU-xNObJAGtVNNruzN4m7BEHdeqsQlhDTRgXQiMvEHFA0vDzsG13F5upW-cGT3VvSnlRFfT4oUMD-dJT_391xk1pI8AAdBno_6DYHdQG7jLsmxtsBOdi5zLe7t1iuWU_b=w1830-h966-s-no-gm
    alt: Disable ASLR
---
# TL;DR
- `launchd` is a userland process. While it's one of the first processes started by the kernel after booting, and is responsible for managing system-level and user-level daemons and agents, it itself runs in user space, not kernel space.
- `launchd` is responsible for launching and acts as the parent process for all (?) user-space processes in iOS
- Userland apps are launched by launchd, and ASLR's flag inherits from its parent
- We need to patch `launchd` ASLR's flag to disable ASLR
- To do that we need kernel read/write primitive with `tfp0` achieved

# Find device XNU version
- ssh into iOS device and run the command
```bash
$ uname -a
Darwin iPhone 22.6.0 Darwin Kernel Version 22.6.0: Tue Jul  2 20:47:35 PDT 2024; root:xnu-8796.142.1.703.8~1/RELEASE_ARM64_T8015 iPhone10,4 arm Darwin
```

# Download XNU source code
- Based on device XNU version, we can find the nearest XNU source code for static analysis [here](https://github.com/apple-oss-distributions/xnu/tags?after=xnu-10002.81.5) 
![XNU source code](https://lh3.googleusercontent.com/pw/AP1GczMvGxjoUrRAheorQXLsmodYBHVyx2ofrvgDgeQrA_iE25-QlpAauybzDDm5o99gVviyVnZ3eIIsjFXEqBd4NbNc6XaB4JiNKb1Wyp6fogm-vX2a5N7c9ZTZunvU0rqCwE0uvpGM9PWpw8O8hm0cxC9K=w2126-h1256-s-no-gm)
_**Figure: XNU source code**_
- As we can see, the XNU version on the iOS device we have is `xnu-8796.142.1.703.8~1`, hence the closest opensourced XNU we can use is [xnu-8796.141.3](https://github.com/apple-oss-distributions/xnu/releases/tag/xnu-8796.141.3)
- Download the source code `.zip` file for static analysis

# Analyse XNU source code
## Find the ASLR flag
- Open downloaded XNU source code in a text editor of your choice, I'm using `Visual Code` in this case. By searching `ASLR` we can see there are multiple results, a few of them caught your eyes that found inside `kern_exec.c` file such as `* Disable ASLR for the spawned process.` or `* Disable ASLR during image activation.  This occurs either if the _POSIX_SPAWN_DISABLE_ASLR attribute was found above or if P_DISABLE_ASLR was inherited from the parent process.`
![XNU P_DISABLE_ASLR](https://lh3.googleusercontent.com/pw/AP1GczPnapSZP4cYybifD24E1vSEYgfQSjiLfhuxmXBHkKdMTSlywZrHN7MaCow-r-BFD0iJSLOVGITSWMSfO_9ywhlH6WWrmqGejMjuWwGotRagDqWdMcrPRbveCl7J6dKoRi6Y7ieMj7p4Ylewaky1cMvt=w2126-h1372-s-no-gm)
_**Figure: XNU P_DISABLE_ASLR**_

- As we can see, it checks `proc::p_flag` with `P_DISABLE_ASLR` (`#define P_DISABLE_ASLR  0x00001000 /* Disable address space layout randomization */` in `proc.h`) to make sure if it needs to apply ASLR for spawned processes or not.

- As we know all userland processes are launched by the parent `launchd (pid=1)`, hence if we are able to modify `launchd` process and change the `p_flag`, we can apply `P_DISABLE_ASLR` for `launchd`, then all children processes will have ASLR disabled.

## `struct proc`
- Navigate to `proc_internal.h` manually or `command + click (on Mac)` on `p_flag` it will navigate to the `struct proc` definition as below.
```c
// proc_internal.h
/*
 * Description of a process.
 *
 * This structure contains the information needed to manage a thread of
 * control, known in UN*X as a process; it has references to substructures
 * containing descriptions of things that the process uses, but may share
 * with related processes.  The process structure and the substructures
 * are always addressible except for those marked "(PROC ONLY)" below,
 * which might be addressible only on a processor on which the process
 * is running.
 */
struct proc {
	LIST_ENTRY(proc) p_list;                /* List of all processes. */

	struct  proc *  XNU_PTRAUTH_SIGNED_PTR("proc.p_pptr") p_pptr;   /* Pointer to parent process.(LL) */
	proc_ro_t       p_proc_ro;
	pid_t           p_ppid;                 /* process's parent pid number */
	pid_t           p_original_ppid;        /* process's original parent pid number, doesn't change if reparented */
	pid_t           p_pgrpid;               /* process group id of the process (LL)*/
	uid_t           p_uid;
	gid_t           p_gid;
	uid_t           p_ruid;
	gid_t           p_rgid;
	uid_t           p_svuid;
	gid_t           p_svgid;
	pid_t           p_sessionid;
	uint64_t        p_puniqueid;            /* parent's unique ID - set on fork/spawn, doesn't change if reparented. */

	lck_mtx_t       p_mlock;                /* mutex lock for proc */
	pid_t           p_pid;                  /* Process identifier for proc_find. (static)*/
	char            p_stat;                 /* S* process status. (PL)*/
	char            p_shutdownstate;
	char            p_kdebug;               /* P_KDEBUG eq (CC)*/
	char            p_btrace;               /* P_BTRACE eq (CC)*/

	LIST_ENTRY(proc) p_pglist;              /* List of processes in pgrp (PGL) */
	LIST_ENTRY(proc) p_sibling;             /* List of sibling processes (LL)*/
	LIST_HEAD(, proc) p_children;           /* Pointer to list of children (LL)*/
	TAILQ_HEAD(, uthread) p_uthlist;        /* List of uthreads (PL) */

	struct smrq_slink p_hash;               /* Hash chain (LL)*/

#if CONFIG_PERSONAS
	struct persona  *p_persona;
	LIST_ENTRY(proc) p_persona_list;
#endif

	lck_mtx_t       p_ucred_mlock;          /* mutex lock to protect p_ucred */

	/* substructures: */
	struct  filedesc p_fd;                  /* open files structure */
	struct  pstats *p_stats;                /* Accounting/statistics (PL) */
	SMR_POINTER(struct plimit *) p_limit;/* Process limits (PL) */
	SMR_POINTER(struct pgrp *XNU_PTRAUTH_SIGNED_PTR("proc.p_pgrp")) p_pgrp; /* Pointer to process group. (LL) */

	struct sigacts  p_sigacts;
	lck_spin_t      p_slock;                /* spin lock for itimer/profil protection */

	int             p_siglist;              /* signals captured back from threads */
	unsigned int    p_flag;                 /* P_* flags. (atomic bit ops) */
    ...
    proc_name_t p_name;
    ...
}

// queue.h
#define LIST_ENTRY(type)                                                \
__MISMATCH_TAGS_PUSH                                                    \
__NULLABILITY_COMPLETENESS_PUSH                                         \
struct {                                                                \
	struct type *le_next;   /* next element */                      \
	struct type **le_prev;  /* address of previous next element */  \
}                                                                       \
__NULLABILITY_COMPLETENESS_POP                                          \
__MISMATCH_TAGS_POP
```

- `struct proc` contains a doubly linked list of all processes, so if we managed to get the `kernel` process, we will able to traverse and find `launchd` process too, that will be our target. We can rely on `p_name` or `p_id` to identify the process we are looking for.

## Find kernel process
- By searching `kernel process` in the XNU source code, we found this in `bsd_init.c`
```c
// bsd_init.c
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

// proc.h
#ifdef KERNEL
__BEGIN_DECLS

extern proc_t kernproc;
...

__END_DECLS
#endif  /* KERNEL */
```

- `kernproc` is a global variable, hence we can find the its address easily when examine the kernel later, and if we know kernel ASLR value (KASLR), we can calculate `kernproc` address at runtime too. The easy way to find KASLR is from the log, most the jailbreak will have the log file that log this value, hence it's trivial.
- From above code, we also observe `initproc = proc_find(1)` which mean find process with id `1` (`launchd`) and assign to another global variable `initproc`, hence later we can directly locate this symbol instead of finding `launchd` from `kernproc` process.

# Download and extract iOS firmware
## Download iOS firmware
- Using [ipsw](https://github.com/blacktop/ipsw) tool, we can easily download expected iOS firmware file. Given above device model (got from `uname -a` and current device iOS version `16.7.11`:
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

## Extract kernel cache
- By running below command, we can extract the kernel cache from downloaded firmware file, which is ready to load into `IDA` for static analysis
```bash
$ ipsw extract --kernel iPhone_4.7_P3_16.7.11_20H360_Restore.ipsw
   • Extracting kernelcache
      • Created 20H360__iPhone10,1_4/kernelcache.release.iPhone10,1_4
$ ls
20H360__iPhone10,1_4
$ ls 20H360__iPhone10,1_4
kernelcache.release.iPhone10,1_4
```

# Anlaysis kernelcache with IDA
![IDA Loading kernelcache](https://lh3.googleusercontent.com/pw/AP1GczNuDEuKx2DbK7kMez10pU2bZjEqcaFkkN_H6pWa2J63ILzgbki3e-vYFSlzdwv8_a3SKcG5kwX_E1oToY_41uXuE6Sw6Thh8o0nF6_CnXfSkuJSm9ZcaTKSSKtwDOEjAmaa96mdsT0e5wXdLftgfNfR=w1564-h1064-s-no-gm)
_**Figure: IDA Loading kernelcache**_

## Finding initproc
- By searching `initproc` in the IDA `Exports` tab, we found this exported symbol `int __cdecl proc_isinitproc(proc_t) at 0xFFFFFFF0075CDF70`
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

## Finding kernproc
Apply same steps earlier, we can find `kernproc` exported in IDA at `0xFFFFFFF0071997E0`:
```asm
__DATA_CONST:__const:FFFFFFF0071997E0                 EXPORT _kernproc
__DATA_CONST:__const:FFFFFFF0071997E0 ; proc_t kernproc
```

## Finding proc->p_flag offset
- As we already identified the address of `launchd` proc, we are able to dump the memory layout of the proc and find out the relative offset to the field.
- Eventhough we got the definition of `struct proc`, we are able to calculate the offset of each field, however it's cumbersome to do that for fields that is too far from the base address, especially the `p_flag` field in this case
- There is another approach to find `p_flag` in this case, as we have XNU source code, we can find the occurrences of `p_flag` and mapping to IDA to identify the offset properly
- By searching `->p_flag` in XNU source code, we found plenty of results. However, let find the result that is a simple method and that method can be found in IDA, in this case found this:
```c
/// kern_prot.c
int
proc_issetugid(proc_t p)
{
	return (p->p_flag & P_SUGID) ? 1 : 0;
}
```

and in IDA:
```c
int __cdecl proc_issetugid(proc_t p)
{
  return (*((_DWORD *)p + 277) >> 8) & 1;
}
```
- It's similar, it casts p to a pointer to 32‑bit integers, calculate offset `277 * 4 = 1108` bytes from the start of `struct proc` which is location corresponds to `p_flag` inside `struct proc`, hence we know `p_flag` offset is `1108` bytes from the start of `struct proc`

# Patching ASLR p_flag
## Retrieve tfp0 or or task_for_pid(0)
`task_for_pid` gets the task port for another "process", named by its process ID on the same host as "target_task". Only permitted to privileged processes, or processes with the same user ID. If `pid == 0`, an error is return no matter who is calling.
```c
kern_return_t
task_for_pid(
	struct task_for_pid_args *args)
{
	mach_port_name_t        target_tport = args->target_tport;
	int                     pid = args->pid;
	user_addr_t             task_addr = args->t;
	proc_t                  p = PROC_NULL;
	task_t                  t1 = TASK_NULL;
	task_t                  task = TASK_NULL;
	mach_port_name_t        tret = MACH_PORT_NULL;
	ipc_port_t              tfpport = MACH_PORT_NULL;
	void                    * sright = NULL;
	int                     error = 0;
	boolean_t               is_current_proc = FALSE;
	struct proc_ident       pident = {0};

	AUDIT_MACH_SYSCALL_ENTER(AUE_TASKFORPID);
	AUDIT_ARG(pid, pid);
	AUDIT_ARG(mach_port1, target_tport);

	/* Always check if pid == 0 */
	if (pid == 0) {
		(void) copyout((char *)&tret, task_addr, sizeof(mach_port_name_t));
		AUDIT_MACH_SYSCALL_EXIT(KERN_FAILURE);
		return KERN_FAILURE;
	}
    ...
}
```
- There is a restriction for `pid` **0**, but luckily most [jailbreaks patch this restriction](https://theapplewiki.com/wiki/Tfp0_patch) to allow calling `task_for_pid(0)` or `tfp0 patch`, hence we can call this to retrieve kernel task directly as below:

```c
#include <mach/mach.h>
#include <stdio.h>

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
}
```
Then compile this followed by signing with specific entitlement `tfp0.plist` to make it work.
```plist
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

```bash
$ 
```

Now we can compile, sign then copy the output file `disable_aslr` to the device under `/usr/bin` location: 
```bash
$ cc disable_aslr.c -o disable_aslr && ldid -Stfp0.plist disable_aslr && scp -P 2222 disable_aslr root@localhost:/usr/bin
root@localhost's password:
disable_aslr                                                                                100%   34KB   5.6MB/s   00:00
```

Then `ssh` into device to test it out, and we are able to get the kernel task port `tfp0`
```bash
$ ssh root@localhost -p 2222
root@localhost's password:
iPhone $ disable_aslr
Got tfp0: 6915
```

## Retrieve KASLR
- For `palera1n` jailbreak, you can get the KASLR value from the log file at `/cores/jbinit.log` when using option `-L` while jailbreaking.
- However, you also can refer this [existing code](https://github.com/Cryptiiiic/x8A4/blob/main/Kernel/slide.c#L57) to retrieve it from palera1n ramdisk. We can copy and add new method `uint64_t palera1n_get_slide(void)` into our `disable_aslr.c`, with a little bit of modification it will be like this

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

- Compile and copy over to device and run it, we are able to get the KASLR is 0x5A54000

```bash
iPhone $ $ disable_aslr
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
- For other kind of jailbreak, you may use [all_image_info_size](https://github.com/bazad/memctl/blob/master/src/libmemctl/kernel_slide.c#L468) to retrieve the KASLR value.

## Adding kernel read/write helper
```c
// Kernel memory read/write
kern_return_t kread(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_read_overwrite(kernel_task, addr, size, (vm_address_t)buf, (vm_size_t *)&size);
}

kern_return_t kwrite(mach_port_t kernel_task, vm_address_t addr, void *buf, size_t size) {
    return vm_write(kernel_task, addr, (vm_offset_t)buf, size);
}
```

## Read `launchd` proc address
```c
int main(void) {
    ...
    // Step 3: Read launchd address
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

## Verify if it is `launchd` proc
- To make sure we are patching the correct `launchd` proc, we can verify it by confirm the value of `p_pid` is `1` or `p_name` is `"launchd"`, let go with `p_name`.
- To find `p_name` offset, we can use the same technique by searching `->p_name` in XNU source code and found one of the usage in `proc_best_name(proc_t p)`
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

- Then we can assure `p_name` field is at offset `1401`, to read the value is become trivial:
```c
int main(void) {
    ...
    // Step 4: Verify if it's launchd
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

## Reading current `launchd->p_flag` value
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
```
- By reading `p_flag` at offset `1108` found earlier, we can leverage XNU flags source code to print out readable flags set, and found out that `launchd` has no `P_DISABLE_ASLR` flag, hence spawned process (child process) will has ASLR enabled

## Patching `launchd->p_flag` with `P_DISABLE_ASLR`
```c
int main(void) {
    ...
     // Step 6: Patch launchd with P_DISABLE_ASLR to disable ASLR
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
    print_p_flag_set(new_p_flag);
}
```

```bash
iPhone $ $ disable_aslr
Got tfp0: 2819
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

## Verify ASLR Disabled
- Let run an app and attach LLDB debugger (either standable `lldb` or `XCode lldb`), and run this `lldb` command: `(lldb) image list`
- You can observe that the ASLR value is `0x0000000000000000` for the main binary, which mean ASLR is disabled
![ASLR Disabled](https://lh3.googleusercontent.com/pw/AP1GczOLdxWqFBcvY1_cq3i34NhR1URbiExtuc4yCveOXCdn_N1jE33wq7iLZxXmKnrQaL83WV-4XoMCLvSLcx-9vk3B4245L4H7MZfscPHABp87dggl9fTF0j-w9a2q0bJJnzMMErKtwf_iLQKUSapPBaGm=w2126-h1366-s-no-gm)
_**Figure: ASLR Disabled**_

- For the process that has the ASLR enabled, the value will not be zero, in this case it's `0x0000000000460000`
![ASLR Enabled](https://lh3.googleusercontent.com/pw/AP1GczM9cxRRaAD4RZ4VI6snx31OGFjZzSAHVD5OvRw68X-JoxvxnJotY-bkA2HImhavCqW2rkcF7f9TyO9X-MZbDgJD0B5FEkNrjha5E8ZOX6S1VSPztrWZk8jTuAGeYpndV_TF9RmdiWrESggs2MSfCYHN=w2126-h1374-s-no-gm?authuser=2)
_**Figure: ASLR Enabled**_

# References
- [ASLR & the iOS Kernel — How virtual address spaces are randomised](https://bellis1000.medium.com/aslr-the-ios-kernel-how-virtual-address-spaces-are-randomised-d76d14dc7ebb)
---
layout: post
title: "Reverse Engineering Flare-On 2025 Challenge 8: FlareAuthenticator"
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczMRu8zOSM7tnklQ0i8-Mio2FBtjTiW7lJJoL69yo30J3Z2KpFAy0yc2YPvfLAqpBc30vD_axzLnaQNbeXk0kUWIva-GHyymhFnIDz93d3mC_oONOChOEVMJAatOMNhm8v1ac9y0SsGoKr5aJyMZaFvA=w680-h736-s-no-gm
    alt: Challenge 8 - FLARE Authenticator
tags: [flareon, flareon12, flareon-2025, ctf, QT, obfuscated, ret-sync, WinDbg, TTD, IDA]
categories: [CTF, Flareon 2025]
---

CTF challenges like those in Flare-On provide excellent opportunities to refine reverse engineering techniques on obfuscated binaries. Challenge 8 from Flare-On 2025, "FlareAuthenticator" involves recovering a 25-digit password from a legacy QT-based authenticator protected by multiple obfuscation layers. The analysis begins with initial exploration and progresses through static and dynamic techniques, leveraging tools such as IDA, x64dbg, WinDbg with Time Travel Debugging (TTD), and the Z3 solver. Key concepts are explained for accessibility while maintaining depth for intermediate practitioners.

# Challenge description
> 8 - FlareAuthenticator
>
>We recovered a legacy authenticator from an old hard disk, but the password has been lost and we only remember that there was a single password to unlock it. We need your help to analyze the program and find a way to get back in. Can you recover the password?
{: .prompt-info}

# Examine the challenge package
Unzipping the package reveals the primary executable `FlareAuthenticator.exe` alongside various dependencies. The presence of `Qt6Core.dll` confirms the application uses the **QT Framework**, a popular choice for cross-platform GUI development. A `run.bat` script accompanies the files to ensure proper launch by configuring environment variables.

```bash
$ ls -1
FlareAuthenticator.exe
msvcp140_1.dll
msvcp140_2.dll
msvcp140.dll
Qt6Core.dll
Qt6Gui.dll
Qt6Widgets.dll
qwindows.dll
run.bat
vcruntime140_1.dll
vcruntime140.dll

$ cat run.bat
@echo off
set QT_QPA_PLATFORM_PLUGIN_PATH=%~dp0
start %~dp0\FlareAuthenticator.exe%
```

# Exploring the Application Interface
Direct execution of `FlareAuthenticator.exe` results in an "Application Error" prompting the use of `run.bat`. Launching via the batch file starts the application correctly.

![FLARE Authenticator GUI](https://lh3.googleusercontent.com/pw/AP1GczMRu8zOSM7tnklQ0i8-Mio2FBtjTiW7lJJoL69yo30J3Z2KpFAy0yc2YPvfLAqpBc30vD_axzLnaQNbeXk0kUWIva-GHyymhFnIDz93d3mC_oONOChOEVMJAatOMNhm8v1ac9y0SsGoKr5aJyMZaFvA=w680-h736-s-no-gm)
_**Figure: FLARE Authenticator GUI**_

The interface functions as a typical authenticator, accepting 25 digits and validating them through alert popups. Input occurs via an on-screen keypad or direct keyboard entry, with deletion supported. Brute-force attempts prove impractical, demanding up to 10^25 trials.

# First Attempt — Intel Pin for Side-Channel Behavior
The Intel Pin framework enables dynamic binary instrumentation for x86 architectures, facilitating custom analysis tools. An initial approach explored instruction counting for side-channel analysis, positing that correct digits might trigger additional logic and higher counts compared to incorrect ones, allowing sequential guessing.

## Preparing the run_pin.bat Script
A `run_pin.bat` script configures the environment and paths necessary for instrumentation.

```bash
:: adjust these 3 paths
set "APPDIR=C:\flareon12\8\8_-_FlareAuthenticator"
set "PIN=C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\intel64\bin\pin.exe"
set "TOOL=C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\source\tools\ManualExamples\obj-intel64\inscount0.dll"

pushd "%APPDIR%"
:: If qwindows.dll is in .\platforms, point at that. If it sits next to the EXE, point at %CD%.
set "QT_QPA_PLATFORM_PLUGIN_PATH=%CD%\platforms"
if exist "%CD%\qwindows.dll" set "QT_QPA_PLATFORM_PLUGIN_PATH=%CD%"

"%PIN%" -t "%TOOL%" -o inscount.log -- "%CD%\FlareAuthenticator.exe"
popd
```

## Running the Binary with Intel Pin
Running the script initiates Pin and launches the challenge.

```bash
C:\flareon12\8\8_-_FlareAuthenticator>set "APPDIR=C:\flareon12\8\8_-_FlareAuthenticator"
C:\flareon12\8\8_-_FlareAuthenticator>set "PIN=C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\intel64\bin\pin.exe"
C:\flareon12\8\8_-_FlareAuthenticator>set "TOOL=C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\source\tools\ManualExamples\obj-intel64\inscount0.dll"
C:\flareon12\8\8_-_FlareAuthenticator>pushd "C:\flareon12\8\8_-_FlareAuthenticator"
C:\flareon12\8\8_-_FlareAuthenticator>set "QT_QPA_PLATFORM_PLUGIN_PATH=C:\flareon12\8\8_-_FlareAuthenticator\platforms"
C:\flareon12\8\8_-_FlareAuthenticator>if exist "C:\flareon12\8\8_-_FlareAuthenticator\qwindows.dll" set "QT_QPA_PLATFORM_PLUGIN_PATH=C:\flareon12\8\8_-_FlareAuthenticator"
C:\flareon12\8\8_-_FlareAuthenticator>"C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\intel64\bin\pin.exe" -t "C:\pin-external-3.31-98869-gfa6f126a8-msvc-windows\pin-external-3.31-98869-gfa6f126a8-msvc-windows\source\tools\ManualExamples\obj-intel64\inscount0.dll" -o inscount.log -- "C:\flareon12\8\8_-_FlareAuthenticator\FlareAuthenticator.exe"
```

Entering '0' and closing logs the count in `inscount.log`.

```bash
$ cat inscount.log
Count 208976372
```

Approximately `208,976,372` instructions execute for '0'. Repeating for digits 1–9 produces comparable figures.

```bash
$ cat track_instructions_count.log
Count 208976372: 0
Count 200884636: 1
Count 208983627: 2
Count 208973627: 3
Count 208947362: 4
Count 209017373: 5
Count 209037462: 6
Count 209103373: 7
Count 209074629: 8
Count 209037462: 9
```

The similarity in counts across inputs renders Pin ineffective for this side-channel strategy. Attention then shifts to static binary analysis.

# Static Analysis
Opening `FlareAuthenticator.exe` in IDA and sorting functions by length highlights several complex ones beyond `main`. Closer inspection uncovers extensive obfuscation.

![Extensive obfuscated functions](https://lh3.googleusercontent.com/pw/AP1GczPOjzuon7DKpWlsNgf_eP77_kMadYWfAJzq3JLbyS06D7mNGm5eB4UzWQIV0wITTII2oXNEcEMFzQ7-hIpQ3N9FqXsI833gzILBJ64JCBzIVxeISHv3th10lwnLbhjuKf1nGPXaE_D4refxx_7KqgpY=w1080-h426-s-no-gm)
_**Figure: Extensive obfuscated functions**_

## Tanged Control Flow Graph (CFG)
Graph view for functions like `sub_140037160` displays an extraordinarily tangled control flow graph (CFG), indicative of intentional disruption. Efforts to simplify it give way to broader exploration of the binary.

![sub_140037160 Obfuscated CFG](https://lh3.googleusercontent.com/pw/AP1GczMyqhDHM0NMB3jmGCmgPkvI6LOxAIvIp5uQY4Gv4p2-kytKTfgngMXWKE6rwdgt3gBUFMKqqxqocnpwZAI4zRCnlZSwTaV7E7TvSrUWPU3_poCMka3Qmwgos2n7l7hPn4eCogG0NmSyv5poxU6z0h8n=w870-h736-s-no-gm)
_**Figure: sub_140037160 Obfuscated CFG**_

![sub_140035EF60 Obfuscated CFG](https://lh3.googleusercontent.com/pw/AP1GczMCUio2MY7LfHxOzFzvRSwIzFZdowA28AC2dUmdDFaq4fXl_kPI_QZGzhy7uRmsppI0nMI96PMJP4qeTSBpMbP_26U1ssRW0vLAIagwcgyVQQQ4vZ4WOtQeXzstUCc8uNwLgUhqmt0Kc1A2ZkZCc6yw=w684-h736-s-no-gm)
_**Figure: sub_140035EF60 Obfuscated CFG**_

## Data-flow obfuscation
Large stack frames allocate via `sub rsp, rax` (with `eax` at `0x7278`), followed by repeated address loads and mirrored stores into local slots. This aliasing confounds decompilers and masks actual data movement through added indirection.

```assembly
.text:000000014003716C                 mov     eax, 7278h
.text:0000000140037171                 call    __alloca_probe
.text:0000000140037176                 sub     rsp, rax
.text:0000000140037179                 lea     rbp, [rsp+80h]
.text:0000000140037181                 mov     [rbp+7230h+var_40], 0FFFFFFFFFFFFFFFEh
.text:000000014003718C                 mov     [rbp+7230h+var_5118], rcx
.text:0000000140037193                 lea     rax, [rbp+7230h+var_7C8]
.text:000000014003719A                 mov     [rbp+7230h+var_4EB0], rax
.text:00000001400371A1                 mov     [rbp+7230h+var_1020], rax
.text:00000001400371A8                 lea     rax, [rbp+7230h+var_5E8]
.text:00000001400371AF                 mov     [rbp+7230h+var_4DE8], rax
.text:00000001400371B6                 mov     [rbp+7230h+var_1040], rax
.text:00000001400371BD                 lea     r10, [rbp+7230h+var_808]
.text:00000001400371C4                 mov     [rbp+7230h+var_4C88], r10
.text:00000001400371CB                 mov     [rbp+7230h+var_1028], r10
.text:00000001400371D2                 lea     rcx, [rbp+7230h+var_3B0]
.text:00000001400371D9                 mov     [rbp+7230h+var_4E90], rcx
.text:00000001400371E0                 mov     [rbp+7230h+var_1038], rcx
.text:00000001400371E7                 lea     rax, [rbp+7230h+var_BC8]
.text:00000001400371EE                 mov     [rbp+7230h+var_4E68], rax
.text:00000001400371F5                 mov     [rbp+7230h+var_1018], rax
.text:00000001400371FC                 lea     r11, [rbp+7230h+var_170]
.text:0000000140037203                 mov     [rbp+7230h+var_4F80], r11
.text:000000014003720A                 mov     [rbp+7230h+var_1050], r11
.text:0000000140037211                 lea     rax, [rbp+7230h+var_528]
.text:0000000140037218                 mov     [rbp+7230h+var_4B60], rax
.text:000000014003721F                 mov     [rbp+7230h+var_1070], rax
.text:0000000140037226                 lea     rax, [rbp+7230h+var_DB0]
.text:000000014003722D                 mov     [rbp+7230h+var_4B78], rax
.text:0000000140037234                 mov     [rbp+7230h+var_1058], rax
.text:000000014003723B                 lea     rax, [rbp+7230h+var_938]
.text:0000000140037242                 mov     [rbp+7230h+var_4F18], rax
.text:0000000140037249                 mov     [rbp+7230h+var_1080], rax
.text:0000000140037250                 lea     rax, [rbp+7230h+var_F70]
.text:0000000140037257                 mov     [rbp+7230h+var_4B10], rax
.text:000000014003725E                 mov     [rbp+7230h+var_10A0], rax
.text:0000000140037265                 lea     rdx, [rbp+7230h+var_A18]
.text:000000014003726C                 mov     [rbp+7230h+var_50D8], rdx
.text:0000000140037273                 mov     [rbp+7230h+var_1088], rdx
.text:000000014003727A                 lea     rdx, [rbp+7230h+var_E98]
.text:0000000140037281                 mov     [rbp+7230h+var_5080], rdx
.text:0000000140037288                 mov     [rbp+7230h+var_10B0], rdx
.text:000000014003728F                 lea     r8, [rbp+7230h+var_280]
.text:0000000140037296                 mov     [rbp+7230h+var_4D68], r8
.text:000000014003729D                 mov     [rbp+7230h+var_10D0], r8
.text:00000001400372A4                 lea     rsi, [rbp+7230h+var_EB0]
.text:00000001400372AB                 mov     [rbp+7230h+var_10E0], rsi
.text:00000001400372B2                 lea     r8, [rbp+7230h+var_258]
.text:00000001400372B9                 mov     [rbp+7230h+var_4D40], r8
.text:00000001400372C0                 mov     [rbp+7230h+var_1100], r8
.text:00000001400372C7                 mov     [rbp+7230h+var_10E8], rcx
.text:00000001400372CE                 mov     [rbp+7230h+var_10F8], rdx
.text:00000001400372D5                 lea     r8, [rbp+7230h+var_2C8]
.text:00000001400372DC                 mov     [rbp+7230h+var_4F68], r8
.text:00000001400372E3                 mov     [rbp+7230h+var_10D8], r8
.text:00000001400372EA                 lea     rcx, [rbp+7230h+var_FE0]
.text:00000001400372F1                 mov     [rbp+7230h+var_1110], rcx
...
```

Though initially daunting, this layer contributes minimally to core logic and is set aside.

## Indirect function-pointer obfuscation
Many function calls are resolved at runtime: a base address is loaded from a global offset table (`cs:off_1400B8440`), then an immediate is added before invocation. This pattern effectively hides callees from static cross-references.

```assembly
.text:000000014003A2A8                 mov     rax, cs:off_1400B8440
.text:000000014003A2AF                 mov     rcx, 0DC4D1CFA717B15E1h
.text:000000014003A2B9                 add     rax, rcx
.text:000000014003A2BC                 lea     rcx, [rbp+7230h+var_3E60]
.text:000000014003A2C3                 call    rax             
.text:000000014003A2C5                 mov     rax, [rax]
.text:000000014003A2C8                 mov     qword ptr [rax+20h], 146Bh
.text:000000014003A2D0                 mov     [rbp+7230h+var_738], 146Bh
.text:000000014003A2DB                 mov     rax, cs:off_1400B1EE0
.text:000000014003A2E2                 mov     rcx, 1E6F4B38BD1901B5h
.text:000000014003A2EC                 add     rax, rcx
.text:000000014003A2EF                 lea     rcx, [rbp+7230h+var_3FB0]
.text:000000014003A2F6                 call    rax             
.text:000000014003A2F8                 mov     rcx, [rax]
.text:000000014003A2FB                 mov     rax, cs:off_1400ADDD0
.text:000000014003A302                 mov     rdx, 0CD5CA65684621C9Ah
.text:000000014003A30C                 add     rax, rdx
.text:000000014003A30F                 call    rax             
.text:000000014003A311                 mov     rax, [rax]
.text:000000014003A314                 add     rax, 10h
.text:000000014003A318                 mov     [rbp+7230h+var_370], rax
.text:000000014003A31F                 mov     rax, cs:off_1400A4428
.text:000000014003A326                 mov     rcx, 0DD4DBD41A35BB9FCh
.text:000000014003A330                 add     rax, rcx
.text:000000014003A333                 lea     rcx, [rbp+7230h+var_4910]
.text:000000014003A33A                 call    rax             
.text:000000014003A33C                 mov     rcx, [rax]
.text:000000014003A33F                 mov     rax, cs:off_1400A6B08
.text:000000014003A346                 mov     rdx, 0AA18916841B77C41h
.text:000000014003A350                 add     rax, rdx
.text:000000014003A353                 call    rax             
.text:000000014003A355                 mov     rcx, [rax]
.text:000000014003A358                 mov     rax, cs:off_1400AEE38
.text:000000014003A35F                 mov     rdx, 4F7DF4F0D44FD46Eh
.text:000000014003A369                 add     rax, rdx
...
```

An IDA Python script automates resolution by parsing these arithmetic patterns, computing target addresses, and inserting proper cross-references (XREFs). After execution, comments reflect actual function names, and navigation via XREFs becomes viable.

```python
# ida_resolve_indirect_calls.py
# IDA 9.x compatible — non-destructive, no renaming.
# Sets call-site comment to CURRENT callee name (no renames), and adds a call xref.

import re
import idc
import idaapi
import ida_bytes
import ida_funcs
import ida_segment
import ida_xref
import idautils

SCAN_LIMIT_BYTES     = 0x80
CALL_LIMIT_AFTER_OP  = 0x40
ONLY_SET_COMMENT_IF_EMPTY = False   # set True to preserve any existing user comments

def get_text_seg():
    seg = ida_segment.get_segm_by_name(".text")
    if seg: return seg.start_ea, seg.end_ea
    for s in idautils.Segments():
        sg = ida_segment.getseg(s)
        if sg and (sg.perm & ida_segment.SEGPERM_EXEC):
            return sg.start_ea, sg.end_ea
    inf = idaapi.get_inf_structure()
    return inf.min_ea, inf.max_ea

def get_opnd(ea, n): return idc.print_operand(ea, n) or ""
def mnem(ea): return (idc.print_insn_mnem(ea) or "").lower()

def is_gpr64(txt: str) -> bool:
    regs = {"rax","rbx","rcx","rdx","rsi","rdi","rbp","rsp",
            "r8","r9","r10","r11","r12","r13","r14","r15"}
    return (txt.strip().lower() in regs)

def reg_name(txt: str) -> str:
    return (txt or "").strip().lower()

def qword_at(ea):
    try: return ida_bytes.get_qword(ea)
    except Exception: return None

def parse_off_addr_from_operand_text(op_text: str):
    m = re.search(r'(off_[0-9A-Fa-f]+|0x[0-9A-Fa-f]+)', op_text or "")
    if not m: return idaapi.BADADDR
    tok = m.group(1)
    if tok.startswith("off_"): return idc.get_name_ea_simple(tok)
    try: return int(tok, 16)
    except Exception: return idaapi.BADADDR

def add_call_xref_safe(call_ea, target_ea):
    if call_ea in (idaapi.BADADDR,) or target_ea in (idaapi.BADADDR,): return False
    # skip if exists
    for xr in idautils.XrefsFrom(call_ea, 0):
        if xr.to == target_ea and xr.type == ida_xref.fl_CF:
            return True
    try:
        ida_xref.add_cref(call_ea, target_ea, ida_xref.fl_CF)
        return True
    except Exception:
        return False

def current_callee_name(ea) -> str:
    """
    Get current visible name of callee without renaming/creating.
    If no name, return formatted address.
    """
    name = idc.get_name(ea, idc.GN_VISIBLE)
    if not name:
        # fall back to function name if it exists (still no rename)
        fn = ida_funcs.get_func_name(ea) or ""
        name = fn if fn else f"{ea:#x}"
    return name

def main():
    print("[*] resolve_indirect_calls_ida9_v5_2.py - resolving + XREF + callee-name comments…")
    text_start, text_end = get_text_seg()
    resolved = 0

    ea = text_start
    while ea < text_end:
        # Anchor: mov rax, cs:off_XXXXXXXX
        if mnem(ea) != "mov": ea = idc.next_head(ea, text_end); continue
        if "rax" not in get_opnd(ea, 0): ea = idc.next_head(ea, text_end); continue
        op1 = get_opnd(ea, 1)
        if not ("off_" in op1 or "qword ptr cs" in op1 or "qword ptr ds" in op1):
            ea = idc.next_head(ea, text_end); continue

        off_addr = idc.get_operand_value(ea, 1)
        if off_addr in (0, idaapi.BADADDR):
            off_addr = parse_off_addr_from_operand_text(op1)
        if off_addr in (0, idaapi.BADADDR):
            ea = idc.next_head(ea, text_end); continue

        base_ptr = qword_at(off_addr)
        if base_ptr is None:
            ea = idc.next_head(ea, text_end); continue

        aliases = {"rax"}   # registers equal to base_ptr
        imm_by_reg = {}     # reg -> imm64

        op_site  = idaapi.BADADDR
        op_kind  = None     # "add"/"sub"
        used_src = None
        imm_val  = None
        call_ea  = idaapi.BADADDR
        call_reg = None

        cursor = idc.next_head(ea, text_end)
        stop   = min(text_end, ea + SCAN_LIMIT_BYTES)

        def propagate_alias(dst, src=None):
            dst = reg_name(dst); src = reg_name(src) if src else None
            if not is_gpr64(dst): return
            if src and is_gpr64(src) and src in aliases:
                aliases.add(dst)
            else:
                if dst in aliases: aliases.discard(dst)

        while cursor != idaapi.BADADDR and cursor < stop:
            mn  = mnem(cursor)
            o0  = reg_name(get_opnd(cursor, 0))
            o1t = reg_name(get_opnd(cursor, 1))

            if mn == "mov":
                if is_gpr64(o0) and idc.get_operand_type(cursor, 1) == idc.o_imm:
                    imm_by_reg[o0] = idc.get_operand_value(cursor, 1) & 0xFFFFFFFFFFFFFFFF
                elif is_gpr64(o0) and is_gpr64(o1t):
                    propagate_alias(o0, o1t)

            elif mn == "xchg":
                if is_gpr64(o0) and is_gpr64(o1t):
                    if o0 in aliases or o1t in aliases:
                        aliases.add(o0); aliases.add(o1t)

            elif mn == "lea":
                op1_full = (idc.print_operand(cursor, 1) or "").strip().lower()
                m = re.fullmatch(r'\[\s*(r[0-9a-z]+)\s*\]', op1_full)
                if is_gpr64(o0) and m:
                    base = m.group(1)
                    if base in aliases: aliases.add(o0)
                    else:
                        if o0 in aliases: aliases.discard(o0)

            elif mn in ("add", "sub"):
                matched = False
                # Case A: add/sub dst, reg_with_known_imm
                if is_gpr64(o0) and o0 in aliases and is_gpr64(o1t) and o1t in imm_by_reg:
                    op_site = cursor; op_kind = mn; used_src = o1t
                    imm_val = imm_by_reg[o1t]; matched = True
                # Case B: add/sub dst, imm
                if (not matched) and is_gpr64(o0) and o0 in aliases and idc.get_operand_type(cursor, 1) == idc.o_imm:
                    op_site = cursor; op_kind = mn; used_src = None
                    imm_val = idc.get_operand_value(cursor, 1) & 0xFFFFFFFFFFFFFFFF
                    matched = True

                if matched:
                    # find 'call <alias>'
                    cur2 = idc.next_head(cursor, text_end)
                    limit2 = min(text_end, cursor + CALL_LIMIT_AFTER_OP)
                    while cur2 != idaapi.BADADDR and cur2 < limit2:
                        if mnem(cur2) == "call":
                            callee = reg_name(get_opnd(cur2, 0))
                            if callee in aliases:
                                call_ea = cur2; call_reg = callee; break
                        cur2 = idc.next_head(cur2, text_end)
                    break

            cursor = idc.next_head(cursor, text_end)

        if op_site == idaapi.BADADDR or call_ea == idaapi.BADADDR or imm_val is None:
            ea = idc.next_head(ea, text_end); continue

        mask   = 0xFFFFFFFFFFFFFFFF
        target = ((base_ptr - imm_val) if op_kind == "sub" else (base_ptr + imm_val)) & mask

        # Add XREF (no rename, no function creation)
        add_call_xref_safe(call_ea, target)

        # Get the CURRENT callee name (no rename). If none, use address string.
        callee_name = current_callee_name(target)

        # Set/keep the call-site comment to just the callee name
        existing = idc.get_cmt(call_ea, 0) or ""
        if ONLY_SET_COMMENT_IF_EMPTY:
            if not existing:
                idc.set_cmt(call_ea, callee_name, 0)
        else:
            if existing != callee_name:
                idc.set_cmt(call_ea, callee_name, 0)

        resolved += 1
        ea = idc.next_head(call_ea, text_end)

    print(f"[*] Done. Resolved {resolved} call sites; call-site comments set to current callee names.")

if __name__ == "__main__":
    main()
```

Post-execution, comments display callee names, and XREFs enable navigation.

```assembly
.text:000000014003A2A8                 mov     rax, cs:off_1400B8440
.text:000000014003A2AF                 mov     rcx, 0DC4D1CFA717B15E1h
.text:000000014003A2B9                 add     rax, rcx
.text:000000014003A2BC                 lea     rcx, [rbp+7230h+var_3E60]
.text:000000014003A2C3                 call    rax             ; sub_14005CB50
.text:000000014003A2C5                 mov     rax, [rax]
.text:000000014003A2C8                 mov     qword ptr [rax+20h], 146Bh
.text:000000014003A2D0                 mov     [rbp+7230h+var_738], 146Bh
.text:000000014003A2DB                 mov     rax, cs:off_1400B1EE0
.text:000000014003A2E2                 mov     rcx, 1E6F4B38BD1901B5h
.text:000000014003A2EC                 add     rax, rcx
.text:000000014003A2EF                 lea     rcx, [rbp+7230h+var_3FB0]
.text:000000014003A2F6                 call    rax             ; sub_14001E7B0
.text:000000014003A2F8                 mov     rcx, [rax]
.text:000000014003A2FB                 mov     rax, cs:off_1400ADDD0
.text:000000014003A302                 mov     rdx, 0CD5CA65684621C9Ah
.text:000000014003A30C                 add     rax, rdx
.text:000000014003A30F                 call    rax             ; sub_140086AB0
.text:000000014003A311                 mov     rax, [rax]
.text:000000014003A314                 add     rax, 10h
.text:000000014003A318                 mov     [rbp+7230h+var_370], rax
.text:000000014003A31F                 mov     rax, cs:off_1400A4428
.text:000000014003A326                 mov     rcx, 0DD4DBD41A35BB9FCh
.text:000000014003A330                 add     rax, rcx
.text:000000014003A333                 lea     rcx, [rbp+7230h+var_4910]
.text:000000014003A33A                 call    rax             ; sub_1400016E0
.text:000000014003A33C                 mov     rcx, [rax]
.text:000000014003A33F                 mov     rax, cs:off_1400A6B08
.text:000000014003A346                 mov     rdx, 0AA18916841B77C41h
.text:000000014003A350                 add     rax, rdx
.text:000000014003A353                 call    rax             ; sub_14000D030
.text:000000014003A355                 mov     rcx, [rax]
.text:000000014003A358                 mov     rax, cs:off_1400AEE38
.text:000000014003A35F                 mov     rdx, 4F7DF4F0D44FD46Eh
.text:000000014003A369                 add     rax, rdx
.text:000000014003A36C                 call    rax             ; sub_14007C620
...
```

For example, a resolved call points to `sub_14005CB50`, a simple accessor returning `rcx + 8`. Restored XREFs now propagate throughout the binary.

```assembly
.text:000000014005CB50 ; =============== S U B R O U T I N E =======================================
.text:000000014005CB50
.text:000000014005CB50
.text:000000014005CB50 sub_14005CB50   proc near               ; CODE XREF: sub_140037160+3163↑P
.text:000000014005CB50                 mov     rax, rcx
.text:000000014005CB53                 add     rax, 8
.text:000000014005CB57                 retn
.text:000000014005CB57 sub_14005CB50   endp
```

## Control-flow mangling
Computed jumps blend memory indirection, bitwise operations, and arithmetic identities to hide destinations, fracturing IDA's CFG and decompiler output.

```assembly
.text:000000014003A44C                 mov     rax, cs:off_1400AA918
.text:000000014003A453                 mov     rcx, 0D9C148F7B07D346Ch
.text:000000014003A45D                 mov     rcx, [rax+rcx]
.text:000000014003A461                 mov     r8, 0F493E10412D3F4FBh
.text:000000014003A46B                 mov     rax, rcx
.text:000000014003A46E                 or      rax, r8
.text:000000014003A471                 mov     r9, rcx
.text:000000014003A474                 sub     r9, rax
.text:000000014003A477                 mov     rdx, 0E927C20825A7E9F6h
.text:000000014003A481                 lea     rdx, [rdx+r9*2]
.text:000000014003A485                 and     rcx, r8
.text:000000014003A488                 sub     rax, rcx
.text:000000014003A48B                 mov     rcx, rax
.text:000000014003A48E                 or      rcx, rdx
.text:000000014003A491                 and     rax, rdx
.text:000000014003A494                 add     rax, rcx
.text:000000014003A497                 jmp     rax
```

Decompiling yields nonsensical expressions involving modular arithmetic and identity operations—classic opaque predicates designed to confuse tools while preserving semantics.
```C
  ...
  v395 = ((v5 | (v4 - ((842453476 * v395 % 0x45DB5247) & 0x6E67061A)))
        + (v5 & (v4 - ((842453476 * v395 % 0x45DB5247) & 0x6E67061A))))
       % 0x45DB5247;
  v6 = *(_QWORD *)((char *)off_1400AA918 - 0x263EB7084F82CB94LL) | 0xF493E10412D3F4FBuLL;
  v7 = 2 * (*(_QWORD *)((char *)off_1400AA918 - 0x263EB7084F82CB94LL) - v6) - 0x16D83DF7DA58160ALL;
  __asm { jmp     rax }
}
```

## Anti-disassembly / Anti-linear-sweep
Unconditional computed jumps (`jmp rax`) frequently land in the middle of subsequent instructions, causing IDA’s linear sweep to misinterpret code bytes as data. Garbage bytes are strategically inserted to exacerbate this effect.

```c
.text:000000014003A6A7                 add     rax, rcx
.text:000000014003A6AA                 jmp     rax
.text:000000014003A6AA ; ---------------------------------------------------------------------------
.text:000000014003A6AC                 dd 8B4800EBh
.text:000000014003A6B0                 dq 1BB9480007736B05h, 4844FC86EAE3452Dh, 20B885894808048Bh
.text:000000014003A6C8                 dq 6ABEF058B480000h, 0C1B92C9DF7B94800h, 8D48C801489B14EEh
.text:000000014003A6E0                 dq 48D0FF00002CE08Dh, 6911F058B48088Bh, 0DE8D950730BA4800h
.text:000000014003A6F8                 dq 0D0FFD00148FD769Fh, 41E058B48088B48h, 98FECFA1BA480008h
.text:000000014003A710                 dq 0FFD00148D74B3204h, 9D058B48088B48D0h, 0ACD59FBA48000771h
.text:000000014003A728                 dq 0D00148E7A955E2ACh, 2118958B48D0FFh, 0B8858B48C1894800h
.text:000000014003A740                 dq 8948098B48000020h, 0BC5C7612DDB94811h, 8948C801480C20C7h
.text:000000014003A758                 dq 58B48000020C085h, 0B587B9480006A29Ch, 148D3A5267D4CC8h
.text:000000014003A770                 dq 2B608D8D48C8h, 58B48088B48D0FFh, 0C610BA4800065D74h
.text:000000014003A788                 dq 14833B864839865h, 8B48C18948D0FFD0h, 98B48000020C085h
.text:000000014003A7A0                 dq 20E88D8948098B48h, 50090010B9480000h, 48C82148F0D0D200h
.text:000000014003A7B8                 dq 8B48000020C88589h, 7BB9480007348B05h, 481D09A93B5D2651h
.text:000000014003A7D0                 dq 0FF00000020B9C801h, 20C88D8B4CD0h, 8948C88948C18948h
.text:000000014003A7E8                 dq 4BB848000020D085h, 490C2225DEA7625Eh, 6AF8858B48C109h
.text:000000014003A800                 dq 5EF2D11C46BA4800h, 694CD131490E42DAh, 68B848252A29BAC0h
.text:000000014003A818                 dq 490E160CAD5A82F6h, 651FDD6DBA48C101h, 0D8958948EA898579h
.text:000000014003A830                 dq 0F748C0894C000020h, 0D8958B48D08948E2h, 481EE8C148000020h
.text:000000014003A848                 dq 294945DB5247C069h, 0F122676AB6B848C0h, 894CC1094969D157h
.text:000000014003A860                 dq 4DC5CF96B10D48C0h, 948D4FC22949C289h, 0E081418B9F2D6212h
.text:000000014003A878                 dq 49C0294C45CF96B1h, 0D0214CD0094DC089h, 20E0858948C0014Ch
```

![Anti-disassembly](https://lh3.googleusercontent.com/pw/AP1GczN4j21Mz5-d0ZyBaNezqnCDu0N2RfTyyjluNbwLowxoYjt8vmHt1Sidqc52Twt7-nF4R-vnLF2x-SrZl8xEBvn3uekyt0WosNkKFc7Bc_HB1WfxU3od2Ywe7JAWd9pbVRcD7EyYiYvv11w5jp8cK29k=w1212-h622-s-no-gm)
_**Figure: Anti-disassembly**_

Manually selecting these regions and pressing 'C' forces IDA to reinterpret them as code, revealing hidden logic. 

```assembly
.text:000000014003A6A7                 add     rax, rcx
.text:000000014003A6AA                 jmp     rax
.text:000000014003A6AC ; ---------------------------------------------------------------------------
.text:000000014003A6AC                 jmp     short $+2
.text:000000014003A6AE ; ---------------------------------------------------------------------------
.text:000000014003A6AE
.text:000000014003A6AE loc_14003A6AE:                          ; CODE XREF: sub_140037160+354C↑j
.text:000000014003A6AE                 mov     rax, cs:off_1400B1A20
.text:000000014003A6B5                 mov     rcx, 44FC86EAE3452D1Bh
.text:000000014003A6BF                 mov     rax, [rax+rcx]
.text:000000014003A6C3                 mov     [rbp+7230h+var_5178], rax
.text:000000014003A6CA                 mov     rax, cs:off_1400A52C0
.text:000000014003A6D1                 mov     rcx, 9B14EEC1B92C9DF7h
.text:000000014003A6DB                 add     rax, rcx
.text:000000014003A6DE                 lea     rcx, [rbp+7230h+var_4550]
.text:000000014003A6E5                 call    rax
.text:000000014003A6E7                 mov     rcx, [rax]
.text:000000014003A6EA                 mov     rax, cs:off_1400A3810
.text:000000014003A6F1                 mov     rdx, 0FD769FDE8D950730h
.text:000000014003A6FB                 add     rax, rdx
.text:000000014003A6FE                 call    rax
.text:000000014003A700                 mov     rcx, [rax]
.text:000000014003A703                 mov     rax, cs:off_1400BAB28
```
However, with hundreds of such instances, automation is essential. A Python script scans for `jmp rax` patterns followed by non-code bytes, patches them with `jmp short $+2` (a two-byte no-op jump), and reconnects the CFG.

```python
# ida_patch_jmp_rax.py
# Purpose: When jmp rax is known to target the *next* instruction (fall-through),
#          replace it with a 2-byte no-op (or a short jmp to $+0) to recover CFG.
#
# Safe-bytes policy:
#   - Original 'jmp rax' = FF E0 (2 bytes)
#   - Replace with '90 90' (NOP; NOP)  -> keeps layout identical
#     OR use 'EB 00' (jmp short next)  -> also 2 bytes, preserves layout
#
# Config:
#   - LIMIT_TO_EAS: set of EAs to patch (default: {0x140016708})
#   - PATCH_ALL_MATCHES: set True to scan/patch all matches in .text
#   - REQUIRE_ADD_RAX_RCX_BEFORE: only patch if previous insn is "add rax, rcx"
#   - MODE: "patch_nop2" | "patch_jmp_next" | "xref_only"

import idaapi
import idautils
import ida_bytes
import ida_segment
import ida_xref
import ida_kernwin
import idc

# --------------------- Configuration ---------------------

LIMIT_TO_EAS = {0x140016708}   # your example site
PATCH_ALL_MATCHES = True       # set True to sweep all .text for jmp rax
REQUIRE_ADD_RAX_RCX_BEFORE = True

# Choose how we "fix" it
# MODE = "patch_nop2"             # "patch_nop2" | "patch_jmp_next" | "xref_only"
MODE = "patch_jmp_next"

# --------------------- Helpers ---------------------------

def is_jmp_rax(ea: int) -> bool:
    return idc.print_insn_mnem(ea).lower() == "jmp" and idc.print_operand(ea, 0).lower() == "rax"

def prev_is_add_rax_rcx(ea: int) -> bool:
    prev = ida_bytes.prev_head(ea, ea - 15)
    if prev == idc.BADADDR:
        return False
    return (
        idc.print_insn_mnem(prev).lower() == "add" and
        idc.print_operand(prev, 0).lower() == "rax" and
        idc.print_operand(prev, 1).lower() == "rcx"
    )

def next_head(ea: int) -> int:
    # conservative bound for instruction max length on x86-64 is < 16 bytes
    return ida_bytes.next_head(ea, ea + 16)

def in_text(ea: int) -> bool:
    seg = ida_segment.getseg(ea)
    return bool(seg) and (seg.perm & ida_segment.SEGPERM_EXEC) and seg.type == ida_segment.SEG_CODE

def add_cref_to_next(ea: int):
    tgt = next_head(ea)
    if tgt != idc.BADADDR:
        ida_xref.add_cref(ea, tgt, ida_xref.fl_JF)  # code xref: jump
        idc.set_cmt(ea, "Added xref to next to help CFG (indirect jmp->next).", 0)

def patch_to_nop2(ea: int):
    # Replace FF E0 with 90 90
    ida_bytes.patch_bytes(ea, b"\x90\x90")
    idc.create_insn(ea)
    idc.create_insn(next_head(ea))
    idc.set_cmt(ea, "Patched jmp rax -> NOP;NOP (fall-through).", 0)

def patch_to_jmp_next(ea: int):
    # Replace with EB 00 (jmp short +0), keeps 2-byte length
    ida_bytes.patch_bytes(ea, b"\xEB\x00")
    idc.create_insn(ea)
    idc.create_insn(next_head(ea))
    idc.set_cmt(ea, "Patched jmp rax -> jmp short $+0.", 0)

def should_consider_site(ea: int) -> bool:
    if not in_text(ea):
        return False
    if not is_jmp_rax(ea):
        return False
    if REQUIRE_ADD_RAX_RCX_BEFORE and not prev_is_add_rax_rcx(ea):
        return False
    if PATCH_ALL_MATCHES:
        return True
    return ea in LIMIT_TO_EAS

# --------------------- Main ------------------------------

def process_site(ea: int):
    if MODE == "xref_only":
        add_cref_to_next(ea)
        return True

    # sanity check: ensure original is 2-byte jmp rax (FF E0)
    orig = ida_bytes.get_bytes(ea, 2) or b""
    if orig != b"\xFF\xE0":
        # If IDA decoded as 'jmp rax' but bytes differ (e.g., thunk?), skip to be safe.
        idc.set_cmt(ea, f"Skipped: bytes {orig.hex()} not FF E0.", 0)
        return False

    if MODE == "patch_nop2":
        patch_to_nop2(ea)
        return True
    elif MODE == "patch_jmp_next":
        patch_to_jmp_next(ea)
        return True
    else:
        ida_kernwin.msg(f"[jmp-rax-fix] Unknown MODE '{MODE}'\n")
        return False

def collect_candidates():
    cands = []
    if PATCH_ALL_MATCHES:
        # Scan executable CODE segments
        for seg_ea in idautils.Segments():
            if not in_text(seg_ea):
                continue
            seg = ida_segment.getseg(seg_ea)
            ea = seg.start_ea
            while ea != idc.BADADDR and ea < seg.end_ea:
                if idc.is_code(idc.get_full_flags(ea)) and is_jmp_rax(ea):
                    if should_consider_site(ea):
                        cands.append(ea)
                ea = ida_bytes.next_head(ea, seg.end_ea)
    else:
        # Only consider explicit addresses
        for ea in LIMIT_TO_EAS:
            if idc.is_code(idc.get_full_flags(ea)) and is_jmp_rax(ea) and should_consider_site(ea):
                cands.append(ea)
    return sorted(set(cands))

def main():
    ida_kernwin.msg("[jmp-rax-fix] Starting...\n")
    cands = collect_candidates()
    ida_kernwin.msg(f"[jmp-rax-fix] Candidates: {', '.join(hex(x) for x in cands) or 'none'}\n")

    patched = 0
    for ea in cands:
        ok = process_site(ea)
        if ok:
            patched += 1

    ida_kernwin.msg(f"[jmp-rax-fix] Done. {patched} site(s) processed in MODE='{MODE}'.\n")

if __name__ == "__main__":
    main()
```

![Patched jmp rax](https://lh3.googleusercontent.com/pw/AP1GczPbde4Hue9eJENT3GbtKafCmvtDtDhW_RCXqW4yEXbZPiSivGJJlWkBHuyb_bY7iuDH_ZhWS3Mqkep_A7BCYGMPy3sDrM1qEegkQ3CU5YVj_IhqnaNm70w2MVTZ1foLt18AntyWSjUphE-DuQ-Vo8sF=w1100-h622-s-no-gm)
_**Figure: Patched jmp rax**_

Post-patching, 656 locations are corrected, dramatically improving graph coherence.
```text
[jmp-rax-fix] Starting...
[jmp-rax-fix] Candidates: 0x140002f29, 0x140003017, 0x1400031ff, 0x140003766, 0x1400037d8, 0x1400039ca, 0x140003f9b, 0x140004209, 0x140004912, 0x140004a49, 0x140004d93, 0x140005100, 0x1400053ae, 0x14000722d, 0x140007499, 0x140007589, 0x140007784, 0x140007891, 
...
0x14007e95e, 0x14007ebe8, 0x14007ed7e, 0x14007f2a0, 0x14007f3fd, 0x14007f56f, 0x14007f6c7, 0x140082713, 0x140082c3f, 0x140082d6e, 0x1400831d8, 0x1400836ba, 0x140083b09, 0x140089845
[jmp-rax-fix] Done. 656 site(s) processed in MODE='patch_jmp_next'.
```

![Patched jmp rax CFG](https://lh3.googleusercontent.com/pw/AP1GczPApBX6Dv5kbl5wsUZDpGqQPMoTJNQ5W6HMQ0zmIqp-mhIjfpa27o7AqHdsuuSznPD96rPd_IzPp3Yh2SLQf75mKTaekn1YPL2_asA70FmdNsgbdkyvzDKI-enHexWn7XjmZudgF_EU6UrA_9jdffG8=w466-h622-s-no-gm)
_**Figure: Patched jmp rax CFG**_

The binary now supports focused tracing of message origins.

## Trace `QMessageBox` Calls
Validation feedback arrives via QT’s `QMessageBox`. Searching for `information` and `warning` in the strings window, then demangling names, locates the imported thunks. Demangling via `Options > Demangled names > Names` clarifies signatures.

```assembly
.text:000000014008DDB0 public: static enum QMessageBox::StandardButton QMessageBox::information(class QWidget *, class QString const &, class QString const &, class QFlags<enum QMessageBox::StandardButton>, enum QMessageBox::StandardButton) proc near
.text:000000014008DDB0                 jmp     cs:QMessageBox::information(QWidget *,QString const &,QString const &,QFlags<QMessageBox::StandardButton>,QMessageBox::StandardButton)
.text:000000014008DDB0 public: static enum QMessageBox::StandardButton QMessageBox::information(class QWidget *, class QString const &, class QString const &, class QFlags<enum QMessageBox::StandardButton>, enum QMessageBox::StandardButton) endp
.text:000000014008DDB0
...

.text:000000014008E030 public: static enum QMessageBox::StandardButton QMessageBox::warning(class QWidget *, class QString const &, class QString const &, class QFlags<enum QMessageBox::StandardButton>, enum QMessageBox::StandardButton) proc near
.text:000000014008E030                 jmp     cs:QMessageBox::warning(QWidget *,QString const &,QString const &,QFlags<QMessageBox::StandardButton>,QMessageBox::StandardButton)
.text:000000014008E030 public: static enum QMessageBox::StandardButton QMessageBox::warning(class QWidget *, class QString const &, class QString const &, class QFlags<enum QMessageBox::StandardButton>, enum QMessageBox::StandardButton) endp
.text:000000014008E030
```

Cross-referencing `warning` leads to a disconnected code island. 

![QMessageBox::warning XREF](https://lh3.googleusercontent.com/pw/AP1GczOO1bW_pKhW55FWBVx-DUkDAtt-N2dbIuU2XQE-oBocsGi9-KeZjH-PBoWgqsz1ZTTIWd2udErLUBcwJrHJt_IJLwMOHgKCEoYJGmy4Z_jcYL6FoG1JEln6qOLz_QyfIDypJcDrVwWFFf3Ro1-DU2Q6=w2364-h602-s-no-gm)
_**Figure: QMessageBox::warning XREF**_

Disassembly shows a single orphan byte (`0x48`) separating a computed jump from its target call—a byproduct of prior `jmp rax` misalignment.

```assembly
.text:000000014002A4DD                 mov     rax, cs:off_1400B7878
.text:000000014002A4E4                 mov     r10, 0B77E22513A2CE62Ch
.text:000000014002A4EE                 add     rax, r10
.text:000000014002A4EE ; ---------------------------------------------------------------------------
.text:000000014002A4F1                 db  48h ; H
.text:000000014002A4F2 ; ---------------------------------------------------------------------------
.text:000000014002A4F2
.text:000000014002A4F2 loc_14002A4F2:                       ; DATA XREF: .rdata:000000014009AB14↓o
.text:000000014002A4F2 ;   try {
.text:000000014002A4F2                 sub     esp, 30h
.text:000000014002A4F5                 mov     r10, rsp
.text:000000014002A4F8                 mov     dword ptr [r10+20h], 0
.text:000000014002A500                 call    rax             ; ?warning@QMessageBox
.text:000000014002A502                 add     rsp, 30h
.text:000000014002A502 ;   } // starts at 14002A4F2
```

Undefining and redefining the region as code restores continuity - the CFG reconnected.

```assembly
.text:000000014002A4DD                 mov     rax, cs:off_1400B7878
.text:000000014002A4E4                 mov     r10, 0B77E22513A2CE62Ch
.text:000000014002A4EE                 add     rax, r10
.text:000000014002A4EE ; ---------------------------------------------------------------------------
.text:000000014002A4F1                 db  48h ; H
.text:000000014002A4F2 ;   try {
.text:000000014002A4F2                 db  83h                 ; DATA XREF: .rdata:000000014009AB14↓o
.text:000000014002A4F3                 db 0ECh
.text:000000014002A4F4                 db  30h ; 0
.text:000000014002A4F5                 db  49h ; I
.text:000000014002A4F6                 db  89h
.text:000000014002A4F7                 db 0E2h
.text:000000014002A4F8                 db  41h ; A
.text:000000014002A4F9                 db 0C7h
.text:000000014002A4FA                 db  42h ; B
.text:000000014002A4FB                 db  20h
.text:000000014002A4FC                 db    0
.text:000000014002A4FD                 db    0
.text:000000014002A4FE                 db    0
.text:000000014002A4FF                 db    0
.text:000000014002A500                 db 0FFh
.text:000000014002A501                 db 0D0h
.text:000000014002A502                 db  48h ; H
.text:000000014002A503                 db  83h
.text:000000014002A504                 db 0C4h
.text:000000014002A505                 db  30h ; 0
.text:000000014002A505 ;   } // starts at 14002A4F2
```

```assembly
.text:000000014002A4DD                 mov     rax, cs:off_1400B7878
.text:000000014002A4E4                 mov     r10, 0B77E22513A2CE62Ch
.text:000000014002A4EE                 add     rax, r10
.text:000000014002A4F1
.text:000000014002A4F1 loc_14002A4F1:                          ; DATA XREF: .rdata:000000014009AB14↓o
.text:000000014002A4F1                 sub     rsp, 30h
.text:000000014002A4F5                 mov     r10, rsp
.text:000000014002A4F8                 mov     dword ptr [r10+20h], 0
.text:000000014002A500                 call    rax             ; ?warning@QMessageBox
.text:000000014002A502                 add     rsp, 30h
.text:000000014002A502 ;   } // starts at 14002A4F2
```

![Orphaned byte fixes CFG](https://lh3.googleusercontent.com/pw/AP1GczPCeXY8gAhtDmbocoJyOGc9ijr0VyFKSv0cgQ-2mzvRRxgmT4SHxkuZ7YZs3cSAadOzgWlKYILKoe9XrqMYijnBryYMyJTjJcB5dxoCJati1ox3duB8I-aX-dMX8S2JVzn8JOkeyyRPXbjkpV4neUNs=w626-h654-s-no-gm)
_**Figure: Orphaned byte fixes CFG**_

Upward traversal reveals a conditional branch controlled by a boolean variable—renamed `v440_is_equal_final_constants`.

![Conditional jump](https://lh3.googleusercontent.com/pw/AP1GczOzBPRqIYQbNfHSUZbddnQ2QzNovO8SrpJU2xptAl0vHh8jSFwnYnahJDCZtrFpcDXHVPTFDmEl6BBUhLir5RlCD0b0UxAly87duI_WB1UpIWlGYriQfvJXZBDz5bqr-87MfP1vEmKXqmbNHU2VwGyf=w1712-h654-s-no-gm)
_**Figure: Conditional jump**_

The check `al & 1` branches if non-zero. The true branch displays the success message via `QMessageBox::information`; the false branch shows the warning.

```assembly
.text:00000001400268A0 loc_1400268A0:                          ; DATA XREF: .rdata:000000014009AAD4↓o
.text:00000001400268A0                 sub     rsp, 30h
.text:00000001400268A4                 mov     r10, rsp
.text:00000001400268A7                 mov     dword ptr [r10+20h], 0
.text:00000001400268AF                 call    rax             ; ?information@QMessageBox
```

This flag is set when a 64-bit value at `[rax+78h]` equals `0x0BC42D5779FEC401`. 

```assembly
.text:0000000140021E1B                 mov     rax, [rbp+2950h+var_3C8]
.text:0000000140021E22                 mov     [rbp+2950h+var_21D0], rax
.text:0000000140021E29                 mov     rax, [rax+78h]
.text:0000000140021E2D                 mov     rcx, 0BC42D5779FEC401h
.text:0000000140021E37                 sub     rax, rcx
.text:0000000140021E3A                 setz    al
.text:0000000140021E3D                 mov     [rbp+2950h+v440_is_equal_final_constants], al
```

```C
uint64_t v = *(uint64_t*)(obj + 0x78);
uint8_t is_equal = (v == 0x0BC42D5779FEC401);
v440_is_equal_final_constants = is_equal;
```

Static XREFs to `[rax+78h]` terminate here, indicating runtime computation. Dynamic analysis is now required to trace how this value accumulates. The enclosing function is renamed `handle_display_flag_sub_1400202B0`.

# Dynamic Analysis
The ret-sync plugin synchronizes x64dbg with IDA, enabling seamless jumps between disassembly and debugger. In IDA, the plugin is activated via `Edit > Plugins > ret-sync`. In x64dbg, `!load sync; !sync` establishes the connection.

![IDA ret-sync](https://lh3.googleusercontent.com/pw/AP1GczMa_h2OH9fKvt6iMScjwS3dec3Ue8Ol5riFkPtmu9CxJDJlLMoDb9YNKHkhSJR4O00LdxnVB6jVeFxsJdLITVIztLBre7OlKffTuVGgxla52niFGqd6Ruex8oBlqs2DITw9Mt6ZVmLRn5qTu9uhFE__=w1918-h590-s-no-gm)
_**Figure: IDA ret-sync**_

## x64dbg - hardware breakpoint
`run_x64dbg.bat` launches debugging appropriately.

```bash
@echo off
set QT_QPA_PLATFORM_PLUGIN_PATH=%~dp0
start C:\x64dbg_snapshot_2025-08-19_19-40\release\x64\x64dbg.exe
```

Attachment to `FlareAuthenticator.exe` sets a breakpoint at the comparison (ASLR-adjusted). Entering 25 digits and confirming hits it. Sync activates via `!sync`.

![x64dbg and IDA are in sync](https://lh3.googleusercontent.com/pw/AP1GczPp_adsQojK2LdQiVsqNml2GM1Bob-TbKP3Uh8Dxm1BSUkcHlRe6cmxgMJrMnmVzVPBfUykbygLY7HkRKhyEostRiLlO_hol1v-TXEWwpuG-Cd94C1zj9TjRsue5ObFDDJ9Zc8kM1pCvVH4q8WIa6kn=w1856-h654-s-no-gm)
_**Figure: x64dbg and IDA are in sync**_

A hardware breakpoint is placed on access to `[rax+78h]` after setting a software breakpoint at the comparison site. Entering 25 arbitrary digits and pressing "OK" triggers the comparison. Following `[rax+78h]` in the memory dump and setting a hardware access breakpoint (`Qword`) captures write operations.

![x64dbg Follow address in dump](https://lh3.googleusercontent.com/pw/AP1GczNN-mLSuUBw-6_-D9Iu2LFA_7Jc3wC_aqQPMpy6lxfHbQUaJndanw7y9uXC8gKNn9LAtN-0QISyTM7VBYEMsdPO73U4eKfTTm9jvf_DwkjR99Z2w89MNp5hTschY8J5viAXQ5x4v3euor5NWK82U69T=w1066-h736-s-no-gm)
_**Figure: x64dbg Follow address in dump**_

![x64dbg set hardware breakpoint access](https://lh3.googleusercontent.com/pw/AP1GczPOMoyPJI-Nfsp5GCP1qnOa2PWrXrV1aHVRYtOgc_ijuHgy4uS93DZ2OYw6hU6M4vYA25eEBWM4h7QP9419nJK7fu2EjK6qTB1O_Su9oQr_Js4DalueS92ZtV2fD6GN3EcLpM0womww2Ju_LqTH-CPs=w776-h736-s-no-gm)
_**Figure: x64dbg set hardware breakpoint access**_

The first hit occurs in `sub_140012E50`, where a 64-bit addition routine updates the global value using a product stored in `var_C78`:

![x64dbg breakpoint hits](https://lh3.googleusercontent.com/pw/AP1GczPadZOUkQ3QdAWovn81NZqghEokt3iN21FL4pABrHWczJPbDWkieosSi5xiXXPDiRuLB9C3SjeIMx3Z4BOZThtyFqKHgZmXD5H7pm62W4suxSQWtD8vxp-eBFQwJHJPQk0xXCf5zyIIV_KXFnxMjlw6=w1378-h736-s-no-gm)
_**Figure: x64dbg hardware breakpoint hits**_

Sync places it in `sub_140012E50`.

```assembly
.text:0000000140016AC9                 mov     r9, [rbp+0FC0h+var_C78]
.text:0000000140016AD0                 mov     rdx, [rax+78h]
.text:0000000140016AD4                 mov     rcx, r9
.text:0000000140016AD7                 not     rcx
.text:0000000140016ADA                 mov     r8, rdx
.text:0000000140016ADD                 not     r8
.text:0000000140016AE0                 or      r8, rcx
.text:0000000140016AE3                 mov     rcx, rdx
.text:0000000140016AE6                 add     rcx, r9
.text:0000000140016AE9                 lea     r8, [r8+rcx+1]
.text:0000000140016AEE                 or      rdx, r9
.text:0000000140016AF1                 sub     rcx, rdx
.text:0000000140016AF4                 mov     rdx, rcx
.text:0000000140016AF7                 or      rdx, r8
.text:0000000140016AFA                 and     rcx, r8
.text:0000000140016AFD                 add     rcx, rdx
.text:0000000140016B00                 mov     [rax+78h], rcx
```

Backtracing `var_C78` reveals it is the result of multiplying two return values from `sub_140081760`:

## Tracing xrefs to **var_C78**
XREFs reveal write locations.

![xrefs to var_C78](https://lh3.googleusercontent.com/pw/AP1GczNuqghVc_lQIxhvEeHhBKsjBIutVaBpq4Xd4B3EJQ35amppuLIy_IruBJNqUrosBm2wwCWfzOSnHk9UOZGY2aH7Dd4weNp0Iln-F6GfotXfCQkLwy4qRICSRqWABM6gnkJWg9RoAPzlabgXHszdvU8f=w1378-h412-s-no-gm)
_**Figure: xrefs to var_C78**_

One update multiplies results from `sub_140081760` with `var_C00`.

```assembly
.text:0000000140016766                 call    rax             ; sub_140081760
.text:0000000140016768                 mov     rcx, rax
.text:000000014001676B                 mov     rax, [rbp+0FC0h+var_C00]
.text:0000000140016772                 imul    rax, rcx
.text:0000000140016776                 mov     [rbp+0FC0h+var_C78], rax
```

`var_C00` similarly derives from `sub_140081760`.

```assembly
.text:0000000140015E99                 call    rax             ; sub_140081760
.text:0000000140015E9B                 mov     rcx, [rbp+0FC0h+var_948]
.text:0000000140015EA2                 mov     [rbp+0FC0h+var_C00], rax
```

Both operands to the multiplication also originate from `sub_140081760`, suggesting that each digit contributes two derived values whose product is accumulated into the final 64-bit sum `global_rax_78h`. Frequent calls to `sub_140081760` lead to TTD for comprehensive tracing. Functions rename to `accumulate_final_value_sub_140012E50` and `derive_value_sub_140081760`.

## Time Travel Debugging with WinDbg TTD
To confirm the per-digit pattern, WinDbg’s Time Travel Debugging (TTD) records full execution traces for offline analysis. After attaching and enabling recording, the application is run with sample input "12345678909876543...". The **ret-sync** plugin again bridges WinDbg and IDA.

Ret-sync loads via `!load sync; !sync`, aligning with IDA.

![WinDbg TTD sync with IDA](https://lh3.googleusercontent.com/pw/AP1GczMWUW23MLuN5vQVfEUQ3C8h6BNiaGjaOCV8IWQ_PlqYAdkDbgYHZQo7JOOIPhmW0ZR-bAECbGCdJysJeZDGUe1XqQM89I-3WX9KfH7XJkxKiRfVBHRL2Q8Swt5NkvlBCaNbBBtMGpQ95_ePZHWty879=w1438-h736-s-no-gm)
_**Figure: WinDbg TTD sync with IDA**_

## TTD Queries - Find sequence calls
Queries analyze call patterns. Module base determination precedes address calculation.

```bash
0:000> lm m FlareAuthenticator
Browse full module list
start             end                 module name
00007ff7`92de0000 00007ff7`92eb1000   FlareAuthenticator C (no symbols)
```

TTD queries enumerate calls to key functions (with ASLR applied). This query identifies which functions are called from a given set of function addresses, counts their call frequency, and sorts them by execution order. The syntax is elegant, but it only works in Binary Ninja TTD; in WinDbg TTD, it fails due to line breaks. To make it work, I had to condense it into a single line as shown below.
```bash
# this version is working in Binary Ninja
dx -g @$cursession.TTD.Calls(
			0x7FF792E17160, 0x7FF792E3EF60, 0x7FF792E002B0, 0x7FF792DE1DE0,
			0x7FF792DF2E50, 0x7FF792E0F8C0, 0x7FF792DED1E0, 0x7FF792E5D290, 0x7FF792E61760
		).Select(c => c.FunctionAddress).Distinct()
		 .Select(addr => new {
			 Func  = addr,
			 Count = @$cursession.TTD.Calls(addr).ToArray().Select(_ => 1).Sum(),
			 First = @$cursession.TTD.Calls(addr).Select(c => c.TimeStart).Min(),
			 Last  = @$cursession.TTD.Calls(addr).Select(c => c.TimeStart).Max()
		 })
		 .OrderByDescending(x => x.First)
```

```bash
# WinDbg does not support line break
0:000>  dx -g @$cursession.TTD.Calls(0x7FF792E17160, 0x7FF792E3EF60, 0x7FF792E002B0, 0x7FF792DE1DE0,0x7FF792DF2E50, 0x7FF792E0F8C0, 0x7FF792DED1E0, 0x7FF792E5D290, 0x7FF792E61760).Select(c => c.FunctionAddress).Distinct().Select(addr => new { Func  = addr, Count = @$cursession.TTD.Calls(addr).ToArray().Select(_ => 1).Sum(), First = @$cursession.TTD.Calls(addr).Select(c => c.TimeStart).Min(), Last  = @$cursession.TTD.Calls(addr).Select(c => c.TimeStart).Max() }) .OrderBy(x => x.First)

=====================================================================
=          = Func              = Count = (+) First   = (+) Last     =
=====================================================================
= [0x0]    - 0x7ff792df2e50    - 25    - 1DD1:25A    - 4678:1BE1    =
= [0x1]    - 0x7ff792e61760    - 50    - 1E9E:A5D    - 4690:15BF    =
= [0x2]    - 0x7ff792e002b0    - 1     - 4EA1:55F    - 4EA1:55F     =
=====================================================================
```

The results align perfectly: 25 accumulations `accumulate_final_value_sub_140012E50` (one per digit) and 50 derivations `derive_value_sub_140081760` (two per digit). And finally once input 25 digits, click `OK` triggered  `handle_display_flag_sub_1400202B0` that only being called exactly 1.

## TTD Replay - Analyse **derive_value_sub_140081760**
Breakpoint at `derive_value_sub_140081760` with replay reveals parameters: `rcx` (object), `dx` (word). Calls alternate between digit index and combined index-digit ASCII.
![WinDbg TTD debug derive_value_sub_140081760](https://lh3.googleusercontent.com/pw/AP1GczMX2Nu19SOaLHiDCeahITQGnDPm0cj3x0MZ7ml2TdLl9208VRcXMerdKVeZBWwfXE2TufwC2xBeH7UITykTS-Wx3wS_EkSZtsS_4WU-itk44yIri4KPNBhTW2TZ4NGjNa3Yx8BLuPKV53HIk2sHuCo_=w1892-h736-s-no-gm)
_**Figure: WinDbg TTD debug derive_value_sub_140081760**_

```bash
O:00> !tt 0; # go back to beiggining of the trace
0:00> bc * ; # clear all breakpoints
0:00> bp 00007ff7`92e61760 "r dx; gc" ; # set breakpoint at derive_value_sub_140081760 and print dx value then resume 
0:00> g ; # start
dx=1
dx=131
dx=2
dx=232
dx=3
dx=333
dx=4
dx=434
dx=5
dx=535
dx=6
dx=636
dx=7
dx=737
dx=8
dx=838
dx=9
dx=939
dx=a
dx=a30
dx=b
dx=b39
dx=c
dx=c38
dx=d
dx=d37
dx=e
dx=e36
dx=f
dx=f35
dx=10
dx=1034
dx=11
dx=1133
dx=12
dx=1232
dx=13
dx=1331
dx=14
dx=1432
dx=15
dx=1533
dx=16
dx=1634
dx=17
dx=1735
dx=18
dx=1836
dx=19
dx=1937
```

The pattern yields:

```C
// ====== For every digit entered ========
secondParam = digitIndex ; // 1-based index
first_result = derive_value_sub_140081760(unknownObject, secondParam)

secondParam = (digitIndex << 8) | ord(digit)
second_result = derive_value_sub_140081760(unknownObject, secondParam)

var_C78 = first_result * second_result

// store to global
global_rax_78h += var_C78 ;

// ======== Once press OK ===========
if (global_rax_78h == 0x0BC42D5779FEC401) {
    QMessageBox::information(maybe_flag)
} else {
    QMessageBox::warning("wrong password")
}
```

Deterministic outputs per input allow black-box treatment and value capture for solving.

## Capturing All Derived Values for Analysis
With 25 positions and 10 possible digits, **250 unique products** must be collected. An x64dbg conditional breakpoint logs the product (`r9`) just before accumulation:

```assembly
.text:0000000140016AC9                 mov     r9, [rbp+0FC0h+var_C78]
.text:0000000140016AD0                 mov     rdx, [rax+78h]
.text:0000000140016AD4                 mov     rcx, r9
.text:0000000140016AD7                 not     rcx
.text:0000000140016ADA                 mov     r8, rdx
.text:0000000140016ADD                 not     r8
.text:0000000140016AE0                 or      r8, rcx
.text:0000000140016AE3                 mov     rcx, rdx
.text:0000000140016AE6                 add     rcx, r9
.text:0000000140016AE9                 lea     r8, [r8+rcx+1]
.text:0000000140016AEE                 or      rdx, r9
.text:0000000140016AF1                 sub     rcx, rdx
.text:0000000140016AF4                 mov     rdx, rcx
.text:0000000140016AF7                 or      rdx, r8
.text:0000000140016AFA                 and     rcx, r8
.text:0000000140016AFD                 add     rcx, rdx
.text:0000000140016B00                 mov     [rax+78h], rcx
```

![x64dbg conditional breakpoint log r9 to file](https://lh3.googleusercontent.com/pw/AP1GczPHHnPS5Eupuc1b-xBvd66nrGpwPHZb6BYNQTlIP5bRLh4CgwfP_5helxiLRPHkrIQL9IJ8C0glsjHSAlMQlTmxf1deXurbuIWf5dghFmoVlbhY8HOs4crSrIYqwH3RZpSj0bbxhasEXuspaCgIitOK=w1566-h736-s-no-gm)
_**Figure: x64dbg conditional breakpoint log r9 to file**_

A Python automation script simulates input across all combinations using Windows API calls to focus the window and send keystrokes such as keys 0, Delete, 1, Delete, ..., 9, 0 , Delete, ... until it finishes 250 posibilities

```python
# safe_focus_and_type.py
#!/usr/bin/env python3
"""
Safe foreground focus + typing script for Windows.

Sends: 0, Delete, 1, Delete, ..., 9 (no Delete after 9)
Repeats that sequence `--repeats` times (default 25).
Sleeps 0.5s after each Delete.

Safety:
 - If --title provided: finds first visible window whose title contains the substring,
   focuses it, and verifies the focused window title still contains the substring.
 - If --exe provided: attempts to verify the target window's process executable name matches.
 - If no --title provided: shows current foreground window title and requires interactive confirmation.
 - During typing, verifies foreground window remains the target window before each keypress. Aborts if not.

Usage examples:
  python safe_focus_and_type.py --title FlareAuthenticator  
"""

import sys
import time
import argparse

try:
    import win32gui
    import win32con
    import win32process
except Exception:
    print("Missing pywin32. Install with: pip install pywin32")
    raise SystemExit(1)

try:
    import pyautogui
except Exception:
    print("Missing pyautogui. Install with: pip install pyautogui")
    raise SystemExit(1)

# psutil is optional but recommended for exe checks
try:
    import psutil
    PSUTIL_AVAILABLE = True
except Exception:
    PSUTIL_AVAILABLE = False

pyautogui.FAILSAFE = False

def enum_windows_by_title_substring(substr):
    substr_lower = substr.lower()
    matches = []

    def _cb(hwnd, _):
        if not win32gui.IsWindowVisible(hwnd):
            return
        title = win32gui.GetWindowText(hwnd) or ""
        if substr_lower in title.lower():
            matches.append((hwnd, title))
    win32gui.EnumWindows(_cb, None)
    return matches  # list of (hwnd, title)

def get_foreground_hwnd():
    return win32gui.GetForegroundWindow()

def bring_window_to_front(hwnd):
    if not hwnd or not win32gui.IsWindow(hwnd):
        raise ValueError("Invalid window handle")
    try:
        win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)
    except Exception:
        pass
    try:
        win32gui.SetForegroundWindow(hwnd)
    except Exception:
        # Try AttachThreadInput trick
        try:
            import ctypes
            user32 = ctypes.windll.user32
            cur_fore = user32.GetForegroundWindow()
            fg_thread = user32.GetWindowThreadProcessId(cur_fore, 0)
            tgt_thread = user32.GetWindowThreadProcessId(hwnd, 0)
            user32.AttachThreadInput(tgt_thread, fg_thread, True)
            win32gui.SetForegroundWindow(hwnd)
            user32.AttachThreadInput(tgt_thread, fg_thread, False)
        except Exception:
            print("Warning: could not force-set foreground window. Proceeding anyway (may fail).")

def hwnd_matches_title(hwnd, substr):
    title = win32gui.GetWindowText(hwnd) or ""
    return substr.lower() in title.lower()

def get_process_name_for_hwnd(hwnd):
    try:
        _, pid = win32process.GetWindowThreadProcessId(hwnd)
        if PSUTIL_AVAILABLE:
            p = psutil.Process(pid)
            return p.name()  # e.g., "notepad.exe"
        else:
            return None
    except Exception:
        return None

def confirm_prompt_foreground(title):
    print("No --title provided. Current foreground window title detected:")
    print("  " + repr(title))
    print("To proceed, type 'yes' and press Enter. Anything else will abort.")
    resp = input("Proceed? (type 'yes' to continue) > ").strip().lower()
    return resp == "yes"

def abort(msg):
    print("ABORT:", msg)
    sys.exit(1)

def send_sequence(target_hwnd, repeats=25, delete_sleep=0.5, key_delay=0.05):
    for r in range(repeats):
        for d in range(10):
            # Verify target still foreground
            cur = get_foreground_hwnd()
            if cur != target_hwnd:
                abort(f"Foreground changed (hwnd {cur}) — aborting to avoid typing into wrong window.")
            digit = str(d)
            pyautogui.press(digit)
            time.sleep(key_delay)
            if d != 9:
                # verify again before delete
                cur = get_foreground_hwnd()
                if cur != target_hwnd:
                    abort(f"Foreground changed before sending Delete — aborting.")
                pyautogui.press('backspace')
                time.sleep(delete_sleep)
        # small pause between whole 0..9 sequences (optional)
        # time.sleep(0.02)

def main():
    parser = argparse.ArgumentParser(description="Safely focus a window and type 0..9 with deletes.")
    parser.add_argument("--title", "-t", help="Window title substring (case-insensitive). If provided, script will locate and focus the first matching visible window.")
    parser.add_argument("--exe", help="Optional process executable name to verify (e.g. notepad.exe). If provided, script will try to verify target HWND's process name matches.")
    parser.add_argument("--repeats", "-r", type=int, default=25, help="How many times to repeat 0..9 sequence. Default 25.")
    parser.add_argument("--dry-run", action="store_true", help="Print planned actions instead of sending keys.")
    args = parser.parse_args()

    target_hwnd = None
    target_title = None

    if args.title:
        matches = enum_windows_by_title_substring(args.title)
        if not matches:
            abort(f"No visible window found with title containing: {args.title!r}")
        target_hwnd, target_title = matches[0]
        print(f"Found window: hwnd={target_hwnd}, title={target_title!r}. Attempting to focus it.")
        bring_window_to_front(target_hwnd)
        time.sleep(0.2)
        # verify the focused window title still contains the substring
        fg = get_foreground_hwnd()
        if fg != target_hwnd:
            print("Warning: after focusing, foreground HWND differs. Will still verify title to be safe.")
        cur_title = win32gui.GetWindowText(get_foreground_hwnd()) or ""
        if not (args.title.lower() in cur_title.lower()):
            abort(f"Focused window title {cur_title!r} does not contain expected substring {args.title!r}. Aborting.")
        print("Focus verified by title match.")
    else:
        # use current foreground window, ask for confirmation
        fg = get_foreground_hwnd()
        if not fg:
            abort("No foreground window detected.")
        cur_title = win32gui.GetWindowText(fg) or "<no title>"
        if not confirm_prompt_foreground(cur_title):
            abort("User did not confirm. Exiting.")
        target_hwnd = fg
        target_title = cur_title
        print(f"Confirmed target HWND {target_hwnd}, title={target_title!r}")

    # optional exe verification
    if args.exe:
        procname = get_process_name_for_hwnd(target_hwnd)
        if procname is None:
            print("Warning: psutil not available or process name couldn't be determined. Skipping exe verification.")
        else:
            if procname.lower() != args.exe.lower():
                abort(f"Process executable name mismatch: expected {args.exe!r}, found {procname!r}. Aborting.")
            else:
                print(f"Process exe verified: {procname!r}")

    if args.dry_run:
        print("DRY RUN: Will send the following sequence:")
        for i in range(args.repeats):
            seq = []
            for d in range(10):
                seq.append(str(d))
                if d != 9:
                    seq.append("<Delete>")
            print("Repeat", i+1, ":", " ".join(seq))
        print("Dry run complete.")
        return

    print("Starting key sending in 2 seconds. Make sure the focused app is ready.")
    time.sleep(2.0)

    try:
        send_sequence(target_hwnd, repeats=args.repeats, delete_sleep=0.5)
    except KeyboardInterrupt:
        abort("Interrupted by user (Ctrl+C).")
    print("Completed successfully.")

if __name__ == "__main__":
    main()
```

The resulting flareauthenticator_x64dbg.log contains 250 lines of precomputed products.

```text
0x19B3240445AA06
0xF2EB6684284AC
0xC38D14DF6D665
0x1D3A557CD3A980
0x26D8E6DC23319D
0x1C544F24D1B429
0x17F846E0621293
0x1B6E4030F71F3D
0x80F42A7139E80
0x1FCAC56F82739C
0x6F63394844DF78
0x3ED168087F3548
0x6391D7049DDE8
0x8114813C91A610
0xA8FA7B20F1E228
0x786666BE83B978
0x6AE188A0BC46F0
0x6FB9A153C78328
0x1B18D762CB44C8
0x921C4279B1A9A8
0x6DF6A4586E71C0
0x49DE34FFC63EC0
0x39FEE368E99380
0x7A11DE14C79E80
0xD8DC90E996E40
0x7146BFC66E2740
0x680673DB489E00
0x6E31309B7B8940
0x316C3AFCB92AC0
0x82DEEFE21A73C0
0x4EA15FC542C9C0
0x1CA2F34F18CD40
0xA614B2DC1C6980
0x60D84A48824900
0x9280B83FADD240
0x60801A9A7374C0
0x49FE057966CC00
0x579189ABC15440
0xB2F907A133B2C0
0x612F00F6EA6080
0x3AC57453ACE252
0x6229B366FE169
0xACB3FF351198AB
0x4C6C9CDBCE1FE5
0x7C9128173EA742
0x47ED36357FF53C
0x321B2C8DECC760
0x436E1CAD683194
0x97DE278C5E1226
0x489730C9AA7E6A
0x6402164C9FDB19
0x483F192B06217F
0x1E5CB35F54FA69
0x6E1E405C548974
0x137E71B08CA2AA
0x6928A14A143271
0x616EB297EDDB28
0x6431716B60DF88
0x2A4A823D6D822D
0x6E4F38A8434132
0x69B5253875B96
0x2BE31F155AB714
0x21AE09901BF552
0xB221597F1D3D8
0x1556DDAA385C92
0x7D821185844EC
0x462D157005089
0x6B104399CEDBC
0x1E79B0A36CDAFF
0xA266DD02D7453
0x9C0D47EAC35D2D
0x6FD7CB84FD4AD7
0x6255B824338303
0xAC26E207A0A6C5
0x23700DD18B0E3C
0xABD8D1A448644C
0x97F36052CAA668
0xA3F30CB6971D36
0x4F56BF7AFE673B
0x708E71FCEE988
0x30B9DA3C1BFE7
0x16F0557C6F1B97
0x105A5256455ED6
0x575F94A1E493B
0xC0C1389DB1C28
0x4D8773736490A
0x1DC3A46D3EB22
0x43AFA29A5046E
0x12101A50F4E1FB
0x737572378FC07
0x3A03C1D1D02F29
0x255E081F63BA07
0x1B8260C83DD73C
0x4188E0049C9EBA
0x527EEC073C111B
0x3DD8DF72E747E1
0x38195731A5D0AE
0x3A28381B02C813
0x162F7A6E9D784F
0x48C67301DAFECB
0x1D392355DF459C
0xBC35FAA41240
0x4D5BA7B7C6DB28
0x26C6DD80773E62
0x3C52C884071124
0x1FD5838C6C7F50
0x188898B9E96D66
0x1D66839EF1A1C0
0x58A12D4900A2FA
0x2DB81C4AE88D20
0x8484A22A795E4
0x5EB45F3E513AC
0x46247570894A4
0x924CE94DF0690
0x1D5530EA52210
0x920A24B9448D0
0x81027F2A79420
0x8B4887801A9FC
0x35E3DA634FB94
0x928D4D1040ED8
0xBE331DD3107AD
0x5E391DD240E89D
0x39874C7FFB294C
0x15E2AB5E23C478
0x312496AE1C4433
0x1356D94E3C824F
0x6FB9313A4228E
0x10CAFC6DD92AE3
0x409BC32BBA4E37
0x13B66FDE55B77C
0x19C7C11DA4E4A2
0x7FA2B9A827FAE
0x42193CC5270058
0x2043847B9EEDD8
0x2EE265CBAEA1AE
0x1D1555557808F2
0x1820C6CA831F20
0x19E699125F8740
0x3D81E379F6B82A
0x2062F49DA45AB4
0x1796E76685E997
0x5DC0C3E3261CDF
0x3A76362613DD95
0x201BB9EA8ED81C
0x33509B969E3741
0x19EB1C9C94E6A8
0x1368E07F2E794F
0x17BFD098C23630
0x448303CFD4DD31
0x1E41E65B7AFC35
0x9BDC1F78073127
0x75583891352145
0x4F18252739A5AC
0xC857C28BE5198
0x32C55970EA1358
0xC421F0A303C70
0x984973478DC5AC
0x56017C15FBDD1
0x5906717CF6C571
0x1A062A307BCABD
0xCCE53B2DF56140
0x926EC5880F9992
0x81FA523E38B793
0x481AF6CCB653B
0x39FC6F33CB770B
0xDB8605E2F4AA0E
0xC34699DB257145
0xD6872C5CC11542
0x6AD52B4C617CF3
0x12C1CA7EAE79A6
0x1DC6931C286DB2
0xED19AD19480BE
0x3A82193F6EF346
0x23394341031AC0
0x2F828795A6E6BE
0x208D625A5FD472
0x1C635AA6DF8398
0x1DE0F95CF827AE
0x3D2149D449636
0x28782F9453EFDE
0x139D946E9D6D82
0xB0E0C55A4D6238
0x97D9725BE8876E
0x26B598F9022C30
0x51C2FA15841D7E
0x18D5FBE4CDA4C8
0xA3F26DF601552
0x13F8DFF2FE8408
0x8A5498F0183BB0
0x3496095EF45282
0x72A31CFDE71EF6
0x46A5661935D9CA
0xBD4C34161B102
0x82A99A5DE4C84F
0xAE5A988393338F
0x825E0AE4D3D5F5
0x6E8EA108204593
0x7A7FE273C35A4C
0x172C91B106B2ED
0x82F69B85B1A2A1
0x40A5DB3578D586
0x22F572E6839826
0x1133B875BED53A
0x4A9B4A38CF1460
0x65C33E4F5A2FC0
0x481234127842A2
0x3BC3C87F8C0454
0x4588CE32DCF25A
0x573D341F00D72
0x48724D79B38078
0xC427156A9E2860
0x88B763CB6FB9F0
0x2F08D6E954E096
0xD9CCF65D9CD7A4
0x17C3E165050BF0
0xCF2B002B1AD140
0xBEA37CEF15E1FA
0xC48CEE7BAE850C
0x489420271C9606
0xDA31DC2C7FDBCE
0x537869C92A42D0
0x1DA1D095B7C0B4
0xBF7B1F10223116
0x6586F47FC6CF52
0x8E40676E4927E0
0x58692F2E100A24
0x4A9CA491835636
0x53CFDDBDD2F61C
0xB2B5098A832538
0x619C35508AEABA
0x8CC856E432BC50
0x62360169143A78
0x55333012D9DD23
0x9C4964E3F15005
0x18A07DDC589B2C
0x9BFE0FEDBF05DC
0x88D54339CF6CF1
0x9463624AACA2C6
0x42E7AF41E9BD7C
0xAB36B7C0777858
0x20CCD008AD41A
0x3684BCD9A0F789
0x2525F943737B70
0x86BB1A4A4AA7D
0x19CA34ADEE1B3A
0x6CC51DF5A1F44
0x4661D5224E5470
0x52CED44ED58CC
0x29A7CADA9C4CB1
0xD0CB5244CA7FD
```

## Solving with Z3
A final Z3 script models the constraint system: select one product per position such that their sum equals the target constant `0x0BC42D5779FEC401`

```python
# z3_solve_input_digits.py
#!/usr/bin/env python3
# pip install z3-solver

from z3 import Solver, BitVec, BitVecVal, Or
from pathlib import Path
import sys

MASK64 = (1 << 64) - 1
TARGET = 0x0BC42D5779FEC401

def accumulate(prev_result: int, derived_num: int) -> int:
    """64-bit wraparound addition (equivalent to the asm block)."""
    return (prev_result + (derived_num & MASK64)) & MASK64

def read_parsed_arrays(path: str):
    """
    Reads 250 lines of 0x-prefixed hex values and chunks into 25 buckets of 10 numbers each.
    Returns: list[list[int]] shape [25][10]
    """
    p = Path(path)
    if not p.exists():
        raise FileNotFoundError(f"Input file not found: {path}")

    with p.open("r", encoding="utf-8") as f:
        # keep non-empty lines, preserve order
        lines = [ln.strip() for ln in f if ln.strip()]

    if len(lines) != 250:
        print(f"[!] Warning: expected 250 lines, got {len(lines)}. Proceeding anyway.", file=sys.stderr)

    # Parse as hex; accept either '0x...' or plain hex
    nums = []
    for i, ln in enumerate(lines):
        try:
            n = int(ln, 0)  # handles 0x-prefixed or decimal/hex as needed
        except ValueError:
            # fallback: strict hex
            n = int(ln, 16)
        nums.append(n & MASK64)

    if len(nums) < 250:
        raise ValueError(f"Need at least 250 numbers; got {len(nums)}")

    parsed = [nums[i * 10:(i + 1) * 10] for i in range(25)]
    return parsed

def solve_buckets(parsed):
    """
    parsed: list of 25 buckets, each a list of 10 ints (< 2^64)
    Returns (chosen_values, chosen_indices_per_bucket_list)
      - chosen_values: list of 25 ints (in order 0..24)
      - chosen_indices_per_bucket_list: list of lists of indices (handles duplicates)
    """
    s = Solver()

    # 25 symbolic 64-bit variables, each must equal one of its bucket’s 10 constants.
    xs = []
    for i in range(25):
        xi = BitVec(f"x_{i}", 64)
        xs.append(xi)
        allowed_vals = [BitVecVal(v & MASK64, 64) for v in parsed[i]]
        s.add(Or([xi == v for v in allowed_vals]))

    # Build 64-bit modular sum: ((((x0 + x1) + x2) + ...) + x24) == TARGET
    acc = xs[0]
    for xi in xs[1:]:
        acc = acc + xi
    s.add(acc == BitVecVal(TARGET & MASK64, 64))

    # Uncomment to cap runtime if needed (ms):
    # s.set(timeout=60_000)

    chk = s.check()
    if str(chk) != "sat":
        raise RuntimeError(f"No solution found: {chk}")

    m = s.model()

    # Extract chosen values in order
    chosen_vals = []
    for i in range(25):
        v = m.evaluate(xs[i], model_completion=True).as_long() & MASK64
        chosen_vals.append(v)

    # Print chosen values first (hex)
    print("Chosen 25 values (hex, in order):")
    for i, v in enumerate(chosen_vals):
        print(f"[{i:02d}] 0x{v:016X}")

    # Verify with our accumulate() from zero (just to be safe)
    acc_py = 0
    for v in chosen_vals:
        acc_py = accumulate(acc_py, v)
    print(f"\nVerification: accumulated = 0x{acc_py:016X}")
    print(f"Target      : 0x{TARGET:016X}")
    if acc_py != (TARGET & MASK64):
        print("[!] Verification mismatch (this should not happen).", file=sys.stderr)

    # For each bucket, find the index/indices where its value equals the chosen one.
    # (If a bucket contains duplicates of the chosen value, we list all matching indices.)
    print("\nIndex/indices of each chosen value within its bucket:")
    idx_lists = []
    for i in range(25):
        bucket = parsed[i]
        chosen = chosen_vals[i]
        matches = [j for j, w in enumerate(bucket) if (w & MASK64) == chosen]
        idx_lists.append(matches)
        # If unique, it will print one index; duplicates => prints multiple
        if len(matches) == 1:
            print(f"bucket {i:02d}: index {matches[0]}")
        else:
            print(f"bucket {i:02d}: indices {matches}")

    return chosen_vals, idx_lists

def main():
    # path = sys.argv[1] if len(sys.argv) > 1 else "dump_possible_derived_each_digit.txt"
    path = sys.argv[1] if len(sys.argv) > 1 else "flareauthenticator_x64dbg.log"
    parsed = read_parsed_arrays(path)
    solve_buckets(parsed)

if __name__ == "__main__":
    main()
```

This script is a **Z3 constraint solver** designed to find one valid 64-bit number from each of 25 predefined “buckets” (each bucket containing 10 possible 64-bit constants) so that their **modular 64-bit sum equals a fixed target value** (`0x0BC42D5779FEC401`).

In short:
- It reads a text file of 250 hex values and divides them into 25 groups of 10 numbers.
- For each group, it declares a 64-bit symbolic variable that must match one of that group’s numbers.
- It then constrains the total 64-bit sum of all 25 chosen numbers to equal the target constant.
- Using Z3, it solves these constraints to find which specific number from each group satisfies the condition.
- Finally, it prints the selected values, verifies their accumulated sum, and lists each chosen value’s index within its respective bucket.

Essentially, it’s automating the brute-force search for a valid combination of 25 constants using symbolic reasoning rather than enumeration.

Execution produces the correct password.

```bash
$ python z3_solve_input_digits.py
Chosen 25 values (hex, in order):
[00] 0x0026D8E6DC23319D
[01] 0x00A8FA7B20F1E228
[02] 0x0082DEEFE21A73C0
[03] 0x00B2F907A133B2C0
[04] 0x00ACB3FF351198AB
[05] 0x006E4F38A8434132
[06] 0x002BE31F155AB714
[07] 0x00AC26E207A0A6C5
[08] 0x0016F0557C6F1B97
[09] 0x00527EEC073C111B
[10] 0x0058A12D4900A2FA
[11] 0x000928D4D1040ED8
[12] 0x005E391DD240E89D
[13] 0x0042193CC5270058
[14] 0x005DC0C3E3261CDF
[15] 0x009BDC1F78073127
[16] 0x00DB8605E2F4AA0E
[17] 0x003A82193F6EF346
[18] 0x00B0E0C55A4D6238
[19] 0x00AE5A988393338F
[20] 0x0065C33E4F5A2FC0
[21] 0x00DA31DC2C7FDBCE
[22] 0x00BF7B1F10223116
[23] 0x00AB36B7C0777858
[24] 0x004661D5224E5470

Verification: accumulated = 0x0BC42D5779FEC401
Target      : 0x0BC42D5779FEC401

Index/indices of each chosen value within its bucket:
bucket 00: index 4
bucket 01: index 4
bucket 02: index 9
bucket 03: index 8
bucket 04: index 2
bucket 05: index 9
bucket 06: index 1
bucket 07: index 3
bucket 08: index 1
bucket 09: index 4
bucket 10: index 8
bucket 11: index 9
bucket 12: index 1
bucket 13: index 2
bucket 14: index 1
bucket 15: index 0
bucket 16: index 5
bucket 17: index 2
bucket 18: index 1
bucket 19: index 4
bucket 20: index 4
bucket 21: index 9
bucket 22: index 2
bucket 23: index 9
bucket 24: index 6
```

Entering this 25-digit sequence triggers the success dialog: `s0m3t1mes_1t_do3s_not_m4ke_any_s3n5e@flare-on.com`.

![Correct password found by z3](https://lh3.googleusercontent.com/pw/AP1GczPPN9dMJl-XszD2tDaY21BorbNdq41h5wPZJBq2AoWeQjx7y6lRjKJnZahK16QDX4H7gc6v4K84krCV_iwdRsAqiN7dLFe4zKx_z9niPyb4wEbXValFJI7sEWJ3f8xFJjHVJL8rF9tUS6lkZwTHDRJI=w964-h736-s-no-gm)
_**Figure: Correct password found by z3**_

# Final thought
Obfuscation initially complicates analysis, yet XREF recovery and flow reconstruction enable a bottom-up strategy without exhaustive deobfuscation. Black-box handling of core functions, paired with Z3 constraints and tools like WinDbg TTD plus ret-sync, accelerates resolution.

### Further Reading
- [Time Travel Debugging Overview](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview) – Official documentation on reversible debugging with WinDbg TTD.
- [Ret-Sync GitHub Repository](https://github.com/bootleg/ret-sync) – Synchronization plugin bridging debuggers and IDA for streamlined workflows.
- [Z3 Guided Tour](https://microsoft.github.io/z3guide/docs/logic/intro/) - State-of-the art theorem prover from Microsoft Research
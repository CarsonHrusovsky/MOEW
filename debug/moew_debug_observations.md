# MOEW — Debugger Observations, Emergent Decode Behavior, and SEH Waterfall Analysis

**Author:** Harrison Edwards  
**Project:** MOEW (Misaligned Opcode Exception Waterfall)  
**Document:** Debugging & Behavioral Notes (based on PoC execution)

---

## 1. Overview

This document summarizes real debugging observations collected while executing the MOEW PoC (`seh_waterfall.exe`) under x32dbg.  
The analysis demonstrates:

- How misaligned opcode entry behaves in practice  
- Why the emergent instruction stream is subtle and difficult to see  
- How the SEH Waterfall appears in a real debugger  
- How call stacks evolve across all 3 stages  
- How the final handler exits cleanly with no crash telemetry  
- Why this matches the real-world behavior that inspired the MOEW whitepaper  

Although the PoC uses benign payloads (notepad, calc, marker file) for validation, the underlying SEH manipulation and misaligned decode pathways are identical to the structure observed in real malware samples.

---

## 2. SEH Waterfall Behavior (Observed)

Across all debugging captures, the execution flow follows the exact pattern defined in the PoC:

### **Stage 0**
- Save original `fs:[0]` (SEH head)  
- Install `SEH[Stage1]`  
- Misaligned jump into `blob1+5`  
- Trigger hardware fault: `#DE (EXCEPTION_INT_DIVIDE_BY_ZERO)`  

### **Stage 1 Handler**
Reached via:

```
KiUserExceptionDispatcher → RtlDispatchException → stage1_handler
```

Handler actions:
- Launch `notepad.exe`  
- Install `SEH[Stage2]`  
- Misaligned jump into `blob2+3`  
- Second `#DE` fault  

### **Stage 2 Handler**
- Reached via SEH dispatch  
- Writes marker file into `%TEMP%`  
- Installs `SEH[Final]`  
- Misaligned jump into `blob3+3`  
- Third `#DE` fault  

### **Final Handler**
- Reached after third dispatched exception  
- Launches `calc.exe`  
- Restores original `fs:[0]`  
- Exits process cleanly (no crash, no WER entry)  

All debugging captures confirm this behavior exactly.

---

## 3. Call Stack Analysis (Per Stage)

The debugger consistently reveals:

### **Stage 0 → Stage 1 transition**
```
ntdll.KiUserExceptionDispatcher
ntdll.RtlDispatchException
seh_waterfall.stage1_handler
```

### **Stage 1 → Stage 2 transition**
```
ntdll.KiUserExceptionDispatcher
ntdll.RtlDispatchException
seh_waterfall.stage2_handler
```

### **Stage 2 → Final handler**
```
ntdll.KiUserExceptionDispatcher
ntdll.RtlDispatchException
seh_waterfall.final_handler
```

### **Finalization**
The process terminates cleanly:
- No unhandled exception  
- No abnormal termination  
- No WER crash bucket  
- Thread disappears with status `0x0`  

This matches the expected behavior of the **clean MOEW** variant.

---

## 4. Why Misaligned Decode Is Hard to See (Critical Insight)

A key insight during debugging:

> **Misaligned entry *is happening*, but the executed instructions look completely normal.**

### Why this occurs:

- The PoC’s blobs are intentionally tiny.  
- The misaligned entry points (`+5`, `+3`, `+3`) land on the first byte of valid x86 opcodes.  
- The debugger disassembles cleanly from those bytes.  
- Thus the emergent stream is **semantically divergent** but **visually aligned**.

### Example

**Linear decode of blob1:**
```
B8 10 00 00 00    mov eax, 0x10
F7 F1             div ecx
C3                ret
```

**Misaligned entry (blob1 + 5):**
```
F7 F1             div ecx    ; same bytes, but MOV is skipped
C3                ret
```

This means:

- The disassembler shows normal code  
- The CPU actually executed an emergent instruction stream  
- A reverse engineer sees nothing suspicious  

This is the foundation of MOEW’s stealth:

> **The emergent stream is logically malicious but visually ordinary.**

---

## 5. “Less Emergent” Streams: A Feature, Not a Bug

Real-world misaligned decode (as seen in the original reversed sample) produced chaotic emergent behaviors:

- Overlapping instructions  
- ModR/M prefix misuse  
- Multi-byte opcode splits  
- Junk bytes becoming valid instructions  

The PoC intentionally uses:

- Clean, compact byte sequences  
- Predictable offsets  
- Valid instructions at misaligned boundaries  

This results in:

- Clean `div` instructions  
- No garbage decode  
- No suspicious padding  
- SEH masking the real control flow  

This is actually *more dangerous*, not less — the lack of visual noise makes the emergent decode nearly undetectable.

---

## 6. Debugger View Anomalies (Important)

Several captures show:

- Stage 2 or Final handler appearing “inside ntdll.dll”
- x32dbg attributing execution to ntdll instead of the Rust blob

This occurs due to:

- Rust’s frame-pointer omission  
- Compiler optimizations  
- SEH frame reconstruction in x32dbg  
- Stack unwinding metadata  

Resulting consequence:

> **Handlers may appear to execute from inside system DLLs, further hiding MOEW’s behavior.**

This completely obscures:

- true EIP  
- real handler location  
- the misaligned entry  
- the manipulated SEH chain  

---

## 7. Why No Crash Telemetry Exists

The Final handler exits via:

```rust
process::exit(0);
```

This eliminates:

- Crash dumps  
- Event Viewer “Application Error” entries  
- WER fault buckets  
- Faulting module attribution  
- Stack traces in logs  

Even though **three hardware faults occurred**, all were:

- caught  
- processed  
- unwound  
- restored cleanly  

Thus leaving **zero forensic footprint**.

---

## 8. Summary of Real Debugging Findings

- MOEW Waterfall executes exactly as designed.  
- Misaligned decode is happening but appears normal visually.  
- SEH dispatch masks the true control flow origin.  
- Debugger shows “clean” call stacks into ntdll.  
- Final exit produces *no* crash logs or telemetry.  
- Handlers may appear inside ntdll due to stack unwinding.  
- Emergent decode is semantically wrong but syntactically valid.  

These findings strongly reinforce the stealth properties described in the MOEW whitepaper and validate the PoC as a faithful implementation of the clean MOEW technique.



# Invisible Threats, Visible Solutions.

# Misaligned Opcode Exception Waterfall (MOEW)  
### 3‑Stage SEH Waterfall PoC (Benign, x86/Wow64)

> **TL;DR:** This PoC demonstrates a 3‑stage Structured Exception Handling (SEH) cascade caused by deliberate **misaligned execution** into raw byte blobs whose first valid instruction is a faulting `div reg` with a zero divisor.  
> Each exception stage installs the next SEH handler, triggers a new misaligned divide‑by‑zero, and executes a benign observable payload before restoring the original SEH chain and terminating cleanly.

---

## 1. Overview

MOEW (Misaligned Opcode Exception Waterfall) is a **defensive research** sample that showcases controlled exception‑driven multi‑stage execution on **x86/Wow64 Windows**. It demonstrates:

- Manual SEH chain manipulation via `fs:[0]`  
- Deliberate misalignment into hand‑crafted byte sequences (`blob1`, `blob2`, `blob3`)  
- Stage‑chained SEH handlers  
- Multi‑fault recursion through:
  - `KiUserExceptionDispatcher`
  - `RtlDispatchException`
  - Custom user‑mode handlers  
- A clean termination path that restores the original SEH chain

**All payloads are benign**:

- **Stage 1:** Launches Notepad  
- **Stage 2:** Writes a marker file to `%TEMP%`  
- **Final Stage:** Launches Calculator  

The PoC is intentionally defanged. No data is encrypted, modified, or destroyed.

---

## 2. Features

- Fully deterministic SEH recursion through 3 staged handlers  
- Naked x86 byte blobs with multiple valid decode paths  
- Misaligned `div` at controlled offsets (`ECX`/`EDX`/`EBX` = 0)  
- Per‑stage global counter for logging and tracing  
- Complete restoration of the original SEH head (`ORIGINAL_SEH`)  
- Rust nightly + inline assembly + naked functions  
- Benign but visible “payloads” for telemetry and debugger testing  

---

## 3. Environment Requirements

### 3.1 Architecture

- **x86 (32‑bit) only**  
- Compiled with the MSVC toolchain  
- Runs on 64‑bit Windows under WoW64 as well  

### 3.2 Rust Nightly

The PoC uses nightly‑only features:

```rust
#![feature(asm_experimental_arch)]
#![feature(naked_functions)]
```

Install the necessary components:

```bash
rustup toolchain install nightly
rustup target add i686-pc-windows-msvc --toolchain nightly
```

### 3.3 Disable SAFESEH

Since the PoC installs custom SEH handlers that are not present in the SAFESEH table, the linker must be instructed to disable SAFESEH validation.

Create `.cargo/config.toml`:

```toml
[target.i686-pc-windows-msvc]
rustflags = [
  "-C", "link-arg=/SAFESEH:NO",
]
```

---

## 4. Build Instructions

Build the binary:

```bash
cargo +nightly build --target i686-pc-windows-msvc --release
```

Output is located at:

```text
target\i686-pc-windows-msvc\release\seh_waterfall.exe
```

---

## 5. Running the PoC

Execute:

```cmd
seh_waterfall.exe
```

Expected control‑flow:

```text
Stage 0 → misaligned blob1 → Stage 1 handler
Stage 1 → misaligned blob2 → Stage 2 handler
Stage 2 → misaligned blob3 → Final handler
Final  → restore SEH       → exit
```

Visible artifacts:

- **notepad.exe** launches (Stage 1)  
- `%TEMP%\moew_stage2.txt` is created (Stage 2)  
- **calc.exe** launches (Final handler)  

---

## 6. Technical Breakdown

### 6.1 SEH Record Layout

On 32‑bit Windows, SEH records form a linked list stored at `fs:[0]`:

```rust
#[repr(C)]
struct SehRec {
    next: *mut SehRec,
    handler: usize,
}
```

Each handler uses the standard SEH signature:

```rust
extern "system" fn handler(
    record: *mut u8,
    frame: *mut u8,
    context: *mut u8,
    dispatcher: *mut u8,
) -> i32
```

### 6.2 Global State

```rust
static STAGE_COUNTER: AtomicU32 = AtomicU32::new(0);
static mut ORIGINAL_SEH: *mut SehRec = std::ptr::null_mut();
```

Used to:

- Count handler recursion depth  
- Restore the original SEH head at the end of the waterfall  

---

## 7. Stage Logic

### **Stage 0 — Initial Frame Setup**

1. Save `fs:[0]` as `ORIGINAL_SEH`.  
2. Build an SEH record pointing to the Stage 1 handler.  
3. Overwrite `fs:[0]` with this new record.  
4. Misalign into `blob1 + 5`, which decodes as `div ecx` (after setting `ECX = 0`).  

### **Stage 1 — First SEH Handler**

1. Launch `notepad.exe`.  
2. Build an SEH record for the Stage 2 handler and chain it on top.  
3. Misalign into `blob2 + 3` → `div edx` (with `EDX = 0`).  

### **Stage 2 — Second SEH Handler**

1. Write `%TEMP%\moew_stage2.txt`.  
2. Install the Final handler on the SEH chain.  
3. Misalign into `blob3 + 3` → `div ebx` (with `EBX = 0`).  

### **Final Handler — Termination**

1. Launch `calc.exe`.  
2. Restore `ORIGINAL_SEH` into `fs:[0]`.  
3. Exit the process cleanly via `process::exit(0)`.  

---

## 8. Misaligned Fault Blobs

### 8.1 `blob1`

```rust
#[unsafe(naked)]
pub extern "C" fn blob1() {
    naked_asm! {
        ".byte 0xB8, 0x10, 0x00, 0x00, 0x00", // mov eax, 0x10
        ".byte 0xF7, 0xF1",                   // div ecx
        ".byte 0xC3",                         // ret
        ".byte 0x90, 0x90, 0x90",             // nop padding
    }
}
```

- Aligned decode: `mov eax, 0x10; div ecx; ret`  
- Misaligned at `+5`: `div ecx` (with `ECX = 0` → `#DE`)  

### 8.2 `blob2`

```rust
#[unsafe(naked)]
pub extern "C" fn blob2() {
    naked_asm! {
        ".byte 0x55",                         // push ebp
        ".byte 0x8B, 0xEC",                   // mov ebp, esp
        ".byte 0xF7, 0xF2",                   // div edx
        ".byte 0xC3",                         // ret
        ".byte 0x90, 0x90, 0x90",             // nop padding
    }
}
```

- Aligned decode: `push ebp; mov ebp, esp; div edx; ret`  
- Misaligned at `+3`: `div edx` (with `EDX = 0` → `#DE`)  

### 8.3 `blob3`

```rust
#[unsafe(naked)]
pub extern "C" fn blob3() {
    naked_asm! {
        ".byte 0x53",                         // push ebx
        ".byte 0x8B, 0xD8",                   // mov ebx, eax
        ".byte 0xF7, 0xF3",                   // div ebx
        ".byte 0xC3",                         // ret
        ".byte 0x90, 0x90, 0x90",             // nop padding
    }
}
```

- Aligned decode: `push ebx; mov ebx, eax; div ebx; ret`  
- Misaligned at `+3`: `div ebx` (with `EBX = 0` → `#DE`)  

---

## 9. Exception Pipeline

Each faulted stage re‑enters the Windows user‑mode exception pipeline:

```text
KiUserExceptionDispatcher
    → RtlDispatchException
        → SEH chain walk (fs:[0])
            → MOEW handler
```

Typical debugger view:

```text
seh_waterfall!blobX+offset
ntdll!KiUserExceptionDispatcher
ntdll!RtlDispatchException
seh_waterfall!stageN_handler
```

The waterfall is entirely driven by genuine hardware faults and SEH dispatch; no synthetic or fake exceptions are used.

---

## 10. Marker File Output

The file written in Stage 2 looks like:

```text
MOEW Stage 2 Marker
-------------------
This file was written by the Stage 2 SEH handler
as a benign demonstration payload.
```

Its presence in `%TEMP%` serves as a simple, observable proof that Stage 2 executed via the SEH chain.

---

## 11. Safety Notes

- PoC is benign and for defensive research only.  
- No persistence, registry modification, or encryption.  
- All exceptions are caught and handled.  
- The original SEH chain is restored before termination.  

---

## 12. Future Enhancements

Potential extensions include:

- YARA and behavioral rules for SEH waterfalls and recursive exception patterns.  
- ETW / EDR signal mapping for staged hardware faults.  
- Graphical diagrams of SEH chain evolution over time.  
- Side‑by‑side comparison with real‑world malware exception chains.  

---

## 13. Full Source Code

The complete PoC implementation is available in this repository (see `src/main.rs`).

---

## 14. License

This project is intended for defensive research and education.
Copyright <2025> <Harryeetsource>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.




---

## 15. Real Sample vs. PoC: SEH Corruption, Waterfall Behavior, and Telemetry Impact

### 15.1 Behavior of the Real Sample: Intentional SEH Corruption

The real‑world MOEW sample that inspired this PoC did **not** restore `fs:[0]` at the end of execution.  
Instead, its final stage:

1. **Overwrote the SEH head (`fs:[0]`)** with either `NULL` or a pointer to invalid memory.  
2. **Triggered one last misaligned exception**, guaranteeing a hardware fault.  
3. Forced Windows to walk an **invalid or truncated SEH chain**.  
4. Caused `RtlDispatchException` to encounter an invalid handler pointer.  
5. Resulted in a crash where:
   - EIP/RIP pointed into **non‑image memory** (heap, stack, or anonymous region).  
   - The **faulting module** could not be resolved and appeared as `unknown`.  
   - The **faulting path** also appeared as `unknown`.  
6. Produced Windows Error Reporting (WER) signatures that were **not attributable** to any loaded module.  
7. Logged Event Viewer `Application Error` events with meaningless fault offsets.  
8. Produced EDR telemetry dominated by:
   - repeated `KiUserExceptionDispatcher` calls,  
   - recursive SEH transitions,  
   - a final anomalous crash lacking module attribution.  

This destructive SEH corruption step serves the sample’s primary anti‑forensic purpose:  
**erasing the causal chain and producing an un‑attributable terminal crash.**

---

### 15.2 Behavior of the PoC: Clean Restoration of SEH Without Terminal Crash

Unlike the real sample, this PoC:

- **Captures** the original SEH head during Stage 0:

  ```rust
  asm!("mov {old}, fs:[0]", old = out(reg) old_head);
  ORIGINAL_SEH = old_head;
  ```

- **Restores** the original SEH head in the final handler:

  ```rust
  asm!("mov fs:[0], {p}", p = in(reg) ORIGINAL_SEH);
  ```

- **Terminates cleanly** via `process::exit(0)` instead of triggering an unhandled fault.

As a result, the PoC:

- Does **not** leave a corrupted SEH chain.  
- Does **not** produce a final unhandled exception.  
- Does **not** generate:
  - WER crash reports,  
  - Event Viewer `Application Error 1000` entries,  
  - “faulting module: unknown” signatures,  
  - invalid exception dispositions,  
  - or dangling SEH records.  

However—critically—the PoC does *not* eliminate MOEW’s **exception‑driven telemetry degradation** behavior.

---

### 15.3 Telemetry Degradation Shared by Both Real Sample and PoC

The PoC absolutely **does** degrade telemetry in the same fundamental way:

- It deliberately triggers **multiple misaligned hardware faults**.  
- It produces **multiple first‑chance exceptions** in rapid succession.  
- It forces Windows to repeatedly execute:

  ```text
  KiUserExceptionDispatcher
  RtlDispatchException
      → custom handler
      → misaligned blob
      → hardware fault
  ```

- It generates **non‑linear, exception‑dominant call stacks**.  
- It distorts control‑flow reconstruction in debuggers and EDR by routing execution through:
  - recursive SEH handlers,  
  - misaligned decode paths,  
  - non‑function‑aligned addresses,  
  - partial instruction boundaries.  

Thus the PoC faithfully reproduces:

- the **recursive waterfall pattern**,  
- the **exception‑driven state machine**,  
- the **misaligned opcode entry behavior**,  
- and the **control‑flow obfuscation**,  

…while avoiding the destructive final crash.

This makes the PoC ideal for instrumentation and research without invoking the full anti‑forensic payload.

---

### 15.4 Why the PoC Is Suitable for Defensive Research

Because the PoC preserves the overall behavior of MOEW *minus* the SEH corruption crash, it is:

- Safe to run repeatedly in lab environments.  
- Deterministic and stable.  
- Suitable for:
  - EDR pipeline analysis,  
  - telemetry research,  
  - incident‑response training,  
  - debugger behavior testing,  
  - side‑by‑side comparison with real malicious samples.  

The PoC models the **exception waterfall** (the essence of MOEW) while removing the final destructive signature.  
It demonstrates that telemetry degradation arises **not only** from SEH corruption, but from the exception‑driven control‑flow model itself.

---

### 15.5 Summary

| Behavior                                   | Real MOEW Sample | PoC Implementation |
|-------------------------------------------|------------------|--------------------|
| Misaligned opcode waterfall               | ✔                | ✔                  |
| Recursive SEH‑driven state machine        | ✔                | ✔                  |
| Dominant exception‑dispatch call stacks   | ✔                | ✔                  |
| Telemetry / stack‑trace degradation       | ✔                | ✔                  |
| Intentional SEH corruption                | ✔                | ❌                  |
| Dangling or invalid SEH pointers          | ✔                | ❌                  |
| Final unhandled exception                 | ✔                | ❌                  |
| WER “unknown module” crash                | ✔                | ❌                  |
| Clean restoration of `fs:[0]`             | ❌                | ✔                  |
| Clean termination                         | ❌                | ✔                  |

This section formalizes the behavioral differences while keeping the PoC’s purpose clear:  
**demonstrate the MOEW waterfall in a safe, research‑friendly form without the anti‑forensic final crash.**

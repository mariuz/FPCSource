# Free Pascal (FPC) LLVM Backend Roadmap & Open TODOs

This document provides a comprehensive review of the current status, open tasks, architectural limitations, and code-level `TODO` / `FIXME` items for the **Free Pascal LLVM Code Generator (`compiler/llvm/`)** in `FPCSource`.

---

## 1. Current Status Overview

The LLVM backend (`-dllvm` / `LLVM=1`) is integrated as a high-level code generator (`thlcgllvm`) on top of FPC's node tree and target parameter managers, emitting textual LLVM IR (`.ll`) which is then assembled/optimized by `clang` (`TLLVMClangAssember`).

### Currently Supported Targets
- **Architectures enabled in `compiler/Makefile.fpc`:** `x86_64`, `aarch64`, `arm` (ARMHF)
  - Note: Partial `cpunode.pas` wiring (`{$ifndef llvm} ... {$else} llvmnode {$endif}`) already exists for `powerpc64`, `riscv64`, `loongarch64`, and `xtensa`.
- **Operating Systems (`agllvm.pas`):**
  - `Darwin/x86_64`, `Darwin/AArch64` (`as_clang_llvm_darwin`, `systems_darwin`; iOS/iphonesim untested)
  - `Linux/x86_64`, `Linux/AArch64`, `Linux/ARM` (`as_clang_llvm`)
  - `FreeBSD/x86_64`, `OpenBSD/x86_64` (`as_clang_llvm`)
- **LLVM Versions (`compiler/llvm/llvminfo.pas`):**
  - Currently defines `llvmver_7_0` through `llvmver_17_0` (plus `Xcode-11.0` through `Xcode-15`).
  - Supports opaque pointers (`llvmflag_opaque_ptr`, default from LLVM 15+), AddressSanitizer (`-Clfsanitize=address`), Link-Time Optimization (`-Clflto`, `-Clfltonosystem`), and DWARF 2–5 debug metadata.

---

## 2. High-Priority Architectural & Roadmap Tasks

### 2.1. Update Supported LLVM / Clang Versions (LLVM 18 – 22+)
- **Location:** `compiler/llvm/llvminfo.pas` (`tllvmversion`, `llvmversionstr`, `llvmversion_properties`)
- **What needs to be implemented:**
  - Add enum entries and property flags for **LLVM 18.0, 19.0, 20.0, 21.0, 22.0** and **Xcode 16+**.
  - Handle newer LLVM IR changes (e.g., new `DIExpression` / debug info transitions removing legacy `llvm.dbg.declare` intrinsics in favor of non-instruction debug records `#dbg_declare` in newer LLVM releases, updated `memory(...)` attributes, and stricter `getelementptr` / `nuw` / `nsw` rules).
  - Add auto-detection of the installed `clang` version when `-Clv` is not explicitly passed.

### 2.2. Platform & Architecture Expansion
- **Locations:** `compiler/Makefile.fpc`, `compiler/llvm/agllvm.pas`, `compiler/llvm/llvmpara.pas`, `compiler/systems/i_*.pas`
- **What needs to be implemented:**
  1. **Complete wired 64-bit / 32-bit Unix targets:**
     - `powerpc64-linux`, `riscv64-linux`, `loongarch64-linux`, and `i386-linux`: `cpunode.pas` already includes `llvmnode` for several of these, and `i_linux.pas` already defines `llvmdatalayout` for `i386`, `powerpc64`, and `riscv64` (note: `riscv32` and `loongarch64` still have `llvmdatalayout: 'todo'` in `i_linux.pas`).
     - Enable `tf_use_psabieh` in `i_linux.pas` / `i_bsd.pas` for these targets under `{$ifdef llvm}`, add them to `as_clang_llvm_info.supported_targets` in `agllvm.pas`, and allow them in `compiler/Makefile.fpc`.
  2. **Generalize Parameter Lowering (`llvmpara.pas`):**
     - `tllvmparamanager` relies on the target `tcpuparamanager` to decompose aggregate parameters/results into ABI-compliant `tcgparalocation` chunks. Targets with complex ABI rules (homogeneous floating-point aggregates, big-endian shifting with `byval`, or split register/stack passing) need verification and target-specific adjustments.
  3. **Windows (`win64` / `win32` / `aarch64-win64`) Support:**
     - Requires implementing **Windows SEH (Structured Exception Handling)** in LLVM IR (`catchswitch`, `catchpad`, `catchret`, `cleanuppad`, `cleanupret` with `__C_specific_handler` or GNU/MSVC personality routines) inside `compiler/llvm/llvmpi.pas` and `nllvmflw.pas` instead of Itanium psABI `landingpad` + `resume`.
     - Requires handling COFF/PE symbol directives, DLL import/export (`dllimport` / `dllexport`), and `ait_seh_directive` (currently triggers `internalerror(2013010713)` in `agllvm.pas:1598`).
  4. **WebAssembly (`wasm32-wasi` / `wasm32-embedded`):**
     - Fill in `llvmdatalayout` in `compiler/systems/i_wasi.pas` (currently `'todo'`) and wire up Wasm exception handling / target triplets.

### 2.3. Exception Handling & Hardware/Signal Exceptions (Try-Block Outlining)
- **Locations:** `compiler/llvm/llvmpi.pas`, `compiler/llvm/nllvmflw.pas`, `rtl/inc/psabiehh.inc`
- **Current Limitation:**
  - LLVM assumes exceptions can only be thrown by `invoke` instructions (`tllvmexceptionstatehandler.new_exception` in `llvmpi.pas:131-137`).
  - If a `try..except` or `try..finally` block contains no calls (or catches hardware exceptions like `SIGSEGV` / `EAccessViolation` or integer division by zero), LLVM may optimize away the landingpad or keep modified variables in SSA registers across a fault.
  - Currently worked around by emitting dummy calls to `FPC_DUMMYPOTENTIALRAISE` at the start and end of every `try` block (`llvmpi.pas:137, 184`). Even with this, local variables modified before a hardware fault are not guaranteed to be spilled to memory unless the body is in a `noinline` routine.
- **What needs to be implemented:**
  - Automatically outline `try` block bodies into `noinline` nested functions (similar to how FPC handles SEH filter/finally outlining or nested routines in `ncgnstfl`), or mark mutated escaping locals within `try` blocks so hardware exceptions safely preserve variable state.

### 2.4. Constrained Floating-Point Intrinsics & Fast-Math / FPU Modes
- **Locations:** `compiler/llvm/nllvmadd.pas`, `compiler/llvm/nllvmcnv.pas`, `compiler/llvm/nllvminl.pas`, `compiler/llvm/hlcgllvm.pas`, `compiler/llvm/llvmfeatures.pas`, `rtl/inc/llvmintr.inc`
- **What needs to be implemented:**
  - Complete support for `llvm.experimental.constrained.*` intrinsics across all floating-point operations and conversions (`fptosi`, `fptoui`, `fptrunc`, `fpext`, `sitofp`, `uitofp`, `frem`, comparisons `fcmp` -> `llvm.experimental.constrained.fcmp` / `fcmps`).
  - Enable `llvmflag_generic_constrained_si64tofp` in `llvminfo.pas` for LLVM versions where non-x86 targets support constrained 64-bit integer-to-float conversion (`llvmfeatures.pas:36-44`).
  - Support `round()` and `trunc()` via constrained intrinsics (`llvm.experimental.constrained.nearbyint` / `rint` / `trunc`) instead of falling back to runtime helpers when fastmath is off.

### 2.5. Debug Information Completeness (`dbgllvm.pas`)
- **Location:** `compiler/llvm/dbgllvm.pas`
- **What needs to be implemented:**
  - **Dynamic Strings (`AnsiString`, `UnicodeString`, `WideString`):** Currently emitted as plain `DW_TAG_pointer_type` to `AnsiChar` / `WideChar` (`dbgllvm.pas:1410-1426`, marked `// Todo: dynamic length "array"`). Implement `DICompositeType` with `dataLocation` / `DISubrange` expressions (similar to dynamic arrays in `dbgllvm.pas:992-1045`) so LLDB/GDB can inspect string lengths and contents natively.
  - **Bitpacked Arrays:** Currently falls back to an array of `u8inttype` bytes when element bit size is not a power-of-two byte multiple (`dbgllvm.pas:976-984`, `{ todo: proper support for bitpacked arrays }`). Add `bitStride` support in `DICompositeType`.
  - **Open Arrays & `array of const`:** Currently hardcoded to a dummy `[0..1]` range (`dbgllvm.pas:946-969`). Connect open array `DISubrange` `upperBound` to the hidden `high` `tparavarsym` metadata node when `llvmflag_array_datalocation` is enabled.
  - **Threadvars (`thread_local`):** Implement DWARF `DIExpression` for TLS / `threadvar` variables (`dbgllvm.pas:2051`, `{ TODO: !!! FIXME: dwarf for thread vars !!!}`).
  - **Procedure Variables (Method Pointers):** Enhance `appenddef_procvar` (`dbgllvm.pas:1430-1446`) to emit the actual subroutine signature and `{proc, self}` struct layout instead of a generic untyped pointer or `array[1..2] of codepointer`.
  - **Nested Procedures & Scoped Local Symbols:** In `appendprocdef` (`dbgllvm.pas:1778-1814`), local types/nested procedure scoping inside `DISubprogram` is commented out and emitted globally.

### 2.6. Thread-Local Storage (TLS) Threadvars
- **Locations:** `compiler/llvm/nllvmutil.pas:72-77`, `compiler/llvm/agllvm.pas:1416-1417`, `compiler/llvm/hlcgllvm.pas`, `compiler/llvm/llvminfo.pas:80` (`llvmflag_no_threadlocal_address`)
- **What needs to be implemented:**
  - `llvminfo.pas` defines `llvmflag_no_threadlocal_address`, and `agllvm.pas` can emit `thread_local`, when `tf_section_threadvars` is enabled.
  - However, in `LLVM >= 16`, taking the address of a `thread_local` global requires calling the `@llvm.threadlocal.address` intrinsic rather than using the global symbol directly as a constant/instruction operand.
  - Enable `tf_section_threadvars` for supported LLVM targets (Linux, Darwin, etc.) and emit `@llvm.threadlocal.address` loads when `not(llvmflag_no_threadlocal_address in llvmversion_properties[...])`.

### 2.7. Optimization Metadata & Code Generation Options Forwarding
- **Locations:** `compiler/llvm/agllvm.pas`, `compiler/llvm/aasmllvmmetadata.pas`, `compiler/llvm/hlcgllvm.pas`
- **What needs to be implemented:**
  - **Metadata for LLVM Optimizer:**
    - Emit `!range` metadata on loads of subrange types, booleans, and enums so LLVM can eliminate redundant bounds checks and dead branches.
    - Emit `!tbaa` (Type-Based Alias Analysis) metadata and `noalias` / `nonnull` / `dereferenceable` / `noundef` parameter attributes.
    - Emit function memory attributes (`memory(none)`, `memory(read)`, `willreturn`, `nounwind`) for pure/const functions.
  - **Forwarding Compiler Switches to Clang (`agllvm.pas:1713-1758`):**
    - Currently `-march=` is restricted to `aarch64` (`agllvm.pas:1744`: `{ restrict march to aarch64 for now to fix x86_64 compilation failure }`). Map FPC `tcputype` names (e.g. `x86_64_v2`, `x86_64_v3`, `core_avx2`, `znver*`) to Clang's `-march=` names on `x86_64` and `arm`.
    - Forward `-Os` (optimize for size, `cs_opt_size`), `-O4`, loop unrolling (`-funroll-loops` / `-fno-unroll-loops`), vectorization switches, and sanitizer options (`thread`, `undefined`, `memory` in addition to `address`).

### 2.8. Direct Bitcode (`.bc`) / In-Process LLVM Emission
- **Location:** `compiler/llvm/agllvm.pas`
- **What needs to be implemented:**
  - FPC currently writes textual `.ll` assembly files and invokes external `clang -x ir`.
  - Textual LLVM IR changes syntax across LLVM releases (requiring `tllvmversionflag` shims), whereas LLVM bitcode (`.bc`) guarantees stronger backward compatibility.
  - Optional future milestone: Add a binary bitcode writer (`TLLVMBitcodeAssembler`) or optional `libLLVM` C-API binding to emit `.bc` or `.o` directly without spawning a `clang` process per unit.

---

## 3. Code-Level `TODO`, `FIXME`, and `InternalError` Inventory in `compiler/llvm/`

| File & Line | Tag / InternalError | Description & Required Implementation |
| :--- | :--- | :--- |
| `compiler/llvm/aasmllvm.pas:762-767` | `internalerror(2013110610)` | Implement register spilling type resolution (`spilling_get_reg_type`) for atomic instructions (`la_fence`, `la_cmpxchg`, `la_atomicrmw`). |
| `compiler/llvm/aasmllvm.pas:779-784` | `{ todo }` / `2013110107` | Implement `spilling_get_reg_type` for vector and aggregate extraction opcodes: `la_extractelement`, `la_insertelement`, `la_shufflevector`, `la_extractvalue`. |
| `compiler/llvm/agllvm.pas:1363-1365` | `{ todo }` / `2014062402` | Handle `ait_symbol` when `tai_symbol(hp).has_value` is true. |
| `compiler/llvm/agllvm.pas:1744-1747` | Comment | Map FPC CPU names to Clang `-march=` strings for `x86_64` and `arm` (currently restricted to `aarch64`). |
| `compiler/llvm/dbgllvm.pas:946-969` | Commented code | Implement dynamic `upperBound` debug info for open arrays and `array of const` using `llvmflag_array_datalocation`. |
| `compiler/llvm/dbgllvm.pas:976-984` | `{ todo }` | Proper DWARF debug info support for bitpacked arrays (`bitStride` instead of byte array fallback). |
| `compiler/llvm/dbgllvm.pas:1410-1426` | `// Todo` | Emit dynamic length array debug info (`dataLocation` + `upperBound` at offset `-sizeof( pint)`) for `st_ansistring`, `st_unicodestring`, and `st_widestring`. |
| `compiler/llvm/dbgllvm.pas:2051` | `{ TODO: !!! FIXME }` | Implement DWARF debug info expressions for `threadvar` (TLS) variables. |
| `compiler/llvm/hlcgllvm.pas:1592-1596` | `{ rest: todo }` | `thlcgllvm.g_proc_entry`: review any remaining prologue tasks beyond `alloclist` concatenation. |
| `compiler/llvm/hlcgllvm.pas:1651-1652` | `{ todo: complex returns }` / `2012111106` | `thlcgllvm.g_proc_exit`: handle non-register/non-void return locations if not lowered to a single aggregate register. |
| `compiler/llvm/hlcgllvm.pas:1753-1784` | `(* TBD *)` | `thlcgllvm.g_set_addr_nonbitpacked_field_ref`: parent struct `getelementptr` traversal is disabled in favor of pointer typecasts due to anonymous function symbol migration; resolve symbol migration so clean GEP chains can be emitted. |
| `compiler/llvm/hlcgllvm.pas:1853-1855` | `{ todo }` / `2013060220` | `thlcgllvm.a_loadmm_reg_ref`: handle `fromsize <> tosize` (insert pointer/value bitcast or aggregate store helper, matching `a_loadmm_ref_reg`). |
| `compiler/llvm/hlcgllvm.pas:1873-1893` | `{ todo }` / `2013060221..23` | Implement `a_opmm_reg_reg` (beyond `OP_XOR` zeroing), `a_loadmm_intreg_reg`, and `a_loadmm_reg_intreg` for SIMD/vector operations. |
| `compiler/llvm/hlcgllvm.pas:2082-2110` | `internalerror(2012090201..06)` | Implement or verify `a_bit_scan_reg_reg`, `g_stackpointer_alloc`, `g_intf_wrapper`, `g_adjust_self_value`, `g_local_unwind`. (Note: interface wrappers in LLVM currently use `ncgutil` / high-level thunks unless target overrides them). |
| `compiler/llvm/hlcgllvm.pas:2112-2128` | Comment | `gen_stack_check_size_para` / `gen_stack_check_call`: stack checking (`-Ct`) is currently a no-op in LLVM because `fpc_stackcheck` parameter patching happens after prologue generation; either patch the `taillvm` call operand or emit LLVM's `"probe-stack"` function attribute. |
| `compiler/llvm/hlcgllvm.pas:2355` | `{ todo }` | `create_hlcodegen_llvm`: clean up `chlcgobj` switching when compiling `po_assembler` routines vs normal routines. |
| `compiler/llvm/llvmbase.pas:116-121` | `// TODO` | `llvm_callingconvention_name`: calling conventions `pocall_pascal`, `pocall_oldfpccall`, `pocall_syscall`, `pocall_far16`, `pocall_softfloat` are not mapped to LLVM CC specifiers. |
| `compiler/llvm/llvmdef.pas:229-231` | `{ todo }` / `2014062202` | `llvmconvop`: update register conversion logic between `R_MMREGISTER` and `R_INTREGISTER` once full hardware vector conversions are enabled. |
| `compiler/llvm/llvmdef.pas:626-628` | `{ maybe todo }` | Generate actual closure/method-pointer data layout for `procvardef` if available instead of generic `{proc*, i8*}`. |
| `compiler/llvm/llvmtype.pas:605-608` | `{ todo: handle mantis #25551 }` | Self-referential `tprocvardef` types (e.g. a procedural type taking or returning a pointer to its own type): Note that with opaque pointers (`ptr` in LLVM 15+), pointer recursion is naturally broken, but legacy typed-pointer modes (< LLVM 15) still hit this. |
| `compiler/llvm/nllvmbas.pas:152-155` | `{ todo }` / `2016102001` | `tllvmasmnode.ResolveRef`: replace `internalerror(2016102001)` with a proper compiler error message when inline assembly attempts to take the stack frame offset (`localgetoffset`) of a local variable under LLVM. |
| `compiler/llvm/nllvmcal.pas:70-85` | Comment | `tllvmcallnode.paraneedsinlinetemp`: clean up `self` parameter type conversion across multi-level inlining so an extra inline temp is not always required. |
| `compiler/llvm/nllvmcal.pas:106-108` | `{ TODO }` | `tllvmcallnode.pushparas`: verify parameter ordering for left-to-right calling conventions (`pocall_pascal`, etc.). |
| `compiler/llvm/tgllvm.pas:49-51` | Comment | `ttgllvm`: investigate reusing temp slots (`R_TEMPREGISTER`) when lifetime intrinsics (`llvm.lifetime.start` / `llvm.lifetime.end`) allow safe slot reuse in very large routines. |

---

## 4. RTL & Compiler Workarounds Disabled Under `CPULLVM`

Several features across `compiler/` and `rtl/` are currently bypassed when `llvm` / `CPULLVM` is defined and represent opportunities for cleanup and parity:

1. **`System.Intrinsics` disabled in `Classes` (`rtl/*/classes.pp:22, 79`):**
   - `FPC_USE_INTRINSICS` is guarded with `{$ifndef CPULLVM}`. Intrinsic units (`System.Intrinsics` for ARM, AArch64, PowerPC, RISC-V, x86) should be enabled or mapped to LLVM vector/atomic intrinsics.
2. **`FreeAndNil(constref obj: TObject)` (`rtl/objpas/sysutils/sysutilh.inc:357`):**
   - Falls back to untyped `procedure FreeAndNil(var obj);` under `{$IFDEF CPULLVM}` with comment `// LLVM does not allow the constref use`. Fixing the underlying type-checking/IR generation issue for `constref TObject` allows removing this divergence.
3. **`min()` / `max()` branchless `select` simplification (`compiler/nflw.pas:1657`):**
   - `HAS_MINMAX_INTRINSICS` is disabled for `{$ifndef llvm}` in `tifnode.internalsimplify`. Either allow `select` (`la_select`) node generation in `nllvmflw` / `nllvminl` or rely on LLVM's `InstCombine` / `SimplifyCFG` (and document why FPC's `HAS_MINMAX_INTRINSICS` is skipped).
4. **Extended 80-bit (`x86_fp80`) Transcendental Math (`rtl/x86_64/math.inc:135-244`):**
   - `fpc_abs_real`, `fpc_sqr_real`, `fpc_sqrt_real`, `fpc_arctan_real`, `fpc_ln_real`, `fpc_sin_real`, `fpc_cos_real` use inline x87 assembly fallbacks on `cpullvm` instead of compiler-internal nodes when `ValReal = Extended`. `nllvminl.pas` already handles `abs`, `sqr`, `sqrt` in firstpass, and LLVM has `@llvm.sin.f80`, `@llvm.cos.f80`, `@llvm.log.f80` intrinsics that can replace the inline asm stubs.

---

## 5. Proposed Implementation Phases

### Phase 1: Quick Wins & Toolchain Modernization
- [ ] Add `llvmver_18_0`, `llvmver_19_0`, `llvmver_20_0`, `llvmver_21_0`, `llvmver_22_0`, and `Xcode-16` to `compiler/llvm/llvminfo.pas`.
- [ ] Replace `internalerror(2016102001)` in `compiler/llvm/nllvmbas.pas:155` with a proper user-facing parser/cg message.
- [ ] Implement `a_loadmm_reg_ref` size-mismatch pointer cast in `compiler/llvm/hlcgllvm.pas:1853` (matching `a_loadmm_ref_reg`).
- [ ] Enable `-march=` mapping for `x86_64` and `arm` in `compiler/llvm/agllvm.pas:1744` and add `-Os` (`cs_opt_size`) forwarding.

### Phase 2: Target Expansion (Linux / BSD)
- [ ] Enable and test `powerpc64-linux`, `riscv64-linux`, `loongarch64-linux`, and `i386-linux` in `compiler/Makefile.fpc`, `agllvm.pas`, and `i_linux.pas`.
- [ ] Add `aarch64-freebsd` and `aarch64-openbsd` to `as_clang_llvm_info.supported_targets` in `agllvm.pas`.

### Phase 3: Debug Info, Threadvars & Optimizer Metadata
- [ ] Implement `@llvm.threadlocal.address` intrinsic support for `thread_local` variables in LLVM 16+ and enable `tf_section_threadvars`.
- [ ] Add DWARF `DIExpression` support for `AnsiString` / `UnicodeString` (`dbgllvm.pas:1410`), open arrays (`dbgllvm.pas:946`), bitpacked arrays (`dbgllvm.pas:976`), and `threadvar`s (`dbgllvm.pas:2051`).
- [ ] Emit `!range` metadata on ordinal subrange/boolean/enum loads and `-Ct` stack probing (`"probe-stack"` attribute).

### Phase 4: Hardware Exceptions & Windows / Wasm Targets
- [ ] Implement automatic outlining of `try..except` / `try..finally` blocks or volatile stack slot promotion for hardware exception safety (`llvmpi.pas`).
- [ ] Implement Windows SEH (`catchswitch` / `catchpad` / `cleanuppad`) in `llvmpi.pas` and COFF symbol handling to bring up `x86_64-win64` and `aarch64-win64` on LLVM.
- [ ] Complete `wasm32-wasi` LLVM target integration.

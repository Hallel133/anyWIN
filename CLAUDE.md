# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project goal

anyWIN converts Windows x86-64 game executables (PE/COFF `.exe` plus the `.dll`s the game ships) into programs that run natively on macOS on Apple Silicon. It follows the model of [AnyPS5](https://github.com/boykopovar/AnyPS5), which does the same for PS5 executables targeting Linux and Windows:

- **No emulator process and no separate runtime.** A static *relinker* rewrites the input executable into the host's native executable format, and the OS loader binds its imports to native reimplementations of the Windows system libraries.
- Only **system** libraries are reimplemented (the ones that reach the kernel or hardware: `ntdll`, `kernel32`/`kernelbase`, `user32`, `gdi32`, `d3d*`/`dxgi`, `xinput`, `xaudio2`/`dsound`, `ws2_32`, ...). DLLs the game ships itself (MSVC runtime `vcruntime140`/`msvcp140`, middleware such as FMOD, Bink, PhysX) are converted by the relinker and loaded as-is, not replaced.

## Current state

The repository is at its initial commit: only `README.md` exists. There is no build system, source, or test suite yet, so no build/test commands work. When scaffolding, mirror AnyPS5's layout and tooling (described below) unless there is a concrete reason not to, and update this file with the real commands once they exist.

## Intended architecture

```
game/                      relinker (core/relinker)                    output (macOS app)
  Game.exe  ─────────►  PE reader → import/export analysis →  ─────►  Game (Mach-O x86-64)
  *.dll (shipped) ───►  x86 rewrite pass → Mach-O writer      ─────►  app0/*.dylib (converted game DLLs)
                                                                        libs/*.dylib (core/libs/win32/* reimplementations)
                                    dyld binds imports by DLL name + symbol / ordinal
```

- `core/relinker/` — CLI conversion tool, C++20 standard library only, buildable without third-party submodules or a GPU (AnyPS5's "relinker-only" build mode). Sub-areas as in AnyPS5: `cli`, `io` (PE parsing), `codegen` (x86 instruction rewriting), `patcher/macos` (Mach-O emission), `relinker` (pipeline).
- `core/libs/win32/<dll>/` — one shared library per reimplemented Windows DLL, exporting the Win32 names (and ordinals, for ordinal-only imports). `api-ms-win-*` API-set DLLs resolve to these.
- `core/shader/` — DXBC/DXIL shader recompiler to Metal (the counterpart of AnyPS5's RDNA→SPIR-V recompiler).

### Hard constraints that shape the design

- **CPU: output is x86-64 Mach-O run under Rosetta 2.** Windows games are x86-64 and Apple Silicon is arm64; Rosetta is the OS-provided translator, which keeps the "no emulator" model. AnyPS5 already emits x86-64 Mach-O for Rosetta (`--macos --to-rosetta`, see `core/relinker/elfpatcher/src/macos` and `core/relinker/codegen` there), including lowering instructions Rosetta lacks (RDSEED→RDRAND, CLWB→NOP, RDPID). Native arm64 static recompilation is a separate, much larger effort; do not mix it into the Rosetta path.
- **64-bit only.** Rosetta 2 does not run 32-bit x86 code, so 32-bit PE (`IMAGE_FILE_MACHINE_I386`) input is rejected.
- **Calling convention.** Game code uses the Microsoft x64 ABI (RCX, RDX, R8, R9, 32-byte shadow space, RSI/RDI/XMM6–15 callee-saved); macOS uses System V. Every export in `core/libs/win32` and every callback into game code must be declared `__attribute__((ms_abi))` (Clang supports it on x86-64 Darwin), and varargs need `__builtin_ms_va_list`.
- **TEB/PEB via GS.** Windows code reads `gs:[0x30]` (TEB), `gs:[0x58]` (TLS array), `gs:[0x60]` (PEB). macOS reserves GS for its own thread-specific data, so the relinker's codegen pass must rewrite GS-relative accesses in game code and shipped DLLs; it cannot rely on setting the GS base.
- **Exceptions.** x64 SEH and C++ exceptions unwind through the PE `.pdata`/`.xdata` tables. The relinker must preserve them, and the `ntdll`/`kernel32` reimplementation must dispatch host signals (SIGSEGV, SIGILL, ...) as Windows exceptions (`RtlVirtualUnwind`, vectored handlers, `__C_specific_handler`).
- **Address space.** 64-bit Mach-O executables reserve a 4 GiB `__PAGEZERO` by default; PE images with a preferred base below 4 GiB must be relocated (via `.reloc`) or the zero page shrunk. A future native arm64 path would also have to deal with 16 KiB pages versus PE's 4 KiB section alignment.
- **Graphics.** Target Metal. Apple's D3DMetal / Game Porting Toolkit is proprietary and must not be bundled or required. DXVK + MoltenVK is a possible early bring-up path but MoltenVK has feature gaps (e.g. geometry shaders, transform feedback), so the long-term backend is a direct D3D→Metal driver with a shader recompiler emitting MSL (or Metal IR).
- Anti-cheat and DRM that inspect the Windows kernel are out of scope.

## Conventions adopted from AnyPS5

- **Fail loudly.** Every function either does exactly what it is supposed to or throws `std::runtime_error`; `what()` is printed to stderr and the process terminates. Unimplemented exports call a `NotImplemented(__func__)`-style helper. Missing imports fail at startup with a clear error naming the DLL and symbol; never add fallbacks that keep running.
- Silent stubs are allowed only when they unblock a title and only affect UI; each one is listed in technical debt.
- Implement the general behaviour of a Win32 function, not what one title happens to need.
- C++ naming: interfaces (pure virtual only) start with `I`; classes and public methods PascalCase; private methods, fields and parameters camelCase; template parameters `TKey`/`TValue`.
- Code comments only for technical debt, `#endif` and end of namespace. Known gaps, unknown behaviour and unverified semantics go in `docs/dev/TechnicalDebt.md`, not in comments.
- Third-party code is a git submodule under `3rdparty/`, built from source.
- Commits follow Conventional Commits.
- Docs are dry, strict and concise; screenshots and investigation logs go in PRs, not the repo.
- Tests must finish within 30 s on CI runners; don't add tests that only check a symbol is exported.

## Planned build commands (mirror AnyPS5; not implemented yet)

```sh
# relinker only, no submodules/GPU (works on Linux and macOS)
cmake -S . -B build-relinker -G Ninja -DCMAKE_BUILD_TYPE=Release -DANYWIN_RELINKER_ONLY=ON -DBUILD_TESTING=ON
cmake --build build-relinker --parallel
ctest --test-dir build-relinker --output-on-failure        # single test: ctest --test-dir build-relinker -R <name>

# full build (macOS, AppleClang)
git submodule update --init --recursive
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=ON
cmake --build build --parallel
```

Converted titles run only on an Apple Silicon Mac; relinker output-format tests should still run on Linux CI by inspecting the produced Mach-O.

## Reference and licensing

AnyPS5 is GPL-2.0-only. Code copied or adapted from it (e.g. its Mach-O writer or Rosetta instruction lowering) makes the affected files GPL-2.0-only, so the project must adopt GPL-2.0-only before any such code is brought in. Wine is LGPL-2.1+ and is a useful behavioural reference for Win32 semantics.

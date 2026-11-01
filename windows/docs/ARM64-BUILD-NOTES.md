# Ghostty for Windows: Windows ARM64 build notes

> Build log from the original ARM64 bring-up (cross-compiled on x86_64 Linux). Paths under `/workspace/` refer to that build machine; the patch described here is commit `4263e89` on this branch.

**Outcome: success.** I cross-compiled a native Windows ARM64 `ghostty.exe` on an x86_64 Linux box with Zig 0.16.0 and its bundled mingw-w64 (`aarch64-windows-gnu`). It needed one small patch (3 files, 8 lines). It has **not been run** on ARM64 hardware yet, because there was no Windows on ARM device available.

Build date: 2026-10-31 (America/Chicago)
Repo: https://github.com/phalladar/ghostty-windows at `c975f493814fbecee378316c50d600ff98e9312f`
("config: default fullscreen to F11 instead of ctrl+enter on Windows")

## Artifacts (in `/workspace/ghostty-arm64-out/`)

| File | What it is |
|---|---|
| `ghostty-arm64.exe` | The app (`PE32+ executable for MS Windows 6.00 (GUI), ARM64, 8 sections`), 27 MB, ReleaseFast, stripped |
| `Ghostty-Windows-arm64.zip` | Portable package laid out the same way as the official `Ghostty-Windows-x64.zip`: `ghostty/bin/{ghostty.exe, conpty.dll, OpenConsole.exe}` (ARM64 ConPTY), plus `share/ghostty` (themes, shell integration), `share/terminfo`, licenses and README |
| `arm64-cross-build.diff` | The source patch needed for the build (`git apply` it) |
| `ci-arm64-release-workflow.diff` | **Draft only, not pushed.** Adds an ARM64 job to `release-windows.yml` on the `windows-11-arm` runner, plus the installer arch change |
| `build-arm64.sh` | Script that reproduces the whole build |
| `SHA256SUMS.txt` | Checksums of the exe and zip |

```
e9aeb84c9f3cfff0ad4d97b3b710a7661b9e7ca32638f0f62573c3aac8a03435  ghostty-arm64.exe
7abf83f19640dc5d095bd53cda81abd66149eb13c469a7b2d5fbaefdf26a4973  Ghostty-Windows-arm64.zip
```

## Environment

- Host: Debian 13 (trixie), x86_64, 8 vCPU, 15 GB RAM
- Zig **0.16.0**, matching `minimum_zig_version` in `build.zig.zon` and what the repo docs/CI use.
  Tarball `zig-x86_64-linux-0.16.0.tar.xz`, sha256 `70e49664a74374b48b51e6f3fdfbf437f6395d42509050588bd49abe52ba3d00`
- No MSVC, no Windows SDK, no Wine. Everything comes from Zig's bundled mingw-w64 and libc++.

## What I read first

- `README.md`: the port officially supports only "Windows 11, x64", and needs an OpenGL 4.3 driver.
- `windows/docs/02-environment.md`: the official toolchain is Zig 0.16.0 plus MSVC 14.44 and SDK 10.0.26100. A documented fallback is the **GNU ABI**, which needs no MSVC: `zig build -Dapp-runtime=win32 -Dtarget=x86_64-windows-gnu`.
- `windows/docs/STATUS.md`: lists "ARM64" under "Not in plan yet".
- `.github/workflows/release-windows.yml`: the release builds on `windows-2025` (x64) with `zig build -Dapp-runtime=win32 -Doptimize=ReleaseFast -Dcpu=baseline`.
- `dist/windows/make-zip.ps1` already maps `-Arch aarch64` to the ARM64 ConPTY binaries, so the packaging script is partly ARM-aware already.

## Step-by-step

```bash
# 1. Clone
cd /workspace && git clone --depth 50 https://github.com/phalladar/ghostty-windows
cd ghostty-windows   # HEAD c975f49

# 2. Zig 0.16.0 (extracting into /opt failed: box not writable there; used ~/tools)
mkdir -p ~/tools && cd ~/tools
curl -fsSL -o zig.tar.xz https://ziglang.org/download/0.16.0/zig-x86_64-linux-0.16.0.tar.xz
tar xf zig.tar.xz && export PATH=~/tools/zig-x86_64-linux-0.16.0:$PATH
zig version            # 0.16.0

# 3. Fetch all deps (~14 s)
cd /workspace/ghostty-windows && zig build --fetch=all

# 4. First ARM64 attempt (unpatched) -> 3 compile errors (see below), ~2m40s
zig build -Dapp-runtime=win32 -Dtarget=aarch64-windows-gnu \
  -Doptimize=ReleaseFast -Dcpu=baseline -p /workspace/build-arm64-gnu

# 5. Apply the patch (arm64-cross-build.diff), then rebuild -> exit 0, ~2m15s
git apply /workspace/ghostty-arm64-out/arm64-cross-build.diff
zig build -Dapp-runtime=win32 -Dtarget=aarch64-windows-gnu \
  -Doptimize=ReleaseFast -Dcpu=baseline -p /workspace/build-arm64-gnu

# 6. Verify
file build-arm64-gnu/bin/ghostty.exe
# -> PE32+ executable for MS Windows 6.00 (GUI), ARM64, 8 sections
```

The first attempt compiled every C/C++ dependency for ARM64 without changes: FreeType, HarfBuzz, libpng, zlib, oniguruma, glslang, SPIRV-Cross, simdutf, highway, dcimgui, wuffs, and the `ghostty-vt.dll` library. Only three errors in Zig code were left.

## Errors hit and fixes

### 1. `pkg/opengl/Buffer.zig:75:17: error: expected type 'c_long', found 'isize'`

```
pkg/opengl/Buffer.zig:75:17: error: expected type 'c_long', found 'isize'
            info.size,
pkg/opengl/Buffer.zig:75:17: note: signed 32-bit int cannot represent all possible signed 64-bit values
```

**Cause:** in `src/build/SharedDeps.zig` the `opengl` package is pulled in with `b.lazyDependency("opengl", .{})`, without passing `target` or `optimize`. So its `translate-c` step (glad `gl.h` and `KHR/khrplatform.h`) runs for the **build host**, not the Windows target. You can see this in the failing command line: the `-Mopengl`/`-Mopengl_c` modules are built with `-ODebug` and no `-target`. Because `_WIN64` is not defined on a Linux host, khrplatform picks `typedef signed long int khronos_ssize_t` → `c_long`. On Windows (LLP64), `c_long` is 32-bit, which does not match `isize`.

This is **not specific to ARM64.** It breaks any Linux→Windows cross-build. I confirmed that by building `x86_64-windows-gnu` with only the other two fixes applied, and it fails with the same error (`/workspace/build-x64-unpatched-opengl.log`). It works on the maintainer's Windows host because there the host *is* Windows, so `_WIN64` is defined.

**Fix:** pass the target and optimize mode through, the same way every other dependency in that file does:

```zig
if (b.lazyDependency("opengl", .{
    .target = target,
    .optimize = optimize,
})) |dep| {
```

### 2 and 3. `@ptrCast increases pointer alignment` (ARM64-specific)

```
src/apprt/win32/dialogs.zig:99:22: error: @ptrCast increases pointer alignment
    task_dialog_fn = @ptrCast(proc);
note: '*const anyopaque' has alignment '1'
note: '?*const fn (...) callconv(.c) i32' has alignment '4'
src/pty.zig:381:27: error: @ptrCast increases pointer alignment
                .create = @ptrCast(create),
```

**Cause:** Zig gives function pointers an alignment of 4 on AArch64 (instructions are 4 bytes wide), but 1 on x86_64. Casting the `*const anyopaque` that `GetProcAddress` returns into a typed function pointer therefore needs an explicit `@alignCast` on ARM64. These casts are the dynamic loads of `TaskDialogIndirect` from comctl32 and of `CreatePseudoConsole`, `ResizePseudoConsole` and `ClosePseudoConsole` from the bundled `conpty.dll`.

**Fix:** use `@ptrCast(@alignCast(x))` at the four call sites. This changes nothing on x64, and on ARM64 the pointers that `GetProcAddress` returns are always 4-byte aligned.

`zig fmt --check` passes on all three changed files.

## Regression check: x64 still builds

With the patch applied, `zig build -Dapp-runtime=win32 -Dtarget=x86_64-windows-gnu -Doptimize=ReleaseFast -Dcpu=baseline` succeeds (2m43s). Output: `PE32+ executable for MS Windows 6.00 (GUI), x86-64`. Log: `/workspace/build-x64-gnu.log`.

## Binary inspection

I checked the PE headers with Python `pefile`:

- `ghostty.exe`: machine `0xAA64` (ARM64), PE32+, subsystem 2 (Windows GUI), and the version/icon resource is present.
- Imports are only system DLLs: USER32, GDI32, SHELL32, ole32, dwmapi, IMM32, OPENGL32, ADVAPI32, KERNEL32, ntdll, dbghelp, WS2_32, MSWSOCK, and the `api-ms-win-crt-*` (UCRT) DLLs. There is no MinGW runtime DLL, because Zig links libc++ and its mingw CRT statically. All of these ship with Windows 11 on ARM.
- The ConPTY in the zip is the same NuGet package and version the repo pins: `microsoft.windows.console.conpty 1.25.260710002-preview`, sha256 `86edb105…b114`, which matches the pin in `make-zip.ps1`. I used the `runtimes/win-arm64/native/conpty.dll` and `build/native/runtimes/arm64/OpenConsole.exe` files from it, and `file` reports both as ARM64.

## Things I tried that did not work, and why

- **MSVC ABI (`-Dtarget=aarch64-windows-msvc`) from Linux.** This fails right away: `'cstring' file not found` (simdutf) and `'stdlib.h' not found` (wuffs). Zig does not bundle the MSVC CRT or the Windows SDK. Getting them on Linux would mean running `xwin`, which downloads Microsoft's SDK and CRT and needs `--accept-license` for the Microsoft license. I did not accept that license for you. The GNU ABI build worked without it.
  GNU ABI tradeoff, from the repo's own docs: debug info is DWARF instead of PDB, so you debug with lldb/gdb rather than WinDbg. For a ReleaseFast build that does not matter much.
- **Box tooling.** `apt-get` was failing because the Debian security mirror returned HTTP 500. I installed `file` 5.46 by fetching the `.deb`s straight from deb.debian.org and installing them with `dpkg -i`. `zip` was not installed, so I built the zip with Python `zipfile`, using sorted entries and a 1980-01-01 timestamp, the same as `make-zip.ps1`.

## Not verified / caveats

1. **Not run on hardware.** I could not launch it, and `zig build test` could not run, because the box is x86_64 Linux and there is no Windows ARM64 machine or emulator here. Wine on x86_64 cannot run ARM64 PE files.
2. **OpenGL 4.3 on Windows on ARM.** The renderer needs OpenGL 4.3. Snapdragon X (Adreno) drivers have no native OpenGL. Qualcomm and Microsoft provide OpenGL up to 4.6 through a mapping layer onto D3D12 (the "OpenCL, OpenGL and Vulkan Compatibility Pack", GLOn12), which users report showing up as GL 4.6. This is the most likely runtime risk. If the app shows a blank window or fails to create a GL context, install or update the Compatibility Pack from the Microsoft Store and check `GHOSTTY_LOG=1` output.
3. The Inno Setup installer was not built, since it needs Windows. The current `.iss` hardcodes `ArchitecturesAllowed=x64compatible`. An ARM64 installer needs `arm64` there; the draft CI diff handles that.
4. The version string is the default dev one: `1.3.2-main-+c975f49`. For a release build, pass `-Dversion-string=…`.

## How to test on a Windows on ARM machine

1. Unzip `Ghostty-Windows-arm64.zip` anywhere and run `ghostty\bin\ghostty.exe`.
2. Task Manager → Details → "Architecture" column should show **ARM64**, not x64 (emulated).
3. Check PowerShell launches, tabs and splits work, and text renders. For logs, run `$env:GHOSTTY_LOG=1; .\ghostty\bin\ghostty.exe +version | Out-String`.

## Recommended upstream path: CI on `windows-11-arm` (draft, not pushed)

`ci-arm64-release-workflow.diff` turns the single `release` job into:

- a `build` matrix with `{x86_64, windows-2025}` and `{aarch64, windows-11-arm}`. `windows-11-arm` is GitHub's hosted Windows 11 ARM64 runner, free for public repos. Building natively there also runs `zig build test` on real ARM64 and keeps the default MSVC ABI and PDBs.
- `make-zip.ps1 -Arch ${{ matrix.arch }}` (the script already supports `aarch64`), and ISCC gets `/DArch` and `/DInnoArch`.
- a `publish` job that downloads both matrices' assets, writes a single `SHA256SUMS.txt`, and creates or updates the GitHub release.
- `ghostty.iss`: `ArchitecturesAllowed` and `ArchitecturesInstallIn64BitMode` become `{#InnoArch}` (default `x64compatible`, `arm64` for the ARM build).

Before merging the draft, check these (I could not test them without pushing):

- that `mlugg/setup-zig` installs `zig-aarch64-windows-0.16.0` on that runner;
- that the runner image's VS 2022 has the ARM64 MSVC tools and Windows SDK (it is expected to). If it does not, add `-Dtarget=aarch64-windows-gnu` to the ARM64 build step;
- that `actions/upload-artifact` and `download-artifact` are pinned to SHAs, matching the repo's convention.

The source patch (`arm64-cross-build.diff`) is still needed in CI. Fix #2 is needed on ARM64 even for a native build. Fix #1 is only needed for cross-builds from Linux, but it is the correct fix either way.

## Logs on the box (copies are also in `ghostty-arm64-out/logs/`)

- `/workspace/build-arm64-gnu.log`: first, failing ARM64 build
- `/workspace/build-arm64-gnu-2.log`: patched, successful ARM64 build
- `/workspace/build-x64-gnu.log`: x64 regression build (success)
- `/workspace/build-x64-unpatched-opengl.log`: proves the opengl error also hits x64 cross-builds
- `/workspace/build-arm64-msvc.log`: MSVC-ABI attempt (missing SDK headers)

## Fork & CI

- **Fork:** https://github.com/clee231/ghostty-windows
- **Branch:** https://github.com/clee231/ghostty-windows/tree/feature/arm64-build

This branch was cut from upstream `main` at `c975f49`. Nothing has been sent upstream: no PR, and no releases created.

### Commits

| Commit | What |
|---|---|
| `4263e89` | win32: fix the Windows ARM64 build (glad target, plus `@alignCast` at the `GetProcAddress` casts) |
| `f421f05` | ci(test): first draft of the arm64 jobs (later reworked) |
| `2301f99` | release-windows: arm64 matrix, `publish` job, `dry_run` input, `ghostty.iss` `InnoArch` |
| `a2c85ca` | docs: these notes |
| `e6ee352` | ci: x86_64 Zig under emulation on windows-11-arm (later replaced) |
| `f6c95bb` | build: run glad translate-c with the GNU ABI on Windows targets (fixes a regression on x64 MSVC) |
| `3e663a5` | win32: add `@alignCast` in the wgl test (an ARM64 compile error that only shows up in tests) |
| `ee7b35e` | build.zig: with `-Demit-test-exe`, set `skip_foreign_checks`, so a test binary the host can't run gets built and installed instead of failing |
| `44f75c1` | ci(test): cross-compile ARM64 on x64 and run the tests on windows-11-arm |
| `350c357` | release-windows: same cross-compile-plus-native-test layout |

### CI issues found and how they were fixed

1. **The native Zig 0.16.0 for aarch64-windows crashes.**
   On `windows-11-arm`, `mlugg/setup-zig` installs `zig-aarch64-windows-0.16.0`. `zig version` and `zig build --fetch` work, but `zig build` exits with code 1 after 0.4 s and prints nothing. This is a known upstream bug, https://codeberg.org/ziglang/zig/issues/31865 (STATUS_ACCESS_VIOLATION).
   - Run with the crash: https://github.com/clee231/ghostty-windows/actions/runs/41261075432
   - Fix: don't run Zig on ARM64 at all (see 3).

2. **x86_64 MSVC regression from my glad fix.**
   Once the real target was passed to the `opengl` dependency, native MSVC builds ran glad's translate-c against MSVC's `winnt.h`. Zig's translate-c can't parse it (`DECLSPEC_ALIGN` and `_CONTEXT` errors).
   - Run: https://github.com/clee231/ghostty-windows/actions/runs/41261338138
   - Fix (`f6c95bb`): on Windows, always translate glad with the GNU ABI. The GL types are the same under both ABIs, and before my patch native Windows builds already resolved this dependency to `native-windows-gnu`.
   - Result: x86_64 build and tests are green in https://github.com/clee231/ghostty-windows/actions/runs/41263757057

3. **x86_64 Zig under emulation on windows-11-arm works, but is too slow.**
   With this approach the app build took about 22 min and the test build about 29 min, so the job hit its 60-min timeout (run 41263757057).
   - The ARM64 test binary did run natively, and tests 1 through 1987 of 3870 had passed (none failed) when the job was cancelled.
   - Along the way it found one more ARM64 compile error, in the wgl test (`3e663a5`).
   - Fix (`44f75c1`, `350c357`): cross-compile the app and test binary with `-Dtarget=aarch64-windows-gnu -Dcpu=baseline` on the x64 Windows runner, upload them as artifacts, and run `ghostty-test.exe` natively on `windows-11-arm` in a dependent job.

### Workflow summary (current)

- **`test.yml`**
  - `build-windows-app` (x86_64) is byte-for-byte the same as upstream.
  - New `build-windows-app-aarch64` job on the same x64 runner: cross-builds `ghostty.exe` (uploaded as artifact `ghostty-windows-aarch64`) and the test exe.
  - New `test-windows-app-aarch64` job on `windows-11-arm`: runs the test exe natively.
  - `build-libghostty-vt` gets an `aarch64-windows-gnu` cross target. This job only runs upstream.
  - `build-libghostty-vt-windows` is left as upstream's x86_64-only job. An ARM64 leg there would hit the native Zig crash, and ARM64 libghostty-vt is already covered by the cross target above.
- **`release-windows.yml`**
  - `build` matrix: both arches run on `windows-2025`, and each leg builds, packages the zip, builds the Inno installer (`/DArch`, `/DInnoArch`), and uploads its assets. The aarch64 leg also builds the test exe.
  - `test-arm64` job on `windows-11-arm` runs that test exe.
  - `publish` waits for both, writes a single combined `SHA256SUMS.txt`, and creates or updates the release. With `dry_run=true` it only uploads the combined assets as a workflow artifact.
- **`ghostty.iss`:** `ArchitecturesAllowed` comes from `InnoArch`, which defaults to `x64compatible`.
- **`build.zig`:** `-Demit-test-exe` now also sets `skip_foreign_checks` on the test Run step.

### Run status (America/Chicago)

- **Test, final:** https://github.com/clee231/ghostty-windows/actions/runs/41267986708 (head `703a287`). **Success.**
  - `build-windows-app` (x86_64, unchanged from upstream): success, 22:09 to 22:36. Build plus full tests.
  - `build-windows-app-aarch64`: success, 22:09 to 22:26. Cross-compiled on windows-2025.
    - Artifact `ghostty-windows-aarch64`: `PE32+ executable for MS Windows 6.00 (GUI), ARM64`.
    - sha256 `8c1f94ec8bfff12483853b533217ec08b22fdd184b2874b05ef266218af49df7`.
    - 
  - `test-windows-app-aarch64`: success, 22:26 to 22:44. Ran natively on windows-11-arm: **3809 passed, 61 skipped, 0 failed** (of 3870).
- **Earlier runs:**
  - https://github.com/clee231/ghostty-windows/actions/runs/41261075432: native aarch64 Zig crash; cancelled.
  - https://github.com/clee231/ghostty-windows/actions/runs/41261338138: x64 glad/MSVC regression, plus the wgl test alignment error.
  - https://github.com/clee231/ghostty-windows/actions/runs/41263757057: x64 green; aarch64 under emulation timed out at 60 min.
- **release-windows dry run: not run.** `gh workflow run release-windows.yml` (retried at 20:30, 21:16 and 22:45) returns `HTTP 404: workflow release-windows.yml not found on the default branch`. GitHub has registered only `nix.yml` and `test.yml` for the fork. To register it, use one of these:
  1. Push any commit to the fork's `main` that touches `.github/workflows/release-windows.yml`, for example by merging this branch into the fork's `main`. Then run `gh workflow run release-windows.yml -R clee231/ghostty-windows --ref feature/arm64-build -f version=1.3.2-windows.0-arm64test -f dry_run=true`.
  2. Push a `windows-v*` tag. Don't use this for testing: on a tag push `dry_run` is false, so it **publishes a real release**.

  The release layout itself is checked only statically (actionlint passes). Its build, cross-compile and test steps are the same commands that passed in the Test run. Packaging (`make-zip.ps1 -Arch aarch64` and ISCC with `InnoArch=arm64`) has not run yet.

### Known limitations

- **`release-windows.yml` can't be manually triggered on the fork yet.** GitHub has only registered `nix.yml` and `test.yml` for this fork, and `gh workflow run release-windows.yml` returns `HTTP 404: workflow release-windows.yml not found on the default branch`. It will register once a commit touching that file lands on the fork's `main`, or when a `windows-v*` tag is pushed (note that a tag push publishes a real release).
- **Not yet launched on Windows on ARM hardware:** the app's GUI, ConPTY, and OpenGL via the Compatibility Pack still need a check on a real device.

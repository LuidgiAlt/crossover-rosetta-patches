# CrossOver Rosetta patches

Four patches for CrossOver 26.3's Wine on Apple Silicon. They fix problems
that only appear because Rosetta 2 translates the x86 code, mostly in 32-bit
Windows programs.

32-bit Windows programs run under Wine's WoW64 inside 64-bit processes, and
Rosetta translates their 32-bit code.

## Origin

The goal was to run Call of Duty: Black Ops II through the Plutonium client on
an M1 Pro. On stock CrossOver:
- the game hung at startup, because Plutonium relies on x86 hardware
  breakpoints and Rosetta cannot deliver them;
- once that was fixed, it froze for seconds at a time.

Both causes turned out to be general:
- anything that uses hardware breakpoints (game clients, mod loaders, some
  DRM and anti-debug code) is affected;
- anything with a module lacking the `NX_COMPAT` flag is affected, which is
  common in 32-bit games from before ~2012.

## The patches

| Patch | Builds into | Fixes | Enabled by |
|---|---|---|---|
| 0001 debug-register-state | `wineserver` | Debug registers read back as set | always |
| 0002 execution-breakpoints-fast64 | `ntdll.so` | Execution breakpoints under Rosetta | `WINE_ROSETTA_EXEC_BREAKPOINTS=1` |
| 0003 wined3d-vk-persistent-pipeline-cache | `wined3d.dll` | Pipeline compile hitches on every launch | always |
| 0004 rosetta-lazy-exec | `ntdll.so` | Multi-second stalls when DEP is off | `WINE_ROSETTA_LAZY_EXEC=1` |

They apply in order to CodeWeavers' `crossover-sources-26.3.0.tar.gz`. Without
their variables, the ntdll changes behave like stock CrossOver.

### 0001 + 0002: hardware breakpoints

Programs arm x86 debug registers (DR0-DR7) to get an exception when an
instruction runs. Rosetta cannot honour them, so CrossOver fakes success and
the breakpoint never fires. Plutonium waits for it forever at
`Loading fastfile code_pre_gfx_zm`.

- **0001** makes wineserver keep the debug registers a thread sets.
- **0002** delivers execution breakpoints by single-stepping the thread
  (trap flag) until it reaches an armed address. It is ported from
  [wine-rosetta-fix](https://github.com/masonmcbride/wine-rosetta-fix) to
  official CrossOver 26.3 sources; six hunks were rewritten for CrossOver's
  older thread-data API.
- **fast64** (in 0002) addresses the cost: 83% of the single-stepped
  instructions were in Wine's 64-bit WoW64 system-call path, where a 32-bit
  breakpoint cannot hit. After 16 consecutive 64-bit steps, stepping pauses
  for the rest of the call and resumes on the first 32-bit instruction
  (TF and `WOW64_CPURESERVED_FLAG_RESET_STATE` in the saved EFlags).
  - Only active while all armed addresses are below 4 GB.
  - 64-bit steps per breakpoint: 706 → 16.
  - wine-rosetta-fix's 104 probe cases pass with and without it.

For 32-bit programs set `WINE_ROSETTA_EXEC_BREAKPOINTS=1`,
`WINE_ROSETTA_WOW64_BREAKPOINTS=1` and `WINE_ROSETTA_CALL_WORKAROUND=1`.
`WINE_ROSETTA_STATS=1` prints a summary per second, `=2` one line per
breakpoint.

### 0003: persistent Vulkan pipeline cache

wined3d's Vulkan backend rebuilt every pipeline on every run (MoltenVK
translates the shaders and Metal compiles them, ~160 ms each). The patch adds a
`VkPipelineCache` stored in `%LOCALAPPDATA%\wined3d-vk-cache\<exe>.pipelines`.
It is saved every 32 new pipelines and on exit. Slow pipeline creations in a
menu: 404 (65 s) → 1.

### 0004: lazy exec

**Cause.** When any loaded module lacks `NX_COMPAT`, Wine turns DEP off for the
process (`loader.c`) and maps all readable memory writable+executable
(`force_exec_prot`). Under Rosetta the first write to such a page traps.
CrossOver's fault handler ("CW Hack 24945") fixes each one with two `mprotect`
calls while holding the process-wide `virtual_mutex`, so other faulting
threads wait.

**Measured** on BO2 (32-bit, wined3d on Vulkan):
- ~13 million page faults in 6 minutes;
- texture and buffer uploads stalled 1-41 s, all of it in the first write to
  each page of GPU-shared upload memory;
- `spindump`: the render thread blocked 32 s in the kernel fault path;
- `vmmap`: upload regions mapped `rwx/rwx SM=SHM` though allocated read/write.

**Fix.** DEP stays off as Wine decides, but memory is no longer made
executable up front:
- a page gets `PROT_EXEC` on its first instruction fetch (also when Rosetta
  reports that fetch as a read fault at the instruction pointer);
- a write to such a page drops `PROT_EXEC`, and the next fetch adds it back,
  so self-modifying and generated code still work;
- data pages stay read/write and never trap.

Result on BO2: no upload stalls left.

**Limits.** It only removes this slowdown. It has been tested in depth with
one game; programs that generate a lot of code (emulators, .NET, script
engines) exercise the write-then-execute path more.

**Checking a program.** Lists the modules without `NX_COMPAT`:

```bash
python3 - "/path/to/game" <<'EOF'
import os, struct, sys
for dp, _, fs in os.walk(sys.argv[1]):
    for f in fs:
        if not f.lower().endswith((".exe", ".dll")): continue
        d = open(os.path.join(dp, f), "rb").read(4096)
        if d[:2] != b"MZ": continue
        pe = struct.unpack_from("<I", d, 0x3c)[0]
        if d[pe:pe+4] != b"PE\0\0": continue
        bits = 64 if struct.unpack_from("<H", d, pe + 24)[0] == 0x20b else 32
        if not struct.unpack_from("<H", d, pe + 94)[0] & 0x100:
            print(f"no NX_COMPAT ({bits}-bit): {os.path.join(dp, f)}")
EOF
```

## Other fixes for the same stall

The root cause of 0004 was reported first by rdbell on September 4, 2026, in
[athei/wine-build#3](https://github.com/athei/wine-build/issues/3). They
measured 12 ms → 2,260 ms per frame in a test program. That build adopted two
fixes: decide DEP from the main executable only, or keep DEP on under Rosetta
(`WINE_DISABLE_NX_COMPAT`). The lock contention in CrossOver's fault handler,
which turned the per-page cost into multi-second whole-process stalls, was not
part of that report.

| | Stock | DEP from main exe | DEP kept on | Lazy exec |
|---|---|---|---|---|
| A DLL lacks `NX_COMPAT` | stalls | fixed | fixed | fixed |
| The main exe lacks `NX_COMPAT` | stalls | **stalls** | fixed, DEP forced on | fixed |
| Program runs code from data memory | works | works | **may crash** | works |
| DEP decision matches Windows | no (DLL case) | yes | no | no (DLL case), as stock |
| Cost | trap on first write to every page | none | none | one fault per page that runs code |

Lazy exec keeps Wine's DEP decision and removes its cost under Rosetta, so
programs that rely on DEP being off behave as before. That covers a main
executable without `NX_COMPAT`, which BO2's Plutonium bootstrapper is and the
main-exe fix does not handle. The two approaches combine: deciding DEP from the
main executable, as Windows does, plus lazy exec for when DEP is off.

## Building

```bash
curl -LO https://media.codeweavers.com/pub/crossover/source/crossover-sources-26.3.0.tar.gz
tar -xzf crossover-sources-26.3.0.tar.gz sources/wine
cd sources/wine && for p in ../../patches/000*.patch; do patch -p1 < "$p"; done
mkdir ../../build && cd ../../build
export SDKROOT=$(xcrun --sdk macosx --show-sdk-path) MACOSX_DEPLOYMENT_TARGET=15.0 \
  CC='clang -arch x86_64' CXX='clang++ -arch x86_64' ac_cv_func_pipe2=no \
  PATH="/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/flex/bin:$PATH"
../sources/wine/configure --host=x86_64-apple-darwin24 --enable-archs=i386,x86_64 \
  --disable-tests --without-x --without-wayland --without-gstreamer --without-ffmpeg \
  --without-freetype --without-gnutls --without-vulkan --without-oss --without-alsa \
  --without-pulse --without-sane --without-usb --without-v4l2 --without-pcap \
  --without-capi --without-opencl --without-cups
make -j10 dlls/ntdll/ntdll.so server/wineserver \
  dlls/wined3d/i386-windows/wined3d.dll dlls/wined3d/x86_64-windows/wined3d.dll
codesign -fs - dlls/ntdll/ntdll.so server/wineserver
i686-w64-mingw32-strip --strip-debug dlls/wined3d/i386-windows/wined3d.dll
x86_64-w64-mingw32-strip --strip-debug dlls/wined3d/x86_64-windows/wined3d.dll
```

Requires Xcode command line tools and Homebrew `mingw-w64 bison flex`.

## Installing

Install into a **copy** of CrossOver's runtime
(`CrossOver.app/Contents/SharedSupport/CrossOver`), not into CrossOver.app:
changing the signed app can stop it from opening, and updates overwrite it.
The builds only work with the CrossOver 26.3 runtime they come from.

| Built file | Replaces |
|---|---|
| `build/dlls/ntdll/ntdll.so` | `lib/wine/x86_64-unix/ntdll.so` |
| `build/server/wineserver` | `bin/wineserver` |
| `build/dlls/wined3d/i386-windows/wined3d.dll` | `lib/wine/i386-windows/wined3d.dll` |
| `build/dlls/wined3d/x86_64-windows/wined3d.dll` | `lib/wine/x86_64-windows/wined3d.dll` |

Keep the originals for rollback, and stop the prefix's wineserver
(`wineserver -k`) before using the new files.

## NotProton

NotProton runs Steam games through its own copy of the CrossOver runtime at
`~/Library/Application Support/notproton/runners/crossover-26.3.0.39832/CrossOver`.
This repository does not include an integration; what one needs:

- **Files.** Replace the four files above in that runner. NotProton modifies
  only the PE `ntdll.dll`, `lsteamclient` and the `wine` loader's signature,
  so there is no conflict. Keep originals as `<file>.notproton-orig`, its own
  backup naming.
- **Reinstall runner** re-copies CrossOver and drops the files; the install
  has to run again after it. App updates keep them.
- **Environment.** Games start through `open -n --env ...`, which forwards
  only a filtered set of variables. In `dylib/feats/compat_run.sh`, add
  `WINE_ROSETTA_` to the forwarding pattern and default
  `WINE_ROSETTA_LAZY_EXEC` to 1. Games then override per title in Steam
  Launch Options, for example `WINE_ROSETTA_LAZY_EXEC=0 %command%`, or the
  three breakpoint variables for programs that need hardware breakpoints.
- **Tests.** `dylib/tests/envcheck.sh` should check that the `WINE_ROSETTA_`
  variables reach the launcher. The run script must not write into
  `$CX_ROOT` (`ShimAgreementTests`), so install the files from the app or a
  separate step.
- Replacing `wineserver` rebuilds NotProton's prefix templates once.

## License

LGPL-2.1-or-later (`LICENSE`), the license of the Wine files the patches
modify. Patches 0001 and 0002 derive from wine-rosetta-fix (LGPL-2.1). Each
patch has a header with its changes, date and origin.

- **Source.** Only the patches are distributed here. If you distribute builds,
  the LGPL requires you to provide the complete source with them: the patches
  plus `crossover-sources-26.3.0.tar.gz` (SHA-256
  `ac99c8ca4b3848f3e81784135f023df266b61c2345726ea55a50b3e030dd6872`).
- **CrossOver.** Nothing here comes from or changes CodeWeavers' proprietary
  code. Running a CrossOver runtime still requires a valid CrossOver license.
- **Programs.** The patches change Wine, not the programs that run on it.
  They do not bypass DRM or anti-cheat.

CrossOver is a trademark of CodeWeavers. This project is not affiliated with
CodeWeavers or Apple.

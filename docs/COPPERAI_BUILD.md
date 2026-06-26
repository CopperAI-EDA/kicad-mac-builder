# CopperAI Mac Builder — Build & Release Guide

Everything needed to build, package, and test the CopperAI macOS app from this repo.

---

## Repo Structure

```
kicad-mac-builder/
├── build-release.sh          # Full clean build from source (slow, ~hours)
├── gen_cmake.sh              # Generates cmake invocation for the KiCad build
├── release.sh                # Orchestrates build + DMG in one shot
├── kicad-mac-builder/        # Upstream cmake modules (wx, python, etc.)
├── dmgbuild/
│   ├── run.sh                # DMG packaging script (use this directly)
│   ├── copperai.icns         # CopperAI app icon
│   ├── copperai.ico          # Windows icon (for reference)
│   ├── background.png        # DMG window background
│   ├── INSTALL.txt           # Shown inside the DMG
│   └── settings.py           # dmgbuild config (fallback mode)
└── build-release/            # Build output (gitignored)
    └── kicad-dest/
        └── CopperAI.app      # The built bundle
```

---

## Runtime Dependency Warning

`build-release/kicad-dest/CopperAI.app` links `libwx`, `libngspice`, and `Python.framework` via absolute `@rpath` pointing into `build-release/`. **Deleting `build-release/` breaks the app** even if the bundle looks complete.

To restore without a full rebuild, copy those three from an installed `/Applications/CopperAI.app`:
```bash
cp -R /Applications/CopperAI.app/Contents/Frameworks/libwx* build-release/kicad-dest/CopperAI.app/Contents/Frameworks/
cp -R /Applications/CopperAI.app/Contents/Frameworks/Python.framework build-release/kicad-dest/CopperAI.app/Contents/Frameworks/
```

---

## Build Variants

### Incremental (during development)

After editing C++ source in `vendor/Kicad_10.0.0/`:

```bash
cd build-release/kicad/src/kicad-build
make -j8 kicommon eeschema
```

Then deploy to the bundle:

```bash
BUNDLE=build-release/kicad-dest/CopperAI.app/Contents
cp kicad/KiCad.app/Contents/PlugIns/_eeschema.kiface $BUNDLE/PlugIns/
cp kicad/KiCad.app/Contents/Frameworks/libkicommon.dylib $BUNDLE/Frameworks/
codesign --sign - --force $BUNDLE/Frameworks/libkicommon.dylib
codesign --sign - --force $BUNDLE/PlugIns/_eeschema.kiface
codesign --sign - --force $BUNDLE/../
```

### Full Build

```bash
bash build-release.sh
```

This rebuilds wx, Python, ngspice, and KiCad from scratch. Takes hours.

---

## Running the Dev Build

```bash
# CORRECT — forces macOS to open this specific bundle, not /Applications/CopperAI.app
open -n /path/to/build-release/kicad-dest/CopperAI.app
```

**Why `-n` is required:** Both the dev build and the installed DMG share bundle ID `com.copperai.copperai` (set by `dmgbuild/run.sh`). Without `-n`, macOS opens whichever instance is already registered (usually `/Applications/CopperAI.app`).

Do NOT run the `kicad` binary directly — it dies at Python init without `KICAD_RUN_FROM_BUILD_DIR=1` and other env vars.

---

## Building a DMG

```bash
cd dmgbuild
SIGNING_IDENTITY=- DMG_NAME="copperai-10.0.0.dmg" bash run.sh
```

**`SIGNING_IDENTITY=-`** means ad-hoc signing. Required unless you have a valid Apple Developer ID cert with network access for the timestamp server. With a real cert, `apple.py` verifies secure timestamps on every binary — `kicad-cli` often fails this check if the build was ad-hoc signed incrementally.

### What `run.sh` does

1. Auto-detects `APP_SOURCE` (prefers `../build-release/kicad-dest/CopperAI.app`)
2. Copies to `stage/CopperAI.app`
3. Patches `Info.plist`:
   - `CFBundleName` → `CopperAI`
   - `CFBundleIdentifier` → `com.copperai.copperai`
   - `CFBundleExecutable` → `kicad`
   - `LSEnvironment` → sets `KICAD_USE_EXTERNAL_PYTHONHOME=1`, `JSC_useJIT=0`, `KICAD_AGENT_CHAT_URL`
4. Patches icons from `vendor/Kicad_9.0.7/` source (stock KiCad icons for sub-tools; KiCad 10 has no sub-apps so those cp commands silently no-op)
5. Hard-links `copperai` → `kicad` if needed (KiCad 10 binary is already named `kicad`, so this is a no-op)
6. Signs with `apple.py` (or ad-hoc)
7. Creates DMG with `hdiutil` in `direct` mode
8. Signs and verifies DMG

### LSEnvironment vars (set by run.sh)

| Var | Value | Purpose |
|---|---|---|
| `KICAD_USE_EXTERNAL_PYTHONHOME` | `1` | Prevents KiCad from using the build-time absolute Python path |
| `JSC_useJIT` | `0` | Disables JavaScriptCore JIT — prevents ~80% crash rate on macOS 15.7+ when opening schematics |
| `KICAD_AGENT_CHAT_URL` | `https://getcopper.dev/chat?copper_client=kicad` | Chat URL loaded in the agent panel WebView |

---

## Adding New Bitmaps to KiCad

When adding a new toolbar icon (e.g. `copper_ai_agent`), three things must be updated:

1. **PNG files** — generate at 16, 24, 32, 48px for light and dark:
   ```bash
   rsvg-convert -w 24 -h 24 icon.svg -o resources/bitmaps_png/png/my_icon_24.png
   cp resources/bitmaps_png/png/my_icon_24.png resources/bitmaps_png/png/my_icon_dark_24.png
   # repeat for 16, 32, 48
   ```

2. **`images.tar.gz`** — add PNGs to the build-tree archive (cmake won't auto-detect new files):
   ```bash
   cd resources/bitmaps_png/png
   cp $BUILD/resources/images.tar.gz /tmp/
   gunzip /tmp/images.tar.gz
   tar -rf /tmp/images.tar my_icon_24.png my_icon_dark_24.png ...
   gzip -9 /tmp/images.tar
   cp /tmp/images.tar.gz $BUILD/resources/images.tar.gz
   ```

3. **`include/bitmaps/bitmaps_list.h`** — add enum entry:
   ```cpp
   my_icon,
   ```

4. **`common/bitmap_info.cpp`** — add cache entries (MANDATORY — forgetting this causes a segfault in `wxImage::ResampleBox` when the toolbar loads):
   ```cpp
   aBitmapInfoCache[BITMAPS::my_icon].emplace_back(BITMAPS::my_icon, wxT("my_icon_24.png"), 24, wxT("light"));
   aBitmapInfoCache[BITMAPS::my_icon].emplace_back(BITMAPS::my_icon, wxT("my_icon_dark_24.png"), 24, wxT("dark"));
   ```

Then rebuild `kicommon` (embeds the bitmap archive) and `eeschema`.

---

## KiCad 10 Bundle Structure (vs KiCad 9)

KiCad 9 had separate `.app` bundles under `Contents/Applications/` (eeschema.app, pcbnew.app, etc.). KiCad 10 eliminated them — everything runs in the main `CopperAI.app` process as `.kiface` plugins:

```
CopperAI.app/Contents/
├── MacOS/kicad              # Main executable
├── PlugIns/
│   ├── _eeschema.kiface
│   ├── _pcbnew.kiface
│   └── ...
├── Frameworks/
│   ├── libkicommon.dylib
│   ├── libwx_osx_cocoau-3.2.0.4.1.dylib
│   ├── Python.framework/
│   └── ...
└── Resources/
    └── kicad.icns           # Replace with copperai.icns for branding
```

The icon patching in `run.sh` that copied to `Contents/Applications/eeschema.app/...` is now all no-ops (those directories don't exist). Only `Contents/Resources/kicad.icns` matters.

---

## Icon Branding

The `kicad.icns` inside the bundle is what macOS shows in the Dock and Finder. The CopperAI icon lives at `dmgbuild/copperai.icns`.

To update the icon in a running dev build:
```bash
cp dmgbuild/copperai.icns build-release/kicad-dest/CopperAI.app/Contents/Resources/kicad.icns
codesign --sign - --force build-release/kicad-dest/CopperAI.app/
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister \
  -f build-release/kicad-dest/CopperAI.app/
killall Finder
```
Then relaunch the app (Dock caches the icon until relaunch).

---

## Crash Triage

### Crash on schematic open (intermittent ~80%)

**Symptom:** App crashes shortly after opening a schematic. Crash log shows JavaScriptCore / JSC in the stack.

**Cause:** JSC JIT initializing on a coroutine fiber stack (heap-allocated, treated by JSC as off-main-thread).

**Fix:** `JSC_useJIT=0` in `LSEnvironment`. Already set by `run.sh`. If it's missing from a dev build's `Info.plist`, add it manually:
```bash
plutil -replace LSEnvironment -json '{"KICAD_USE_EXTERNAL_PYTHONHOME":"1","JSC_useJIT":"0","KICAD_AGENT_CHAT_URL":"https://getcopper.dev/chat?copper_client=kicad"}' \
  CopperAI.app/Contents/Info.plist
```

### "KiCad bridge is not connected"

**Symptom:** Agent panel loads but the web app shows a bridge connectivity error.

**Cause:** `RunWebViewScriptFireAndForget` in `kiplatform/port/wxosx/ui.mm` returns `false`. When this happens, all bridge injection scripts (`window.kicad.ipc`, etc.) queue in `m_deferredScripts` and are retried every 100ms — but the retry also calls `RunWebViewScriptFireAndForget`, so they loop forever and never execute.

**Fix:** Implement `RunWebViewScriptFireAndForget` using `[WKWebView evaluateJavaScript:completionHandler:nil]`.

### Crash in `wxImage::ResampleBox` on startup

**Symptom:** Crash immediately when eeschema opens, stack shows `ACTION_TOOLBAR::Add → BITMAP_STORE::GetBitmapBundleDef → resampleImage → wxImage::ResampleBox`.

**Cause:** A new `BITMAPS::` enum entry was added to `bitmaps_list.h` but the corresponding entry was not added to `common/bitmap_info.cpp`. The bitmap store returns null/empty for the unknown ID.

**Fix:** Add entries to both `bitmaps_list.h` AND `bitmap_info.cpp`.

---

## Disk Space Notes

The build tree is large:
- `build-release/kicad/` — ~10-20G (full KiCad build with all deps)
- `build-release/kicad-dest/CopperAI.app` — ~1.5G
- `dmgbuild/stage/` — another ~1.5G copy created during DMG build (deleted each run)

Keep `/tmp` and the home partition monitored — DMG builds that copy large repos will fail silently when disk fills.

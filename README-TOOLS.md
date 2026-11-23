# xsystem4 Tools (xsystem4-android 1.0.8 + FSR / SGSR upscaling + cheat engine)

This is xsystem4-android 1.0.8 with three additions:

1. **FSR 1.0** upscaling (AMD FidelityFX Super Resolution)
2. **SGSR 1.0** upscaling (Qualcomm Snapdragon Game Super Resolution)
3. **Quality modes** for both (Ultra Quality / Quality / Balanced / Performance)
4. **A cheat engine**: search the game's variables for a value, change it, freeze it, or browse
   the game's named variables (gold, HP, flags...) directly.

Everything is reached from a small gear button that floats over the game.

> **This zip does not contain a built APK.** The environment I worked in cannot download the
> Android SDK / NDK / Gradle, so I could not compile an APK. Section 1 below gets you one with a few
> clicks, using GitHub's free build servers. Everything else (the engine code, the shaders and the
> Android UI) was written and tested as described in section 6.

---

## 1. Getting the APK

The engine (`xsystem4/`) is already included, so there are no submodules to fetch.

1. Create a free account at github.com and a new **empty** repository (private is fine).
2. Upload this whole folder, **including the hidden `.github` folder**. The most reliable way is git:

   ```
   cd xsystem4-android-tools
   git init
   git add -A
   git commit -m "xsystem4 tools"
   git branch -M main
   git remote add origin https://github.com/YOUR-NAME/YOUR-REPO.git
   git push -u origin main
   ```

3. Open the repository's **Actions** tab. The *Build APK* workflow starts automatically on push
   (or press **Run workflow**). It compiles the engine for three CPU types and then the app, so it
   takes a while.
4. When it finishes, open the run and download the **app-release.apk** artifact.
5. Copy it to the phone and install it (allow "install unknown apps" for your file manager/browser).

The workflow signs the APK with a key it generates itself. That is fine for installing and using it,
but each build gets a *different* key, so to install a newer build you must first uninstall the
older one **from this app** (see section 2 about data). If you want updates to install over each
other, create a keystore once and add the repository secrets `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`,
`KEY_ALIAS`, `KEY_PASSWORD` (the same ones the original project used).

**If the build fails**, open the failed step in the Actions log and send me the last ~50 lines. I
could not run the Gradle/NDK build myself, so a first-run problem there is possible.

## 2. Installing next to the original app (important)

This build has its **own app ID** (`io.github.kichikuou.xsystem4.tools`) and is named
*xsystem4 Tools*, so it installs **alongside** the original xsystem4 and does not replace it.

I did this on purpose: games and saves live in `Android/data/<app id>/files/`, and Android
**deletes that folder when an app is uninstalled**. If this build reused the original app's ID you
would have to uninstall the original first and would lose the games and save data stored there.

So: copy your game folders to
`Android/data/io.github.kichikuou.xsystem4.tools/files/` (same USB procedure as the original app;
the in-app help shows the new path). The game folder includes its `SaveData`, so saves come along.
Your original install is left untouched as a backup.

(If you really want the original ID, delete the `applicationId` line in `project/app/build.gradle`
and back your games up first.)

## 3. Using it

A translucent gear button sits at the top-right of the game. **Tap** it to open the panel;
**drag** it to move it out of the way. On Android 7+ the game keeps running while the panel is
open (on Android 5/6 the system pauses it while the panel has focus; everything still works).

### Graphics tab

* **Upscaler**: Off (default, the game's normal smooth stretch), FSR 1.0, or SGSR 1.0.
* **Quality mode**: the resolution the upscaler works at, as a share of your screen:

  | Mode | Works at | Notes |
  |---|---|---|
  | Ultra Quality | 100% | sharpest, most GPU work |
  | Quality | 85% | |
  | Balanced | 70% | |
  | Performance | 50% | lightest |

  It never works below the game's own resolution, and the result is stretched the rest of the way
  with the normal smooth filter. Settings apply immediately and are remembered. The status line
  shows what is actually happening, e.g. `FSR 1.0, Quality: 1280x720 -> 1632x918 (screen 1920x1080)`.
  If your screen is not larger than the game's resolution the upscaler reports "inactive".

  The default is **Off** because I could not test on a real device; switch it on once and compare.
  If a GPU rejects the shaders, the game falls back to the normal picture and the status line says
  why - it does not crash.

### Scan tab (cheat engine)

Classic search-and-narrow:

1. Note a number in the game (e.g. your gold: 1500). Enter `1500`, press **First scan**.
2. Spend or earn some gold so it changes (e.g. now 1320). Enter `1320`, press **Next scan**.
3. Repeat until a few results remain. Tap one to **Set** a new value, or **Freeze** it.

Besides *Exact value* there are *Unknown initial value* (first scan) and *Changed / Unchanged /
Increased / Decreased / Greater / Less* (next scans) for values you cannot see. Whole numbers and
decimals are supported; booleans count as whole numbers. "Include local variables" also searches
function-local variables (off by default; usually unnecessary and noisy).

### Variables tab

Search the game's own variable names (`gold`, `hp`, Japanese names work too). Because it reads the
game's symbol table it shows real names such as `g_Party[2].HP`, and lets you edit without scanning.
Games built without symbol names show generic labels instead.

### Frozen tab

Lists frozen values. Tap one to change or release it.

## 4. Things worth knowing

* **Freeze** re-applies the value every time the engine services events or sleeps (many times per
  second). If the game reads a value in the instant between its own change and the next re-apply, it
  can see the new value for that instant, so very fast damage/consumption may still register once.
* Only numbers are editable (int, float, bool). Strings and objects are not.
* Modified values are written into your save files like any other value. Make a backup of saves
  before experimenting. Use on single-player games you own.
* Scans list at most 3,000,000 candidates; if you hit that, narrow with a more specific value.
* The panel shows "The game is busy" if the game does not reach a safe point within 2 seconds;
  just try again.
* The panel text is in English only (the rest of the app keeps its Japanese translation).

## 5. How it works (for the curious)

* **Upscaling** hooks the final "stretch the game surface to the window" step (`gfx_swap` in
  `xsystem4/src/video.c`). The game keeps rendering at its fixed logical resolution; when an upscaler
  is on, that surface is first run through it (`xsystem4/src/upscale.c`, shaders in
  `xsystem4/shaders/`), then drawn as before (letterboxing and screen-shake still work).
  FSR = EASU upscale + RCAS sharpen. SGSR = single pass (version 1; version 2 needs motion
  vectors/depth, which a 2D game doesn't have).
* **Cheat engine** (`xsystem4/src/cheat.c`) addresses variables as (heap slot, index, slot
  sequence number); the sequence number detects objects that were freed or replaced, so stale
  addresses are refused instead of corrupting something unrelated. The UI thread takes the VM
  from the game thread only at safe points (event pump / sleep), guarded by a mutex.
* Android UI: `ToolsPanel.kt`, `NativeBridge.kt` (JNI), `GraphicsSettings.kt`.
  JNI functions live in `xsystem4/src/android_jni.c`.

## 6. What was and was not verified

Done, with real execution (software GPU / real JVM / sanitizers):

* **Shaders**: compile as GLSL ES 3.00 (and 1.40). FSR EASU is **bit-identical** to AMD's
  unmodified reference shader on a test image; RCAS within 2/255 of AMD's reference; SGSR matches
  Qualcomm's original except for ~0.1-0.3% of pixels that I traced to knife-edge threshold ties
  and cases where the reference run itself disagreed with an independent float64 implementation
  of the same algorithm.
* **Engine path**: the real `gfx_init()` / `gfx_swap()` ran headless; FSR/SGSR output through the
  engine was bit-identical to the standalone shader run; switching Off restored the exact original
  image; pillarboxed windows kept clean black bars; 120 frames of mode/quality changes produced
  no GL errors; missing/rejected shaders fall back to the normal picture with a status message.
* **Quality measurement** (synthetic anime-style test image, upscaled 3x, scored against the
  true full-resolution image; one image, so treat as indicative):

  | | PSNR |
  |---|---|
  | Off (plain stretch) | 25.73 dB |
  | FSR Ultra / Quality / Balanced / Performance | 27.22 / 27.05 / 26.88 / 26.36 dB |
  | SGSR Ultra / Balanced | 27.73 / 27.03 dB |

* **Cheat engine**: 12 test groups against the real heap/page code with a hand-built game
  (Japanese Shift-JIS names, structs, arrays, arrays of structs): scans, labels, writes, freeze
  against constant damage, stale-address handling, busy timeout, 400 UI operations against a
  running game thread. Clean under ThreadSanitizer, and AddressSanitizer + UBSan.
* **Kotlin <-> native boundary**: the compiled `NativeBridge` was called through real JNI in a
  JVM with `-Xcheck:jni` (no warnings), including Japanese text and an emoji label.
* **Kotlin** type-checks against the Android API 34 jar with the project's Kotlin version (1.7.20).
  This caught one real bug, which is fixed.

**Not done:**

* No APK was built, so **the Gradle/NDK build has never been run** on this code.
* **Nothing has run on a real Android device or emulator.** The panel's layout, the floating
  button, and the real GPU drivers (Adreno/Mali) are untested. The C code was compiled with gcc on
  Linux, not with the NDK's clang.
* No real AliceSoft game was available, so the cheat engine was tested on a simulated game, not
  an actual one.
* The desktop (Meson) build of the engine was not built.

## 7. Files changed

`xsystem4/` (engine): new `src/upscale.c`, `src/cheat.c`, `src/android_jni.c`, `include/upscale.h`,
`include/cheat.h`, `shaders/{upscale.v,fsr_easu.f,fsr_rcas.f,sgsr.f}.glsl`, `licenses/`;
modified `src/video.c`, `src/vm.c`, `src/input.c`, `src/system4.c`, `include/gfx/gfx.h`,
`CMakeLists.txt`, `src/meson.build`. (`engine-tools.patch` next to this zip is that diff against
upstream commit `de65718`.)

Android: new `ToolsPanel.kt`, `NativeBridge.kt`, `GraphicsSettings.kt`; modified
`XSystem4Activity.kt`, `LicensesMenuActivity.kt`, `app/build.gradle` (app ID, version name),
`strings.xml` (app name, install path), `collect-licenses.sh`, `.github/workflows/build.yml`.

## 8. Licenses

FSR (MIT, (c) AMD) and SGSR (BSD-3-Clause, (c) Qualcomm) notices are in `xsystem4/licenses/`,
bundled into the app and listed under the app's Licenses screen. The rest is under the original
projects' licenses (xsystem4: GPL-2.0-or-later).

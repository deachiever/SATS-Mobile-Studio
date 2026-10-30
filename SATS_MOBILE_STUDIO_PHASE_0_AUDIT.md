# SATS Mobile Studio — Phase 0 Technical Audit Report

**Date:** October 30, 2026  
**Auditor:** Antigravity Agent  
**Environment:** Android ARM64 / Ubuntu 20.04.6 LTS (PRoot Environment)  
**Target Project:** SATS Mobile Studio (Fork / Evolution of Mobile Harness)  
**Status:** Read-Only Technical Audit Complete  

---

## Executive Summary

This audit assesses the technical feasibility, architectural components, dependency prerequisites, build systems, licensing constraints, and development pathways for **SATS Mobile Studio**, an Android-first AI-powered development environment based on the upstream open-source project **Mobile Harness** (`com.jarves.mh` / `techjarves/Mobile-Harness`).

The current execution environment is running *inside* an active instance of Mobile Harness, specifically within its userland PRoot Ubuntu container. This provides first-hand observation of how the host Android application, native execution layer, PRoot container, and Antigravity agent interact in practice.

---

## 1. Repository and Source Directory Identification

### 1.1 Upstream Source Repository
* **Repository Name:** Mobile Harness ("Claude Code on Android / Mobile Coding IDE")
* **GitHub URL:** `https://github.com/techjarves/Mobile-Harness`
* **Default Branch:** `main`
* **Latest Verified Commit SHA (HEAD):** `96b1da08d03af4f2d63970a3794c7fda068948c4`
* **Primary Language:** Kotlin (with C / ASM native components)
* **License:** MIT License (`LICENSE`)

### 1.2 Host Application & Package Identification
* **Application Package Namespace:** `com.jarves.mh`
* **Application Name:** Mobile Harness (internal codename / legacy references: `PocketDev`)
* **Host App Directory:** `/data/user/0/com.jarves.mh/` (Android internal private storage)
* **Host Native Libraries:** `/data/app/~~.../com.jarves.mh-.../lib/arm64`
* **Host Workspaces Root:** `/data/user/0/com.jarves.mh/files/workspaces/`
* **Host Runtime Rootfs:** `/data/user/0/com.jarves.mh/files/runtime/ubuntu/`

### 1.3 Guest Container Layout (PRoot)
* **Guest Root:** `/` (maps to `/data/user/0/com.jarves.mh/files/runtime/ubuntu/`)
* **Workspaces Mount:** `/workspace/` (maps to `/data/user/0/com.jarves.mh/files/workspaces/`)
* **Current Active Project Workspace:** `/workspace/swift-bose`
* **User Home:** `/root`
* **Android SDK Location:** `/root/android-sdk`
* **Local Maven Repository:** `/root/maven/localMvnRepository`
* **Antigravity CLI Binary:** `/root/.gemini/antigravity-cli/bin/agy` (symlinked into `/usr/local/bin` and user PATH)

---

## 2. Architecture Overview

Mobile Harness employs a four-tier architecture designed to run terminal tools, Linux toolchains, and AI coding agents on non-rooted Android ARM64 devices:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Android Host Application                          │
│   • Kotlin 2.2.21 + Jetpack Compose (Material 3)                      │
│   • MainActivity (Single-Activity UI)                                  │
│   • ViewModels: MainViewModel, LocaleHelper, UI Theme                  │
│   • RuntimeExecutionService & RuntimeSetupService (Foreground Services)│
│   • PackageInstaller API Bridge (ADB-free on-device APK installation)  │
│   • ApiKeyVault (Android KeyStore AES-GCM Encrypted Storage)           │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │ JNI Calls
┌────────────────────────────────────▼───────────────────────────────────┐
│                    Native Bridge & Carrier Layer                       │
│   • libpocketspawn.so: PTY / process execution / signal handling       │
│   • libandroid-shmem.so: SysV IPC emulation over ashmem/memfd          │
│   • libtalloc.so: Hierarchical memory allocator for PRoot              │
│   • libproot.so & libprootloader.so: Custom PRoot binary & loader      │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │ Ptrace Emulation
┌────────────────────────────────────▼───────────────────────────────────┐
│                     Ubuntu 20.04 PRoot Rootfs                          │
│   • Linux userspace: bash, coreutils, git 2.25.1, node 24.19.0         │
│   • Java Temurin JDK 17.0.20.1+1 (/opt/jdk-17)                         │
│   • Gradle 8.14.3 (/opt/gradle/gradle-8.14.3)                          │
│   • Android SDK (API 35 & 36 Platforms, Build-Tools 35.0.0)            │
│   • Local Maven Pre-cache (/root/maven/localMvnRepository)             │
│   • AAPT2 ARM64 Override (/root/android-sdk/build-tools/35.0.0/aapt2)  │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │ stdio stream-json / CLI
┌────────────────────────────────────▼───────────────────────────────────┐
│                       AI Agent Integrations                            │
│   • Google Antigravity CLI (agy) via stream-json protocol              │
│   • Claude Code CLI via terminal/stream bridge                         │
│   • OpenRouter Gateway & Custom OpenAI/Anthropic Compatible APIs       │
│   • DeepSeek DSH Bridge                                                │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.1 UI Layer (Kotlin & Jetpack Compose)
* **Single Activity:** `MainActivity.kt` with `PocketDevApp.kt` declarative navigation.
* **Screens:**
  * `AgentScreen.kt`: Conversational agent interface with streaming text deltas, rich markdown rendering (`MarkdownText.kt`), collapsible tool call accordions, and status banners.
  * `TerminalScreen.kt`: Interactive Linux terminal with virtual keyboard bar (ESC, TAB, CTRL, ALT, cursor keys).
  * `SettingsScreenModern.kt`: AI model selector, provider configuration, stack installations, and diagnostics.
  * `AndroidBuildText.kt`: Dedicated build progress visualization showing real-time Gradle phases, compilation issues, and installation triggers.

### 2.2 Native Execution & JNI Layer
* **`pocket_spawn.c` (`libpocketspawn.so`):**
  * Spawns processes with true pseudo-terminals (`openpty`/PTY).
  * Configures process groups, controlling terminals, and signal forwarding (SIGINT/Ctrl+C, SIGTERM, SIGKILL).
  * Pipes stdin/stdout/stderr directly to file descriptors or log files.
* **AGP Shared Library Trick (`GenerateLoaderInfo.cmake` & `executable_carrier.c`):**
  * Android Gradle Plugin refuses to package executable ELF binaries directly into APK `lib/` directories.
  * Mobile Harness declares `proot-carrier` and `prootloader-carrier` as dummy shared libraries (`.so`), then replaces them post-link with the actual `proot` and `prootloader` ELF executables.
  * Compiles with `-Wl,-z,max-page-size=16384` to guarantee compatibility with Android 15/16 16 KiB page memory systems.

### 2.3 Foreground Services
* **`RuntimeExecutionService`:**
  * Foreground service type: `specialUse` (`android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE` = "User-started local development task").
  * Flags: `stopWithTask = false`, `WAKE_LOCK`.
  * Guarantees that agent reasoning, long-running Gradle compilations, and terminal tasks are not killed by Android's Low Memory Killer (LMK) or background limits when the app is in the background.
* **`RuntimeSetupService`:**
  * Foreground service type: `specialUse` ("User-started installation of the local coding environment").
  * Manages downloading, verifying (SHA-256), and decompressing runtime bundles via `commons-compress` and `zstd-jni`.

### 2.4 PRoot Virtualization Layer
* Uses PRoot v5.1 with extensions (`fake_id0`, `sysvipc`, `ashmem_memfd`, `link2symlink`, `mountinfo`).
* Intercepts `execve`, `ptrace`, and filesystem system calls via `seccomp` and `ptrace` in user space without requiring root privileges.
* Binds the host project directory (`/data/user/0/com.jarves.mh/files/workspaces/<slug>`) directly to `/workspace/<slug>`.

---

## 3. Build Prerequisites

To compile the **Mobile Harness host application** from source, the build system requires:

| Component | Required Version | Status in Current Environment |
|---|---|---|
| **OS Host** | Linux / macOS / Windows with Android Build Support | Android 13 ARM64 inside PRoot |
| **JDK** | Java 17 (Temurin 17.0.20.1+1) | **Installed** (`/opt/jdk-17`) |
| **Gradle** | 8.14 (wrapper) / 8.14.3 | **Installed** (`/opt/gradle/gradle-8.14.3`) |
| **Android Gradle Plugin (AGP)** | 8.13.2 | **Missing** (local cache only has 8.11.0) |
| **Kotlin Gradle Plugin** | 2.2.21 | **Missing** (local cache only has 1.9.22) |
| **Android SDK Platform** | API 36 (`compileSdk = 36`) | **Installed** (`/root/android-sdk/platforms/android-36`) |
| **Android SDK Min/Target** | minSdk 28, targetSdk 36 (Play) / 28 (Direct) | **Supported** |
| **Android Build-Tools** | 35.0.0 | **Installed** (`/root/android-sdk/build-tools/35.0.0`) |
| **Android NDK** | **28.2.13676358** (NDK r28) | **MISSING** (`/root/android-sdk/ndk` does not exist) |
| **CMake** | **3.22.1** (Android SDK CMake) | **Missing SDK CMake** (system has 3.16.3) |
| **Bundled Agent Tarball** | `pocketdev-agy-arm64-2026.09.1.tar.zst` | **MISSING** in `dist/runtime-bundles/` |

---

## 4. Missing or Incompatible Dependencies

An in-depth inspection of the current environment against the upstream repository reveals major blockers preventing the compilation of Mobile Harness *inside* the current PRoot environment:

### 4.1 Missing Android NDK (Blocker #1)
* Mobile Harness requires native compilation of `pocket_spawn.c`, `proot` (50+ C files), `talloc`, `android-shmem`, and `prootloader`.
* `app/build.gradle.kts` enforces:
  ```kotlin
  ndkVersion = providers.gradleProperty("mhNdkVersion").orNull ?: "28.2.13676358"
  ```
* NDK 28 is explicitly required because NDK 26 emits 4 KiB segment headers that crash on modern 16 KiB page Android devices.
* **Current status:** The NDK is not installed anywhere in `/root/android-sdk` or the filesystem.

### 4.2 Missing SDK CMake 3.22.1 (Blocker #2)
* `app/build.gradle.kts` specifies `externalNativeBuild { cmake { version = "3.22.1" } }`.
* The environment only has Ubuntu 20.04's system CMake `3.16.3` (`/usr/bin/cmake`). AGP will fail when attempting to resolve Android SDK CMake `3.22.1`.

### 4.3 Missing AGP 8.13.2 and Kotlin 2.2.21 Artifacts (Blocker #3)
* The pre-cached offline Maven repository (`/root/maven/localMvnRepository`) was provisioned for building standard client apps (e.g. DayMate):
  * Local AGP: `8.11.0` (Mobile Harness requires `8.13.2`)
  * Local Kotlin: `1.9.22` (Mobile Harness requires `2.2.21`)
  * Local Compose BOM: `2022.10.00` (Mobile Harness requires `2025.02.00`)
* Compiling Mobile Harness requires downloading dozens of new Maven artifacts, which would fail in an offline build.

### 4.4 Missing Asset Bundles (Blocker #4)
* Gradle tasks `prepareBundledAgentAssets` and `prepareOfflineRuntimeAssets` expect pre-built rootfs tarballs in `dist/runtime-bundles/`:
  * `pocketdev-agy-arm64-2026.09.1.tar.zst`
  * `pocketdev-core-arm64-2026.09.5.tar.zst`
  * `pocketdev-claude-arm64-2026.09.1.tar.zst`
  * `pocketdev-python-arm64-2026.09.2.tar.zst`
  * `pocketdev-android-arm64-2026.09.1.tar.zst`
  * `pocketdev-dsh-arm64-2026.09.1.tar.zst`
* These archives are not present in the workspace.

### 4.5 AAPT2 Architecture Constraint
* The AAPT2 binary packaged in standard Maven coordinates is compiled for x86_64.
* On ARM64 Linux, AGP must be overridden with the native ARM64 binary using `-Pandroid.aapt2FromMavenOverride=/root/android-sdk/build-tools/35.0.0/aapt2`, which is configured in `/root/.gradle/init.d/pocketdev-android.gradle`.

---

## 5. Feasibility of Compiling Mobile Harness in Current Environment

### 5.1 Verification: Can the Existing Android SDK Compile Mobile Harness?
**Conclusion: NO.** It cannot compile Mobile Harness in this environment.

### 5.2 Technical Failure Points
1. **Compilation Step 1 (Gradle Evaluation):** Fails immediately due to missing AGP 8.13.2 and Kotlin 2.2.21 in the offline repository.
2. **Compilation Step 2 (Asset Sync):** Task `prepareBundledAgentAssets` fails because `dist/runtime-bundles/pocketdev-agy-arm64-2026.09.1.tar.zst` does not exist.
3. **Compilation Step 3 (External Native Build):** Fails because `ndkVersion 28.2.13676358` is not installed, and CMake 3.22.1 is missing.
4. **Hardware Constraint (RAM):** The device has ~1.0 GB of available RAM. Compiling the C++ native sources (over 50 C source files) together with Kotlin compiler daemon and AGP worker threads will trigger out-of-memory (OOM) process termination.
5. **Hardware Constraint (Disk Space):** Disk space is 94% full (7.6 GB free). An NDK 28 installation takes ~3.5 GB uncompressed, Gradle caches take ~1 GB, and build intermediates take ~1.5 GB, risking catastrophic disk exhaustion.

---

## 6. Existing APK Build Procedure

Mobile Harness has two distinct APK build workflows:

### 6.1 Host Mobile Harness Application Build (Workstation / CI)
The repository includes dedicated release scripts intended for host workstations (e.g. macOS / Linux x86_64 or ARM64):
* **Script:** `scripts/build-play-release.sh`
* **Command:**
  ```bash
  ./gradlew \
    -PplayBuild=true \
    -PappVersionCode="8" \
    -PappVersionName="1.0.7" \
    playReadinessCheck testOnlineDebugUnitTest lintOnlineDebug assembleRelease bundleRelease
  ```
* **Output Targets:**
  * Direct Online APK: `app/build/outputs/apk/online/release/app-online-release.apk`
  * Direct Offline APK: `app/build/outputs/apk/offline/release/app-offline-release.apk`
  * Play Store App Bundles: `app/build/outputs/bundle/onlineRelease/app-online-release.aab`
* **Development Install:**
  * Script `scripts/install-phone.sh` compiles and pushes to a device via ADB:
    ```bash
    ./gradlew :app:installOnlineDebug -x lint
    adb shell am start -n com.jarves.mh/.MainActivity --activity-single-top
    ```

### 6.2 On-Device Android Projects Build (Inside PRoot)
When user applications (such as DayMate or SATS Radio Studio) are compiled *inside* Mobile Harness:
* **Command Pattern:**
  ```bash
  timeout 15m gradle -Dorg.gradle.jvmargs= --no-daemon --max-workers=2 \
    --init-script /root/.gradle/init.d/pocketdev-android.gradle \
    -Pandroid.aapt2FromMavenOverride=/root/android-sdk/build-tools/35.0.0/aapt2 \
    assembleDebug --console=plain --stacktrace
  ```
* **Key flags explained:**
  * `--no-daemon`: Prevents orphan Gradle daemons from staying attached to deleted PRoot mounts or consuming background RAM.
  * `--max-workers=2`: Limits parallel execution to avoid exceeding 1 GB free RAM.
  * `-Dorg.gradle.jvmargs=`: Overrides heavy default heap sizes to keep JVM within device memory boundaries.
  * `-Pandroid.aapt2FromMavenOverride`: Forces Gradle to use the native ARM64 `aapt2` binary instead of Maven's x86_64 binary.

### 6.3 On-Device Installation Procedure (No ADB Required)
* Handled by `AndroidAppInstaller.kt`:
  * Direct communication with Android OS via `android.content.pm.PackageInstaller`.
  * Opens an installer session (`installer.createSession(params)`), streams APK bytes into `session.openWrite()`, and commits the session with a PendingIntent callback to `AndroidAppInstallReceiver`.
  * Sets `setRequireUserAction(USER_ACTION_NOT_REQUIRED)` for updates on Android 12+ (API 31+).
  * Uses `FileProvider` (`${applicationId}.files`) fallback for MIUI or restricted devices.
  * **Result:** No ADB daemon or USB debugging connection is needed to install newly compiled APKs.

---

## 7. Estimated Additional Storage Requirements

If Mobile Harness were to be compiled natively on this device, the estimated storage footprint would be:

| Component | Compressed Download | Extracted / Cache Size |
|---|---|---|
| Android NDK r28 (`28.2.13676358`) | ~1.1 GB | ~3.8 GB |
| Android SDK CMake 3.22.1 | ~85 MB | ~210 MB |
| AGP 8.13.2 + Kotlin 2.2.21 + Maven Dependencies | ~350 MB | ~850 MB |
| Runtime Bundles (`dist/runtime-bundles/*.tar.zst`) | ~800 MB | ~800 MB |
| Build Intermediates & C++ Object Files | — | ~1.2 GB |
| **Total Estimated Additional Storage** | **~2.3 GB** | **~6.86 GB** |

> **Risk Assessment:** Available storage is currently **7.6 GB** (94% used). Downloading and building the entire native stack on the device would leave less than **750 MB** of free disk space, causing high risk of filesystem exhaustion, database corruption, or OS instability.

---

## 8. AI Provider Integration Architecture

Mobile Harness connects to multiple AI providers through modular runtime bridges:

```
                      ┌──────────────────────┐
                      │    MainViewModel     │
                      └──────────┬───────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
┌────────▼────────────┐ ┌────────▼───────────┐ ┌──────────▼───────────┐
│AntigravityRuntime-  │ │ClaudeRuntimeBridge │ │ProviderApiClient /   │
│Bridge (Google agy)  │ │(Claude Code CLI)   │ │OpenRouterGateway     │
└────────┬────────────┘ └────────┬───────────┘ └──────────┬───────────┘
         │ stdio stream          │ stdio pty              │ HTTP / REST
┌────────▼────────────┐ ┌────────▼───────────┐ ┌──────────▼───────────┐
│ /root/.gemini/      │ │ /usr/local/bin/    │ │ https://openrouter.ai│
│ antigravity-cli/    │ │ claude             │ │ Custom OpenAI /      │
│ bin/agy             │ │                    │ │ Anthropic Base URLs  │
└─────────────────────┘ └────────────────────┘ └──────────────────────┘
```

### 8.1 Antigravity Integration (`AntigravityRuntimeBridge.kt`)
* **Execution:** Spawns `agy` inside PRoot via `NativeSpawnProcess`.
* **CLI Command Invocation:**
  ```bash
  agy --input-format stream-json --output-format stream-json --print-timeout 2m --dangerously-skip-permissions
  ```
* **Event Protocol:** Bidirectional JSON streaming over stdio:
  * Inbound events: `init` (conversation ID), `step_update` (text deltas, tool executions, outputs), `result` (completion, status, final text).
  * Redaction: Sensitive credentials (tokens, passwords, Authorization headers) are automatically masked before entering the UI event bus.
  * Process lifecycle: Driven by the protocol rather than process termination (`stopAfterTerminalResult`).
* **Authentication (`AntigravityAuthController.kt`):**
  * Spawns an interactive login session with `SSH_CONNECTION="127.0.0.1 1 127.0.0.1 1"` to trigger agy's manual browser OAuth URL and one-time code entry.
  * Tokens are saved inside the guest filesystem at `/root/.gemini/antigravity-cli/antigravity-oauth-token`. Mobile Harness checks credential file presence without inspecting the private key.

### 8.2 Other AI Providers
* **Claude Code (`ClaudeRuntimeBridge.kt`):** Drives Anthropic's `claude` CLI via pseudo-terminal.
* **OpenRouter & Custom APIs (`ProviderApiClient.kt` & `OpenRouterRoutingGateway.kt`):** Direct HTTP API requests for OpenAI-compatible and Anthropic-compatible endpoints.
* **Credential Security (`ApiKeyVault.kt`):**
  * Employs the Android KeyStore provider (`AndroidKeyStore`) with AES-256 GCM encryption (`KeyProperties.KEY_ALGORITHM_AES`, `BLOCK_MODE_GCM`).
  * Stores encrypted ciphertexts in private SharedPreferences (`pocket_secrets`).
  * Keys never enter plain text files in the project workspace or Git history.

---

## 9. Existing Project and Workspace Storage Locations

### 9.1 Filesystem Mapping
| PRoot Container Path | Host Android Path | Purpose |
|---|---|---|
| `/workspace/` | `/data/user/0/com.jarves.mh/files/workspaces/` | User projects root directory |
| `/workspace/swift-bose/` | `.../files/workspaces/swift-bose/` | Current active workspace |
| `/workspace/daymate/` | `.../files/workspaces/daymate/` | DayMate project |
| `/workspace/sats-radio-studio/` | `.../files/workspaces/sats-radio-studio/` | SATS Radio Studio project |
| `/workspace/antigravity-auth/` | `.../files/workspaces/antigravity-auth/` | Temporary sandbox for Antigravity OAuth |
| `/workspace/antigravity-models/` | `.../files/workspaces/antigravity-models/` | Model verification sandbox |
| `/workspace/github-auth/` | `.../files/workspaces/github-auth/` | GitHub authentication sandbox |
| `/root/` | `/data/user/0/com.jarves.mh/files/runtime/ubuntu/root/` | Container root user home |
| `/root/.gradle/` | `.../runtime/ubuntu/root/.gradle/` | Gradle daemon, caches, init scripts |
| `/root/android-sdk/` | `.../runtime/ubuntu/root/android-sdk/` | Android SDK platforms and build-tools |
| `/root/maven/localMvnRepository/` | `.../runtime/ubuntu/root/maven/localMvnRepository/`| Pre-cached offline dependencies |
| `/root/.gemini/antigravity-cli/` | `.../runtime/ubuntu/root/.gemini/antigravity-cli/` | Antigravity logs, state, conversation transcripts |

---

## 10. Licensing Requirements and Redistribution Considerations

| Component | License | Redistribution & Compliance Requirements |
|---|---|---|
| **Mobile Harness** | MIT License | Permissive. Requires copyright notice preservation. Can be forked, rebranded, and commercially distributed. |
| **PRoot** (`third_party/proot`) | **GPL v2.0** | **Copyleft.** Distribution of compiled `proot` and `prootloader` binaries requires providing corresponding complete source code of PRoot and any applied patches. |
| **Talloc** (`third_party/talloc`) | **LGPL v3.0+** | Dynamically linked (`libtalloc.so`). Permitted under LGPL conditions provided source is available upon request. |
| **libandroid-shmem** (`third_party/libandroid-shmem`)| **BSD 3-Clause** | Permissive. Retain BSD 3-Clause license notice. |
| **Ubuntu Rootfs** | Various (GPL, LGPL, Apache, BSD) | Ubuntu base packages are subject to Canonical's distribution policies and individual package licenses. |
| **Antigravity CLI** | Proprietary Google Toolchain | Distributed via official packaging; runtime bundles extract official binaries. Custom forks should not re-host or alter proprietary binaries. |
| **Claude Code** | Anthropic Terms | Distributed as `@anthropic-ai/claude-code` npm package. |

> [!IMPORTANT]
> Because Mobile Harness incorporates PRoot (GPL v2.0) and Talloc (LGPL v3.0), any public distribution of SATS Mobile Studio APKs bundling these native binaries must maintain an open-source repository containing the exact native source code and build recipes under GPL v2.0 compliance.

---

## 11. Recommended Safest Path to Create SATS Mobile Studio

Based on the audit findings, attempting to compile the host application inside this on-device container is neither technically possible (missing NDK 28, missing CMake 3.22.1, missing AGP 8.13.2) nor operationally safe (1 GB RAM, 7.6 GB disk).

The recommended, robust strategy consists of **two parallel tracks**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Track A: Host Development                       │
│                       (Workstation / CI Pipeline)                      │
│                                                                        │
│   1. Fork `techjarves/Mobile-Harness` -> `SATS-Mobile-Studio`.        │
│   2. Rebrand UI: Theme, app names, icons, splash screen, docs.        │
│   3. Refactor package namespace: `com.sats.mobilestudio`.             │
│   4. Compile APK & AAB releases on workstation with NDK 28 + AGP 8.13.2│
│   5. Publish release APK for deployment to mobile devices.            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Install Release APK on Device
┌───────────────────────────────────▼────────────────────────────────────┐
│                       Track B: On-Device Studio                        │
│                  (Inside Current PRoot Environment)                    │
│                                                                        │
│   1. Continue using current PRoot container for developing Android     │
│      client applications (DayMate, SATS Radio Studio, etc.).           │
│   2. Develop and verify SATS custom agent skills, rules, and scripts.  │
│   3. Author architectural specs, UI mockups, and feature definitions.  │
│   4. Maintain strict adherence to local memory (--max-workers=2) and   │
│      offline build paths.                                              │
└────────────────────────────────────────────────────────────────────────┘
```

### 11.1 Key Architectural Enhancements for SATS Mobile Studio
When forking Mobile Harness to create SATS Mobile Studio:
1. **Dedicated SATS Agent Preset:** Pre-configure Antigravity and Claude prompts with SATS design guidelines and memory-conscious Gradle compilation parameters.
2. **Modernized UI Theme:** Apply SATS brand aesthetics (color palette, typography, custom icons, dark mode refinement) in Jetpack Compose.
3. **Optimized Local Dependency Bundling:** Include AGP 8.11+ and Kotlin 1.9.22/2.0+ pre-cached artifacts in the runtime bundle to allow offline compilation of modern Compose client projects without requiring internet access.
4. **Enhanced Diagnostics:** Add active memory and storage gauges directly into the IDE status bar so users can monitor RAM and disk space before starting heavy compilation tasks.

---

## 12. Clear Recommendation for Phase 1

### Phase 1 Scope & Action Items
1. **Repository Setup:**
   * Fork `https://github.com/techjarves/Mobile-Harness` into the official SATS organization repository (e.g., `sats-core/sats-mobile-studio`).
   * Preserve upstream commit history and license headers.
2. **Branding & Package Strategy:**
   * Prepare package rename specification from `com.jarves.mh` to `com.sats.mobilestudio`.
   * Update application strings, icons (`app/src/main/res/mipmap-*`), and notification channels.
3. **CI/CD Pipeline Setup:**
   * Establish a GitHub Actions workflow targeting Android NDK 28.2.13676358, CMake 3.22.1, and Java 17 to build `SATS Mobile Studio` debug and release APKs automatically.
4. **On-Device Development Rule Enforcement:**
   * Keep the active on-device workspace (`/workspace/swift-bose`) dedicated to application-layer projects, agent skill development, and architectural verification.
   * Do not attempt local NDK installation on this Android device.

---
*Report compiled and verified in `/workspace/swift-bose/SATS_MOBILE_STUDIO_PHASE_0_AUDIT.md`.*

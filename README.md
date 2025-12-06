# 🛡️ FridaDetector-Demo

[![Android](https://img.shields.io/badge/Platform-Android-3ddc84?style=for-the-badge&logo=android&logoColor=white)](https://developer.android.com)
[![C++](https://img.shields.io/badge/Native-C%2B%2B17-00599c?style=for-the-badge&logo=cplusplus&logoColor=white)](https://isocpp.org/)
[![License](https://img.shields.io/badge/License-Research-yellow?style=for-the-badge)](LICENSE)
[![Security](https://img.shields.io/badge/Focus-RASP-red?style=for-the-badge&logo=security&logoColor=white)]()

> **Runtime Application Self-Protection against Dynamic Instrumentation**

A proof-of-concept demonstrating advanced runtime protection techniques for Android applications against dynamic analysis and instrumentation frameworks.

---

## 📋 Overview

AntiFrida RASP implements a multi-layered defense strategy that operates at the native level, utilizing **direct system calls** to bypass standard library hooks. The detection engine employs multiple orthogonal vectors to identify instrumentation attempts, making evasion significantly more difficult.

**Key Innovations:**
- 🔐 **Syscall-level I/O** — Bypasses libc hooks using raw `SVC #0` (ARM64), `int $0x80` (x86)
- 🔍 **RAM vs Disk Integrity** — Compares function preambles in memory against original ELF on disk
- 🧠 **Heuristic Pattern Analysis** — Detects GLib/V8/QuickJS signatures in anonymous memory
- ⚡ **Immediate Response** — Option to terminate before detection results reach managed code

---

## ✨ Features

### 🔬 Advanced Memory Forensics
Scans process memory (`/proc/self/maps`) for suspicious artifacts using **direct syscalls** to bypass hooked `open()`/`read()` functions. Detects:
- Anonymous executable regions (`r-xp` with inode 0)
- ELF headers in non-file-backed memory
- GLib type system markers (`GObject`, `GMainLoop`, `GThread`)
- JavaScript engine signatures (V8, QuickJS)
- Fake JIT cache regions (`[anon:jit-cache]` without `dalvik` prefix)
- Deleted file mappings (`(deleted)`)
- `memfd:` backed executables

### 🔐 Native Integrity Verification
Validates critical system library functions by comparing **RAM bytes vs disk bytes**:
- Reads libc.so from disk using raw syscalls
- Compares function preambles (`open`, `read`, `mmap`, `dlsym`, etc.) against file contents
- Detects inline hooks and trampolines without relying on signature patterns
- ELF program header parsing for accurate file offset calculation

### 🌐 Heuristic Port & Communication Scanning
Detects instrumentation servers through active port analysis:
- Scans `/proc/net/tcp` and `/proc/net/tcp6` for listening ports
- Detects default ports (27042, 27043) via hash matching
- Detects patched/custom ports (28042, 28052)
- Uses FNV-1a hash comparison to avoid plaintext signatures

### 🧵 Thread Behavioral Analysis
Monitors for suspicious background threads spawned by injection agents:
- Scans `/proc/self/task/*/comm` for each thread
- Detects characteristic thread names (`gmain`, `gum-js-loop`, `pool-*`, etc.) via hash comparison
- Uses directory enumeration with `getdents64` syscall

### 🔬 Inline Hook Detection
Analyzes function preambles for trampoline patterns:
- **ARM64**: LDR+BR sequences, suspicious B (branch) instructions, BRK patterns
- **ARM32**: LDR PC instructions, BX register patterns, BKPT instructions
- **x86/x64**: JMP rel32, JMP [mem] patterns
- Checks critical libc functions for modifications

### 📊 Statistical Timing Analysis
Detects instrumentation overhead through execution timing:
- Rolling window of timing samples (16 measurements)
- Median Absolute Deviation (MAD) based anomaly detection
- CPU cycle counter based measurement (ARM64: `CNTVCT_EL0`)
- Adaptive threshold: `median + 3σ`

### 🎯 GOT/PLT Hook Detection
Validates Global Offset Table entries:
- Parses `.dynamic` section from ELF headers
- Checks if GOT entries point to suspicious memory regions
- Flags anonymous executable targets

### 🔍 Proc-Mem Injection Detection
Counters sophisticated injection techniques that bypass ptrace:
- Scans `/proc/self/fd` for suspicious file descriptors
- Detects RWX memory regions (shellcode indicators)
- Monitors for libraries loaded from `/data/local/tmp`
- Checks for timezone variable manipulation (injection signaling)

### 💀 Active Countermeasures
Features a **"Delayed Lockdown"** mechanism with multiple termination strategies:
- `SIGILL` — Illegal instruction (inline execution, cannot be caught)
- `SIGKILL` — Direct syscall (bypasses signal handlers)
- Stack corruption — Overwrites return address
- **Disguised crashes** — Appear as normal bugs (null deref, div-by-zero, double-free)
- Random selection with random delay to frustrate analysis

---

## 🎯 Detection Coverage

| Variant | Detection Method |
|---------|-----------------|
| **Frida Server** | Port scan, thread analysis, memory patterns |
| **Frida Gadget** | Memory forensics, ELF in anonymous regions, GLib patterns |
| **Injected Agents** | Anonymous exec memory, inline hooks, RAM vs disk comparison |
| **StrongR-Frida** | GLib + JS engine heuristics, fake JIT-cache detection |
| **Frida-Patchs** | Custom port detection, memfd detection |
| **Linjector** | RWX memory, `/data/local/tmp` libraries, malloc hook check |

---

## 🔧 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kotlin UI Layer                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ MainActivity │  │ RaspManager │  │ Detection Callbacks      │ │
│  └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘ │
├─────────┴────────────────┴──────────────────────┴───────────────┤
│                         JNI Bridge                               │
├─────────────────────────────────────────────────────────────────┤
│                     Native RASP Engine (C++)                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                      lib file                               ││
│  │  - Memory Maps Scan      - Inline Hook Detection            ││
│  │  - Tracer PID Check      - Libc Integrity Check             ││
│  │  - Thread Analysis       - GOT/PLT Validation               ││
│  │  - Port Scanning         - Statistical Timing               ││
│  │  - Anonymous Exec Scan   - Linjector Detection              ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐  │
│  │syscall_utils.h │  │ kill_methods.h │  │string_obfuscator │  │
│  │ Direct SVC #0  │  │ Multi-strategy │  │  OBFUSCATE()     │  │
│  └────────────────┘  └────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ Direct Syscalls
┌─────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                              │
│  /proc/self/maps  │  /proc/self/status  │  /proc/net/tcp        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Installation

### Prerequisites
- Android device/emulator (API 21+)
- ADB installed and configured

### Install APK
```bash
# Download the latest release APK
adb install antifrida-debug.apk

```

---

## 🎮 Usage

### Operation Modes

| Mode | Behavior |
|------|----------|
| 🔵 **DETECTOR** | Monitors and reports threats; application continues running |
| 🔴 **ENFORCER** | Actively protects; terminates immediately upon detection |

Toggle modes via the on-screen button.

### Threat Levels

| Level | Detection Count | Response |
|-------|-----------------|----------|
| ✅ SAFE | 0 | Clean environment |
| 🟡 MEDIUM | 1 | Single indicator detected |
| 🟠 HIGH | 2 | Multiple indicators detected |
| 🔴 CRITICAL | 3+ | Active instrumentation confirmed |

---

## 🔒 Security Considerations

### Why Direct Syscalls?
Standard library functions (`open`, `read`, `fopen`) can be hooked by instrumentation frameworks. This project bypasses libc entirely using:

```cpp
// ARM64 direct syscall example
__asm__ volatile("svc #0" : "=r"(x0) : "r"(x8), "r"(x1) : "memory");
```

### Why Hash-Based Detection?
Plaintext strings like `"frida"` are trivially patched. This project uses **compile-time FNV-1a hashes**:

```cpp
constexpr uint32_t FRIDA_AGENT = HASH("frida-agent");  // Resolved at compile time
```

### Why RAM vs Disk Comparison?
Inline hooking **must** modify function preambles in memory. By reading the original bytes from disk using direct syscalls, we can detect any modification regardless of the hook implementation.

---

## ⚠️ Disclaimer

> **This project is provided for educational and research purposes only.**
>
> It is intended to demonstrate runtime application self-protection techniques and should be used responsibly. The authors are not responsible for any misuse of this software.
>
> - Do not use this in production applications without proper legal review
> - Testing should only be performed on applications you own or have permission to analyze
> - This software is provided "as-is" without warranty of any kind

---

## 📚 Technical References

- [Frida Dynamic Instrumentation Toolkit](https://frida.re/)
- [ARM64 SVC Instruction Reference](https://developer.arm.com/documentation/ddi0596/latest/)
- [ELF Format Specification](https://refspecs.linuxfoundation.org/elf/elf.pdf)
- [Linux /proc Filesystem](https://www.kernel.org/doc/Documentation/filesystems/proc.txt)

---

## 📄 License

This project is licensed for research and educational purposes. See [LICENSE](LICENSE) for details.

---

<p align="center">
  <b>Built with 🔐 for the security research community</b>
</p>

# Remote Attestation on NVIDIA Jetson AGX Orin using OP-TEE and IMA

> **Platform:** NVIDIA Jetson AGX Orin Developer Kit  
> **Kernel:** 5.15.148 (custom build with IMA enabled)  
> **JetPack:** R36.4.4  
> **TEE:** OP-TEE 4.2  
> **TPM:** Microsoft fTPM (firmware TPM inside OP-TEE)

---

## Overview

This repository documents the design, implementation, and findings of a
remote attestation system on NVIDIA Jetson AGX Orin. The system allows a
remote verifier to request cryptographic proof of the device's software
state — what kernel is running, which modules are loaded, and whether the
platform is correctly configured.

The work is built on three components:

- **IMA (Integrity Measurement Architecture)** — Linux kernel subsystem
  that measures every binary, module, and firmware blob at runtime
- **fTPM (firmware TPM)** — Microsoft's TPM 2.0 implementation running
  as a Trusted Application inside OP-TEE
- **OP-TEE** — ARM TrustZone-based Trusted Execution Environment that
  holds the attestation signing key and signs evidence

---

## Key Finding — fTPM + IMA Initialization Ordering Problem

During implementation, a fundamental architectural conflict was discovered
between IMA and fTPM on Jetson AGX Orin.

### The Problem

```
[1.806s] ima: No TPM chip found, activating TPM-bypass!
[3.764s] optee: probing for conduit method
[3.824s] optee: initialized driver
```

IMA initializes at **1.806 seconds** and immediately looks for a TPM.
OP-TEE (which hosts the fTPM) does not initialize until **3.824 seconds**.
By the time fTPM becomes accessible, IMA has already given up and activated
bypass mode. As a result, IMA measurements are not extended into TPM PCR[10].

### Why This Happens

On Jetson AGX Orin, there is no physical TPM chip. Instead, TPM 2.0 is
implemented as Microsoft's fTPM Trusted Application running inside OP-TEE.

```
Physical TPM (other platforms):
  Separate chip, powered independently
  Ready at power-on → IMA always finds it ✅

fTPM on Jetson (this platform):
  Software TA inside OP-TEE
  OP-TEE depends on TrustZone + TEE bus init
  TEE bus probes AFTER kernel security subsystems
  IMA initializes BEFORE TEE bus is ready ❌
```

This is a known problem across multiple ARM platforms. See related issues:

- [OP-TEE issue #7248](https://github.com/OP-TEE/optee_os/issues/7248) — fTPM + IMA on Xilinx ZCU104
- [Raspberry Pi Linux issue #3291](https://github.com/raspberrypi/linux/issues/3291) — IMA/TPM load order broken
- [Cybersecurity-LINKS patch](https://github.com/Cybersecurity-LINKS/tpm-ima-patch) — Fix for RPi 5.15.y (merged into RPi kernel, not mainline)
- [LKML patch 2021](https://lkml.rescloud.iu.edu/2101.1/10112.html) — fTPM: make sure TEE is initialized before fTPM
- [TI TDA4VM forum](https://e2e.ti.com/support/processors-group/processors/f/processors-forum/1375425/tda4vm-ima-vs-tpm-builtin-driver-boot-order) — Same issue on TI platform

**This repository is the first documented case of this problem on
NVIDIA Jetson AGX Orin with JetPack R36.4.4.**

---

## Platform Details

| Property | Value |
|---|---|
| Device | NVIDIA Jetson AGX Orin Developer Kit |
| JetPack | R36.4.4 |
| Kernel | 5.15.148-tegra (OOT variant) |
| OP-TEE | 4.2 (d78bc5fa) |
| TPM type | firmware TPM (microsoft,ftpm) |
| TPM driver | tpm_ftpm_tee |
| Storage | eMMC 59.3G (mmcblk0) |
| RAM | 64GB |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Jetson AGX Orin                          │
│                                                             │
│  Normal World                  Secure World (OP-TEE)        │
│  ──────────────                ─────────────────────        │
│                                                             │
│  Linux kernel                  fTPM TA                      │
│    │                             │                          │
│    IMA subsystem                 PCR[0..23] variables        │
│    measures files                                           │
│    │                           Attestation TA (planned)     │
│    Normal World Agent            │                          │
│    │ (dumb pipe)                 reads PCR[0-7]             │
│    │                             reads IMA log hash         │
│    sends nonce ────────────────► reads ECID + fuse state    │
│                                  signs with private key      │
│    receives token ◄──────────────│                          │
│    │                                                        │
│    forwards to verifier                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## What Is Attested

### Boot Chain — PCR[0-7] (firmware measured, stored in fTPM)

EDK2 UEFI firmware measures the boot chain before the kernel starts
and extends values into fTPM PCRs:

```
PCR[0] = 0x3D458CFE...  ← UEFI firmware hash
PCR[1] = 0x22DC145D...  ← UEFI configuration
PCR[4] = 0xA8147957...  ← bootloader hash
PCR[7] = 0x65CAF8DD...  ← secure boot state
```

These are hardware-enforced. The kernel cannot modify them.

### Runtime — IMA Measurement Log (kernel space)

IMA measures every binary and module as it executes:

```
10  0adefe76...  ima-ng  sha256:000...000  boot_aggregate
10  3d55998e...  ima-ng  sha256:29a31aa3...  /init
10  65566a30...  ima-ng  sha256:c5f8a98c...  /usr/bin/bash
10  8fe75730...  ima-ng  sha256:d93175bf...  /usr/lib/ld-linux-aarch64.so.1
...
```

**6000+ measurements** recorded per boot. Userspace cannot fake these
entries — IMA runs in kernel space.

### Device Identity — Hardware registers (TEE reads directly)

- ECID (Electronic Chip ID) — unique per device, from Security Engine
- Fuse state — ODM production mode, secure boot configuration

---

## Current Status

| Component | Status | Notes |
|---|---|---|
| IMA enabled | ✅ Working | 6000+ measurements per boot |
| IMA policy (tcb) | ✅ Active | Measures binaries, modules, firmware |
| fTPM accessible | ✅ Working | /dev/tpm0, tpm2_pcrread works |
| PCR[0-7] | ✅ Non-zero | Firmware-extended boot measurements |
| PCR[10] extension | ❌ Not working | fTPM/IMA timing ordering problem |
| OP-TEE TA | 🔄 In progress | Attestation TA being built |
| Normal World agent | 🔄 Planned | |
| Verifier | 🔄 Planned | |

---

## Enabling IMA — Build Steps

### Prerequisites

```bash
# verify IMA is not enabled on your Jetson
grep "CONFIG_IMA" /proc/config.gz 2>/dev/null || \
zcat /proc/config.gz | grep "CONFIG_IMA"
# should show: # CONFIG_IMA is not set
```

### Step 1 — Extract kernel source

```bash
mkdir -p ~/kernel_build
cd ~/kernel_build
tar -xjf ~/Linux_for_Tegra/source/kernel_src.tbz2
tar -xjf ~/Linux_for_Tegra/source/kernel_oot_modules_src.tbz2
```

### Step 2 — Get current running config as base

```bash
cd ~/kernel_build/kernel/kernel-jammy-src/
zcat /proc/config.gz > .config
```

### Step 3 — Enable IMA options

```bash
./scripts/config --enable CONFIG_INTEGRITY
./scripts/config --enable CONFIG_INTEGRITY_SIGNATURE
./scripts/config --enable CONFIG_INTEGRITY_ASYMMETRIC_KEYS
./scripts/config --enable CONFIG_IMA
./scripts/config --set-val CONFIG_IMA_MEASURE_PCR_IDX 10
./scripts/config --enable CONFIG_IMA_NG_TEMPLATE
./scripts/config --set-str CONFIG_IMA_DEFAULT_HASH "sha256"
./scripts/config --enable CONFIG_IMA_DEFAULT_HASH_SHA256
./scripts/config --enable CONFIG_IMA_WRITE_POLICY
./scripts/config --enable CONFIG_IMA_READ_POLICY
./scripts/config --disable CONFIG_IMA_APPRAISE
./scripts/config --enable CONFIG_TCG_FTPM_TEE
make olddefconfig
```

### Step 4 — Build kernel

```bash
export ARCH=arm64
make -j$(nproc) Image modules dtbs 2>&1 | tee ~/kernel_build/build.log
```

### Step 5 — Install

```bash
sudo make modules_install
sudo cp /boot/Image /boot/Image.backup.original
sudo cp arch/arm64/boot/Image /boot/Image
```

### Step 6 — Enable IMA via kernel cmdline

```bash
sudo vim /boot/extlinux/extlinux.conf
# Add to APPEND line:
# ima_policy=tcb ima_hash=sha256 ima_template=ima-ng
```

### Step 7 — Remove duplicate fTPM module

```bash
# required because CONFIG_TCG_FTPM_TEE=y (built-in)
# conflicts with old .ko file from modules_install
sudo rm /lib/modules/5.15.148/kernel/drivers/char/tpm/tpm_ftpm_tee.ko
sudo depmod 5.15.148
sudo reboot
```

### Verify IMA is Working

```bash
# IMA filesystem present
ls /sys/kernel/security/ima/

# measurement count (should be > 0)
sudo cat /sys/kernel/security/ima/runtime_measurements_count

# first few measurements
sudo head -5 /sys/kernel/security/ima/ascii_runtime_measurements

# TPM PCR values
sudo tpm2_pcrread sha256:0,1,2,3,4,5,6,7,8,9,10
```

---

## The fTPM + IMA Timing Problem — Deep Analysis

### Boot Timeline on Jetson AGX Orin

```
0.000s  Kernel starts
0.261s  device-mapper: IMA_DISABLE_HTABLE disabled
1.806s  ima: No TPM chip found, activating TPM-bypass!  ← IMA gives up
1.806s  ima: Allocated hash algorithm: sha256
3.764s  optee: probing for conduit method               ← OP-TEE starts
3.824s  optee: initialized driver                       ← OP-TEE ready
        fTPM TA becomes accessible (after OP-TEE)
        IMA already in bypass mode — too late
```

### Root Cause

```
IMA is a security subsystem — initializes as late_initcall
fTPM is a TEE bus device — probes after TEE bus init
TEE bus init happens after PCI/USB/platform buses
Platform buses init after security subsystems

Chain:
  security_init (very early)
    └── IMA init → looks for TPM → not found
  bus init (later)
    └── TEE bus → OP-TEE driver → fTPM probe
        → TPM available — but IMA already gave up
```

### Why Setting CONFIG_TCG_FTPM_TEE=y Does Not Help

Even with fTPM built-in (not a module), the device probe is still
deferred because it depends on the TEE bus, which cannot initialize
before OP-TEE, which cannot start before TrustZone is configured by
TF-A. This chain is hardware-determined and cannot be shortcut by
changing module vs built-in status alone.

### What PCR[10] = 0 Means for Security

Without PCR[10] extension:

- IMA log exists and is correct ✅
- IMA log cannot be proven complete via hardware ⚠️
- A kernel-level attacker could modify the IMA log without TPM detecting it
- Userspace attackers still cannot fake IMA entries (kernel space protection)

This is documented as a research limitation, consistent with standard
attestation literature that assumes an uncompromised kernel.

---

## Attestation Token Design

```json
{
  "device_identity": {
    "ecid": "<hardware chip ID>",
    "fuse_state": {
      "odm_production": "true/false",
      "secure_boot": "true/false"
    }
  },
  "boot_measurements": {
    "pcr0": "0x3D458CFE...",
    "pcr4": "0xA8147957...",
    "pcr7": "0x65CAF8DD...",
    "source": "EDK2-firmware-extended-fTPM"
  },
  "runtime_measurements": {
    "ima_measurement_count": 6024,
    "ima_log_aggregate": "<sha256 of IMA log>",
    "ima_policy": "tcb",
    "source": "IMA-kernel-space"
  },
  "nonce": "<verifier-supplied>",
  "timestamp": "<ISO8601>",
  "signature": "<ECDSA-P256 signed by OP-TEE TA private key>"
}
```

---

## Related Work and References

### Same Problem on Other Platforms

| Platform | Kernel | Status |
|---|---|---|
| Raspberry Pi | 5.15.y | Fixed — patch merged into RPi kernel |
| Xilinx ZCU104 | 5.x | Open — OP-TEE issue #7248 |
| TI TDA4VM | 6.1.80 | Open — TI E2E forum |
| NVIDIA Jetson AGX Orin | 5.15.148 | **This work** |

### Key References

- Sailer et al., "Design and Implementation of a TCG-based Integrity
  Measurement Architecture," USENIX Security 2004 — original IMA paper
- Microsoft fTPM research paper:
  https://www.microsoft.com/en-us/research/wp-content/uploads/2017/06/ftpm1.pdf
- Linux kernel IMA documentation:
  https://www.kernel.org/doc/html/latest/security/IMA-templates.html
- Linux kernel fTPM documentation:
  https://docs.kernel.org/6.3/security/tpm/tpm_ftpm_tee.html
- IETF RATS (Remote ATtestation procedureS): RFC 9334
- ARM PSA Attestation API specification

---

## Repository Structure

```
jetson-ima-attestation/
├── README.md                    ← this file
├── docs/
│   ├── problem-analysis.md      ← deep dive on fTPM+IMA timing
│   ├── threat-model.md          ← security analysis
│   └── architecture.md          ← system design
├── evidence/
│   ├── dmesg_boot.txt           ← full boot log
│   ├── ima_measurements.txt     ← IMA log sample
│   ├── pcr_values.txt           ← TPM PCR readings
│   ├── timing_evidence.txt      ← critical timing output
│   ├── kernel_config_ima.txt    ← IMA kernel config
│   └── ima_policy.txt           ← active IMA policy
├── kernel/
│   └── ima-build-steps.md       ← reproducible build guide
├── ta/                          ← OP-TEE Attestation TA (planned)
├── host/                        ← Normal World agent (planned)
└── verifier/                    ← Remote verifier (planned)
```

---

## Contributing / Related Issues

If you are facing the same fTPM + IMA timing problem on another ARM
platform, please open an issue or reference this repository.

The upstream fix discussion is tracked at:
- linux-integrity mailing list: linux-integrity@vger.kernel.org
- OP-TEE GitHub: https://github.com/OP-TEE/optee_os/issues/7248

---

## License

MIT License — see LICENSE file.

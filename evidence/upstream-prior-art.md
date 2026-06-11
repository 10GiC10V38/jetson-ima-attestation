# Upstream Prior Art — The fTPM + IMA / tee-supplicant Limitation

**This is not a Jetson-specific bug, and it is not a novel discovery.**
It is a known OP-TEE architectural limitation: when the fTPM relies on
RPMB secure storage mediated by the userspace `tee-supplicant` daemon,
the fTPM is unavailable during kernel init — exactly when Linux IMA
performs its one-shot TPM lookup. As a result, IMA never extends PCR[10].

The same root cause has been reported independently on multiple ARM
platforms by different people, starting in 2022. This file consolidates
that prior art accurately. The contribution of *this* repository is a
detailed, evidence-backed characterization of how the limitation
manifests specifically on **NVIDIA Jetson AGX Orin (JetPack R36.4.4)**,
including the platform-specific load mechanism (module denylist +
`nv-tee-supplicant.service`), plus an evaluation of the available
workarounds.

---

## 1. Original report — OP-TEE mailing list (July 2022)

Thread: *"Current design of tee-supplicant can't support ftpm requests
during kernel space bootup"*, OP-TEE mailing list, 20 July 2022.

Link:
https://lists.trustedfirmware.org/archives/list/op-tee@lists.trustedfirmware.org/thread/CUP5PF37EYVAOFUYQF6AUWXCX5HCRZIX/

The original poster (Judy Wang) described the exact problem this
repository encountered:

> "It has to collect data when kernel space is booting up, so we cannot
> delay these requests further until user space is up ... tee-supplicant
> context is not yet initialized, which results in IMA detection of TPM
> devices failed."

OP-TEE maintainers (Jens Wiklander, Jérôme Forissier, Sumit Garg)
participated in the thread.

**Note on attribution:** the poster's affiliation is not stated on the
public archive page, and the message headers use a Microsoft Outlook /
Exchange domain. This file therefore does **not** attribute the report
to any specific company. The point that matters is that the problem was
raised upstream, in the OP-TEE community, in 2022.

---

## 2. Other attestation project — OP-TEE issue #5766 (2023)

Link: https://github.com/OP-TEE/optee_os/issues/5766

A developer attempting essentially the same thing as this project —
IMA-based remote attestation using the Microsoft fTPM TA over OP-TEE —
hit the same tee-supplicant / RPMB / initramfs ordering wall on
**RockPi4B** and **STM32MP157C-DK2**. Their stated goal:

> "to have a functional fTPM early on boot for IMA module to take
> advantage or even for a measured boot procedure leveraging PCRs and
> Event log."

Their workaround direction: fTPM as an **Early TA**, `tee-supplicant`
started from the **initramfs**, and OP-TEE configured RPMB-only
(`CFG_REE_FS=n CFG_RPMB_FS=y`) so the fTPM TA can reach secure storage
before the root filesystem mounts.

---

## 3. Same problem on other platforms

| Platform        | Source                                            |
| --------------- | ------------------------------------------------- |
| RockPi4B        | OP-TEE issue #5766                                 |
| STM32MP157C-DK2 | OP-TEE issue #5766                                 |
| Xilinx ZCU104   | OP-TEE issue #7248                                 |
| BlueField-3 DPU | NVIDIA networking docs (OP-TEE/fTPM, RPMB-gated)   |
| Jetson AGX Orin | This repository                                    |

The recurrence across unrelated SoCs and vendors confirms the cause is
the OP-TEE + fTPM + RPMB + userspace-`tee-supplicant` architecture, not
any single platform.

---

## 4. A fix landed in mainline Linux 6.12 (2024) — but not on this kernel

This limitation has been **addressed upstream**. The **RPMB subsystem**
(`drivers/misc/rpmb`) was merged into **mainline Linux 6.12 (2024)**, with
the **OP-TEE driver as its first user** (Jens Wiklander / Linaro). It lets the
kernel access the eMMC RPMB partition **directly via the MMC subsystem**,
removing the dependency on the userspace `tee-supplicant`. As Linaro put it:
*"the fTPM now works whether it's compiled into the kernel, or loaded as a
module, and thereby the userspace dependencies have been removed"* — so on a
kernel with this enabled, the fTPM is available during kernel init and IMA can
extend PCR[10].

- Linaro write-up: https://www.linaro.org/blog/linaro-enables-op-tee-rpmb-access-directly-from-the-linux-kernel/
- Phoronix (Linux 6.12 merge): https://www.phoronix.com/news/Linux-6.12-RPMB-MMC
- RPMB subsystem series (v7, incl. `optee: probe RPMB device using RPMB subsystem`):
  https://patchwork.kernel.org/project/linux-mmc/cover/20240527121340.3931987-1-jens.wiklander@linaro.org/
- linux-integrity discussion:
  https://www.mail-archive.com/linux-integrity@vger.kernel.org/msg01749.html

**Why it does not help *this* device (yet):** this Jetson runs **kernel
5.15.148 (JetPack R36.4.4)**, which predates 6.12 and does not have the RPMB
subsystem or the OP-TEE integration. The limitation documented here is therefore
real and current on this platform. The fix applies only once a kernel ≥6.12 with
the RPMB subsystem + OP-TEE RPMB integration (and it enabled) runs on the device
— which would require NVIDIA to ship or backport it in JetPack. Evaluating that
on Jetson AGX Orin is future work for this project.

---

## 5. NVIDIA's official documentation — the mechanism, not the IMA caveat

NVIDIA's Jetson Linux Developer Guide (Firmware TPM) documents the
measured-boot model and the userspace fTPM bring-up:

https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/Security/FirmwareTPM.html

> "Before the fTPM is ready, the measurements can be stored in the format
> of TPM event log ... After fTPM brings up, all the measurements should
> go through the TPM2 PCR extend command only."

The documentation describes the userspace `modprobe tpm_ftpm_tee` step,
the EKB provisioning flow, and the "fTPM not ready early" window. It
does **not** mention Linux IMA, PCR[10], or the runtime-attestation
consequence. The design is documented; the IMA limitation is not
surfaced as a product caveat.

---

## Takeaway

A fundamental OP-TEE constraint — fTPM secure storage requires the
userspace `tee-supplicant`/RPMB path — collides with IMA's need for a
TPM during kernel init. Reported upstream since 2022, seen across
multiple platforms, and fixed in mainline Linux 6.12 (2024) by the kernel
RPMB subsystem — but that fix is newer than the 5.15 kernel this Jetson
(JetPack R36.4.4) runs, so the limitation still applies on this platform.

Given this constraint, this project does not rely on PCR[10]. Instead it
signs the IMA log aggregate with a hardware-rooted key held in a custom
OP-TEE Trusted Application, bound to a verifier nonce, under the standard
uncompromised-kernel threat model.

# Root-Cause Analysis — Why IMA Never Connects to the fTPM on Jetson

**Platform:** NVIDIA Jetson AGX Orin, JetPack R36.4.4, kernel 5.15.148, OP-TEE 4.2

The limitation documented here is **structural**, not a tunable timing gap. Raw
command output backing every citation is in
[`evidence/root_cause_source_trace.txt`](../evidence/root_cause_source_trace.txt)
and [`evidence/nvidia_ftpm_userspace_binding.txt`](../evidence/nvidia_ftpm_userspace_binding.txt).

---

## TL;DR

The IMA log on this device is never anchored into TPM PCR[10] because the fTPM
**does not exist as a TPM device during kernel init at all**. It is registered
only *after the userspace `tee-supplicant` daemon starts*, because the fTPM
relies on supplicant-mediated **RPMB** secure storage. IMA's `tpm_default_chip()`
lookup runs in `late_initcall`, entirely within kernel init — seconds before
userspace exists — so it can never find the chip. No initcall reordering, build
option, or boot parameter can close a kernel-init-vs-userspace gap.

The same property (userspace-bound, RPMB-backed fTPM) is the single root cause
behind **both** of this project's headline findings:

1. **PCR[10] = 0** — fTPM absent when IMA initializes.
2. **TPM AK not persistent across reboot** — *empirically observed*. The fTPM is
   brought up and torn down from userspace, and this dev kit is unprovisioned (no
   EKB/EK). The non-persistence is consistent with that userspace-managed,
   unprovisioned lifecycle; the precise mechanism (unprovisioned NV vs. shutdown
   teardown ordering) was not fully isolated.

One architectural property, two symptoms.

---

## Authoritative confirmation — NVIDIA's own config files

The kernel-source argument below explains *how* the fTPM could be late. NVIDIA's
own platform config proves it is late **by design** — which is the stronger and
more interesting result. (Verbatim files:
[`evidence/nvidia_ftpm_userspace_binding.txt`](../evidence/nvidia_ftpm_userspace_binding.txt).)

**1. The fTPM module is denylisted from boot autoload.**
`/etc/modprobe.d/denylist-tpm-ftpm-tee.conf` blocks `tpm_ftpm_tee` from loading
during kernel init. So the chip is *deliberately* absent when IMA looks.

**2. It is loaded manually from userspace, only after tee-supplicant.**
`nv-tee-supplicant.service` (`systemctl cat`):

```ini
# Wait for tee-supplicant to start and load tpm module
# add "-" to avoid service launch failed with failing to probe tpm_ftpm_tee
ExecStartPost=-/bin/sh -c '\
    for i in $(seq 1 ${RETRY_COUNT}); do \
        if pgrep -f "tee-supplicant" > /dev/null; then \
            logger "tee-supplicant is running, loading tpm module"; \
            modprobe tpm_ftpm_tee && exit 0; \
        fi; \
        sleep ${RETRY_INTERVAL}; \
    done; ...'
# Workaround for fTPM TA: stop kernel module before tee-supplicant
ExecStop=-/bin/sh -c "/sbin/modprobe -v -r tpm_ftpm_tee ; /bin/kill $MAINPID"
```

The fTPM is `modprobe`'d **after** `pgrep tee-supplicant` succeeds, and on
shutdown the module is removed **before** the supplicant is killed (clean RPMB
teardown order). The fTPM's entire load/unload lifecycle is userspace-managed.
The leading `-` on `ExecStartPost` is also why a built-in `CONFIG_TCG_FTPM_TEE=y`
kernel just logs a harmless `modprobe: FATAL ... not found`.

**3. The fTPM's backing store isn't even ready until after IMA.**
`dmesg`: `[3.387907] mmcblk0rpmb … ready` — RPMB comes up at **3.388s**, ~1s
after IMA's 2.333s lookup. The fTPM cannot function without RPMB, and RPMB
postdates IMA. This is a storage dependency, not a tunable timing race.

**4. NVIDIA ships a userspace, EKB-provisioned attestation path.**
`/etc/systemd/nv-ftpm-device-provision.sh` provisions the fTPM via `nvftpm-helper-app`
+ EKB (Encrypted Key Blob), producing RSA/EC EK certs and an owner hierarchy.
That is a *userspace-mediated, provisioned* attestation model — **not** the
kernel-space IMA → PCR[10] model.

### The conclusion this forces

The two attestation models are **structurally incompatible on this platform**:

- **NVIDIA's intended model:** userspace, RPMB-backed, EKB-provisioned fTPM,
  consumed by `nvftpm-helper-app` + a dedicated PTA. Brought up after userspace.
- **Linux IMA's model:** kernel-space measurement extended into PCR[10] at
  `late_initcall`, requiring a TPM present during kernel init.

There is no overlap in time: the fTPM is deliberately kept out of kernel init,
exactly where IMA needs it. This is **by design, not a defect** — and it is why
PCR[10] runtime attestation is unavailable here without abandoning the fTPM's
secure storage entirely.

---

## Upstream prior art & cross-platform recurrence

This is **not a Jetson bug, and not a novel discovery** — it is a known OP-TEE
architectural limitation, raised upstream since 2022 and hit independently on
several ARM platforms. Full quotes + links:
[`evidence/upstream-prior-art.md`](../evidence/upstream-prior-art.md).

- **Raised upstream (OP-TEE list, 2022-07-20).** A thread titled *"Current design
  of tee-supplicant can't support ftpm requests during kernel space bootup"*
  describes the exact problem:
  > "It has to collect data when kernel space is booting up, so we cannot delay
  > these requests further until user space is up … tee-supplicant context is not
  > yet initialized, which results in IMA detection of TPM devices failed."
  OP-TEE maintainers (Wiklander, Forissier, Garg) participated. *(The poster's
  affiliation is not stated on the public archive — the message headers use a
  Microsoft Exchange domain — so this is not attributed to any specific vendor.)*

- **Xilinx ZCU104 — identical wall, 2025 (OP-TEE #7248, open).** A different
  platform, OP-TEE 4.3.0:
  > "the device probe can be done only if the tee-supplicant [is running], which
  > makes it impossible to have an instantiated tpm device at IMA initialization
  > time."

- **RockPi4B / STM32MP157C-DK2 — same goal, same failure, 2023 (OP-TEE #5766).**
  A developer attempting *exactly this* (fTPM as Early TA for IMA / remote
  attestation) hit the tee-supplicant/RPMB/initramfs wall.

- **Fixed in mainline Linux 6.12 (2024) — but newer than this Jetson's kernel.**
  The kernel **RPMB subsystem** (`drivers/misc/rpmb`) was merged in **Linux 6.12**,
  with the OP-TEE driver as its first user (Wiklander / Linaro). It lets the kernel
  reach RPMB directly via the MMC subsystem, removing the userspace `tee-supplicant`
  dependency, so the fTPM becomes usable during kernel init (compiled-in or module)
  and IMA can extend PCR[10]. **However, this Jetson runs kernel 5.15.148 (JetPack
  R36.4.4), which predates 6.12 and lacks it** — so the limitation is real and
  current here until a kernel ≥6.12 with this support runs on the device (NVIDIA
  would need to ship/backport it). Evaluating that on Jetson is future work. See
  [`evidence/upstream-prior-art.md`](../evidence/upstream-prior-art.md) for links.

- **NVIDIA's official docs document the mechanism, not the IMA consequence.** The
  [Jetson Linux Developer Guide — Firmware TPM](https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/Security/FirmwareTPM.html)
  describes the userspace `modprobe tpm_ftpm_tee`, EKB provisioning, and states
  *"Before the fTPM is ready, the measurements can be stored in the format of TPM
  event log"* — but never mentions IMA or PCR[10]. The design is documented; the
  runtime-attestation limitation is not surfaced as a product caveat.

**Takeaway:** a fundamental OP-TEE constraint (fTPM secure storage requires the
userspace tee-supplicant/RPMB path) collides with IMA's need for a TPM at kernel
init. Reported upstream since 2022, seen across multiple platforms, and fixed in
mainline Linux 6.12 (2024) via the kernel RPMB subsystem — but that fix postdates
the 5.15 kernel this Jetson runs, so it does not yet apply on this platform. Given
this constraint, this project does not rely on PCR[10]; it signs the IMA log
aggregate with a hardware-rooted key held in a custom OP-TEE Trusted Application.

---

## Boot timeline — "OP-TEE-ready" is not "fTPM-ready"

A crucial subtlety: OP-TEE's *core* driver initializing is **not** the same as
the fTPM being available. Across boots, OP-TEE core comes up **before** IMA's
lookup, yet IMA still finds no TPM:

| Source | OP-TEE init | IMA "No TPM" | Order |
|---|---|---|---|
| `dmesg_boot.txt` / `timing_evidence.txt` | 3.824s | 3.896s | IMA 72 ms after OP-TEE |
| screenshot | 2.287s | 2.333s | IMA 46 ms after OP-TEE |

```
[2.287706] optee: initialized driver               <-- OP-TEE core READY
[2.333121] ima: No TPM chip found, activating TPM-bypass!   <-- 46 ms later, still no chip
```

OP-TEE core is fully initialized *before* IMA runs, and IMA still finds no TPM.
The boot-to-boot variance in the small gap (72 ms vs 46 ms) is itself the tell:
the gap size is irrelevant, because the fTPM is gated on something that happens
*after kernel init entirely* — the userspace tee-supplicant daemon (below).
**OP-TEE-core-ready is not fTPM-ready.**

---

## The three facts that pin the cause

### Fact 1 — IMA's TPM lookup is a permanent one-shot, run at `late_initcall`

`security/integrity/ima/ima_init.c`:

```c
int __init ima_init(void)
{
	ima_tpm_chip = tpm_default_chip();          // line 123 — looked up ONCE
	if (!ima_tpm_chip)
		pr_info("No TPM chip found, activating TPM-bypass!\n");  // line 125
	...
}
```

`security/integrity/ima/ima_main.c:1077`:

```c
late_initcall(init_ima);   /* Start IMA after the TPM is available */
```

So IMA runs at `late_initcall` — one of the *last* kernel-init phases — and it
still misses the fTPM, which tells us the fTPM appears even later than
`late_initcall`. The lookup is never retried: if `ima_tpm_chip` is NULL it stays
NULL for the whole session.

### Fact 2 — A lazy re-acquire is not a clean patch

It is tempting to "just retry `tpm_default_chip()` later." That does not work
cleanly, because IMA's init *branches on the chip being present at init time* in
three coupled places:

- **boot_aggregate** (`ima_init.c`, ~line 75): `if (ima_tpm_chip) ima_calc_boot_aggregate(...)`.
  With no chip, the first IMA log record is written as all-zeros, permanently.
- **crypto bank arrays** (`ima_crypto.c`, the `NR_BANKS(ima_tpm_chip)` loops):
  `NR_BANKS` is 0 when the chip is NULL, so `ima_algo_array` is built for the
  *no-TPM* layout. Every template entry's digests are computed against that layout.
- **extend path** (`ima_queue.c:142`, 220–231): `if (!ima_tpm_chip) return;`,
  and the digests array is sized from `ima_tpm_chip->nr_allocated_banks`.

A late re-acquire would therefore require re-running `ima_init_crypto()`,
re-emitting boot_aggregate, and defining replay semantics for the measurements
taken before the chip appeared (which were never extended). That is an IMA
re-architecture, not a small fix — and it would still leave a window of
un-anchored early measurements, breaking the clean "replay whole log == PCR[10]"
property that is the entire point.

### Fact 3 — The fTPM is supplicant-gated, so it appears only in userspace

The fTPM driver is a passive `tee_client_driver` (`tpm_ftpm_tee.c:382`,
registered at `module_init`, line 415). It only probes when a matching device
is put on the TEE bus by the OP-TEE driver. OP-TEE has **two** enumeration paths
(`drivers/tee/optee/core.c`):

```c
core.c:747:  rc = optee_enumerate_devices(PTA_CMD_GET_DEVICES);        // SYNC, inside optee_probe
core.c:217:  WARN_ON(optee_enumerate_devices(PTA_CMD_GET_DEVICES_SUPP)); // SUPP, on a workqueue
```

- `GET_DEVICES` runs synchronously inside `optee_probe`. If the fTPM were in this
  list it would be on the bus by the time `optee: initialized driver` prints.
  **It is not** — proven by Fact's dmesg (OP-TEE ready, yet IMA finds no chip).
- `GET_DEVICES_SUPP` is the **supplicant-dependent** list. It runs from
  `optee_bus_scan`, which is dispatched on a workqueue that is `queue_work`'d
  **only when the userspace `tee-supplicant` daemon opens a supplicant context**:

```c
// core.c:215..251 (optee_open_session / context-open path)
static void optee_bus_scan(struct work_struct *work) { ... }
...
if (teedev == optee->supp_teedev) {          // the SUPPLICANT device
	if (!optee->supp.ctx) {
		optee->supp.ctx = ctx;           // userspace tee-supplicant connected
		...
		INIT_WORK(&optee->scan_bus_work, optee_bus_scan);
		optee->scan_bus_wq = create_workqueue("optee_bus_scan");
		queue_work(optee->scan_bus_wq, &optee->scan_bus_work);  // THEN enumerate SUPP devices
	}
}
```

The fTPM is in the `_SUPP` list because it needs supplicant-mediated **RPMB**
secure storage for its persistent state. Confirming evidence: the kernel-init
dmesg contains **no `tpm tpm0` registration line at all** — the fTPM is simply
not enumerated during boot. It comes up later, once userspace `tee-supplicant`
runs.

---

## Why this is unfixable without breaking the fTPM

Putting the facts together:

```
kernel init:
  ...
  late_initcall(init_ima)
    └── tpm_default_chip() -> NULL  (no fTPM yet; SUPP devices not enumerated)
  ...
kernel hands off to userspace (systemd / init)
  └── tee-supplicant daemon starts
        └── opens supplicant context on OP-TEE
              └── queue_work(optee_bus_scan)
                    └── enumerate PTA_CMD_GET_DEVICES_SUPP
                          └── fTPM device registered on TEE bus
                                └── ftpm_tee probe -> tpm_add_char_device() -> /dev/tpm0
```

IMA's window is *inside kernel init*. The fTPM's earliest possible appearance is
*after userspace starts*. These are separated by the kernel→userspace handoff,
which is unconditional. The only way to make the fTPM available during kernel
init is to remove its dependency on tee-supplicant — i.e. strip its RPMB secure
storage — which destroys fTPM persistence (NV indexes, persistent EK/AK). That
is not a fix; it dismantles the TPM.

This is also why the cross-platform "fixes" do **not** transfer here:

- **Raspberry Pi (kernel 5.15.y) — fixed for its hardware, irrelevant to ours.**
  The RPi fix moved the bcm2835 clock driver from `postcore_initcall` to
  `subsys_initcall` and forced the SPI TPM driver to probe synchronously when IMA
  is enabled:

  ```c
  static struct spi_driver tpm_tis_spi_driver = {
      .driver = {
          .name = "tpm_tis_spi",
  #ifdef CONFIG_IMA
          .probe_type = PROBE_FORCE_SYNCHRONOUS,
  #else
          .probe_type = PROBE_PREFER_ASYNCHRONOUS,
  #endif
      },
      ...
  };
  ```

  This works because the RPi TPM is a real, synchronous **SPI** bus device that
  can be forced to finish probing before IMA's check. The Jetson fTPM is not on
  SPI — it is denylisted and hand-loaded by a userspace daemon — so there is no
  driver `probe_type` to force.

- **Wei Liu Makefile patch (2021) — link-order only, doesn't apply.**

  ```makefile
  # TPM drivers must come after TEE, otherwise fTPM initialization
  # will be deferred, which causes IMA to not get a TPM device in time
  obj-$(CONFIG_TCG_TPM) += char/tpm/
  ```

  This only reorders built-in initcall *link order*. It cannot move a userspace
  event (the `modprobe` issued by `nv-tee-supplicant.service`), which is what
  actually gates the Jetson fTPM.

---

## Consequence for the attestation design (and why it is fine)

Because PCR[10] cannot be fed at measurement time, the design does **not** rely
on it. The runtime-integrity evidence is the IMA log, carried as an aggregate
hash and signed by the OP-TEE attestation TA's hardware-rooted key, bound to a
verifier nonce. Under the standard attestation threat model (uncompromised
kernel), this provides authenticity and freshness of the log as presented.

What it does **not** provide — and what only a synchronous in-kernel TPM could —
is *load-time, extend-only* anchoring: the guarantee that each measurement was
folded into tamper-proof hardware at the moment of load, unrollback-able. On
this platform that property is structurally unavailable. Documenting it as a
characterized constraint of supplicant-gated fTPM is more honest and more useful
than a fragile post-hoc workaround (e.g. having the TA "replay" the
NW-supplied log into PCR[10] — that adds no integrity, since the TA only ever
sees the untrusted, NW-supplied log).

---

## Reproduce

On the device, against `~/kernel_build/kernel/kernel-jammy-src`:

```bash
K=~/kernel_build/kernel/kernel-jammy-src
O=$K/drivers/tee/optee

# IMA one-shot lookup + late_initcall
grep -rn "TPM-bypass\|tpm_default_chip\|ima_tpm_chip" $K/security/integrity/ima/ima_init.c
grep -rn "late_initcall" $K/security/integrity/ima/ima_main.c

# fTPM is a passive tee_client_driver
grep -n "tee_client_driver\|module_init" $K/drivers/char/tpm/tpm_ftpm_tee.c

# Two enumeration paths; SUPP one is on a supplicant-triggered workqueue
grep -rn "optee_enumerate_devices\|INIT_WORK\|queue_work\|optee_bus_scan\|supp_teedev" $O/core.c

# Decisive: OP-TEE ready before IMA, yet IMA finds no chip; no tpm0 line at boot
sudo dmesg | grep -iE "tpm|ftpm|optee|ima:|supplicant"
```

Full captured output: [`evidence/root_cause_source_trace.txt`](../evidence/root_cause_source_trace.txt).

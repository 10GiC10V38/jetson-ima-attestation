# fTPM + IMA Initialization Ordering Problem — Deep Analysis

**Platform:** NVIDIA Jetson AGX Orin, JetPack R36.4.4, Kernel 5.15.148

---

## The Core Problem in One Paragraph

On NVIDIA Jetson AGX Orin, the TPM is implemented as a firmware TPM
(fTPM) — a Microsoft Trusted Application running inside OP-TEE. When the
Linux kernel boots, the IMA subsystem initializes and immediately looks
for a TPM device to extend PCR[10] with runtime measurements. However,
fTPM depends on OP-TEE, which depends on the TEE bus, which initializes
significantly later than IMA. IMA gives up finding a TPM and activates
bypass mode. Even though the fTPM becomes fully functional shortly after,
IMA never connects to it. The result: IMA runs correctly and measures
everything, but PCR[10] remains all zeros for the entire session.

---

## Precise Boot Timeline

From dmesg on Jetson AGX Orin R36.4.4:

```
[0.000000]  Kernel starts
[0.261754]  device-mapper: IMA_DISABLE_HTABLE disabled
[1.806174]  ima: No TPM chip found, activating TPM-bypass!  ← IMA gives up
[1.806195]  ima: Allocated hash algorithm: sha256
[3.764685]  optee: probing for conduit method               ← OP-TEE starts
[3.764735]  optee: revision 4.2 (d78bc5fa)
[3.823894]  optee: dynamic shared memory is enabled
[3.824153]  optee: initialized driver                       ← OP-TEE ready
            (fTPM TA accessible from here)
```

Gap between IMA check and fTPM availability: **~2 seconds**

---

## Initialization Chain — Why the Gap Exists

```
Kernel boot sequence (simplified):

  security_init()           ← very early, before drivers
    └── ima_init()
          └── tpm_default_chip()  ← looks for TPM RIGHT NOW
                return NULL       ← nothing registered yet
          └── "No TPM chip found, activating TPM-bypass!"

  ... many driver initializations ...

  tee_init()                ← TEE core bus
    └── optee_driver_init()
          └── optee_probe()
                └── optee_enumerate_devices()
                      └── registers fTPM device on TEE bus

  ftpm_tee_driver_probe()   ← fTPM probes TEE bus
    └── opens session with fTPM TA in OP-TEE
    └── registers /dev/tpm0
    └── calls tpm_add_char_device()
          └── idr_replace()  ← NOW tpm0 is in the TPM idr table
                                but IMA already checked and left
```

---

## Why CONFIG_TCG_FTPM_TEE=y Does Not Solve It

A natural first attempt is to build the fTPM driver into the kernel
instead of as a module, hoping it probes earlier. This does not work
because the probe ordering is determined by the TEE bus, not by whether
the driver is built-in or modular.

The fTPM driver cannot probe until:
1. OP-TEE driver has initialized
2. TEE bus has enumerated devices
3. fTPM device node is registered on TEE bus
4. fTPM driver's probe() is called
5. Session with fTPM TA is opened in OP-TEE
6. tpm_add_char_device() registers the chip

Steps 1-6 happen in bus initialization order, which is after
security subsystem initialization where IMA runs.

Result after setting CONFIG_TCG_FTPM_TEE=y on Jetson:
```
[5.446524] Error: Driver 'ftpm-tee' is already registered, aborting...
```
The built-in driver AND the old .ko file both try to register.
After removing the .ko file, the conflict resolves, but IMA
still runs before fTPM is ready.

---

## Comparison with Physical TPM

```
Physical TPM (SPI/I2C chip):           fTPM on Jetson:

Power ON                                Power ON
  │                                       │
  TPM chip self-initializes              OP-TEE boots (needs TF-A first)
  PCRs reset to zero                       │
  │                                       fTPM TA starts inside OP-TEE
  kernel boots                             │
  │                                       TEE bus initializes
  ima_init()                               │
  tpm_default_chip() ──────────────────── tpm_add_char_device() called
  FOUND ✅                                 too late ❌
  PCR[10] extension begins
```

Physical TPM is available before kernel starts.
fTPM is available only after multiple kernel subsystems initialize.

---

## What Fixes Exist on Other Platforms

### Raspberry Pi (kernel 5.15.y) — Fixed

The Raspberry Pi fix used two changes:
1. Postponed the bcm2835 clock driver from postcore_initcall to
   subsys_initcall, giving TPM more time to probe
2. Forced the SPI TPM driver to use PROBE_FORCE_SYNCHRONOUS when
   IMA is enabled, ensuring probe completes before IMA checks

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

This fix works for SPI-connected TPM chips. It does not apply to
fTPM because fTPM is not a SPI device.

### Linux Kernel Makefile Patch (2021, Wei Liu) — Partial

```makefile
# TPM drivers must come after TEE, otherwise fTPM initialization
# will be deferred, which causes IMA to not get a TPM device in time
obj-$(CONFIG_TCG_TPM) += char/tpm/
```

This ensures TEE initializes before TPM drivers when both are built-in.
Helps with link-order deferred initialization but does not solve the
fundamental TEE bus timing issue.

### Jetson (this work) — Not yet fixed

No existing patch addresses fTPM + IMA ordering on Jetson AGX Orin.
The fix would require one of:

- Making IMA retry TPM detection after a delay
  (changes security/integrity/ima/ima_init.c)
- Making fTPM probe synchronously before IMA
  (requires TEE bus early init — architecturally complex)
- Adding a deferred TPM registration callback in IMA
  (cleanest solution, needs upstream discussion)

---

## Security Implications

### What PCR[10] = 0 Means

Without PCR[10] extension, the IMA measurement log lacks a
hardware-enforced integrity proof. The log exists and is correct, but
a kernel-level attacker could theoretically:

1. Load a malicious kernel module
2. Remove its IMA log entry from the kernel linked list
3. Present a clean-looking log to the verifier
4. No PCR[10] mismatch would be detected

With PCR[10] extension, this attack fails because:
- The extend operation is one-way (cannot undo)
- Removing a log entry does not un-extend the PCR
- Verifier replays log → simulated PCR ≠ actual PCR → detected

### Threat Model Scope

This work assumes the kernel is not compromised (standard assumption
in attestation literature). Under this assumption:

- IMA log is trustworthy (kernel space, userspace cannot fake)
- PCR[0-7] prove boot chain integrity (firmware-extended)
- OP-TEE TA signs both, providing a single signed evidence package
- Normal World agent is a dumb pipe and cannot tamper with signed token

---

## Our Workaround

Since PCR[10] cannot be connected to IMA on this platform, we use
OP-TEE as a direct trust anchor for runtime measurements:

```
Normal system:
  IMA log → extends → PCR[10] → signed by TPM quote

Our system:
  IMA log → aggregate hash → signed by OP-TEE TA key
```

The OP-TEE TA reads:
- PCR[0-7] directly from fTPM (via inter-TA communication)
- IMA log aggregate hash from kernel memory
- ECID and fuse state from hardware registers

It hashes all of this with a verifier-supplied nonce and signs with
an ECDSA private key that never leaves the TEE. The security property
is equivalent for our threat model — the private key is hardware-rooted
in TrustZone, which is stronger protection than most software solutions.

---

## Upstream Contribution Opportunity

This problem has not been reported specifically for Jetson AGX Orin
with JetPack R36.4.4. A potential upstream fix would be:

```c
// security/integrity/ima/ima_init.c
// Add deferred TPM probe support

static void ima_tpm_found(struct tpm_chip *chip) {
    /* called when TPM becomes available after IMA init */
    if (ima_used_chip == NULL) {
        ima_used_chip = chip;
        /* replay early measurements into PCR */
        ima_replay_measurements_to_tpm();
    }
}

// register callback with TPM core
tpm_register_notifier(&ima_tpm_notifier);
```

This would allow IMA to connect to fTPM when it becomes available
rather than failing permanently at boot time.

---

## Evidence Files

| File | Description |
|---|---|
| evidence/dmesg_boot.txt | Complete kernel boot log |
| evidence/timing_evidence.txt | IMA and OP-TEE timing lines |
| evidence/pcr_values.txt | TPM PCR values showing PCR[10]=0 |
| evidence/ima_measurements.txt | Sample IMA measurement log entries |
| evidence/ima_policy.txt | Active IMA policy (tcb) |
| evidence/kernel_config_ima.txt | IMA-related kernel config options |

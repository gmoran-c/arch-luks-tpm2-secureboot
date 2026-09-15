# Boot Hardening on Arch Linux: LUKS2 + TPM2 + Custom Secure Boot

Documentation of a personal project to harden the boot process on an Arch Linux laptop, combining full-disk encryption, TPM 2.0-backed automatic unlocking, and Secure Boot signed with user-owned keys.

## Goal

Replace a basic disk encryption setup (LUKS2 + manual passphrase) with a full chain of trust where:

- The encryption key is released automatically **only if the firmware has not been tampered with** (TPM-based verification tied to Secure Boot state).
- The kernel and bootloader are cryptographically signed with keys generated and controlled by the user, not a third party.
- A strong fallback passphrase (properly tuned Argon2id) remains available for any scenario where the TPM is unavailable.

## Trust chain diagram

```text
UEFI Firmware (Secure Boot enabled, custom keys)
        |
        v
Signed systemd-boot --> verified by Secure Boot
        |
        v
Signed kernel --> verified by Secure Boot
        |
        v
TPM2 measures boot state (PCR 7)
        |
        +-- PCR 7 matches --> automatically releases the LUKS key
        |
        +-- PCR 7 does NOT match --> refuses to release the key
                                      -> falls back to manual passphrase (Argon2id)
```

## Hardware and software used

- Laptop with UEFI + TPM 2.0 (firmware TPM / Intel PTT)
- Arch Linux
- Bootloader: `systemd-boot`
- Encryption: LUKS2 (native format since installation)

> **Portability note**: this guide applies to virtually any machine with UEFI + TPM 2.0 + Arch (or another systemd-based distro) using `systemd-boot`. If your machine uses **GRUB** instead, the Secure Boot steps differ substantially (it requires a signed shim and a different signing workflow).

---

## Prerequisites

```bash
sudo pacman -S sbctl tpm2-tools
```

Verify the system detects the TPM:

```bash
sudo tpm2_getcap properties-fixed
ls /dev/tpm*
```

Verify your systemd version (need >= 248 for `systemd-cryptenroll`):

```bash
systemctl --version
```

---

## Step-by-step guide

### 1. Tune the passphrase KDF (Argon2id)

Before touching TPM or Secure Boot, make sure your passphrase keyslot uses a KDF with parameters reasonable for early boot (initramfs):

```bash
sudo cryptsetup luksConvertKey --pbkdf argon2id \
  --pbkdf-memory 524288 \
  --pbkdf-parallel 4 \
  --iter-time 4000 \
  /dev/LUKS_DEVICE
```

Verify:

```bash
sudo cryptsetup luksDump /dev/LUKS_DEVICE
```

It should show `PBKDF: argon2id` and `Memory: 524288` on the relevant slot.

### 2. Confirm the correct hook in `mkinitcpio.conf`

`sd-encrypt` (the modern systemd hook) is **not compatible** with the classic `udev` hook. They must coexist under the `systemd` hook instead:

```text
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt resume filesystems fsck)
```

Regenerate after any change:

```bash
sudo mkinitcpio -P
```

### 3. Use the correct syntax in boot entries

With `sd-encrypt`, `systemd-boot` entries (in `/boot/loader/entries/*.conf`) must use `rd.luks.name=`, **not** `cryptdevice=` (which belongs to the legacy hook):

```text
options rd.luks.name=LUKS_UUID=root root=/dev/mapper/root rw
```

### 4. Install and configure custom Secure Boot with `sbctl`

```bash
sudo sbctl status
```

If the firmware is in **Setup Mode**, you can proceed directly. Otherwise, existing Secure Boot keys must be cleared from the machine's UEFI settings first.

```bash
sudo sbctl create-keys
sudo sbctl enroll-keys -m   # -m also includes Microsoft's keys (recommended)
```

### 5. Sign the kernel and bootloader

```bash
sudo sbctl verify
```

Sign every file reported as unsigned:

```bash
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /boot/vmlinuz-linux
```

### 6. Enable Secure Boot in UEFI

Reboot, enter UEFI settings, and enable Secure Boot (with your own keys already enrolled).

Verify after booting:

```bash
bootctl status
sudo sbctl status
mokutil --sb-state
```

All three should confirm `Secure Boot: enabled (user)`.

### 7. Enroll the TPM sealed to Secure Boot state (PCR 7)

```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/LUKS_DEVICE
```

Verify:

```bash
sudo cryptsetup luksDump /dev/LUKS_DEVICE | grep -A5 "Tokens:"
```

You should see `systemd-tpm2` with `tpm2-hash-pcrs: 7`.

### 8. Automate re-signing after kernel updates

Create `/etc/pacman.d/hooks/95-sbctl-sign.hook`:

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux
Target = linux-hardened
Target = systemd

[Action]
Description = Signing EFI files with sbctl for Secure Boot...
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
```

Without this hook, every kernel update will lose its signature and Secure Boot will refuse to boot it.

---

## Troubleshooting: real issues encountered

This section documents three real failures during implementation, along with diagnosis and fix — the value-add over a purely theoretical guide.

### Issue 1: Manual unlock took 1–2 minutes

**Symptom**: after entering the correct passphrase, boot would hang on `A start job is running for Cryptography Setup for <name> (Xs / no limit)` for over a minute.

**Diagnosis**: the passphrase keyslot used a KDF with no memory cost tuned for the constrained initramfs environment.

**Fix**: recalibrate with `cryptsetup luksConvertKey` using Argon2id with explicit memory settings (see step 1).

### Issue 2: System dropped into an emergency shell after fixing the KDF

**Symptom**: `Failed to stat resume device`, followed by `ERROR: device '/dev/mapper/root' not found`, ending in an initramfs emergency shell with no `cryptsetup` tools available.

**Diagnosis**: `/etc/mkinitcpio.conf` mixed the `udev` hook (legacy system) with `sd-encrypt` (systemd-exclusive) in the same `HOOKS` array. The two boot systems are incompatible with each other; the encryption hook never executed correctly.

**Fix**: replace `udev` with `systemd`, and `keymap`/`consolefont` with `sd-vconsole`, in the HOOKS array, then regenerate the initramfs.

### Issue 3: Timeout waiting for `/dev/mapper/root` despite the fixed hook

**Symptom**: after fixing the hooks, boot still failed with `[TIME] Timed out waiting for device /dev/mapper/root`, dropping into systemd's emergency mode (compounded by the root account being locked by default, blocking access to the emergency shell).

**Diagnosis**: the boot entries (`/boot/loader/entries/*.conf`) still used the kernel parameter `cryptdevice=PARTUUID=...`, which belongs to the legacy `encrypt` hook. The `sd-encrypt` hook doesn't understand this parameter; it expects `rd.luks.name=` or `rd.luks.uuid=`.

**Fix**: edit the boot entries to replace `cryptdevice=PARTUUID=...:root` with `rd.luks.name=<LUKS_UUID>=root`.

**General lesson**: all three failures are pieces of the same chain (KDF → initramfs hooks → kernel parameters) and must remain consistent with each other. Changing one part without updating the others produces boot failures that are hard to diagnose without cross-checking `journalctl` logs and boot entries together.

---

## Comparative analysis (qualitative)

> **Methodology**: there is no standardized public benchmark that scores "disk encryption security" as a percentage. The scores below are a qualitative self-assessment based on documented technical criteria (KDF type, hardware isolation of the key, boot integrity verification, code transparency), intended as a comparative and study guide — not a certified metric.

| Criterion | LUKS2 (basic config) | LUKS2 + TPM2 + custom Secure Boot | Apple FileVault (Secure Enclave) |
|---|---|---|---|
| Offline brute-force resistance | Medium (depends on KDF) | High (tuned KDF + hardware) | High (hardware rate-limiting) |
| Hardware key protection | None | Yes (TPM2, sealed to PCR) | Yes (dedicated Secure Enclave) |
| Boot integrity verification | None | Yes (Secure Boot + PCR 7) | Yes (Apple's own Verified Boot) |
| Transparency / auditability | Full (open source) | Full (open source) | Low (closed ecosystem) |
| Configuration flexibility | High | High | None (fixed by Apple) |

---

## Certification concepts applied

This project puts into practice concepts covered in the CompTIA Security+ and CySA+ exam domains:

- **Hardware root of trust**: using the TPM as a trust anchor that cannot be tampered with from the OS software layer.
- **Defense in depth**: multiple independent layers (Secure Boot, TPM, strong KDF) where a single failure doesn't compromise the whole chain.
- **Measured boot**: verifying system state at each boot stage via TPM PCRs.
- **Key derivation hardening**: tuning Argon2id parameters to resist GPU/ASIC brute-force attacks.
- **Attack surface reduction**: removing reliance on a single passphrase as the only unlock factor.

---

## Known limitations and future work

- The TPM on this device is a **firmware TPM (fTPM)** via Intel PTT, sharing silicon with the CPU — offering less physical isolation than a discrete TPM or an Apple Secure Enclave-equivalent solution.
- The `pacman` hook for automatic re-signing depends on `sbctl sign-all` not failing silently; it's worth re-checking `sbctl verify` after major system updates.
- No physical disk-extraction or real "evil maid" attack scenario has been tested against this setup — the analysis relies on the theoretical guarantees documented by systemd and TPM2, not on original penetration testing.
- Possible future improvement: migrate to a **Unified Kernel Image (UKI)** to also seal against **PCR 11** (exact hash of the loaded kernel), adding an extra verification layer.
- Possible future improvement: add a physical second factor (YubiKey/FIDO2) via `systemd-cryptenroll --fido2-device=auto`.

---

## Disclaimer

This documentation describes a personal configuration for educational and learning purposes. Applying it carries the risk of losing access to the system if a mistake is made during the process; it is strongly recommended to always keep an external boot medium (live USB) and a recovery key (`systemd-cryptenroll --recovery-key`) stored safely offline before making any of these changes.

## License

This documentation is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any scripts or config snippets included (e.g., the pacman hook) are additionally available under the [MIT License](https://opensource.org/licenses/MIT). See `LICENSE` for the full terms.

# Secure Boot and Trusted Computing

Everything we've covered so far assumes the OS kernel is trustworthy. But what if the kernel itself has been tampered with? What if a rootkit replaced the bootloader? All the access controls, sandboxes, and MAC policies in the world are meaningless if the foundation they run on has been compromised. Secure boot and trusted computing solve the boot trust problem: establishing a chain of trust from hardware to running software.

## The Boot Trust Problem

A computer starts from firmware and works its way up:

```
Power On
   |
   v
Firmware (UEFI/BIOS)
   |
   v
Bootloader (GRUB, Windows Boot Manager)
   |
   v
Kernel (Linux, Windows NT)
   |
   v
Init System (systemd, init)
   |
   v
User Applications

If an attacker replaces ANY of these stages, they control everything above.
A tampered bootloader can load a tampered kernel.
A tampered kernel can disable all security mechanisms.
```

This is why boot security is the foundation of everything else. Without it, MAC policies and sandboxes are just theater -- the rootkit can disable them before they load.

## Secure Boot (UEFI)

Secure Boot verifies **digital signatures** at each stage of the boot process:

```
Chain of Trust:

+-------------------+
| UEFI Firmware     |  Contains Platform Key (PK) and trusted certificates
| (in flash ROM)    |  Verifies bootloader signature
+--------+----------+
         | Signature valid?
         v
+-------------------+
| Bootloader        |  Signed by a trusted key (e.g., Microsoft, distro)
| (GRUB, systemd-   |  Verifies kernel signature
|  boot, etc.)      |
+--------+----------+
         | Signature valid?
         v
+-------------------+
| OS Kernel         |  Signed by the OS vendor
| (vmlinuz)         |  (Optional: kernel verifies modules)
+--------+----------+
         | 
         v
+-------------------+
| Kernel modules,   |  Optionally verified by kernel
| initramfs         |  (module signing enforcement)
+-------------------+

At EACH step: if signature verification fails --> REFUSE TO BOOT.
Tampered bootloader? Won't load.
Tampered kernel? Won't load.
```

### Secure Boot Key Hierarchy

```
Platform Key (PK)
  |  Owner: hardware manufacturer or system owner
  |  Controls who can modify the key database
  |
  +---> Key Exchange Keys (KEK)
          |  Owner: OS vendors (Microsoft, Red Hat, Canonical)
          |  Can add/remove signatures from the DB
          |
          +---> Signature Database (db)
          |       Contains: trusted signing certificates
          |       "These bootloaders and kernels are allowed"
          |
          +---> Forbidden Signatures (dbx)
                  Contains: revoked certificates and hashes
                  "These are known-bad -- never boot them"
```

```bash
# Check Secure Boot status on Linux
$ mokutil --sb-state
SecureBoot enabled

# List enrolled keys
$ mokutil --list-enrolled

# On UEFI systems, Secure Boot settings are in firmware setup
# (usually accessible via F2/Del during POST)
```

### The Linux Shim

Most x86 PCs ship with Microsoft's keys. Linux distributions use a signed "shim" bootloader:

```
UEFI Firmware (trusts Microsoft's key)
  |
  v
Shim (signed by Microsoft)        <-- Gets past Secure Boot
  |
  v
GRUB (signed by distro's key)     <-- Shim trusts the distro
  |
  v
Linux kernel (signed by distro)   <-- GRUB trusts the distro
```

The shim is a small bootloader that Microsoft signs. It contains the distribution's own key, so it can then verify GRUB and the kernel without Microsoft being in the loop for every kernel update.

## Measured Boot and TPM

Secure Boot says "only run trusted code." Measured Boot goes further: "record exactly what code ran, so you can prove it later."

### TPM (Trusted Platform Module)

The TPM is a dedicated hardware chip (or firmware module) that provides:

```
+------------------------------------------+
|            TPM Chip                       |
|                                          |
|  +------------------+                    |
|  | PCR Registers    |  Platform          |
|  | PCR[0]: BIOS     |  Configuration     |
|  | PCR[1]: Config   |  Registers --      |
|  | PCR[2]: ROM      |  store boot        |
|  | PCR[3]: ROM cfg  |  measurements      |
|  | PCR[4]: MBR/Boot |                    |
|  | PCR[5]: Boot cfg |                    |
|  | PCR[6]: Events   |                    |
|  | PCR[7]: SecBoot  |                    |
|  | ...              |                    |
|  +------------------+                    |
|                                          |
|  +------------------+                    |
|  | Key Storage      |  Stores encryption |
|  | (sealed keys)    |  keys that can     |
|  |                  |  only be released   |
|  |                  |  if PCRs match     |
|  +------------------+                    |
|                                          |
|  +------------------+                    |
|  | RNG              |  Hardware random   |
|  | (random number   |  number generator  |
|  |  generator)      |                    |
|  +------------------+                    |
+------------------------------------------+
```

### How Measured Boot Works

Each boot stage **measures** (hashes) the next stage before loading it:

```
Boot Stage           Action                              TPM PCR

1. UEFI Firmware     Hash(bootloader) --> extend PCR[4]  PCR[4] = hash1
                     Load bootloader

2. Bootloader        Hash(kernel) --> extend PCR[8]      PCR[8] = hash2
                     Hash(initramfs) --> extend PCR[9]   PCR[9] = hash3
                     Load kernel

3. Kernel            Hash(modules) --> extend PCR[10]    PCR[10] = hash4
                     Load modules

PCR "extend" operation:
  PCR_new = Hash(PCR_old || measurement)
  
  This is one-way: you can't set a PCR to a specific value.
  You can only extend it. This prevents tampering with measurements.
```

The PCR values form a **cryptographic fingerprint** of the entire boot chain. If any component changes (tampered bootloader, modified kernel), the PCR values will be different.

### Remote Attestation

Remote attestation lets a server prove to a remote party exactly what software is running:

```
+------------------+                    +------------------+
|  Client Machine  |                    | Verification     |
|                  |                    | Server           |
|  TPM chip with   |    "Prove what    |                  |
|  PCR values      |    you're running"|  "Here are the   |
|                  | <-----------------+  expected PCR    |
|                  |                    |  values for a    |
|  TPM signs PCRs  |                    |  clean system"  |
|  with its key    | ------------------>|                  |
|                  |    Signed PCR      |  Verify:         |
|                  |    quote           |  1. TPM signature |
+------------------+                    |  2. PCR values   |
                                        |     match        |
                                        |     expected?    |
                                        +------------------+

If PCRs don't match expected values --> machine may be compromised.
```

Use cases: corporate compliance (prove endpoints run approved OS), DRM, cloud infrastructure integrity verification.

## dm-verity: Filesystem Integrity

dm-verity verifies the integrity of **read-only filesystem partitions** using a Merkle tree (hash tree):

```
Filesystem blocks:
[Block 0] [Block 1] [Block 2] [Block 3] [Block 4] [Block 5] ...

Hash tree:
                    Root Hash (stored in trusted location)
                   /                                \
          Hash(H0+H1)                          Hash(H2+H3)
         /          \                         /          \
    Hash(B0+B1)  Hash(B2+B3)           Hash(B4+B5)    ...
    /       \     /       \             /       \
 H(B0)   H(B1) H(B2)   H(B3)       H(B4)   H(B5)

When Block 2 is read:
  1. Hash Block 2
  2. Combine with sibling hash, verify parent
  3. Walk up to root
  4. Compare with trusted root hash
  5. Match? Data is authentic. Mismatch? Corruption or tampering.
```

```bash
# dm-verity is used by:
# - Android: verified boot for system partition
# - ChromeOS: verified boot for rootfs
# - Container images: integrity verification

# Set up dm-verity on Linux
$ veritysetup format /dev/sda1 /dev/sda2    # Create hash device
Root hash: 4a3c2b1d...

$ veritysetup open /dev/sda1 verified-root /dev/sda2 4a3c2b1d...
# Now /dev/mapper/verified-root is a verified read-only device
```

**Key property:** dm-verity only works for read-only partitions. The hash tree is computed once; any change to any block changes the root hash. This is perfect for system partitions that shouldn't change.

## Full Disk Encryption

Full disk encryption (FDE) protects **data at rest** -- if a device is stolen, the data is unreadable without the encryption key.

| Solution | Platform | Algorithm | Key Storage |
|----------|----------|-----------|-------------|
| **LUKS** | Linux | AES-256-XTS | Passphrase-derived (or TPM-sealed) |
| **BitLocker** | Windows | AES-256 (XTS or CBC) | TPM + PIN/USB key |
| **FileVault** | macOS | AES-256-XTS | Secure Enclave / iCloud recovery |

### LUKS on Linux

```
Disk Layout with LUKS:
+-----------------------------------------------------+
| LUKS Header     | Encrypted Partition                |
| (key slots,     | (looks like random data without   |
|  cipher info,   |  the correct passphrase)           |
|  salt, UUID)    |                                     |
+-----------------------------------------------------+

Boot process with LUKS:
  1. UEFI loads bootloader (unencrypted /boot partition)
  2. Bootloader loads kernel + initramfs (unencrypted)
  3. initramfs asks for passphrase (or reads from TPM)
  4. LUKS decrypts the root partition
  5. System boots normally
```

```bash
# Set up LUKS encryption
$ cryptsetup luksFormat /dev/sda2
$ cryptsetup open /dev/sda2 cryptroot
$ mkfs.ext4 /dev/mapper/cryptroot
$ mount /dev/mapper/cryptroot /mnt

# LUKS + TPM: seal the key to TPM PCR values
# If boot chain is tampered with, TPM refuses to release the key
$ systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/sda2
```

### TPM-Sealed Disk Encryption

The most secure configuration ties disk encryption to the TPM:

```
Clean boot:                         Tampered boot:
  Firmware --> Bootloader --> Kernel   Firmware --> EVIL Bootloader --> ?
  PCRs match expected values          PCRs DON'T match
  TPM releases disk encryption key    TPM REFUSES to release key
  System boots normally               System cannot decrypt disk
                                      Attacker is locked out
```

This ensures that even if someone steals your laptop and replaces the bootloader with a keylogger, the TPM won't release the disk encryption key because the boot measurements don't match.

## Supply Chain Attacks and Boot Integrity

Why all of this matters more than ever:

**Boot-level attacks persist across OS reinstalls.** A rootkit in the firmware or bootloader survives a complete OS reinstall. Secure Boot prevents loading tampered bootloaders. Measured Boot detects if firmware itself was tampered with.

**Supply chain attacks** compromise software before it reaches the user. If an attacker compromises a build system and injects malicious code into a bootloader update, Secure Boot's signature verification catches it (the malicious update won't be signed by the trusted key). Measured Boot creates an audit trail of what actually ran.

**Firmware attacks are the new frontier.** As OS-level security improves, attackers move down the stack. UEFI rootkits, BMC compromises, and hardware implants are real threats in high-security environments. This is why technologies like Intel Boot Guard (which verifies firmware integrity using a key fused into the CPU) exist.

## Confidential Computing

The newest frontier extends trust boundaries into the cloud:

```
Traditional cloud:
  You trust: your code, the cloud provider's hypervisor, firmware, hardware
  Cloud provider can theoretically read your VM's memory

Confidential computing:
  AMD SEV (Secure Encrypted Virtualization):
    - VM memory is encrypted with a key the hypervisor can't access
    - Even the cloud provider can't read your data in memory

  Intel TDX (Trust Domain Extensions):
    - Hardware-enforced isolation between VMs and hypervisor
    - Remote attestation proves the VM is running on genuine hardware

  AWS Nitro Enclaves:
    - Isolated compute environment with no persistent storage
    - No network access, no admin access from the host
    - Attestation proves the enclave code hasn't been tampered with
```

This matters for regulated industries: you can process sensitive data in the cloud without trusting the cloud provider with your plaintext data.

## Real-World Connection

**Secure Boot on cloud VMs**: Cloud providers (AWS, Azure, GCP) offer Secure Boot for VMs. Azure Trusted Launch verifies the VM's boot chain. AWS Nitro uses a dedicated security chip. This prevents boot-level persistence -- if an attacker compromises a VM, they can't install a bootkit that survives reboots.

**TPM for disk encryption keys**: BitLocker on Windows uses the TPM to seal the disk encryption key to the boot configuration. If someone removes the disk and puts it in another machine (no matching TPM) or tampers with the boot chain (PCRs don't match), the key isn't released. This is why enterprise laptops with BitLocker + TPM are considered acceptable even if stolen.

**Android Verified Boot**: Android uses dm-verity to verify the system partition on every read. Combined with Secure Boot, this creates a chain of trust from the bootloader to every file on the system partition. If a rootkit modifies any system file, dm-verity detects it and the device either refuses to boot or displays a warning.

**ChromeOS security model**: ChromeOS is built entirely around verified boot. The rootfs is read-only and verified with dm-verity. Updates are applied to an alternate partition and verified before switching. If verification fails, ChromeOS recovers from a known-good image. This is why Chromebooks are considered one of the most secure consumer platforms.

## Interview Angle

**Q: How does UEFI Secure Boot work?**

A: Secure Boot creates a chain of trust during boot. The UEFI firmware contains trusted certificates (Platform Key, Key Exchange Keys, Signature Database). When loading the bootloader, the firmware verifies its digital signature against the trusted certificates. If valid, the bootloader loads and verifies the kernel's signature. Each stage cryptographically verifies the next before executing. If any signature check fails, the system refuses to boot. This prevents loading tampered bootloaders or kernels -- a rootkit can't survive if it can't pass signature verification.

**Q: What is a TPM and how does it relate to disk encryption?**

A: A TPM (Trusted Platform Module) is a hardware chip that securely stores cryptographic keys and platform measurements. During boot, each stage hashes the next component and extends the hash into TPM PCR (Platform Configuration Register) values. For disk encryption, the encryption key is "sealed" to specific PCR values. The TPM only releases the key if the current PCR values match what was expected -- meaning the boot chain hasn't been tampered with. If an attacker modifies the bootloader, the PCRs change, and the TPM refuses to release the disk encryption key.

**Q: What is dm-verity and where is it used?**

A: dm-verity is a Linux kernel feature that verifies the integrity of read-only block devices using a Merkle hash tree. Each block is hashed, and hashes are combined into a tree structure with a single root hash. When any block is read, its hash is verified up the tree to the trusted root hash. If a single byte is changed, verification fails. It's used by Android (verified boot for system partitions), ChromeOS (rootfs integrity), and container runtimes (image verification). The limitation is it only works for read-only data -- you can't use it on partitions that need to be written to.

**Q: Why does boot integrity matter for cloud environments?**

A: In cloud environments, you're running on shared hardware you don't control. Without Secure Boot, a compromised VM could install a bootkit that persists across reboots and OS reinstalls, potentially accessing data from other tenants or extracting secrets from memory. Secure Boot ensures only signed, trusted code runs from firmware through the kernel. Combined with confidential computing technologies (AMD SEV, Intel TDX), even the cloud provider's hypervisor can't read your VM's memory. This is critical for regulated industries processing sensitive data in the cloud.

---

This concludes the Security and Protection section. These mechanisms form a defense-in-depth stack: hardware roots of trust (Secure Boot, TPM) at the bottom, through kernel enforcement (MAC, seccomp, capabilities), up to application-level sandboxing (namespaces, containers). No single layer is sufficient -- security comes from combining all of them.

**Return to:** [Security and Protection Overview](README.md)

# Running x86_64 CTF Practice Images on an Apple Silicon Mac

How I got my Cybersecurity Club's VMware practice images (built for Intel/AMD PCs) running on an ARM-based MacBook Pro, and the troubleshooting process behind it.

**Author:** Teylor Bean ([@dull-scythe](https://github.com/dull-scythe)) · **Status:** ✅ Working (Linux and Windows images)

---

## TL;DR

| | |
|---|---|
| **Problem** | Practice images are x86_64 VMs; my M1 Pro is ARM64. |
| **Root cause** | Virtualization requires matching CPU architectures, so x86_64 guests can't be virtualized on Apple Silicon. |
| **Solution** | Emulate x86_64 with **UTM** (QEMU) in *Emulate* mode, after converting each `.vmdk` to `.qcow2` with `qemu-img`. |
| **Trade-off** | Works reliably but is slower than native, since every instruction is translated in software. |

---

## Environment

| Item | Details |
|---|---|
| Host | MacBook Pro, Apple M1 Pro (ARM64), macOS _<version>_ |
| Emulator | UTM (QEMU 10.0 backend), Emulate mode |
| CLI tools | QEMU via Homebrew (`qemu-img`) |
| Image 1 | `Debian10-BloonsTD6`: Debian 10, x86_64, VirtIO disk |
| Image 2 | Windows 10 (`guestOS = windows9-64`), x86_64, UEFI |

---

## Troubleshooting Process

Each step follows **Observation → Hypothesis → Test → Result.**

### Step 1: Identify why the images won't run
- **Observation:** The VMware images (`.vmdk` + `.vmx`) wouldn't open on my Mac.
- **Hypothesis:** I first thought the images needed Windows.
- **Analysis:** The real constraint is the **CPU instruction set**. The images contain an OS compiled for x86_64, and Apple Silicon is ARM64.

| | Virtualization | Emulation |
|---|---|---|
| How it works | Guest code runs directly on the real CPU | Software translates each instruction |
| Requirement | Guest ISA must match host ISA | Any ISA on any host |
| Speed | Near-native | Much slower |

- **Result:** These images must be **emulated**, not virtualized.

### Step 2: Rule out a Windows 11 ARM VM as a middle layer
- **Hypothesis:** Run Windows 11 ARM in a VM, then open the image inside it.
- **Analysis:** Windows on ARM translates individual x86 *apps*, not a hypervisor running a full x86 OS. Apple Silicon also lacks the nested virtualization this would need.
- **Result:** ❌ Rejected. An extra OS layer doesn't fix an architecture mismatch.

### Step 3: Rule out VMware Fusion
- **Hypothesis:** The images are VMware-format, so VMware Fusion should open them.
- **Test:** Opened Fusion's new-VM wizard with the image. It only offered **"Debian 10.x 64-bit Arm"** guest types.
- **Analysis:** On Apple Silicon, Fusion only virtualizes ARM guests and does not emulate x86. Rosetta 2 translates Mac apps, not virtual machines.
- **Result:** ❌ Rejected. *Matching file format ≠ matching CPU architecture.*

### Step 4: Choose QEMU emulation via UTM
- **Analysis:** QEMU's TCG (Tiny Code Generator) translates x86_64 instructions to ARM64 at runtime. UTM is a macOS front-end that bundles QEMU.
- **Correction to a generic AI answer:** A Google AI Overview suggested `-enable-kvm`, `-accel hvf`, and `if=virtio`. None of these fit my setup. KVM is Linux-only, HVF only accelerates guests that match the host architecture, and Windows has no built-in VirtIO drivers.
- **Result:** ✅ Selected UTM in **Emulate** mode with architecture **x86_64**.

### Step 5: Fix a failed disk import (Image 1, Debian)
- **Observation:** UTM error on boot:
  `Could not open '.../Data/2-Debian10-BloonsTD6.qcow2': No such file or directory`
- **Analysis:** The VM's size showed **524 KB**, so the bundle held only config files and no disk. UTM's built-in `.vmdk` import had **silently failed**.
- **Test:** Inspected and converted the disk manually:
  ```bash
  brew install qemu
  qemu-img info disk.vmdk
  qemu-img convert -p -f vmdk -O qcow2 disk.vmdk Debian10-BloonsTD6.qcow2
  ```
  Deleted the broken drive entry in UTM and imported the new `.qcow2`.
- **Result:** ✅ **Booted successfully.** The image turned out to be **Debian 10**, not Windows, which confirmed the real requirement was x86_64, not a specific OS.

### Step 6: Address performance
- **Observation:** Noticeable input lag when typing in LibreOffice.
- **Analysis:** Expected under full software emulation; GUI redraws are especially costly.
- **Tuning options tested or considered:**

  | Setting | Effect | Notes |
  |---|---|---|
  | More CPU cores | More parallel translation | |
  | Force Multicore | Can improve speed | ⚠️ UTM docs warn correctness isn't guaranteed for x86-on-ARM; disable if unstable |
  | JIT cache size | Caches translated code | Diminishing returns past a point |
  | `virtio-vga-gl` display | Offloads 3D drawing to the Mac's GPU | Linux guests only |
  | Retina Mode off | Fewer pixels to render | |
  | SSH instead of GUI | Skips desktop rendering entirely | Best for terminal-based challenges |

- **Result:** ⚙️ Usable, and good enough for practice.

### Step 7: Fix "No bootable device" (Image 2, Windows)
- **Observation:** After converting and importing the second image, the VM reported **"No bootable device."**
- **Hypotheses, ranked by likelihood:** (1) BIOS/UEFI mismatch, (2) incomplete conversion, (3) drive order or interface.
- **Test:** Checked the VMware config:
  ```bash
  grep -i -E 'firmware|guestOS' image.vmx
  ```
  - `firmware = "efi"` meant the image boots with UEFI, but I had turned UEFI off during setup.
  - `guestOS = "windows9-64"` is VMware's internal ID for **Windows 10** (64-bit), so no TPM or Secure Boot is needed.
- **Fix:** Re-enabled **UEFI Boot** (VM settings → QEMU).
- **Result:** ✅ **Booted successfully.** Firmware is set per image, so it must be checked every time.

---

## Final Working Configuration

| Setting | Linux guest (Debian 10) | Windows guest (Windows 10) |
|---|---|---|
| Mode / OS type | Emulate / Other | Emulate / Other |
| Architecture / System | x86_64 / Q35 | x86_64 / Q35 |
| Memory | 8192 MiB | 8192 MiB |
| UEFI Boot | Per `.vmx` | **On** (`firmware = "efi"`) |
| Drive interface | **VirtIO** | **SATA** or **IDE** (no VirtIO drivers) |
| Display | `virtio-vga` / `virtio-vga-gl` | `VGA` (fallback `cirrus-vga`) |
| Network | e1000 | e1000 |
| Clock | Default | Use local time for base clock |

## New Image Checklist

```bash
# 1. Unpack if needed
tar -xvf image.ova

# 2. Check for VMware encryption (no output = not encrypted)
grep -i encryption image.vmx

# 3. Identify the guest OS and firmware
grep -i -E 'guestOS|firmware' image.vmx

# 4. Verify the disk is readable
qemu-img info image.vmdk

# 5. Convert (keeps the original untouched)
qemu-img convert -p -f vmdk -O qcow2 image.vmdk image.qcow2
```
Then create the UTM VM (Emulate → Other → x86_64), import the `.qcow2`, match the UEFI setting to the `.vmx`, choose the drive interface for the guest OS, confirm the VM size shows **GB, not KB**, and clone the VM as a clean baseline before practicing.

---

## Lessons Learned

1. **Identify the architecture first.** It decides between virtualization and emulation and rules out most tools immediately.
2. **Verify assumptions about the target.** I assumed Windows; the first image was Debian.
3. **Read the evidence, not just the error.** "No such file" pointed to a missing disk; the 524 KB size explained *why*.
4. **GUI tools can fail silently; CLI tools report why.** Converting manually with `qemu-img` gave a verifiable result.
5. **Config files hold the answers.** Two `grep` commands on the `.vmx` revealed the firmware type and OS version.
6. **Adapt generic AI answers to your environment, and verify them against primary docs.**
7. **Protect the original.** Convert to a new file and keep the source image intact.

---

## References

- [UTM: QEMU System settings](https://docs.getutm.app/settings-qemu/system/): architecture matching, Force Multicore, JIT cache
- [UTM: QEMU settings](https://docs.getutm.app/settings-qemu/qemu/): UEFI Boot, local-time clock
- [QEMU: System emulation introduction](https://www.qemu.org/docs/master/system/introduction.html): accelerators (KVM, HVF, TCG)
- [QEMU: `qemu-img` manual](https://www.qemu.org/docs/master/tools/qemu-img.html)
- [Airbus Security Lab: QEMU internals blog](https://github.com/airbus-seclab/qemu_blog): how TCG translation works
- [Broadcom: VMware Guest OS identifiers](https://developer.broadcom.com/xapis/vsphere-web-services-api/8.0U2/vim.vm.GuestOsDescriptor.GuestOsIdentifier.html)
- [Red Hat Bugzilla: VMware firmware in `.vmx`](https://bugzilla.redhat.com/show_bug.cgi?id=1217444)
- [Proxmox Wiki: Windows VirtIO Drivers](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)
- [Broadcom Community: x86 VM on an Arm host error](https://community.broadcom.com/vmware-cloud-foundation/discussion/error-message-this-virtual-machine-cannot-be-powered-on-because-it-requires-the-x86-machine-architecture-which-is-incompatible-with-this-arm-machine-architecture-host)

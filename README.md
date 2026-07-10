# Ubuntu 26.04 + NVIDIA Driver + CUDA Setup

**Hardware:** Dell workstation, NVIDIA RTX PRO 6000 Blackwell (96 GB)
**OS:** Ubuntu 26.04 LTS

This is the working procedure, in the order that actually succeeds. Doing the
BIOS steps *first* avoids the display-hang ordeal entirely.

---

## Key lessons (read first)

1. **The monitor must be plugged into the NVIDIA card's port**, and BIOS must
   initialize display on the discrete GPU. If the monitor is on the motherboard
   (onboard) output, the NVIDIA driver's mode-setting appears to "hang" — it's
   actually rendering to a port with no monitor attached.
2. **Above 4G Decoding + Resizable BAR must be enabled** in BIOS. The 96 GB card
   needs a large memory window; without it the driver hangs at display init.
3. **Keep Secure Boot in one state the whole time.** Toggling it mid-setup caused
   keyboard/boot problems. If Secure Boot is ON, you must enroll the driver's MOK
   key (see below) or the module is rejected ("Key was rejected by service").
4. **`nomodeset` is temporary.** Use it only to boot until the driver works, then
   remove it. `nvidia-smi` working does NOT mean the display works — that's a
   separate step.
5. **Change one thing at a time and verify before moving on.**

---

## Step 0 — BIOS settings (do these first) - GPU Optimization

Enter BIOS (tap **F2** at the Dell logo) and set:

- **Above 4G Decoding → Enabled**  *(critical for the 96 GB card)*
- **Resizable BAR / Smart Access Memory → Enabled**
- **Primary Display / Video → leave on Auto for now.** You switch this to the
  NVIDIA GPU in **Step 8**, *after* the driver + MOK are in place. Switching it
  too early can leave the Ubuntu installer with a blank screen. (If the installer
  is blank anyway, set it back to Auto/onboard to get through the install.)
- **CSM / Legacy Option ROMs → Disabled** (pure UEFI)
- **Secure Boot →** pick a state and keep it (ON is fine; enroll MOK in Step 7)

**Connect the monitor to a port on the NVIDIA card**, not the motherboard.

Save and exit.

---

## Step 1 — BIOS setting - Storage initialization

Enter BIOS again, or keep the same session as **Step 0**:

- Click on **Advanced Setup**, if you didn't before (top left corner)
- Set **SATA/NVMe Operation** (Disk mode) to **AHCI / NVMe** — **not "RAID On"**.
  If it's left on RAID, the Ubuntu installer won't detect the SSD.
- Apply changes (bottom)
- Exit

---

## Step 2 — Install Ubuntu

- Boot the Ubuntu 26.04 installer. If the installer screen is blank/unstable,
  at the GRUB boot menu press **`e`**, add **`nomodeset`** to the `linux` line,
  then **Ctrl+X**.
- During install, do **NOT** tick "Install third-party graphics drivers"
  (it ships a driver too old for Blackwell).
- Consider skipping full-disk encryption unless you need it (the LUKS passphrase
  prompt caused keyboard trouble early in boot).

First boot: if it hangs, use the GRUB `e` → add `nomodeset` trick again.

---

## Step 3 — Make GRUB recoverable + update system

Always keep the GRUB menu visible so you're never stranded:

```bash
sudo nano /etc/default/grub
```
Set:
```
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
```
Then:
```bash
sudo update-grub
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y build-essential dkms linux-headers-$(uname -r)
sudo reboot
```

---

## Step 4 — Blacklist Nouveau

```bash
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
```

---

## Step 5 — Add NVIDIA's CUDA repo (Ubuntu 26.04)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
```

---

## Step 6 — Install the open driver (Blackwell needs the 610+ branch)

```bash
sudo apt install -y nvidia-open
```

The repo `nvidia-open` is DKMS-built and signed with a **local key** (not
Canonical's). With Secure Boot ON, the kernel rejects it until you enroll that
key (Step 7), so right now `sudo modprobe nvidia` will fail with **"Key was
rejected by service"** — that's expected. Continue to Step 7.

---

## Step 7 — Enroll the driver's signing key (MOK) — required with Secure Boot ON

Skip this step **only** if Secure Boot is disabled (`mokutil --sb-state` tells
you). With Secure Boot on, this is required or the driver won't load.

**1. Find the key and import it:**
```bash
mokutil --sb-state                      # confirm "SecureBoot enabled"
ls -la /var/lib/shim-signed/mok/        # look for MOK.der
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
```
Set a **one-time password** (something short you'll remember for two minutes).
If `MOK.der` isn't there, check `/var/lib/dkms/mok.pub` and import that instead.
To confirm which key signed the module: `modinfo nvidia | grep sig_key`.

**2. Reboot and enroll at the blue screen:**
```bash
sudo reboot
```
At the blue **MOK Management** screen: **Enroll MOK → Continue → Yes → enter the
password → Reboot**. (Your keyboard works here because Secure Boot is on.)

**3. Verify the driver loads** (do this *after* you can log in — see the note):
```bash
sudo modprobe nvidia
nvidia-smi            # should list the RTX PRO 6000 Blackwell, no key error
```
> After the MOK reboot the machine will most likely **get stuck at boot** —
> that's expected, because the driver now loads and tries to drive the display
> while the BIOS is still outputting on Auto/onboard. Go to **Step 8** to point
> the display at the GPU, *then* come back and run this verification.
> `nvidia-smi` working confirms compute; the display is enabled in Steps 8–9.

---

## Step 8 — Changing Display to run on GPU

After the MOK reboot (Step 7) the machine will most likely **get stuck at boot**,
because the NVIDIA driver is now loading and trying to drive the display while
the BIOS is still set to Auto/onboard. Fix it by pointing the display at the
NVIDIA GPU:

- Press **F2** while booting to enter the BIOS;
- Go to **Display** (or Primary Display / Video);
- Select the **NVIDIA GPU** option (instead of **Auto**);
- Apply changes;
- Exit.

The machine should now boot to the login. Go back to **Step 7 → part 3** and run
`nvidia-smi` to confirm the driver loaded, then continue to Step 9.

---

## Step 9 — Remove `nomodeset` and enable the desktop

With the monitor on the GPU + BIOS set to init display on the GPU (Step 8), the
driver can now drive the display.

**If the desktop already came up after Step 8**, `nvidia_drm` is already loaded
and driving the display. `sudo modprobe -r nvidia_drm` will then report
**"Module nvidia_drm is in use"** — that's fine and expected. **Skip the reload
test below** and jump straight to "make it permanent".

Otherwise, test it safely **before** editing GRUB:

```bash
sudo systemctl enable gdm
sudo systemctl set-default graphical.target
sudo systemctl stop nvidia-persistenced
sudo modprobe -r nvidia_drm
sudo modprobe nvidia_drm modeset=1     # should NOT hang now (thanks to BIOS fix)
sudo systemctl start gdm               # graphical login should appear
```

Make it permanent (remove `nomodeset`):
```bash
sudo sed -i 's|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"|' /etc/default/grub
sudo update-grub
sudo reboot
```

It should now boot straight to the graphical login on its own.

---

## Step 10 — Install the CUDA toolkit

Install the toolkit from the NVIDIA repo (gives `nvcc` + CUDA 13.x libraries):

```bash
sudo apt install -y cuda-toolkit
```

Then add CUDA to your PATH. **Edit `~/.bashrc` by hand** — do NOT use
`echo '...' >> ~/.bashrc` for this (chaining two `echo` appends on one line
mangles them into a single broken line and corrupts your PATH):

```bash
nano ~/.bashrc
```
Add these two lines, on **separate lines**, at the end (quotes included):
```
export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda/lib64:$LD_LIBRARY_PATH"
```
Save, then:
```bash
source ~/.bashrc
nvcc --version        # should show CUDA 13.x
```

Notes:
- **Do NOT install Ubuntu's `nvidia-cuda-toolkit`** — it ships an older CUDA
  that can't target Blackwell and mismatches the 610 driver. Use the repo
  `cuda-toolkit` above (installs to `/usr/local/cuda`).
- `nvcc` is only needed if you **compile CUDA code yourself**. PyTorch /
  TensorFlow / vLLM bundle their own CUDA runtime and just need the working
  driver — so you can skip this step if you only run ML frameworks.

### PATH recovery (if you ever break it)
If a shell says a command "is available in /usr/bin but not in PATH", `export`
is a built-in and still works with a broken PATH — restore it, then fix
`~/.bashrc`:
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/cuda/bin
```

---

## Step 11 — Final verification

```bash
cat /proc/cmdline        # nomodeset gone
nvidia-smi               # GPU + driver 610, driving the display
nvcc --version           # only if you installed the toolkit
# optional compute test if PyTorch is installed:
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

Reboot once more to confirm it comes up to the desktop unattended. Done.

---

## Recovery cheat-sheet (if a boot fails)

- **Force GRUB menu:** tap **`Esc`** right after the Dell logo. If you land at
  `grub>`, type `normal` (or `configfile (hd0,gpt2)/boot/grub/grub.cfg`).
- **Boot to safe text mode:** at GRUB press `e`, add to the `linux` line:
  `nomodeset nvidia-drm.modeset=0 systemd.unit=multi-user.target`, then Ctrl+X.
- **Repair GRUB from within Ubuntu:** `sudo grub-install /dev/nvme0n1 && sudo update-grub`
- **Rule:** don't remove `nomodeset` until `nvidia-smi` works AND the monitor is
  on the GPU with BIOS display-init set to the GPU.

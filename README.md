# Ubuntu 26.04 + NVIDIA Driver + CUDA Setup

**Hardware:** Dell workstation, NVIDIA RTX PRO 6000 Blackwell (96 GB)
**OS:** Ubuntu 26.04 LTS

This is the working procedure, in the order that actually succeeds. The single
biggest simplification: **plug the monitor's DisplayPort straight into the GPU.**
With that, the install is smooth — no `nomodeset`, no BIOS display juggling.

---

## Key lessons (read first)

1. **Connect the monitor directly to a DisplayPort on the NVIDIA card** (not the
   motherboard). This is the most important thing — it makes the whole install
   go smoothly, with **no `nomodeset` needed** and BIOS **Primary Display on Auto**.
2. **Above 4G Decoding + Resizable BAR** — enable them if your BIOS exposes them.
   On many Dell Precision workstations they're **auto-enabled and not shown**,
   which is fine; don't hunt for a setting that isn't there.
3. **Keep Secure Boot in one state the whole time.** Toggling it mid-setup causes
   keyboard/boot problems. If Secure Boot is ON, you must enroll the driver's MOK
   key (Step 6) or the module is rejected ("Key was rejected by service").
4. **`nomodeset` is only a fallback.** With the monitor on the GPU you shouldn't
   need it. If a boot ever hangs with a blank screen, it's the temporary escape
   hatch (see Recovery cheat-sheet), not a normal step.
5. **Change one thing at a time and verify before moving on.**

---

## Step 0 — BIOS settings: GPU

Enter BIOS (tap **F2** at the Dell logo) and set:

- **Above 4G Decoding → Enabled** *(if present; often auto/hidden on Dell — fine)*
- **Resizable BAR / Smart Access Memory → Enabled** *(if present)*
- **Primary Display / Video → Auto** (leave it — with the monitor on the GPU,
  Auto works throughout install and after)
- **CSM / Legacy Option ROMs → Disabled** *(if present; pure UEFI)*
- **Secure Boot →** pick a state and keep it (ON is fine; enroll MOK in Step 6)

**Connect the monitor to a DisplayPort on the NVIDIA card**, not the motherboard.

Save and exit.

---

## Step 1 — BIOS settings: Storage

Enter BIOS again, or keep the same session as **Step 0**:

- Click **Advanced Setup** if you haven't (top-left / top-right toggle)
- Click **Storage**
- Set **SATA/NVMe Operation** (Disk mode) to **AHCI / NVMe** — **not "RAID On"**.
  If it's left on RAID, the Ubuntu installer won't detect the SSD.
- Apply changes (bottom)
- Exit

---

## Step 2 — Install Ubuntu

- On boot, press **F12** to get the one-time boot menu.
- Choose the USB entry (e.g. **UEFI USB SanDisk**).
- Select **Ubuntu (safe graphics)** to be safe.
- Run the installer.
  - During install, do **NOT** tick "Install third-party graphics drivers"
    (it ships a driver too old for Blackwell).
  - Consider skipping full-disk encryption unless you need it (the LUKS
    passphrase prompt can cause keyboard trouble early in boot).
- *Fallback only:* if the installer or first boot is blank/stuck, at the GRUB
  menu press **`e`**, add **`nomodeset`** to the `linux` line, then **Ctrl+X**.
  With the monitor on the GPU you normally won't need this.

Then update the system:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y build-essential dkms linux-headers-$(uname -r)
sudo reboot
```

---

## Step 3 — Blacklist Nouveau

```bash
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
```

---

## Step 4 — Add NVIDIA's CUDA repo (Ubuntu 26.04)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
```

---

## Step 5 — Install the open driver (Blackwell needs the 610+ branch)

```bash
sudo apt install -y nvidia-open
```

The repo `nvidia-open` is DKMS-built and signed with a **local key** (not
Canonical's). With Secure Boot ON, the kernel rejects it until you enroll that
key (Step 6), so right now `sudo modprobe nvidia` will fail with **"Key was
rejected by service"** — that's expected. Continue to Step 6.

---

## Step 6 — Enroll the driver's signing key (MOK) — required with Secure Boot ON

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

---

## Step 7 — Verify the driver and desktop

After the MOK reboot, with the monitor on the GPU it should boot straight to the
graphical login. Confirm everything is up:

```bash
sudo modprobe nvidia
nvidia-smi            # should list the RTX PRO 6000 Blackwell, no key error
cat /proc/cmdline     # should be clean (no nomodeset)
```

`nvidia-smi` listing the card = compute works and the driver is driving the
display. If instead the boot hangs on a blank screen, see the Recovery
cheat-sheet.

---

## Step 8 — Install the CUDA toolkit

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

## Step 9 — Final verification

```bash
nvidia-smi               # GPU + driver 610, driving the display
nvcc --version           # only if you installed the toolkit
# optional compute test if PyTorch is installed:
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

Reboot once more to confirm it comes up to the desktop unattended. Done.

---

## Firefox crashes on this new GPU (optional)

If Firefox crashes on launch, force software rendering (write the prefs from a
terminal, no GUI needed):
```bash
pkill -i firefox; sleep 2
PROFILE=$(ls -d ~/snap/firefox/common/.mozilla/firefox/*.default* 2>/dev/null | head -1)
[ -z "$PROFILE" ] && PROFILE=$(ls -d ~/.mozilla/firefox/*.default* 2>/dev/null | head -1)
cat >> "$PROFILE/user.js" <<'EOF'
user_pref("gfx.webrender.software", true);
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.all", false);
EOF
firefox &
```
Remove those lines later once you want GPU acceleration back.

---

## Recovery cheat-sheet (if a boot fails)

- **Force GRUB menu:** tap **`Esc`** right after the Dell logo. If you land at
  `grub>`, type `normal` (or `configfile (hd0,gpt2)/boot/grub/grub.cfg`).
- **Boot to safe text mode:** at GRUB press `e`, add to the `linux` line:
  `nomodeset nvidia-drm.modeset=0 systemd.unit=multi-user.target`, then Ctrl+X.
- **Repair GRUB from within Ubuntu:** `sudo grub-install /dev/nvme0n1 && sudo update-grub`
- **Make the GRUB menu always show** (easier recovery): in `/etc/default/grub`
  set `GRUB_TIMEOUT_STYLE=menu` and `GRUB_TIMEOUT=10`, then `sudo update-grub`.
- **If the display hangs:** confirm the monitor is on the **GPU DisplayPort**.
  That single connection is what makes this whole setup work without `nomodeset`.

# About
Hi! This repository is a short guide to let you:
- turn on devices remotely
- establish ssh connection "outside" of your LAN

As an end result you should be able to connect to your Omarchy desktop from anywhere! Tested on **Omarchy V3.8.2**. Why a special guide? Because the LUKS encryption makes the Omarchy pretty hard to setup.

> [!WARNING]
> Remember this is just a random respository, my proposed solution can be unsafe and might require some tweaking. Proceed with caution, make frequent backups :)

The guide will be split into two parts:
- Remote Power-On
- Remote Access
Each of them is independant, so if you just want one of the mechanism feel free to skip the other. 

> [!TIP]
> If you will encounter any problems please look into [Issues Section](#issues), I provided some of the solutions I found useful

## My Setup
Just to be clear, this is one of many possible solutions, but to keep the assumptions and constraints of this guide clear please acquiant yourself with this part. 

The setup includes:
- **router**
- **client device** -- the remote device that will connect to the target device
- **secondary device** -- can be microcontrolelr, other PC, server, anything else that is able to hold a ssh connection. This device is always turned on and achievable. 
- **target device** -- the device we will connect to

All devices besides client device are in the same LAN.  

### Connecitivty and LAN
To connect to the target the client has to first access the secondary-device. In my setup it's made so that it's achievable outside of my home network. To achieve something similar you can use VPNs, SSH tunneling or whatever else you would like. There are easy to use clients such as Headscale or NetBird. Again choice is up to you :)

Then from the secondary-device it's possible to connect to the target.

# Part 1: Remote Power-On (WOL)
There are at least two most common solutions to the remote power-on problem:
- smart plugs -- simply make the power-on when voltage is present + smart plug that can be remotely operated 
- WOL (Wake-On-LAN) -- the idea is that the device can be listening to the traffic and if *magic packet* is detected, the device will begin booting. 

For my setup I chose the WOL approach (yes I am avid smarthouse hater), meaning that I always have to have one powered-on device in my LAN. 

## Requirements and Warnings
Requirements for this part:
- device in same LAN as the target
- motherboard supporting WOL (Wake-On-Lan) -- please consult documentation or/and manufacturer websites

> [!WARNING]
> This part doesn't include hardening and in-depth cybersecurity advice. Please educate yourself about the risks that come with WOL.

## Step 1: Setting up the interface
Before we start messing up with the BIOS, let's configure the interface. 

> [!TIP]
> Don't know your motherboard or details about the controller? Try: 
> ```sh
> lspci -k | grep -A4 -i ethernet
> ```
> If there is not enough info expand the N -A<N> parameter 

First confirm which interface you will be using -- it's probably best that you pick the wired connection. You can get your interface name and other details by running:
```sh
ip link
```
From this step note: *interface name* and *MAC Address*. Next let's inspect the state of the interface, run:
```sh
sudo ethtool <interface_name>
``` 
Now look for two settings:
- `Supports Wake-on` expected value after config: `pumbg` 
- `Wake-on` expected value after config: `g`
For now you might get values like `d` or `none`, if that's the case run:
```sh
sudo ethtool -s <interface_name> wol g
```
Re-check if the state of the interface now looks as expected. 

Now we changed the settings temporarily, but to make sure that they are propagated for each boot, we will create a service and enable it. First, we will create the configuration file:
```ini
sudo tee /etc/systemd/system/wol@.service << 'EOF'
[Unit]
Description=Wake-on-LAN for %i
Requires=network.target
After=network.target

[Service]
ExecStart=/usr/bin/ethtool -s %i wol g
Type=oneshot

[Install]
WantedBy=multi-user.target
EOF
```
> [!NOTE]
> The configuration files such as the one above can be found in the `/configFile` directory of this repository :)

After that we start the service:
```sh
sudo systemctl enable wol@<interface_name>.service
sudo systemctl start wol@<interface_name>.service
```
Great! Now the service should be up and running. We are ready for manipulating the BIOS.

## Step 2: BIOS Settings
All BIOSes look different and the options depend on your parts, however there are  settings you should pay attention to:
- Wake-On-LAN -- state: `enable`
- ErP Ready -- state: `disable`
- Deep Sleep -- state: `disable`
Now there might be additional settings that you need to include, the idea is that while your device is in S5 state not enough power can be delivered to your NIC. That will cause the WOL mechanism to fail as it won't have enough energy to bring the system up. 

## Step 3: Final Checkup
Now it's the time to check if everything works as it should. After rebooting the device, power it off. We will send magic packets from your client device, for that you usually need to have some package/app. For Linux a pretty popular choice is `wakeonlan`. 
> [!Note]
> The packet needs to be send by device in your LAN, make sure that requirement is satisfied. 

So how does it work? Simple!
```sh
wakeonlan <MAC_ADDR> 
```
Or 
```sh
wakeonlan -i <IP_ADDR> <MAC_ADDR>
```
Voila! Your device should turn on now! 

### Troubleshooting
If it doesn't, turn the device manually and perform following test. Install `tcpdump` and start listening on the target device:
```sh
sudo tcpdump -vv -s0 -n udp port 9
```
Now resend the magic packet from client device. If you see a hit such as:
```sh
<sender-ip>.<port> > <your-ip-or-broadcast>.9: UDP, length 102
```
Then reinvestigate the target device settings:
1. Check the interface settings
2. Check if the created service is up
3. Recheck BIOS settings, look for options that might limit the NIC power.

**If the packet doesn't arrive** then the issue is most likely rooted in your router. Go to step 4.

## Step 4: Router Config
Some routers might restrict WOL, so this is a good first failpoint to inspect if the magic packets don't arrive. It's hard to give exact guide, since the interface depends heavily on the model and manufacturer, so you need to look for WoL. 

Additionally you might consider:
- **Static IP/Address Reservation** -- Useful for port forwarding and more repeatable experience
- **Port Forwarding** -- you might need to forward the packets. Create following service on your router:
```ini
        External Port: 9
        Internal Port: 9
        Protocol:      UDP
        IP:            <insert static IP>
```

# Part 2: Remote Access
Now, this part of the guide is the most risky, make sure to have a live-USB with Arch in your hands reach. We will manipulate the kernel configs so make sure to:
a) Not panic
b) Keep backup of the files
c) If you feel uneasy make backup of the whole drive

Before I take you through the steps, I will explain the overall idea of the setup. Main issue with setting up the remote Omarchy access is the disk encryption (LUKS), only after you log-in the disk is decrypted and you can access the data within. So the idea is to access the PC twice. 

First access is during the boot process, we setup a small SSH server to be able to put in the passphrase. After that the system boots normally and we begin the second session.

The second session has nothing special about it, you do your work and exit (if you also have remote power-on you can of course shutdown the device). 

> [!WARNING]
> Since the first step we are imapcting the ability to build good kernel image = ability to boot. **REALLY MAKE SURE YOU HAVE LIVE-USB**

## Step 1: Installing Packages 
First, I want you to know that we will change the dependencies for the initramfs. So the booting will proceed a bit differently than the usual, unfortunetly we will have to get rid of some of the default dependecies as a result. I couldn't make my interfaces go up in boot stage with those packages, hence the modification. We will install two packages:
```sh
sudo pacman -S mkinitcpio-systemd-extras dropbear
```
*You will most probably get warning about package conflict, proceed with the installation*. `mkinitcpio-systemd-extras` provides systemd utilities needed for LUKS encryption and networking. While `dropbear` will be used to provided the early SSH server. 

> [!NOTE]
> If you dislike dropbear for any reason, feel free to get any other lighweight SSH-server package

## Step 2: Writing Hooks Configuration
> [!CAUTION]
> There is a lot of paths and file names in this and next steps, please make sure you type everything correctly. Otherwise you will be stuck with debugging the most annoying thing known to man.

You have to know that Omarchy has two files that keep track of the hooks, namely `/etc/mkinitcpio.conf` and `/etc/mkinitcpio.conf.d/omarchy_hooks.conf`. Make copy of them now:
```sh
cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.bak
cp /etc/mkinitcpio.conf.d/omarchy_hooks.conf /etc/mkinitcpio.conf.d/omarchy_hooks.conf.bak
```
Great now we can work on creating the new config. 
```ini
etc/mkinitcpio.conf:

# HOOKS = (...)    #<- Comment this line -- we will define hooks in the omarchy_hooks.conf

# Add lines below -- They will add necessary files to the image
SD_NETWORK_CONFIG=/etc/systemd/network-initramfs

SD_DROPBEAR_AUTHORIZED_KEYS=/etc/dropbear/initrd.authorized_keys
SD_DROPBEAR_COMMAND="/bin/sh"

# You can change to port 22
SD_DROPBEAR_PORT=2222          

#Necessary to be able to decrypt the disk
BINARIES=(/usr/bin/cryptsetup)    
```

And in the second file we add the hooks, to not tempt with copying you I will just show you the things you need to modify. When you open your conf file you probably see something like:
```ini
HOOKS=(base udev plymouth keyboard autodetect microcode modconf kms keymap consolefont block encrypt filesystems fsck btrfs-overlayfs)
```
And we need to add some and delete some of them as seen below:
```ini
omarchy_hooks.conf:

HOOKS=(base <systemd> udev (...) block <sd-network> <sd-dropbear> <sd-encrypt> filesystems (...) )
```
So to sum up:
- add `systemd` after `base` 
- add `sd-network sd-dropbear sd-encrypt` after block 
- delete `encrypt` 

Small note, if you happen to have a bugged login screen you can get rid of `plymouth` too. 

## Step 3: Network Config
After we added the systemd-network we need to fill-out the profile for our interface. To do that we create new directory (please check if it's not already present):

```sh
sudo mkdir -p /etc/systemd/network-initramfs
```
And now we create profile for our interface:
```ini
sudo tee /etc/systemd/network-initramfs/10-wired.network << 'EOF'
[Match]
MACAddress=<nic-mac-address>

[Network]
DHCP=yes
EOF
```

## Step 4: Dropbear Setup
After we defined our interface we must also configure our dropbear SSH server. First, create SSH key on your *secondary-device* (the one that is on the same LAN as the target). Then on the target device create directory that will hold the keys, for example:
```sh
sudo mkdir -p /etc/dropbear
```
Copy the public key into the target device:
```sh
cp <path_to_pub_key> /etc/dropbear/initrd.authorized_keys
```

> [!CAUTION]
> If you want to change the name of the key or its location make sure it matches what we wrote in the config files in the previous step

## Step 5: Rebuilding Kernel Image
Now run two last commands:
```sh
sudo limine-mkinitcpio
```
Look for any errors and warnings during the building, troubleshoot accordingly. If everything seems fine:
```sh
sudo limine-update
```

## Step 6: Verification
Now what should happen is that you build your image correctly. And the next step is to confirm that all needed files are included in the build:

```sh
lsinitcpio /boot/EFI/Linux/omarchy_linux.efi | grep -E 'etc/systemd/network-initramfs|dropbear|authorized_keys'
```
You should see something like this:
```sh
etc/credstore/dropbear_ecdsa_host_key
etc/credstore/dropbear_rsa_host_key
etc/systemd/network-initramfs/
etc/systemd/network-initramfs/10-wired.network
root/.ssh/authorized_keys
usr/bin/dropbear
usr/lib/systemd/system/dropbear@.service
usr/lib/systemd/system/dropbear@.socket
```

## Step 7: Final Test
Now before the last step make sure you know the names of your disk partitions, if unsure run:
```sh
lsblk
```
Note the name of the partition with data.
---
If everything seems fine reboot your target device. 

---

Try connecting with your secondary-device, example:
```sh
ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes -p 2222 root@<machine-lan-ip>
```
That should establish connection with your device and the target device in the boot stage. After that you need to decrypt the disk:
```sh
cryptsetup open /dev/<parition_name> root  
```
Success!!!

# Issues 

## 1. Early-boot networking not working

**Symptoms**

- Machine never appears on LAN during initramfs.
- Router does not show the host.
- Ping and SSH to the machine’s LAN IP fail until after full OS boot.

**Causes**

- Wrong NIC driver loaded early (`rXXXX` instead of the desired driver).
- Busybox `netconf` hook was configured, but the interface name in initramfs did not match the name used in `ip=...` kernel parameter.

**Fixes**

- Ensure correct NIC driver in `MODULES` in `/etc/mkinitcpio.conf`, e.g.:

  ```sh
  MODULES=(<nic-driver>)
  ```

- Debug interface name in initramfs using a one-time kernel parameter (`break` or `break=netconf`), then adjust `ip=:::::<initramfs-iface>:dhcp` in the bootloader config to match the actual initramfs name.
- Ultimately, switch from busybox `netconf` to systemd-based initramfs networking via `systemd` + `sd-network` and a MAC-matched `.network` file.

---

## 2. netconf hook brittle / “no such device” errors

**Symptoms**

- Boot messages like:

  ```text
  ipconfig: eno1: SIOCGIFINDEX: No such device
  ipconfig: no device to configure
  ```

- Machine hangs at early boot, no usable network interface.

**Causes**

- netconf hook hard-coded to the wrong interface name (e.g. `eno1`) which did not exist in initramfs.
- Interface naming semantics differed between full OS and initramfs.

**Fixes**

- Inspect initramfs with `break`, use `ip link` inside initramfs to see actual names.
- Attempt to adjust `ip=:::::<initramfs-iface>:dhcp`.
- In practice, netconf proved fragile for this setup; replaced with `systemd` + `sd-network` (systemd-networkd in initramfs) and a MAC-based match unit:

  ```ini
  [Match]
  MACAddress=<nic-mac>

  [Network]
  DHCP=yes
  ```

---

## 3. mkinitcpio config layering (drop-ins overriding main config)

**Symptoms**

- Rebuild hooks showed `sd-network` and `sd-dropbear`, but `lsinitcpio` did not contain the expected config files (e.g. `systemd/network-initramfs/10-wired.network`).
- SD_* variables apparently ignored.

**Causes**

- `HOOKS` and SD_* variables defined in both `/etc/mkinitcpio.conf` and `/etc/mkinitcpio.conf.d/omarchy_hooks.conf`, with drop-ins overriding or shadowing main config.
- mkinitcpio was using the drop-in HOOKS but not the SD_* variables defined only in the main config, or vice versa.

**Fixes**

- Make `/etc/mkinitcpio.conf.d/omarchy_hooks.conf` the *only* source of HOOKS.
- Keep SD_* variables (`SD_NETWORK_CONFIG`, `SD_DROPBEAR_AUTHORIZED_KEYS`, `SD_DROPBEAR_COMMAND`, `SD_DROPBEAR_PORT`) in `/etc/mkinitcpio.conf` only.
- After rebuild, verify contents with:

  ```sh
  lsinitcpio /boot/EFI/Linux/<uki>.efi | grep -E 'systemd/network-initramfs|dropbear|authorized_keys|<nic-driver>'
  ```

---

## 4. SSH key and host key issues (Dropbear in initramfs)

**Symptoms**

- `Permission denied (publickey)` when SSHing into initramfs.
- Host key mismatch warnings:

  ```text
  WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
  Host key verification failed.
  ```

**Causes**

- `/etc/dropbear/initrd.authorized_keys` did not contain the correct client public key.
- Old known_hosts entry pointed to the fully-booted OS’s OpenSSH host key; Dropbear in initramfs uses different host keys.

**Fixes**

- Ensure `initrd.authorized_keys` has the exact line from the client’s `~/.ssh/id_ed25519.pub`.
- Set `SD_DROPBEAR_AUTHORIZED_KEYS=/etc/dropbear/initrd.authorized_keys` in `/etc/mkinitcpio.conf`.
- Clear old host key entry and accept the new one:

  ```sh
  ssh-keygen -R "[<lan-ip>]:2222"
  ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes -p 2222 root@<lan-ip>
  ```

---

## 5. Hidden / unusable LUKS prompt (black screen at boot)

**Symptoms**

- Machine appears stuck at a black screen during boot.
- Typing the disk password “blind” sometimes works; no visible prompt or feedback.

**Causes**

- Plymouth splash screens plus encrypt/sd-encrypt hooks obscured the password UI.
- Systemd initramfs crypt prompts rendered behind Plymouth or on a different VT.

**Fixes**

- Remove `plymouth` from initramfs HOOKS and use plain systemd text prompt.
- Use SSH into initramfs to unlock manually via `cryptsetup` when needed.

---

## 6. Manual unlock not continuing boot

**Symptoms**

- `cryptsetup luksOpen ...` appeared to succeed, but the system stayed at a black screen.
- No progress to root filesystem mount or user space.

**Causes**

- LUKS mapping name mismatched `cryptdevice=...:<name>` in kernel cmdline:
  - Bootloader expected `:root`.
  - Initramfs unlock used `cryptroot`, creating `/dev/mapper/cryptroot` instead of `/dev/mapper/root`.
- systemd initramfs only recognizes the device name specified in `cryptdevice`.

**Fixes**

- Unlock LUKS using the exact name expected by `cryptdevice`:

  ```sh
  cryptsetup luksOpen /dev/disk/by-partuuid/<partuuid> root
  cryptsetup status root
  ```

- Once `cryptsetup status root` reports `is active`, systemd can proceed to switch_root.

---

## 7. Missing `cryptsetup` binary in initramfs

**Symptoms**

- SSH into initramfs works.
- `cryptsetup` command is not found (`/bin/sh: cryptsetup: not found`), making manual unlock impossible.

**Causes**

- `sd-encrypt` hook present, but the CLI binary `/usr/bin/cryptsetup` not included in initramfs image.

**Fixes**

- Add `cryptsetup` to mkinitcpio BINARIES:

  ```sh
  BINARIES=(/usr/bin/cryptsetup)
  ```

- Rebuild UKI/initramfs with mkinitcpio.
- Verify presence with:

  ```sh
  lsinitcpio /boot/EFI/Linux/<uki>.efi | grep cryptsetup
  ```

---

## 8. Early remote unlock UX (systemd-tty-ask-password-agent hanging)

**Symptoms**

- Running `systemd-tty-ask-password-agent --query --watch` from the initramfs shell appears to “hang” (no prompt displayed).
- `--watch` version waits indefinitely.

**Causes**

- `--watch` uses a long-running mode designed to wait for future prompts; this looks like a hang if there is no pending job.
- Password prompt may be present but still not attached to the right TTY.

**Fixes**

- Use simple `--query`:

  ```sh
  systemd-tty-ask-password-agent --query
  ```

- If prompts remain unreliable, fall back to manual `cryptsetup` unlock from initramfs shell and let systemd proceed once `/dev/mapper/root` is active.

---

## 9. WOL regression after driver/initramfs changes

**Symptoms**

- WOL previously worked.
- After driver/initramfs changes, machine no longer wakes on magic packet.
- `ethtool <iface>` shows `Wake-on: g`, and `tcpdump` sees magic packets while the system is running.

**Causes**

- Deep sleep or ErP options in BIOS cut NIC power in S5, even though WOL is enabled.
- PCIe ASPM or other power-management quirks on some NICs can prevent wake despite correct WOL flags.

**Fixes**

- Confirm driver (`<nic-driver>`), WOL flags (`Wake-on: g`), and magic packets arriving:
  - `lspci -k | grep -A3 -i ethernet`
  - `sudo ethtool <iface>`
  - `sudo tcpdump -vv -s0 -n udp port 9`
- Check NIC LEDs after shutdown:
  - Lit/blinking LEDs → NIC has power; WOL possible.
  - Dark LEDs → NIC unpowered; WOL impossible until BIOS changes.
- In BIOS:
  - Enable “Wake on LAN”.
  - Disable “Deep Sleep”, “ErP Ready”, or similar options that cut NIC power in S5.
- Optionally disable PCIe ASPM via kernel parameter (`pcie_aspm=off`) if WOL remains unreliable.
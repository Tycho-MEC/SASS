# SendSpin Alpine - Manual Installation Guide

This guide walks you through installing SendSpin on Alpine Linux from scratch. This is useful for:
- Building your own custom image
- Understanding how the system works
- Troubleshooting issues
- Creating variations for different hardware

## Prerequisites

- Raspberry Pi 3 or 4
- MicroSD card (8GB minimum, 16GB recommended)
- USB Audio DAC
- Network connection (Ethernet recommended)
- Another computer to prepare the SD card

## Step 1: Download Alpine Linux

1. Visit https://alpinelinux.org/downloads/
2. Download the **armv7** version for Raspberry Pi: `alpine-rpi-{version}-armv7.tar.gz`
3. For Raspberry Pi 3, use armv7 (recommended) or armhf

## Step 2: Prepare SD Card

### Format SD Card

**Linux:**
```bash
# Find your SD card device (e.g., /dev/sdb)
lsblk

# Format as FAT32
sudo mkfs.vfat -F 32 /dev/sdX1
```

**Windows:**
- Use SD Card Formatter tool
- Format as FAT32

**macOS:**
- Use Disk Utility
- Format as MS-DOS (FAT)

### Extract Alpine to SD Card

```bash
# Extract the outer tar.gz
tar -xzf alpine-rpi-3.21.0-armv7.tar.gz

# You'll get another .tar file - extract that too
tar -xf alpine-rpi-3.21.0-armv7.tar

# Copy the contents (not the tar file itself) to SD card root
# The SD card should contain folders like: boot/, apks/, config.txt, etc.
```

## Step 3: First Boot and Basic Setup

1. Insert SD card into Raspberry Pi
2. Connect Ethernet cable
3. Power on and wait ~30 seconds
4. Find the Pi's IP address (check your router or use `nmap`)
5. SSH into the Pi:

```bash
ssh root@<pi-ip-address>
# Default password: (none, just press Enter)
```

### Run Alpine Setup

```bash
setup-alpine
```

Answer the prompts:
- **Keyboard layout:** us (or your preference)
- **Hostname:** sendspin (or your choice)
- **Network:** Use DHCP for eth0
- **Root password:** Set a secure password
- **Timezone:** Your timezone (e.g., America/New_York)
- **Proxy:** none
- **NTP client:** chrony
- **APK mirror:** Choose fastest (usually default)
- **SSH server:** openssh
- **Disk:** Choose your SD card (mmcblk0)
- **Mode:** sys (full installation to disk)

The system will install and reboot.

## Step 4: Enable Community Repository

After reboot, SSH back in and edit repositories:

```bash
vi /etc/apk/repositories
```

Uncomment the `community` line (remove the `#`):
```
http://dl-cdn.alpinelinux.org/alpine/v3.21/main
http://dl-cdn.alpinelinux.org/alpine/v3.21/community
```

Update package index:
```bash
apk update
```

## Step 5: Install System Dependencies

```bash
apk add python3 py3-pip alsa-utils alsa-lib
```

## Step 6: Install Build Dependencies

These are needed to compile Python packages:

```bash
apk add gcc musl-dev python3-dev build-base linux-headers \
    openblas-dev gfortran ffmpeg-dev pkgconfig libffi-dev \
    jpeg-dev zlib-dev portaudio-dev portaudio
```

## Step 7: Install SendSpin

```bash
pip install sendspin --break-system-packages
```

This will take 10-20 minutes on a Raspberry Pi 3 as it compiles numpy and other packages.

**Note:** The `--break-system-packages` flag is needed on Alpine. This is safe for a dedicated appliance.

## Step 8: Add SendSpin to PATH

If installing as a non-root user:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.profile
source ~/.profile
```

For system-wide installation, install as root and it will be in `/usr/local/bin`.

## Step 9: Configure USB Audio

### Check USB DAC is detected

```bash
aplay -l
```

You should see your USB device listed as card 0 or card 1.

### Create ALSA Configuration

```bash
vi /etc/asound.conf
```

Add this content (adjust card number if needed):

```
defaults.pcm.card 0
defaults.ctl.card 0

pcm.!default {
    type plug
    slave.pcm "hw:0,0"
}

ctl.!default {
    type hw
    card 0
}
```

**Important:** Change `card 0` to match your USB DAC number from `aplay -l`.

### Test Audio

```bash
speaker-test -c 2 -r 48000 -D hw:0,0
```

You should hear pink noise. Press Ctrl+C to stop.

## Step 10: Create SendSpin Service

Create the OpenRC service script:

```bash
vi /etc/init.d/sendspin
```

Add this content:

```bash
#!/sbin/openrc-run

name="SendSpin Audio Player"
description="SendSpin streaming audio player for Home Assistant"

command="/usr/local/bin/sendspin"
command_args="--headless"
command_user="root"
command_background=true
pidfile="/run/${RC_SVCNAME}.pid"

depend() {
    need net
    after firewall
}

start_pre() {
    checkpath --directory --owner $command_user --mode 0755 /run
}
```

**Note:** If you installed as a user, change the command path to `/home/username/.local/bin/sendspin` and adjust `command_user`.

Make it executable:

```bash
chmod +x /etc/init.d/sendspin
```

Enable and start the service:

```bash
rc-update add sendspin default
rc-service sendspin start
```

Check status:

```bash
rc-service sendspin status
```

## Step 11: Optional - Install sudo/doas

For easier administration:

```bash
apk add doas

# Configure doas
echo "permit persist :wheel" > /etc/doas.d/doas.conf

# Add a user (if desired)
adduser yourname
addgroup yourname wheel
```

## Step 12: Test SendSpin

SendSpin should now be running and visible to Music Assistant on your network. 

Check logs:
```bash
cat /var/log/messages | grep sendspin
```

Manual test (stop service first):
```bash
rc-service sendspin stop
sendspin --headless
# Press Ctrl+C to stop
rc-service sendspin start
```

## Step 13: Cleanup (Optional)

If you want to reduce image size, you can remove build dependencies after sendspin is installed:

```bash
apk del gcc musl-dev python3-dev build-base linux-headers \
    openblas-dev gfortran ffmpeg-dev pkgconfig libffi-dev \
    jpeg-dev zlib-dev portaudio-dev
```

**Warning:** Don't do this if you plan to install more Python packages later!

## Troubleshooting

### SendSpin command not found

- Check PATH: `echo $PATH`
- Find sendspin: `find / -name sendspin 2>/dev/null`
- Add to PATH or use full path

### Audio pops/crackles

Try increasing buffer size in `/etc/asound.conf`:

```
pcm.!default {
    type plug
    slave.pcm {
        type dmix
        ipc_key 1024
        slave {
            pcm "hw:0,0"
            period_size 2048
            buffer_size 16384
        }
    }
}
```

### Out of space during pip install

You might be in diskless mode. Rerun `setup-alpine` and choose `sys` mode for disk installation.

### USB DAC not detected

- Check USB connection
- Try different USB port
- Check: `dmesg | grep -i usb`
- Check: `lsusb`

### Service won't start

- Check logs: `cat /var/log/messages`
- Test manually: `sendspin --headless`
- Verify permissions on init script: `ls -la /etc/init.d/sendspin`

## Next Steps

Once you have a working system:
1. Test thoroughly with your USB DAC
2. Configure WiFi if needed: `setup-interfaces`
3. Change default passwords
4. Consider creating an image backup of your SD card
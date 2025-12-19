# SASS - Simple Apline SendSpin - Raspberry Pi Audio Streaming Player

A minimal Alpine Linux image for Raspberry Pi that runs [SendSpin](https://github.com/Sendspin/sendspin-cli) - a high-quality audio streaming protocol for Home Assistant's Music Assistant.

## Features

- **Minimal footprint** - Runs Alpine Linux in sys mode with only essential packages
- **USB DAC support** - Automatic ALSA configuration for USB audio devices
- **Auto-start service** - SendSpin starts automatically on boot in headless mode
- **Low latency** - Direct ALSA hardware access for minimal audio buffering
- **Raspberry Pi optimized** - Built specifically for Pi 3/4 hardware

## Supported Hardware

- Raspberry Pi 3 Model B / B+
- Raspberry Pi 4 Model B
- USB Audio DACs (USB Audio Class compliant devices)

## Quick Start

### Download Pre-built Image

Download the latest image from [Releases](https://github.com/yourusername/sendspin-alpine-image/releases)

### Flash to SD Card

**Linux/macOS:**
```bash
dd if=sendspin-alpine-0.1.0-armv7.img of=/dev/sdX bs=4M status=progress
sync
```

**Windows:**
Use [Rufus](https://rufus.ie/)

### First Boot

1. Insert SD card into Raspberry Pi
2. Connect USB DAC
3. Power on
4. The player will automatically connect to Music Assistant on your network
5. Default credentials: `root` / `alpine` (change immediately!)

## Building from Source

### Automated Build (Coming Soon)

An automated build script is in development. For now, use the manual installation method.

### Manual Installation

See [INSTALL.md](INSTALL.md) for complete step-by-step instructions to build your own image from scratch.

This method is recommended if you want to:
- Customize the installation
- Understand how the system works
- Build for different hardware configurations
- Troubleshoot issues

## Configuration

### Network Configuration

**Ethernet:** Works automatically via DHCP

**WiFi:** SSH into the device and configure:
```bash
setup-interfaces
rc-service networking restart
```

### Hostname

```bash
setup-hostname mynewname
rc-service hostname restart
```

### Audio Device

By default, the system uses the first USB audio device (card 0). To use a different device:

1. List audio devices: `aplay -l`
2. Edit `/etc/asound.conf` and change `card 0` to your device number
3. Restart sendspin: `rc-service sendspin restart`

## Project Structure

```
sendspin-alpine-image/
├── build.sh              # Main build script
├── config/
│   ├── asound.conf      # ALSA audio configuration
│   ├── sendspin.initd   # OpenRC service script
│   └── packages.txt     # List of packages to install
├── scripts/
│   └── setup.sh         # Post-install configuration
└── output/              # Build artifacts
```

## Troubleshooting

### Audio Issues (Pops/Crackles)

Check buffer settings in `/etc/asound.conf`. Increase buffer_size if needed:
```
buffer_size 16384
```

### SendSpin Not Starting

```bash
# Check service status
rc-service sendspin status

# View logs
cat /var/log/messages | grep sendspin

# Manual test
sendspin --headless
```

### USB DAC Not Detected

```bash
# List USB devices
lsusb

# Check ALSA devices
aplay -l

# Check kernel messages
dmesg | grep -i audio
```

## Roadmap

- [ ] Web UI for configuration
- [ ] Audio input capture and streaming
- [ ] WiFi configuration via boot partition config file
- [ ] Multiple USB DAC support
- [ ] Automatic updates

## Contributing

Contributions welcome! Please open an issue or pull request.

## License

MIT License - See LICENSE file for details

## Credits

- [SendSpin](https://github.com/Sendspin/sendspin-cli) by the SendSpin team
- [Alpine Linux](https://alpinelinux.org/) 
- [Home Assistant](https://www.home-assistant.io/) and [Music Assistant](https://music-assistant.io/)
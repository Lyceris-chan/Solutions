# CachyOS Configuration and Troubleshooting Guide

This guide provides optional configuration steps and troubleshooting fixes for 
system stability, hardware, and gaming performance on CachyOS. These are 
personal preferences and recommendations based on experience. Apply only the 
configurations that are relevant to your specific hardware and use case.

## System optimization

### Remove Plymouth boot splash (optional)

Plymouth can cause boot hangs and increase boot time on some systems. This is 
an optional optimization.

**Source:** [CachyOS Forum](https://discuss.cachyos.org/t/tutorial-disable-or-remove-plymouth-boot-splash/10922)

To remove Plymouth:

1.  Remove Plymouth packages:

    ```bash
    sudo pacman -Rns plymouth-git plymouth-kcm cachyos-plymouth-theme cachyos-plymouth-bootanimation
    ```

2.  Edit `/etc/mkinitcpio.conf` and remove `plymouth` from the `HOOKS` array.

3.  Rebuild the initramfs:

    ```bash
    sudo mkinitcpio -P
    ```

### Configure bootloader (optional)

You can reduce boot delay and set the default boot entry:

```bash
sudo bootctl set-timeout 0
sudo bootctl set-default linux-cachyos-rc.conf
```

### Add kernel parameters (recommended for Ryzen 7000 series)

The following kernel parameters can improve system stability and power 
management. Add these parameters to your boot configuration file (such as 
`/boot/loader/entries/linux-cachyos.conf` or `/etc/kernel/cmdline`):

```text
LINUX_OPTIONS="zswap.enabled=0 nowatchdog gpiolib_acpi.ignore_interrupt=AMDI0030:00@3,AMDI0030:00@10 mem_sleep_default=deep quiet"
```

**Parameter explanations:**

- `zswap.enabled=0` - Disables the zswap compressed swap cache. Disabling 
  zswap can improve performance and reduce memory overhead on systems with 
  adequate RAM.

- `nowatchdog` - Disables hardware watchdog timers. Watchdog timers can cause 
  unexpected system reboots or boot delays if they trigger incorrectly.

- `gpiolib_acpi.ignore_interrupt=AMDI0030:00@3,AMDI0030:00@10` - Ignores 
  specific GPIO interrupts from the AMDI0030 ACPI device. These GPIO pins can 
  cause spurious wakeups from suspend, which prevents your system from staying 
  asleep. The specific pins (3 and 10) are known to trigger false wake events 
  on Ryzen 7000 series systems.
  
  **Source:** [Arch Linux Wiki - Power Management Wakeup Triggers (Ryzen 7000 Series)](https://wiki.archlinux.org/title/Power_management/Wakeup_triggers#Ryzen_7000_Series)
  
  **For Ryzen 7000 series users:** If you have a Ryzen 7000 series CPU and 
  experience suspend issues, consult the Arch Wiki source above. The page 
  explains how to identify which specific GPIO pins are causing wakeup problems 
  on your system. The page also documents other common causes of broken suspend 
  and their solutions.

- `mem_sleep_default=deep` - Sets the default suspend mode to deep sleep (S3). 
  Deep sleep provides better power savings during suspend compared to shallow 
  sleep modes.

- `quiet` - Reduces boot messages shown on screen for a cleaner boot experience.

### Remove systemd timeout configuration (optional)

Strict systemd timeouts can cause slow shutdowns or boot delays on some 
systems.

To remove the timeout configuration:

```bash
sudo rm /usr/lib/systemd/system.conf.d/00-timeout.conf
```

### Configure time synchronization (preference)

This configuration removes connections to Cloudflare and Google NTP servers 
and uses only the Arch Linux NTP pool. This is a personal preference to avoid 
connecting to third-party corporate time servers.

To use only the Arch Linux NTP pool:

```bash
sudo sed -i \
  -e 's|^NTP=.*|NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org|' \
  -e 's|^FallbackNTP=.*|#FallbackNTP=|' \
  /usr/lib/systemd/timesyncd.conf.d/10-timesyncd.conf

sudo systemctl restart systemd-timesyncd
```

### Automount secondary drives (optional)

Use `x-systemd.automount` to prevent boot blocking when secondary drives fail 
to mount.

To configure automount:

1.  Find the UUID of your drive:

    ```bash
    lsblk -f
    ```

2.  Create the mount directory. Replace the path with your desired location:

    ```bash
    mkdir -p /home/sleepy/Desktop/Stuff
    ```

3.  Verify that the directory exists and the path is correct:

    ```bash
    ls -ld /home/sleepy/Desktop/Stuff
    ```

4.  Add the following entry to `/etc/fstab`. Replace the UUID with your drive's 
    UUID and update the mount point path to match the directory you created:

    ```fstab
    UUID=E252-E723  /home/sleepy/Desktop/Stuff  exfat  defaults,uid=1000,gid=1000,noauto,x-systemd.automount  0  2
    ```

**Important:** The mount point path must exist before you add the fstab entry. 
If the path doesn't exist, the automount will fail.

### Fix Polkit authentication on COSMIC DE

Admin dialog boxes might not accept input on COSMIC Desktop Environment.

**Source:** [CachyOS Forum](https://discuss.cachyos.org/t/polkit-authentication-issue-with-cosmic-desktop-environment/20592)

To fix Polkit authentication:

```bash
sudo chmod 4755 /usr/lib/polkit-1/polkit-agent-helper-1
```

## Applications and environment

### Disable Avahi (optional)

Avahi is a zeroconf implementation that enables service discovery on local 
networks using mDNS (multicast DNS). It implements Apple's Bonjour/Zeroconf 
protocol, allowing automatic discovery of printers, file shares, and other 
network services without manual configuration.

**Source:** [Arch Linux Wiki - Avahi](https://wiki.archlinux.org/title/Avahi)

**What Avahi is used for:**
- Automatic network printer discovery
- .local hostname resolution (e.g., `hostname.local`)
- Network service discovery (SSH, VNC, file shares)
- Bonjour protocol support for applications
- Zero-configuration networking

**Why you might want to disable it:**
- Not needed if you don't use .local hostnames or network service discovery
- Reduces attack surface on systems that don't need network service advertising
- Prevents unnecessary mDNS traffic on port 5353
- systemd-resolved provides similar functionality in some configurations
- Boot time optimization (minor)

**Important:** Avahi is a dependency for many essential packages including 
`pipewire-pulse`, `libcups`, `geoclue`, and various desktop applications. 
**Do not attempt to remove the Avahi package** as this will break critical 
system functionality. Only disable the daemon if you don't use network service 
discovery features.

To disable and mask the Avahi daemon:

1.  Stop the Avahi services:

    ```bash
    sudo systemctl stop avahi-daemon.service avahi-daemon.socket
    ```

2.  Disable the services from starting at boot:

    ```bash
    sudo systemctl disable avahi-daemon.service avahi-daemon.socket
    ```

3.  Mask the services to prevent them from being started by other services:

    ```bash
    sudo systemctl mask avahi-daemon.service avahi-daemon.socket
    ```

4.  (Optional) Remove nss-mdns from `/etc/nsswitch.conf` if you had previously 
    configured it for .local hostname resolution. Edit the file:

    ```bash
    sudo nano /etc/nsswitch.conf
    ```

    Remove `mdns_minimal [NOTFOUND=return]` or `mdns` from the `hosts:` line.

To re-enable Avahi later:

```bash
sudo systemctl unmask avahi-daemon.service avahi-daemon.socket
sudo systemctl enable --now avahi-daemon.service avahi-daemon.socket
```

### Enable Wayland in Helium browser (optional)

Configure Wayland and hardware acceleration globally for Helium.

**Reference:** [CachyOS Wiki - Hardware Acceleration](https://wiki.cachyos.org/configuration/enabling_hardware_acceleration_in_google_chrome/)

1.  Edit `/etc/environment`:

    ```bash
    sudo nano /etc/environment
    ```

2.  Add or modify the following line:

    ```bash
    HELIUM_USER_FLAGS="--enable-wayland-ime --ozone-platform=wayland --enable-features=AcceleratedVideoDecodeLinuxGL,AcceleratedVideoDecodeLinuxZeroCopyGL,AcceleratedVideoEncoder,WaylandSessionManagement,WaylandTextInputV3,WaylandUiScale"
    ```

### Prevent Steam black screen on idle (optional)

Steam can display a black screen after running in the background for an 
extended period.

To prevent this issue:

1.  In Steam, click **Settings > Interface**.
2.  Clear the **Enable GPU accelerated rendering in web views** checkbox.

### Disable Steam auto-update on launch (preference)

Prevent the Steam bootstrapper from checking for updates every time you launch 
Steam.

To disable auto-update on launch:

1.  Verify that the Steam directory exists. The directory should exist if 
    you've launched Steam at least once:

    ```bash
    ls -ld ~/.local/share/Steam
    ```

2.  Edit `~/.local/share/Steam/steam.cfg`. Create the file if it doesn't exist:

    ```bash
    nano ~/.local/share/Steam/steam.cfg
    ```

3.  Add the following line:

    ```ini
    BootStrapperInhibitAll=enable
    ```

## Audio and hardware

### Prevent microphone auto-adjustment (recommended for Discord/Electron apps)

Discord and other Electron-based apps can automatically change your input 
volume without your permission.

**Source:** [Lumeh.org](https://www.lumeh.org/wiki/audio/stop-adjusting-my-microphone/)

To prevent automatic microphone adjustment:

1.  Create the WirePlumber configuration directory if it doesn't exist:

    ```bash
    mkdir -p ~/.config/wireplumber/wireplumber.conf.d
    ```

2.  Create the configuration file:

    ```bash
    nano ~/.config/wireplumber/wireplumber.conf.d/90-prevent-adjusting.conf
    ```

3.  Add the following rule:

    ```lua
    access.rules = [
      {
        matches = [ { application.process.binary = "electron" } ]
        actions = { update-props = { default_permissions = "rx" } }
      }
    ]
    ```

### Fractal Scape headphones USB connection (hardware-specific)

Fractal Scape headphones might not be recognized when connected to certain USB 
ports. This issue affects front-panel USB ports on some cases (such as the 
Corsair 4000D), USB hubs built into monitors, and external USB hubs.

**Workaround:** Connect the headphones directly to the rear USB 3.0 ports on 
your motherboard. This approach reduces the available range from your PC.

**Extending the range with the charging base:** The charging base can function 
as a USB extender when you plug it into your PC's USB port. However, if you 
connect the charging base to your PC, it can't charge your headphones when 
your PC is fully turned off.

**Recommended setup:** Connect the charging base to a wall outlet instead of 
your PC. This configuration lets you charge your headphones even when your PC 
is completely powered off, but you can't use the charging base as a USB 
extender.

**Alternative solution:** If you need more range and want to keep the charging 
base connected to a wall outlet, use a USB extension cable. The Logitech G502X 
PLUS Lightspeed mouse includes a suitable extension cable. If you upgraded 
from the original G502X Lightspeed mouse, you can repurpose that extension 
cable because the new dongle is compatible with it.

## Gaming configuration

### Network performance tuning for gaming (optional, experimental)

These network optimizations may reduce latency for competitive gaming. Results 
vary significantly based on hardware and network conditions. Apply these 
settings individually and test to determine if they improve your specific 
situation.

**Important:** These settings were tested for Overwatch 2 packet loss issues 
but did not resolve that specific problem. They may still be beneficial for 
other games or network scenarios.

#### Disable Energy Efficient Ethernet (EEE)

EEE can cause latency spikes as the network interface transitions between power 
states. Replace `enp6s0` with your interface name:

```bash
sudo ethtool --set-eee enp6s0 eee off
```

To make persistent, add to `/etc/systemd/system/network-tuning.service` 
(create similar to the WoL service in Power Management section).

#### Disable network offload features

Generic Receive/Segmentation Offload can add latency in some configurations:

```bash
sudo ethtool -K enp6s0 gro off gso off
```

**Note:** Disabling these features increases CPU usage for network processing.

#### Disable network interface power management

Prevents the network interface from entering power-saving modes:

```bash
echo "on" | sudo tee /sys/class/net/enp6s0/device/power/control
```

#### Verify current settings

Check your current network interface settings:

```bash
sudo ethtool enp6s0
sudo ethtool --show-eee enp6s0
sudo ethtool -c enp6s0
```

### Fix cursor issues in games with Gamescope

Some games can experience cursor warping, incorrect cursor behavior, or cursor 
not being properly confined to the game window when running under Gamescope.

**Source:** [Arch Linux Wiki - Gamescope Cursor Issues](https://wiki.archlinux.org/title/Gamescope#Cursor_doesn't_behave_properly)

To fix cursor issues, add the following to your Steam launch options:

```text
gamescope -f --force-grab-cursor -- %command%
```

The `-f` flag enables fullscreen mode, and `--force-grab-cursor` forces the 
cursor to be grabbed and confined to the game window.

### Fix alt-tab crashes in games with Gamescope

Games can freeze or crash when you alt-tab between windows. Use Gamescope to 
provide a stable compositor layer that prevents these issues.

**Note on performance:** Gamescope runs as a nested compositor, which means it 
runs a compositor within a compositor. This introduces a small performance 
overhead and potentially minor additional input latency. For most systems and 
games, this overhead is negligible, but it may be noticeable on lower-end 
hardware or in competitive gaming scenarios where every millisecond matters.

**Sources:** 
- [Arch Linux Wiki - Gamescope](https://wiki.archlinux.org/title/Gamescope)

To use Gamescope, add the following to your Steam launch options.

```text
gamescope -W 1920 -H 1080 --adaptive-sync --borderless -- %command%
```

**Parameter explanations:**

- `-W 1920 -H 1080` - Output resolution (the resolution Gamescope displays)
- `--adaptive-sync` - Enables FreeSync/G-Sync if supported
- `--borderless` - Runs in borderless window mode

### Enable NTSync (experimental)

NTSync is a kernel driver that provides Windows NT synchronization primitives 
in the Linux kernel. When used with Wine or Proton, NTSync can reduce overhead 
and latency by allowing synchronization operations to run in kernel space 
instead of user space.

**Important:** Performance improvements vary by game. Some games see 
significant improvements (5-15% FPS gains in CPU-bound scenarios), while others 
see minimal or no benefit. Games that heavily rely on synchronization 
primitives (such as games with many threads or complex physics) tend to benefit 
most.

**Source:** [Phoronix - Linux 6.14 NTSYNC Driver](https://www.phoronix.com/news/Linux-6.14-NTSYNC-Driver-Ready)

To enable NTSync:

1.  Verify that your system supports NTSync (requires Linux kernel 6.14 or 
    later with NTSync enabled):

    ```bash
    lsof /dev/ntsync
    ```

2.  If the `/dev/ntsync` device exists, add the following to your Steam launch 
    options:

    ```text
    PROTON_USE_NTSYNC=1 %command%
    ```

## Power management

### Disable Wake-on-LAN (optional)

Wake-on-LAN (WoL) allows a computer to be powered on remotely over the network. 
Disabling it can prevent spurious wakeups and reduce power consumption when the 
system is off.

To disable Wake-on-LAN for your network interface:

1.  Identify your network interface name:

    ```bash
    ip link show
    ```

2.  Disable Wake-on-LAN (replace `enp6s0` with your interface name):

    ```bash
    sudo ethtool -s enp6s0 wol d
    ```

3.  To make this persistent across reboots, create a systemd service:

    ```bash
    sudo nano /etc/systemd/system/disable-wol.service
    ```

4.  Add the following content (replace `enp6s0` with your interface name):

    ```ini
    [Unit]
    Description=Disable Wake-on-LAN
    After=network.target
    
    [Service]
    Type=oneshot
    ExecStart=/usr/bin/ethtool -s enp6s0 wol d
    
    [Install]
    WantedBy=multi-user.target
    ```

5.  Enable the service:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable disable-wol.service
    ```

### Fix suspend crashes caused by USB devices (hardware-specific)

Some USB devices (such as the HyperX QuadCast microphone and Logitech wireless 
receivers) can wake your system immediately after suspend, which can cause GPU 
crashes or prevent the system from entering sleep mode. This script 
automatically disables these devices before sleep and re-enables them after 
wake.

**Why this method works:** This configuration uses USB device authorization 
control instead of disabling wakeup triggers through `/proc/acpi/wakeup`. The 
`/proc/acpi/wakeup` method doesn't work reliably for USB devices that trigger 
spurious wakeups. By directly setting the device's `authorized` attribute to 
`0` before suspend, the USB controller effectively disconnects the device at 
the hardware level. This prevents the device from generating any wakeup events. 
After resume, setting `authorized` back to `1` reconnects the device.

**Reference:** See the GPIO interrupt section above for the related Arch Linux 
Wiki article on wakeup triggers, which discusses the broader context of suspend 
issues on Ryzen 7000 series systems.

To configure USB soft-unplug:

1.  Verify that the `/usr/local/bin` directory exists:

    ```bash
    sudo mkdir -p /usr/local/bin
    ```

2.  Create the script file `/usr/local/bin/usb-soft-unplug.sh` with the 
    following content:

    ```bash
    #!/bin/bash
    # Targets: Logitech G502X (046d:c547), HyperX Hub (214b:7250), 
    # QuadCast Audio (03f0:0d8b), QuadCast LED (03f0:0f8b)
    TARGET_IDS=("046d:c547" "214b:7250" "03f0:0d8b" "03f0:0f8b")
    ACTION=$1
    
    if [ "$ACTION" != "pre" ] && [ "$ACTION" != "post" ]; then
        echo "Usage: $0 {pre|post}"
        exit 1
    fi
    
    set_authorization() {
        target_state=$1
        for dev_path in /sys/bus/usb/devices/*; do
            if [ -f "$dev_path/idVendor" ] && [ -f "$dev_path/idProduct" ]; then
                vid=$(cat "$dev_path/idVendor")
                pid=$(cat "$dev_path/idProduct")
                current_id="$vid:$pid"
                for target in "${TARGET_IDS[@]}"; do
                    if [ "$current_id" == "$target" ] && [ -f "$dev_path/authorized" ]; then
                        echo "$target_state" > "$dev_path/authorized"
                    fi
                done
            fi
        done
    }
    
    case "$ACTION" in
        pre) set_authorization 0 ;;
        post) set_authorization 1 ;;
    esac
    ```

3.  Make the script executable:

    ```bash
    sudo chmod +x /usr/local/bin/usb-soft-unplug.sh
    ```

4.  Create the systemd service file 
    `/etc/systemd/system/usb-unplug-suspend.service` with the following content:

    ```ini
    [Unit]
    Description=Soft-unplug USB devices to prevent suspend crashes
    Before=sleep.target suspend.target hybrid-sleep.target
    StopWhenUnneeded=yes
    
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/local/bin/usb-soft-unplug.sh pre
    ExecStop=/usr/local/bin/usb-soft-unplug.sh post
    
    [Install]
    WantedBy=sleep.target suspend.target hybrid-sleep.target
    ```

5.  Enable the service:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable usb-unplug-suspend.service
    ```

**Finding your USB device IDs:** To find the vendor:product ID for your own 
USB devices, use:

```bash
lsusb
```

Look for entries like `ID 046d:c547` - the part before the colon is the vendor 
ID, and the part after is the product ID.

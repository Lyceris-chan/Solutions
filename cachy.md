# CachyOS Configuration and Troubleshooting Guide

This guide provides configuration steps and troubleshooting fixes for system 
stability, hardware, and gaming performance on CachyOS.

## System optimization

### Remove Plymouth boot splash

Plymouth can cause boot hangs and increase boot time.

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

### Configure bootloader

To reduce boot delay and set the default boot entry:

```bash
sudo bootctl set-timeout 0
sudo bootctl set-default linux-cachyos-rc.conf
```

### Add kernel parameters

The following kernel parameters improve system stability and power management. 
Add these to your boot configuration file, such as 
`/boot/loader/entries/linux-cachyos.conf` or `/etc/kernel/cmdline`:

```text
LINUX_OPTIONS="zswap.enabled=0 nowatchdog gpiolib_acpi.ignore_interrupt=AMDI0030:00@3,AMDI0030:00@10 mem_sleep_default=deep quiet"
```

**Parameter explanations:**

- `zswap.enabled=0` - Disables zswap compressed swap cache. This can improve 
  performance and reduce memory overhead on systems with adequate RAM.

- `nowatchdog` - Disables hardware watchdog timers. Watchdog can cause 
  unexpected system reboots or boot delays if it triggers incorrectly.

- `gpiolib_acpi.ignore_interrupt=AMDI0030:00@3,AMDI0030:00@10` - Ignores 
  specific GPIO interrupts from the AMDI0030 ACPI device. These GPIO pins can 
  cause spurious wakeups from suspend, preventing the system from staying 
  asleep. The specific pins (3 and 10) are known to trigger false wake events.

- `mem_sleep_default=deep` - Sets the default suspend mode to deep sleep (S3). 
  This provides better power savings during suspend compared to shallow sleep 
  modes.

- `quiet` - Reduces boot messages shown on screen for a cleaner boot experience.

### Remove systemd timeout configuration

Strict systemd timeouts can cause slow shutdowns or boot delays.

To remove the timeout configuration:

```bash
sudo rm /usr/lib/systemd/system.conf.d/00-timeout.conf
```

### Configure time synchronization

To remove Cloudflare and Google NTP servers and use only the Arch Linux NTP 
pool:

```bash
sudo sed -i \
  -e 's|^NTP=.*|NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org|' \
  -e 's|^FallbackNTP=.*|#FallbackNTP=|' \
  /usr/lib/systemd/timesyncd.conf.d/10-timesyncd.conf

sudo systemctl restart systemd-timesyncd
```

### Automount secondary drives

Use `x-systemd.automount` to prevent boot blocking when secondary drives fail 
to mount.

To configure automount:

1.  Find the UUID of your drive:

    ```bash
    lsblk -f
    ```

2.  Create the mount directory (replace the path with your desired location):

    ```bash
    mkdir -p /home/sleepy/Desktop/Stuff
    ```

3.  Verify the directory exists and the path is correct:

    ```bash
    ls -ld /home/sleepy/Desktop/Stuff
    ```

4.  Add the following entry to `/etc/fstab` (replace the UUID with your drive's 
    UUID and update the mount point path to match the directory you created):

    ```text
    UUID=E252-E723  /home/sleepy/Desktop/Stuff  exfat  defaults,uid=1000,gid=1000,noauto,x-systemd.automount  0  2
    ```

**Important:** Ensure the mount point path exists before adding the fstab entry, 
otherwise the automount will fail.

### Fix Polkit authentication on COSMIC DE

Admin dialog boxes may not allow input on COSMIC Desktop Environment.

**Source:** [CachyOS Forum](https://discuss.cachyos.org/t/polkit-authentication-issue-with-cosmic-desktop-environment/20592)

To fix Polkit authentication:

```bash
sudo chmod 4755 /usr/lib/polkit-1/polkit-agent-helper-1
```

## Applications and environment

### Enable Wayland in Helium browser

To configure Wayland and hardware acceleration globally for Helium:

**Reference:** [CachyOS Wiki - Hardware Acceleration](https://wiki.cachyos.org/configuration/enabling_hardware_acceleration_in_google_chrome/)

1.  Edit `/etc/environment`:

    ```bash
    sudo nano /etc/environment
    ```

2.  Add or modify the following line:

    ```bash
    HELIUM_USER_FLAGS="--enable-wayland-ime --ozone-platform=wayland --enable-features=AcceleratedVideoDecodeLinuxGL,AcceleratedVideoDecodeLinuxZeroCopyGL,AcceleratedVideoEncoder,WaylandSessionManagement,WaylandTextInputV3,WaylandUiScale"
    ```

### Prevent Steam black screen on idle

Steam can render a black screen after running in the background.

To prevent this issue:

1.  Open **Steam > Settings > Interface**.
2.  Clear the **Enable GPU accelerated rendering in web views** checkbox.

### Disable Steam auto-update on launch

To prevent the Steam bootstrapper from checking for updates every time you 
launch it:

1.  Verify the Steam directory exists (it should if you've launched Steam at 
    least once):

    ```bash
    ls -ld ~/.local/share/Steam
    ```

2.  Edit `~/.local/share/Steam/steam.cfg` (create the file if it doesn't exist):

    ```bash
    nano ~/.local/share/Steam/steam.cfg
    ```

3.  Add the following line:

    ```text
    BootStrapperInhibitAll=enable
    ```

## Audio and hardware

### Prevent microphone auto-adjustment

Discord and other Electron apps can automatically change your input volume.

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

### Fractal Scape headphones USB connection

Fractal Scape headphones may not be recognized when connected to front-panel 
USB ports on the Corsair 4000D case.

To resolve this issue, connect the headphones directly to rear USB 3.0 ports. 
This reduces the available range from the PC. While the charging base could be 
used as a USB hub to extend range, it must remain plugged into an outlet to 
stay powered, making this impractical.

**Solution:** Use the USB extension cable that comes with the Logitech G502X 
PLUS Lightspeed mouse. If you upgraded from the original G502X Lightspeed, you 
can repurpose the old extension cable since the new dongle works with it.

## Gaming configuration

### Fix Overwatch packet loss

Overwatch can experience 10-20% outbound packet loss with the default Realtek 
driver.

To fix packet loss:

1.  Install the improved driver:

    ```bash
    paru -S r8125-dkms
    ```

2.  Create the modprobe configuration file:

    ```bash
    sudo nano /etc/modprobe.d/realtek.conf
    ```

3.  Add the following lines:

    ```text
    options r8125 aspm=0
    blacklist r8169
    ```

4.  Reboot your system.

### Fix mouse warping in Paladins and UE3 games

Paladins and other Unreal Engine 3 games can experience mouse cursor warping 
issues on Linux.

To fix mouse warping, add the following to Steam launch options:

```text
WINE_MOUSE_WARP_POINTER=1 %command%
```

Alternatively, you can force the mouse warp override through the Wine registry:

```bash
WINEPREFIX="$HOME/.local/share/Steam/steamapps/compatdata/444090/pfx" wine reg add "HKCU\Software\Wine\DirectInput" /v MouseWarpOverride /t REG_SZ /d force /f
```

### Fix alt-tab crashes in games

Games can freeze or crash when you use alt-tab. Use Gamescope to prevent this 
issue.

Add the following to Steam launch options. Replace `240` with your monitor's 
refresh rate:

```text
gamescope -W 1920 -H 1080 -r 240 --adaptive-sync --borderless -- %command%
```

### Enable NTSync

NTSync provides kernel-backed synchronization for lower latency.

To enable NTSync:

1.  Verify that your system supports NTSync:

    ```bash
    lsof /dev/ntsync
    ```

2.  Add the following to Steam launch options:

    ```text
    PROTON_USE_NTSYNC=1 %command%
    ```

## Power management

### Fix suspend crashes caused by USB devices

HyperX QuadCast and Logitech receivers can wake the system instantly after 
suspend, which can cause GPU crashes. This script automatically unplugs these 
devices before sleep and reconnects them after wake.

To configure USB soft-unplug:

1.  Create the script file `/usr/local/bin/usb-soft-unplug.sh`. First, ensure 
    the directory exists:

    ```bash
    sudo mkdir -p /usr/local/bin
    ```

    Then create the file with the following content:

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

2.  Make the script executable:

    ```bash
    sudo chmod +x /usr/local/bin/usb-soft-unplug.sh
    ```

3.  Create the systemd service file 
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

4.  Enable the service:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable usb-unplug-suspend.service
    ```

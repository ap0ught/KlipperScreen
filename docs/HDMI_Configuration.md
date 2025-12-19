# HDMI Display Configuration

This guide covers everything you need to know about configuring HDMI displays with KlipperScreen on Raspberry Pi and other Linux systems.

## Overview

KlipperScreen supports HDMI displays through the GTK/GDK display system. It can automatically detect connected monitors and provides options for multi-monitor setups, power management, and display configuration.

## Display Detection

### How Monitors Are Detected

KlipperScreen uses the GDK (GIMP Drawing Kit) display system to detect monitors:

```python
# From screen.py
display = Gdk.Display.get_default()
monitor_amount = Gdk.Display.get_n_monitors(display)
```

When KlipperScreen starts, it:
1. Gets the default display (usually `:0` for X11)
2. Enumerates all connected monitors
3. Logs each monitor's resolution
4. Selects a monitor based on command-line arguments or defaults to monitor 0

### Display Numbering

Monitors are numbered starting from 0. The numbering depends on:
- Physical connection order (HDMI-1, HDMI-2, etc.)
- Display driver detection order
- X11 or Wayland display configuration

To see your monitor configuration:
```bash
DISPLAY=:0 xrandr
```

Output example:
```
Screen 0: minimum 320 x 200, current 1920 x 1080, maximum 8192 x 8192
HDMI-1 connected primary 1920x1080+0+0 (normal left inverted right x axis y axis) 800mm x 450mm
   1920x1080     60.00*+  50.00    59.94  
HDMI-2 connected 1280x720+1920+0 (normal left inverted right x axis y axis) 700mm x 400mm
   1280x720      60.00*+  50.00    59.94
```

### X11 Display Environment Variable

KlipperScreen uses the `DISPLAY` environment variable to identify which X11 display to use:

```python
# From screen.py
self.display_number = os.environ.get('DISPLAY') or ':0'
```

- Default: `:0` (first display)
- For remote displays: `hostname:0`
- For specific screens: `:0.1` (second screen on first display)

### Monitor Selection Logic

The `get_monitor()` logic in `screen.py`:

```python
# Select monitor based on command-line argument
mon_n = int(args.monitor)  # Default: 0
monitor = display.get_monitor(mon_n)

# Get monitor geometry
self.width = monitor.get_geometry().width
self.height = monitor.get_geometry().height
```

## Configuration Options

### Command-Line Arguments

Start KlipperScreen with specific display options:

```bash
# Use monitor 0 (default)
~/KlipperScreen/screen.py

# Use monitor 1 (second display)
~/KlipperScreen/screen.py -m 1

# Specify config file
~/KlipperScreen/screen.py -c ~/printer_data/config/KlipperScreen.conf

# Specify log file location
~/KlipperScreen/screen.py -l ~/printer_data/logs/KlipperScreen.log

# Combine options
~/KlipperScreen/screen.py -m 1 -c ~/my_config.conf
```

#### Available Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `-m, --monitor` | Monitor number to display KlipperScreen | `0` |
| `-c, --configfile` | Path to configuration file | Auto-detected |
| `-l, --logfile` | Path to log file | `~/printer_data/logs/KlipperScreen.log` |

### Configuration File Settings

In `~/printer_data/config/KlipperScreen.conf`:

```ini
[main]
# Screen width (windowed mode)
# Setting width or height disables fullscreen
# width: 1024

# Screen height (windowed mode)
# height: 600

# Show mouse cursor
# show_cursor: False

# Use DPMS for power management
# use_dpms: True

# Screen blanking timeout (seconds, or "off")
# screen_blanking: 600

# Screen blanking while printing
# screen_blanking_printing: off
```

### Default HDMI Monitor

By default, KlipperScreen uses **monitor 0** which is typically:
- The first HDMI port on Raspberry Pi (HDMI0)
- The primary display if multiple monitors are connected
- The only display if just one monitor is connected

### Multiple Monitor Support

KlipperScreen can run on any detected monitor:

**Single Instance on Specific Monitor:**
```bash
# Show on second monitor
~/KlipperScreen/screen.py -m 1
```

**Multiple Instances:**
For running KlipperScreen on multiple monitors simultaneously, see the [Multi-Instance Documentation](Multi_instance.md).

### Fullscreen vs Windowed Mode

**Fullscreen Mode (Default):**
- Uses entire monitor resolution
- Best for dedicated touchscreen displays
- Monitor selection with `-m` works only in fullscreen

**Windowed Mode:**
- Set `width` or `height` in config to enable
- Useful for development and testing
- Monitor selection (`-m`) is ignored in windowed mode

```ini
[main]
# Enable windowed mode
width: 1024
height: 600

# Show cursor in windowed mode
show_cursor: True
```

## DPMS and Power Management

DPMS (Display Power Management Signaling) allows KlipperScreen to control display power states.

### The `set_dpms()` Function

Located in `screen.py`, this function enables or disables DPMS:

```python
def set_dpms(self, use_dpms):
    """Enable or disable DPMS power management"""
    if not use_dpms:
        # Disable DPMS
        subprocess.run(
            f"xset -display {self.display_number} dpms 0 0 0",
            shell=True, check=True
        )
        subprocess.run(
            f"xset -display {self.display_number} -dpms",
            shell=True, check=True
        )
    self.use_dpms = use_dpms
```

### The `wake_screen()` Function

Wakes the display from power-saving mode:

```python
def wake_screen(self):
    """Wake the screen from standby"""
    subprocess.run(
        f"xset -display {self.display_number} dpms force on",
        shell=True, check=True
    )
```

### Using `xset` Commands for Power Control

Manual DPMS control:

```bash
# Get current DPMS status
xset -display :0 q | grep "DPMS"

# Turn DPMS on
xset -display :0 +dpms

# Turn DPMS off
xset -display :0 -dpms

# Set DPMS timeouts (standby, suspend, off in seconds)
xset -display :0 dpms 600 0 0

# Force screen state
xset -display :0 dpms force on      # Wake up
xset -display :0 dpms force standby # Sleep
xset -display :0 dpms force off     # Turn off
```

### Screen Blanking Configuration

Control when the screen blanks:

```ini
[main]
# Blank screen after 10 minutes of inactivity
screen_blanking: 600

# Never blank screen while printing
screen_blanking_printing: off

# Disable screen blanking entirely
# screen_blanking: off
```

**Blanking Behavior:**
- `off`: Screen never blanks
- `<seconds>`: Blank after specified seconds of inactivity
- Printing mode: Separate timeout for when printer is active
- Touch/mouse activity resets the timeout

### Automatic DPMS Control

KlipperScreen automatically manages DPMS:

1. **During Printing**: 
   - Uses `screen_blanking_printing` timeout
   - Default: never blank to monitor print progress

2. **Idle State**:
   - Uses `screen_blanking` timeout
   - Default: 10 minutes (600 seconds)

3. **On Activity**:
   - Any touch or mouse input wakes screen
   - Timer resets on interaction

### Disabling DPMS

If you experience issues with DPMS:

```ini
[main]
use_dpms: False
screen_blanking: off
```

Or via the UI: Settings → Screen → DPMS

## Hardware Support

### Raspberry Pi HDMI Outputs

#### Raspberry Pi 4/5
- **Two micro-HDMI ports**
- HDMI0: First port (closest to USB-C power)
- HDMI1: Second port
- Supports 4K resolution
- Can run dual displays with multi-instance setup

#### Raspberry Pi 3 and Earlier
- **Single full-size HDMI port**
- 1080p maximum resolution
- Uses HDMI-1 identifier

### DPI Displays

Some displays connect via DPI (Display Parallel Interface):
- DSI displays (ribbon cable connection)
- Direct GPIO connection displays
- Generally work without additional HDMI configuration

See the [Hardware Documentation](Hardware.md) for tested DPI displays.

### Tested Hardware

The following HDMI displays are confirmed working:

- **BTT HDMI5/7**: Plug-and-play, touchscreen support
- **5" HDMI Display-B**: 800x480 resolution
- **Generic HDMI monitors**: Most standard monitors work
- **4K displays**: Supported on Raspberry Pi 4/5

For a complete list, see [Hardware Documentation](Hardware.md).

### Common Hardware Compatibility Issues

**Issue**: Screen shows `No Signal`
- **Cause**: Incorrect resolution or timing
- **Fix**: Configure resolution in `/boot/firmware/cmdline.txt`

**Issue**: Touchscreen not working
- **Cause**: Separate touch driver needed
- **Fix**: See [Touch Issues Troubleshooting](Troubleshooting/Touch_issues.md)

**Issue**: Screen flickers or artifacts
- **Cause**: Insufficient power or bad cable
- **Fix**: Use proper power supply and quality HDMI cable

## Resolution and Display Settings

### Setting Custom Resolutions

Edit the kernel command line on Raspberry Pi OS Bookworm:

```bash
sudo nano /boot/firmware/cmdline.txt
```

!!! warning "Important"
    Do not add newlines to the file. All options must be on a single line separated by spaces.

### Resolution Syntax

Basic format:
```
video=<identifier>:<xres>x<yres>[@<refresh-rate>]
```

Examples:
```
# Simple resolution
video=HDMI-A-1:1920x1080

# With refresh rate
video=HDMI-A-1:1920x1080@60

# With rotation
video=HDMI-A-1:1920x1080@60,rotate=90

# Multiple options
video=HDMI-A-1:1920x1080M@60,rotate=90,reflect_x
```

### Find Your Display Identifier

```bash
DISPLAY=:0 xrandr
```

Output:
```
Screen 0: minimum 320 x 200, current 1024 x 600, maximum 8192 x 8192
HDMI-1 connected primary 1024x600+0+0 (normal left inverted right x axis y axis) 800mm x 450mm
```

The identifier is `HDMI-1` in this case.

### Mode Specifiers

Full syntax:
```
<xres>x<yres>[M][R][-<bpp>][@<refresh-rate>][i][m][eDd]
```

| Option | Description |
|--------|-------------|
| `M` | Calculate timings using CVT |
| `R` | CVT reduced blanking (requires 60Hz) |
| `-<bpp>` | Bits per pixel (usually 24) |
| `@<refresh-rate>` | 50, 60, 70, or 85 Hz |
| `e` | Force enable |
| `D` | Force digital mode |
| `d` | Disable this output |

For more information: [Kernel modedb documentation](https://docs.kernel.org/fb/modedb.html)

### Aspect Ratio Configuration

KlipperScreen automatically detects aspect ratio:

```python
# From screen.py
self.aspect_ratio = self.width / self.height
self.vertical_mode = self.aspect_ratio < 1.0
```

**Portrait Mode** (`aspect_ratio < 1.0`):
- Layout automatically adjusts
- UI elements stack vertically
- Menu and status panels reorganize

**Landscape Mode** (`aspect_ratio >= 1.0`):
- Default horizontal layout
- Side-by-side panels

### Vertical/Portrait Mode

To use portrait orientation:

1. **Set display rotation** in `/boot/firmware/cmdline.txt`:
   ```
   video=HDMI-A-1:1080x1920@60,rotate=90
   ```

2. **Fix touch rotation** if needed:
   See [Touch Issues - Rotation](Troubleshooting/Touch_issues.md#touch-rotation-and-matrix)

3. **KlipperScreen auto-detects** portrait mode and adjusts layout

### Fullscreen Settings

In `KlipperScreen.conf`:

```ini
[main]
# Fullscreen (default)
# Don't set width or height

# Windowed mode
width: 800
height: 480

# Minimum resolution: 480x320
```

**Note**: Setting `width` OR `height` disables fullscreen mode.

## Troubleshooting

### No Display Detected

**Symptoms**: 
- Log shows "WARNING: No monitors detected by Gdk"
- Black screen
- No output

**Causes and Solutions**:

1. **X11 not running**
   ```bash
   # Check if X is running
   ps aux | grep X
   
   # Start X if needed
   startx
   ```

2. **Wrong DISPLAY variable**
   ```bash
   # Set correct display
   export DISPLAY=:0
   
   # Verify
   echo $DISPLAY
   ```

3. **Display driver issues**
   ```bash
   # Check display configuration
   DISPLAY=:0 xrandr
   
   # Reinstall display drivers (Raspberry Pi)
   sudo apt-get install --reinstall xserver-xorg
   ```

4. **Permissions issue**
   ```bash
   # Add user to video group
   sudo usermod -a -G video $USER
   ```

### Wrong Monitor Selected

**Symptoms**: KlipperScreen appears on wrong display

**Solution**:
```bash
# Check monitor numbers
DISPLAY=:0 xrandr

# Start on specific monitor
~/KlipperScreen/screen.py -m 1
```

**Systemd Service**: Edit service file to use `-m` argument:
```bash
sudo nano /etc/systemd/system/KlipperScreen.service
```

Update `ExecStart` line:
```ini
ExecStart=/home/pi/KlipperScreen/screen.py -m 1
```

Restart service:
```bash
sudo systemctl daemon-reload
sudo systemctl restart KlipperScreen
```

### HDMI Not Showing Output

**Check these in order:**

1. **Physical connection**
   - Ensure HDMI cable is firmly connected
   - Try a different HDMI cable
   - Test display with another device

2. **Power supply**
   - Use official Raspberry Pi power supply
   - Underpowered Pi may not output HDMI

3. **Boot configuration**
   ```bash
   sudo nano /boot/firmware/config.txt
   ```
   
   Ensure these are present:
   ```ini
   # HDMI settings
   hdmi_force_hotplug=1
   hdmi_drive=2
   ```

4. **Resolution configuration**
   - Set explicit resolution in cmdline.txt (see above)
   - Try safe mode resolution: `video=1024x768@60`

5. **Check logs**
   ```bash
   cat ~/printer_data/logs/KlipperScreen.log | grep -i "screen\|display\|monitor"
   ```

### Display Geometry Issues

**Symptoms**: 
- Incorrect aspect ratio
- Stretched or squashed UI
- Clipped edges

**Solutions**:

1. **Set correct resolution** in cmdline.txt
2. **Disable overscan** in `/boot/firmware/config.txt`:
   ```ini
   disable_overscan=1
   ```

3. **Check display mode**:
   ```bash
   DISPLAY=:0 xrandr
   ```
   Ensure the active mode (marked with `*`) matches your display.

4. **Test different resolutions**:
   ```bash
   # Temporarily change resolution
   DISPLAY=:0 xrandr --output HDMI-1 --mode 1920x1080
   ```

### DPMS Issues

**Screen doesn't wake**:
```bash
# Manually wake
xset -display :0 dpms force on

# Check DPMS status
xset -display :0 q | grep DPMS
```

**Screen blanks too quickly**:
```ini
[main]
screen_blanking: 1800  # 30 minutes
```

**Disable DPMS**:
```ini
[main]
use_dpms: False
screen_blanking: off
```

### Logging and Debugging

**Enable detailed logging**:

1. Check KlipperScreen log:
   ```bash
   tail -f ~/printer_data/logs/KlipperScreen.log
   ```

2. Look for display-related messages:
   ```bash
   grep -i "monitor\|display\|hdmi\|dpms" ~/printer_data/logs/KlipperScreen.log
   ```

3. X server log:
   ```bash
   cat /var/log/Xorg.0.log | grep -i "HDMI"
   ```

4. System log:
   ```bash
   sudo journalctl -xe | grep -i "display\|hdmi"
   ```

**Debug display detection**:
```bash
# List all displays and their properties
DISPLAY=:0 xrandr --verbose

# Check display capabilities
DISPLAY=:0 xrandr --current

# Monitor connection status
DISPLAY=:0 xrandr --query
```

### Getting Help

When reporting HDMI issues, include:

1. Output of `xrandr`:
   ```bash
   DISPLAY=:0 xrandr
   ```

2. KlipperScreen log:
   ```bash
   tail -100 ~/printer_data/logs/KlipperScreen.log
   ```

3. Display model and connection type
4. Raspberry Pi model
5. Contents of `/boot/firmware/cmdline.txt`
6. Contents of `/boot/firmware/config.txt`

## Additional Resources

- [Hardware Documentation](Hardware.md) - Tested display hardware
- [Display Rotation Guide](Troubleshooting/Rotation.md) - Configure screen rotation
- [Touch Issues](Troubleshooting/Touch_issues.md) - Touchscreen calibration
- [Multi-Instance Setup](Multi_instance.md) - Multiple displays
- [Configuration Guide](Configuration.md) - General configuration options
- [Extending Interface](Extending_Interface.md) - Custom UI development

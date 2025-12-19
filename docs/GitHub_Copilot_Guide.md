# GitHub Copilot Guide for KlipperScreen Development

This guide helps developers effectively use GitHub Copilot and other AI coding assistants when working with KlipperScreen.

## Overview

GitHub Copilot can significantly accelerate KlipperScreen development by:
- Generating panel boilerplate code
- Creating UI layouts and widgets
- Implementing HDMI configuration logic
- Writing GTK event handlers
- Suggesting Klipper-specific patterns

### How to Use This Guide

1. **Understand the architecture** first using [Extending Interface](Extending_Interface.md)
2. **Use the prompts below** as starting points
3. **Iterate and refine** the AI-generated code
4. **Test thoroughly** before deploying

## Extending the UI with Copilot

### Best Practices

**DO:**
- ✅ Provide context about the KlipperScreen architecture
- ✅ Reference existing panels as examples
- ✅ Specify GTK 3.0 compatibility requirements
- ✅ Request error handling and logging
- ✅ Ask for inline comments explaining complex logic
- ✅ Validate generated code against actual printer state

**DON'T:**
- ❌ Blindly accept generated code without testing
- ❌ Skip error handling in printer commands
- ❌ Forget to clean up resources in deactivate()
- ❌ Ignore the existing code style
- ❌ Use blocking operations in UI code

### Recommended General Prompts

#### Understanding Existing Code

```
Explain how panels are loaded and registered in KlipperScreen. 
Reference screen.py and the _load_panel() method.
```

```
How does the BasePanel class work? Show me the lifecycle methods 
(activate, deactivate, process_update) and when they're called.
```

```
What are the key differences between ScreenPanel and BasePanel? 
When should I inherit from each?
```

#### Code Generation Setup

Before generating code, provide Copilot with context:

```
I'm creating a custom panel for KlipperScreen. The project uses:
- GTK 3.0 via PyGObject (gi)
- Python 3.8+
- Panel architecture inheriting from ScreenPanel or BasePanel
- Panel file location: panels/<name>.py
- Each panel must have a class named "Panel"

Generate code following these conventions:
- Use self._screen for accessing core services
- Use self._printer for printer state
- Use self._gtk for UI helpers
- Add logging statements for debugging
- Include proper error handling
- Clean up resources in deactivate()
```

## Panel Development with Copilot

### Example Prompts

#### Basic Panel Creation

```
Create a KlipperScreen panel that displays printer status. The panel should:
- Inherit from ScreenPanel
- Show current printer state (ready, printing, paused)
- Display current X, Y, Z position from toolhead
- Show extruder and bed temperatures
- Include a refresh button
- Update every 2 seconds when active
- Clean up timer in deactivate()
- Follow KlipperScreen conventions from panels/example.py
```

Expected output: A complete panel with lifecycle methods, UI layout, and update logic.

#### Interactive Panel with Buttons

```
Create a KlipperScreen panel for controlling printer movement with:
- GTK Grid layout (3x3 buttons)
- Arrow buttons for X+, X-, Y+, Y-, Z+, Z-
- Home button in center
- Distance selector (1mm, 5mm, 10mm, 25mm)
- Buttons use self._gtk.Button() helper
- Send G-code commands via self._screen._ws.klippy.gcode_script()
- Confirm dialogs for home operations
- Style buttons with 'color1', 'color2', etc.
- Handle vertical and horizontal layouts
```

#### Settings Panel with Toggles

```
Generate a KlipperScreen settings panel with:
- ScrolledWindow container for long content
- Toggle switches for enable/disable features
- GTK Switch widgets connected to callbacks
- Save settings to KlipperScreen.conf via self._config
- Load current settings on activate()
- Show confirmation messages on save
- Settings to include:
  * DPMS on/off
  * Screen blanking timeout
  * Show cursor
  * 24-hour time format
```

#### Panel with Keyboard Input

```
Create a KlipperScreen panel for manual G-code entry:
- GTK Entry for text input
- Set input purpose to FREE_FORM for full keyboard
- Connect entry to show_keyboard on focus
- Submit button to send G-code
- History list showing recent commands (last 10)
- ScrolledWindow for history
- Clear button to clear history
- Error handling for invalid G-code
- Use self._screen.show_keyboard() method
```

#### Custom Widget Development

```
Create a custom GTK widget for KlipperScreen that shows a temperature gauge:
- Inherit from Gtk.Box
- Display temperature value and target
- Progress bar showing percentage to target
- Color changes based on temperature (blue=cold, red=hot)
- Accept update_temperature(current, target) method
- Show heater name as label
- Match KlipperScreen theming
```

### Panel with Live Updates

```
Create a KlipperScreen panel that monitors print progress:
- Show current layer and total layers
- Display print filename
- Show elapsed time and estimated time remaining  
- Progress bar (0-100%)
- Thumbnail preview using self.get_file_image()
- Pause/Resume/Cancel buttons
- Update on process_update() callback
- Handle print_stats data from self._printer.data
- Responsive layout for vertical/horizontal modes
```

## HDMI Configuration with Copilot

### Recommended Prompts

#### Monitor Detection

```
Write Python code for KlipperScreen to detect and list all connected HDMI monitors using GTK/GDK:
- Use Gdk.Display.get_default()
- Get monitor count with get_n_monitors()
- Iterate through monitors and get geometry
- Log each monitor's resolution
- Return a list of monitor info dictionaries
- Include error handling for no monitors
```

#### DPMS Control Implementation

```
Implement DPMS (Display Power Management) control for KlipperScreen:
- Create enable_dpms() function using xset command
- Create disable_dpms() function
- Create wake_screen() function to force display on
- Use subprocess.run() with proper error handling
- Accept display number parameter (e.g., ':0')
- Log success/failure of each operation
- Handle CalledProcessError exceptions
- Return status boolean
```

#### Multi-Monitor Support

```
Add multi-monitor support to KlipperScreen initialization:
- Parse command-line argument for monitor number (-m flag)
- Validate monitor number against available monitors
- Select monitor using display.get_monitor(mon_n)
- Get monitor geometry (width, height)
- Handle invalid monitor numbers with fallback to monitor 0
- Log selected monitor info
- Support fullscreen on specific monitor
```

#### Display Configuration Settings

```
Create a KlipperScreen settings panel for display configuration:
- DPMS enable/disable toggle
- Screen blanking timeout input (seconds or 'off')
- Separate timeout for printing state
- Monitor selection dropdown (if multiple monitors)
- Resolution display (read-only)
- Show cursor toggle
- Save to KlipperScreen.conf [main] section
- Apply changes without restart where possible
```

#### Screen Blanking Timer

```
Implement screen blanking timer for KlipperScreen:
- Create set_screenblanking_timeout(seconds) function
- Use GLib.timeout_add_seconds() for timer
- Call wake_screen() on user interaction
- Different timeouts for idle vs printing states
- Cancel timer on panel deactivate
- Store timeout_id for cleanup
- Handle 'off' value to disable blanking
- Update DPMS timeout via xset command
```

## Architecture Questions for Copilot

### Understanding Panel Registration

```
How does KlipperScreen discover and load panels from the panels/ directory?
Show the _load_panel() method and explain the import mechanism.
```

```
How are panels added to menus in KlipperScreen.conf? 
Show example configuration for custom panel menu entry.
```

### Display System

```
Explain the KlipperScreen display initialization sequence:
- GTK window creation
- Monitor detection
- Display geometry setup
- Fullscreen vs windowed mode
- Wayland vs X11 handling
```

### State Management

```
How does printer state flow through KlipperScreen?
Trace the path from Moonraker WebSocket update to panel UI update.
Include: websocket callback -> printer.process_update() -> panel.process_update()
```

### Configuration System

```
How does KlipperScreen configuration work?
- Config file location and search order
- Reading values with get_main_config()
- Saving user changes with save_user_config_options()
- Printer-specific vs global settings
```

## Code References

When asking Copilot to generate code, reference these key files:

### Core Files

**screen.py** - Main application
```
Reference screen.py for:
- KlipperScreen(Gtk.Window) class structure
- Monitor detection and selection
- DPMS control methods
- Panel loading and navigation
- Keyboard management
```

**panels/base_panel.py** - Base panel class
```
Reference base_panel.py for:
- BasePanel lifecycle methods
- Action bar button setup
- Title bar configuration
- Emergency stop handling
```

**ks_includes/screen_panel.py** - Core panel class
```
Reference screen_panel.py for:
- ScreenPanel base functionality
- Common panel methods
- Menu loading/unloading
- Emergency stop implementation
```

### Example Panels

**panels/example.py** - Minimal template
```
Use panels/example.py as template for:
- Basic panel structure
- Minimal working panel
- Required imports and class definition
```

**panels/temperature.py** - Complex panel
```
Reference panels/temperature.py for:
- Graph widgets
- Device management
- Popover menus
- Keypad input
- Complex layouts
```

**panels/move.py** - Input handling
```
Reference panels/move.py for:
- Keyboard integration
- Numeric input
- Distance/speed controls
- G-code movement commands
```

**panels/main_menu.py** - Menu system
```
Reference panels/main_menu.py for:
- Menu item arrangement
- Vertical/horizontal layouts
- Graph integration
- Status display
```

### Widget Examples

```
Reference ks_includes/widgets/ for:
- keyboard.py - On-screen keyboard
- keypad.py - Numeric keypad
- heatergraph.py - Temperature graphs
- prompts.py - Confirmation dialogs
```

## Testing and Validation

### Testing Generated Panels

```
Generate a test procedure for a custom KlipperScreen panel:
1. Panel loads without errors
2. UI elements render correctly
3. Buttons respond to clicks
4. Keyboard shows/hides properly
5. Updates work in activate()
6. Cleanup works in deactivate()
7. Process_update() handles printer state
8. Error handling prevents crashes
9. Logging outputs debug info
10. Memory leaks from timers are prevented
```

### Validating HDMI Configuration

```
Create a diagnostic script for KlipperScreen HDMI setup:
- Check if X11 is running (ps aux | grep X)
- Verify DISPLAY environment variable
- List all monitors with xrandr
- Check current DPMS status
- Test xset commands work
- Verify display geometry detection
- Check config file for display settings
- Log all findings for troubleshooting
```

### Debugging with Copilot Assistance

```
I'm debugging a KlipperScreen panel that doesn't update. Help me:
1. Add debug logging at key points
2. Verify process_update() is being called
3. Check if GLib timeout is properly registered
4. Ensure deactivate() removes timeouts
5. Validate printer data access
6. Add try-except blocks around printer data access
7. Log printer data structure for inspection
```

### Code Review Prompts

```
Review this KlipperScreen panel code for:
- Memory leaks (unremoved GLib timeouts)
- Error handling completeness
- Resource cleanup in deactivate()
- GTK 3.0 API compatibility
- Code style consistency with KlipperScreen
- Security issues (G-code injection, etc.)
- Performance concerns (blocking operations)
```

## Common Patterns to Request

### Pattern: Confirmation Dialog

```
Generate code for a KlipperScreen confirmation dialog before sending G-code:
- Use self._screen._confirm_send_action()
- Translatable message with _() function
- Send G-code via printer.gcode.script
- Handle user confirmation/cancellation
```

### Pattern: Periodic Updates

```
Generate code pattern for periodic UI updates in KlipperScreen panel:
- GLib.timeout_add_seconds() in activate()
- Update function returns True to continue
- GLib.source_remove() in deactivate()
- Store timeout_id as instance variable
- Handle None timeout_id safely
```

### Pattern: Configuration Access

```
Show pattern for reading/writing KlipperScreen configuration:
- Read: self._config.get_main_config().get(key, fallback)
- Write: self._config.set(section, key, value)
- Save: self._config.save_user_config_options()
- Get boolean: .getboolean(key, fallback)
- Get integer: .getint(key, fallback)
```

### Pattern: Printer Data Access

```
Generate code pattern for safely accessing printer data:
- Check if key exists: 'heater_bed' in self._printer.data
- Access with fallback: data.get('temperature', 0.0)
- Handle missing data gracefully
- Log when expected data is missing
- Use try-except for nested data
```

## Example Workflow

### Creating a New Panel with Copilot

**Step 1: Architecture Context**
```
I'm creating a new KlipperScreen panel. Explain the panel lifecycle:
- __init__(): Create UI
- activate(): Called when panel shown
- deactivate(): Called when panel hidden
- process_update(action, data): Printer state changes
```

**Step 2: Generate Scaffold**
```
Generate a KlipperScreen panel skeleton for monitoring bed mesh:
- File: panels/bed_mesh_monitor.py
- Class: Panel(ScreenPanel)
- Show mesh profile name
- Display mesh min/max values
- Show probed points count
- Refresh button
- Follow KlipperScreen conventions
```

**Step 3: Add Functionality**
```
Add to the bed mesh monitor panel:
- Visualize mesh with color gradient
- Use Gtk.DrawingArea for drawing
- Calculate colors based on height values
- Draw grid of mesh points
- Add legend showing height scale
- Update on process_update() when bed_mesh changes
```

**Step 4: Polish and Test**
```
Add to the bed mesh monitor panel:
- Error handling for missing mesh data
- Loading state while probing
- Tooltips on hover (if possible)
- Responsive layout for vertical mode
- Save/load mesh profiles
- Consistent styling with other panels
```

**Step 5: Integration**
```
Generate KlipperScreen.conf menu entry for bed mesh monitor panel:
- Add to calibration menu
- Appropriate icon
- Translatable name
- Show only when bed_mesh is configured
```

## Learning Resources

### Effective Prompt Engineering

**Be Specific:**
❌ "Create a panel"
✅ "Create a KlipperScreen panel that inherits from ScreenPanel, displays extruder temperature, and updates every 5 seconds"

**Provide Context:**
❌ "Add a button"
✅ "Add a button using self._gtk.Button() with 'home' icon, 'Home All' label, 'color1' style, and connect it to a method that sends G28 command"

**Request Examples:**
❌ "How do I show the keyboard?"
✅ "Show example code for displaying the KlipperScreen on-screen keyboard when a Gtk.Entry gets focus, using the show_keyboard() method"

### Iterative Refinement

Start broad, then refine:
1. "Create a temperature control panel"
2. "Add target temperature input with number pad"
3. "Add preheat presets from config"
4. "Add temperature graph"
5. "Optimize layout for vertical mode"

### Understanding AI Limitations

Copilot may not know:
- Latest KlipperScreen API changes
- Project-specific conventions
- Hardware-specific behaviors
- Moonraker WebSocket protocol details

Always:
- Verify generated code against actual codebase
- Test with real printer
- Check documentation for API changes
- Review for security issues

## Additional Resources

- [Extending Interface Documentation](Extending_Interface.md) - Comprehensive panel development guide
- [HDMI Configuration Documentation](HDMI_Configuration.md) - Display setup and troubleshooting
- [KlipperScreen Configuration](Configuration.md) - Configuration file reference
- [Developer Setup](Developers.md) - Development environment setup
- [Panel Gallery](Panels.md) - Screenshots and examples of existing panels

## Getting Help

When Copilot-generated code doesn't work:

1. **Check the logs**: `~/printer_data/logs/KlipperScreen.log`
2. **Review existing code**: Look at similar panels for working examples
3. **Simplify**: Remove complexity until basic functionality works
4. **Ask specific questions**: "Why does this GTK callback fail?" instead of "Fix my code"
5. **Consult documentation**: This guide and [Extending Interface](Extending_Interface.md)

Remember: AI is a tool to accelerate development, not replace understanding. Always review and test generated code thoroughly.

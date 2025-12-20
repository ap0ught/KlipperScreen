# Extending the KlipperScreen Interface

This guide provides comprehensive information for developers who want to extend KlipperScreen by creating custom panels, widgets, and interfaces.

## Architecture Overview

KlipperScreen is built on GTK 3.0 and follows a panel-based architecture that makes it easy to extend and customize.

### Core Components

#### KlipperScreen Window
The main application class `KlipperScreen(Gtk.Window)` in `screen.py` manages:
- GTK windowing system initialization
- Display and monitor configuration
- Panel loading and navigation
- WebSocket connections to Moonraker
- Keyboard and input management
- Power management (DPMS)

#### Panel System
Panels are the primary UI components in KlipperScreen. Each panel is a self-contained module that handles:
- Its own layout and widgets
- Updates from the printer state
- User interactions
- Lifecycle management

The panel hierarchy:
```
ScreenPanel (ks_includes/screen_panel.py)
    ↓
BasePanel (panels/base_panel.py) 
    ↓
Custom Panels (panels/*.py)
```

#### Key Architectural Patterns
- **Panel Registration**: Panels are loaded dynamically from the `panels/` directory
- **State Management**: Printer state is managed centrally and pushed to active panels
- **Event-Driven**: Panels react to printer state changes via callbacks
- **Reusable Widgets**: Common UI elements are provided by `KlippyGtk`

## Creating Custom Panels

### Basic Panel Structure

All panels should inherit from `ScreenPanel` or `BasePanel`. Here's a minimal example:

```python
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk
from ks_includes.screen_panel import ScreenPanel


class Panel(ScreenPanel):
    def __init__(self, screen, title, **kwargs):
        title = title or _("My Custom Panel")
        super().__init__(screen, title)
        
        # Create your UI elements here
        label = Gtk.Label(label="Hello from Custom Panel!")
        button = self._gtk.Button("home", _("Test Button"))
        button.connect("clicked", self.on_button_clicked)
        
        # Layout
        grid = Gtk.Grid(row_homogeneous=True, column_homogeneous=True)
        grid.attach(label, 0, 0, 2, 1)
        grid.attach(button, 0, 1, 2, 1)
        
        self.content.add(grid)
    
    def on_button_clicked(self, widget):
        self._screen.show_popup_message("Button was clicked!")
```

### Panel Class Requirements

Every panel must:
1. Have a class named `Panel`
2. Accept `screen` and `title` parameters in `__init__`
3. Call `super().__init__(screen, title)` 
4. Add content to `self.content` (a `Gtk.Box`)

### Accessing Core Services

Through `self._screen`, you can access:

```python
# Configuration
self._config.get_main_config()
self._config.get_printer_config(section)

# Printer state
self._printer.state  # 'ready', 'printing', 'paused', etc.
self._printer.get_temp_devices()
self._printer.get_tools()

# GTK utilities
self._gtk.Button(icon, label, style, scale)
self._gtk.Image(name, scale)
self._gtk.Label(text)

# Files and metadata
self._files.get_file_list()
self._files.get_file_info(filename)

# Show other panels
self._screen.show_panel(panel_name, **kwargs)

# Display messages
self._screen.show_popup_message(message)
```

### Panel Lifecycle Methods

Override these methods to hook into the panel lifecycle:

```python
def activate(self):
    """Called when panel becomes visible"""
    # Start updates, timers, etc.
    pass

def deactivate(self):
    """Called when panel is hidden"""
    # Stop updates, remove timers
    pass

def process_update(self, action, data):
    """Called when printer state changes"""
    # Update UI based on new printer data
    pass
```

### Complete Working Example

Here's a complete example of a custom status panel:

```python
import logging
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GLib
from ks_includes.screen_panel import ScreenPanel


class Panel(ScreenPanel):
    def __init__(self, screen, title, **kwargs):
        title = title or _("Printer Status")
        super().__init__(screen, title)
        
        self.update_timeout = None
        
        # Create status labels
        self.labels = {}
        self.labels['state'] = Gtk.Label(label="State: Unknown")
        self.labels['state'].get_style_context().add_class("temperature_entry")
        
        self.labels['position'] = Gtk.Label(label="Position: ---")
        self.labels['temperature'] = Gtk.Label(label="Hotend: ---°C")
        self.labels['bed'] = Gtk.Label(label="Bed: ---°C")
        
        # Create action buttons
        home_button = self._gtk.Button("home", _("Home All"), "color1")
        home_button.connect("clicked", self.home_all)
        
        refresh_button = self._gtk.Button("refresh", _("Refresh"), "color2")
        refresh_button.connect("clicked", self.refresh_status)
        
        # Layout
        grid = Gtk.Grid(row_homogeneous=True, column_homogeneous=True)
        grid.set_row_spacing(10)
        grid.set_column_spacing(10)
        
        # Status section
        grid.attach(self.labels['state'], 0, 0, 2, 1)
        grid.attach(self.labels['position'], 0, 1, 2, 1)
        grid.attach(self.labels['temperature'], 0, 2, 2, 1)
        grid.attach(self.labels['bed'], 0, 3, 2, 1)
        
        # Buttons section
        grid.attach(home_button, 0, 4, 1, 1)
        grid.attach(refresh_button, 1, 4, 1, 1)
        
        self.content.add(grid)
    
    def activate(self):
        """Start periodic updates when panel is shown"""
        self.update_status()
        self.update_timeout = GLib.timeout_add_seconds(2, self.update_status)
    
    def deactivate(self):
        """Stop updates when panel is hidden"""
        if self.update_timeout is not None:
            GLib.source_remove(self.update_timeout)
            self.update_timeout = None
    
    def update_status(self):
        """Update status labels from printer state"""
        # Update state
        state = self._printer.state
        self.labels['state'].set_text(f"State: {state}")
        
        # Update position
        if 'toolhead' in self._printer.data:
            pos = self._printer.data['toolhead']['position']
            self.labels['position'].set_text(
                f"Position: X:{pos[0]:.1f} Y:{pos[1]:.1f} Z:{pos[2]:.1f}"
            )
        
        # Update temperatures
        if 'extruder' in self._printer.data:
            temp = self._printer.data['extruder']['temperature']
            target = self._printer.data['extruder']['target']
            self.labels['temperature'].set_text(
                f"Hotend: {temp:.1f}°C / {target:.1f}°C"
            )
        
        if 'heater_bed' in self._printer.data:
            temp = self._printer.data['heater_bed']['temperature']
            target = self._printer.data['heater_bed']['target']
            self.labels['bed'].set_text(
                f"Bed: {temp:.1f}°C / {target:.1f}°C"
            )
        
        return True  # Continue periodic updates
    
    def home_all(self, widget):
        """Send home all command"""
        self._screen._ws.klippy.gcode_script("G28")
        self._screen.show_popup_message(_("Homing all axes..."))
    
    def refresh_status(self, widget):
        """Manually refresh status"""
        self.update_status()
        self._screen.show_popup_message(_("Status refreshed"))
```

### Registering Your Panel

To make your panel available:

1. Save it as `panels/my_custom_panel.py` (filename must match panel name)
2. The panel is automatically discovered by KlipperScreen
3. Add it to menus in `KlipperScreen.conf`:

```ini
[menu __main]
name: Main Menu

[menu __main custom_status]
name: My Status
icon: info
panel: my_custom_panel
```

## Widget System

### Using Built-in Widgets

KlipperScreen provides many pre-built widgets through `self._gtk`:

```python
# Buttons
button = self._gtk.Button(
    icon_name="home",      # Icon name from theme
    label="Home",          # Button text
    style="color1",        # CSS style class
    scale=1.0              # Size scale
)

# Images
image = self._gtk.Image("printer", scale=2.0)

# Scrollable containers
scroll = self._gtk.ScrolledWindow()
scroll.add(content_widget)

# Dialog/confirmation
self._screen._confirm_send_action(
    widget,
    _("Are you sure?"),
    "printer.gcode.script",
    {"script": "M112"}
)
```

### Creating Custom Widgets

Custom widgets should inherit from GTK widgets:

```python
import gi
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk


class CustomWidget(Gtk.Box):
    def __init__(self, screen):
        super().__init__(orientation=Gtk.Orientation.VERTICAL)
        self._screen = screen
        
        # Build widget UI
        self.label = Gtk.Label(label="Custom Widget")
        self.pack_start(self.label, True, True, 0)
    
    def update_data(self, value):
        """Public method to update widget"""
        self.label.set_text(str(value))
```

Usage in panel:
```python
self.custom_widget = CustomWidget(self._screen)
self.content.add(self.custom_widget)
```

### GTK Widget Hierarchy

Common GTK containers used in KlipperScreen:

- **Gtk.Box**: Linear layout (horizontal or vertical)
- **Gtk.Grid**: Grid-based layout with rows and columns
- **Gtk.ScrolledWindow**: Scrollable container
- **Gtk.Overlay**: Stack widgets on top of each other
- **Gtk.Paned**: Adjustable split view

Layout example:
```python
# Create a grid layout
grid = Gtk.Grid(
    row_homogeneous=True,    # Equal row heights
    column_homogeneous=True  # Equal column widths
)
grid.set_row_spacing(5)
grid.set_column_spacing(5)

# Attach widgets: (widget, left, top, width, height)
grid.attach(widget1, 0, 0, 2, 1)  # Span 2 columns
grid.attach(widget2, 0, 1, 1, 1)
grid.attach(widget3, 1, 1, 1, 1)
```

## Keyboard Integration

KlipperScreen provides a built-in on-screen keyboard for touch devices.

### Showing the Keyboard

For text entry fields, use `Gtk.Entry` with the keyboard:

```python
# Create entry field
entry = Gtk.Entry()
entry.set_text("Default value")

# Set input type for specialized keyboards
entry.set_input_purpose(Gtk.InputPurpose.NUMBER)  # Number pad
# or Gtk.InputPurpose.DIGITS for integers only
# or Gtk.InputPurpose.FREE_FORM for full keyboard

# Connect to show keyboard on focus
entry.connect("focus-in-event", self._screen.show_keyboard)

# Add to layout
self.content.add(entry)
```

### Keyboard Input Purposes

```python
Gtk.InputPurpose.FREE_FORM  # Full QWERTY keyboard
Gtk.InputPurpose.DIGITS     # Integer number pad (0-9)
Gtk.InputPurpose.NUMBER     # Decimal number pad (0-9, ., +/-)
```

### Custom Keyboard Handling

For more control:

```python
def show_custom_keyboard(self, widget, event):
    """Custom keyboard display handler"""
    entry = widget
    
    # Show keyboard with custom callback
    self._screen.show_keyboard(
        entry=entry,
        event=event,
        box=self.content,  # Container to add keyboard to
        close_cb=self.keyboard_closed  # Called when closed
    )

def keyboard_closed(self, widget, event=None):
    """Handle keyboard dismissal"""
    # Get value from entry
    value = widget.get_text()
    logging.info(f"User entered: {value}")
    
    # Process the input
    self.process_input(value)
    
    # Remove keyboard
    self._screen.remove_keyboard()
```

### Touch and Mouse Events

Handle user interactions:

```python
# Button click
button.connect("clicked", self.on_clicked)

# Long press
button.connect("button-press-event", self.on_press_event)

def on_press_event(self, widget, event):
    if event.button == 1:  # Left mouse button / touch
        # Start timer for long press
        GLib.timeout_add(500, self.long_press_action, widget)
    return False

# Entry changes
entry.connect("changed", self.on_text_changed)

def on_text_changed(self, entry):
    text = entry.get_text()
    # Validate or process text
```

## Best Practices

### Panel Development

1. **Keep panels focused**: Each panel should have a single, clear purpose
2. **Use lifecycle methods**: Always clean up in `deactivate()`
3. **Handle state changes**: Implement `process_update()` for real-time updates
4. **Responsive design**: Test in both landscape and portrait modes using `self._screen.vertical_mode`
5. **Error handling**: Use try-except blocks around printer commands
6. **Logging**: Use Python's `logging` module for debugging

### Performance Tips

1. **Limit update frequency**: Use `GLib.timeout_add_seconds()` instead of `timeout_add()`
2. **Remove timeouts**: Always remove GLib timeouts in `deactivate()`
3. **Batch updates**: Update multiple UI elements in single function call
4. **Lazy loading**: Only create heavy widgets when panel is activated

### Code Organization

```python
class Panel(ScreenPanel):
    def __init__(self, screen, title, **kwargs):
        """Initialize UI and state"""
        super().__init__(screen, title)
        self.create_ui()
    
    def create_ui(self):
        """Separate UI creation from initialization"""
        pass
    
    def activate(self):
        """Start services when shown"""
        pass
    
    def deactivate(self):
        """Stop services when hidden"""
        pass
    
    def process_update(self, action, data):
        """Handle printer state updates"""
        pass
    
    # UI event handlers
    def on_button_clicked(self, widget):
        pass
    
    # Helper methods
    def update_display(self):
        pass
```

## Common Patterns

### Sending G-code Commands

```python
# Simple G-code
self._screen._ws.klippy.gcode_script("G28")

# G-code with confirmation
self._screen._confirm_send_action(
    widget,
    _("Home all axes?"),
    "printer.gcode.script",
    {"script": "G28"}
)

# Multiple commands
script = "G28\nG1 Z10 F300"
self._screen._ws.klippy.gcode_script(script)
```

### Reading Printer Configuration

```python
# Get config value
max_velocity = self._printer.config.getfloat(
    'printer', 'max_velocity', fallback=300
)

# Check if feature exists
has_bed_mesh = 'bed_mesh' in self._printer.config.sections()
```

### Periodic Updates

```python
def activate(self):
    # Update every 2 seconds
    self.update_timeout = GLib.timeout_add_seconds(2, self.periodic_update)

def deactivate(self):
    if self.update_timeout:
        GLib.source_remove(self.update_timeout)
        self.update_timeout = None

def periodic_update(self):
    # Update UI
    self.update_display()
    return True  # Continue calling
```

## Anti-patterns to Avoid

1. **Don't modify core files**: Create new panels instead of modifying existing ones
2. **Don't block the UI thread**: Use async operations for network/file I/O
3. **Don't hardcode values**: Use configuration files or printer settings
4. **Don't assume printer state**: Always check if devices exist before accessing
5. **Don't forget cleanup**: Memory leaks from unremoved timeouts are common

## Debugging

### Logging

```python
import logging

logging.debug("Debug information")
logging.info("General information")
logging.warning("Warning message")
logging.error("Error occurred")
```

Logs are written to `~/printer_data/logs/KlipperScreen.log`

### GTK Inspector

Enable GTK inspector for debugging UI:
```bash
gsettings set org.gtk.Settings.Debug enable-inspector-keybinding true
```

Then press `Ctrl+Shift+D` in KlipperScreen to open the inspector.

### Print Debugging

```python
# Print to console (when running from terminal)
print(f"Value: {value}")

# Show on screen
self._screen.show_popup_message(f"Debug: {value}")
```

## Examples from Core Panels

Study these panels for real-world examples:

- **panels/example.py**: Minimal panel template
- **panels/temperature.py**: Complex layout with graphs and controls
- **panels/move.py**: Numeric input and keyboard usage
- **panels/main_menu.py**: Menu system and navigation
- **panels/console.py**: Scrolling list and text input
- **panels/settings.py**: Configuration and toggles

## Additional Resources

- [GTK 3 Python Tutorial](https://python-gtk-3-tutorial.readthedocs.io/)
- [KlipperScreen Configuration Guide](Configuration.md)
- [Panel Gallery](Panels.md)
- [HDMI Display Configuration](HDMI_Configuration.md)
- [Developer Setup](Developers.md)

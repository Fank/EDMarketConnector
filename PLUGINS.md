# EDMC Plugins

Plugins allow you to customise and extend the behavior of EDMC.

# Installing a Plugin

EDMC loads all plugins it finds in it's `plugins` folder.  You can easily find this on your system via the Plugins tab
of the Settings window.

# Writing a Plugin

Plugins are loaded when EDMC starts up.

Each plugin has it's own folder in the `plugins` directory:

* Windows: `%LOCALAPPDATA%\EDMarketConnector\plugins`
* Mac: `~/Library/Application Support/EDMarketConnector/plugins`
* Linux: `$XDG_DATA_HOME/EDMarketConnector/plugins`, or `~/.local/share/EDMarketConnector/plugins` if `$XDG_DATA_HOME` is unset.

Plugins are python files. The plugin folder must have a file named `load.py` that must provide one module level function and optionally provide a few others.

EDMC will import the `load.py` file as a module and then call the `plugin_start()` function.

```python
def plugin_start(plugin_dir):
   """
   Load this plugin into EDMC
   """
   print "I am loaded! My plugin folder is {}".format(plugin_dir.encode("utf-8"))
   return "Test"
```

Any errors or print statements from your plugin will appear in `%TMP%\EDMarketConnector.log` on Windows or `$TMPDIR/EDMarketConnector.log` on Mac.

This gets called when the user closes the program:

```python
def plugin_stop():
    """
    EDMC is closing
    """
    print "Farewell cruel world!"
```

If your plugin uses one or more threads to handle Events then stop and join() the threads before returning from this function.

# Plugin Hooks
## Configuration 

If you want your plugin to be configurable via the GUI you can define a frame (panel) to be displayed on its own tab in EDMC's settings dialog. The tab title will be the value that you returned from `plugin_start`. Use widgets from EDMC's myNotebook.py for the correct look-and-feel. You can be notified when the settings dialog is closed so you can save your settings.

You can use `set()`, `get()` and `getint()` from EDMC's `config.config` object to retrieve your plugin's settings in a platform-independent way.

Use `numberFromString()` from EDMC's `l10n.Locale` object to parse input numbers in a locale-independent way.

```python
import Tkinter as tk
import myNotebook as nb
from config import config

this = sys.modules[__name__]	# For holding module globals

def plugin_prefs(parent, cmdr, is_beta):
   """
   Return a TK Frame for adding to the EDMC settings dialog.
   """
   this.mysetting = tk.IntVar(value=config.getint("MyPluginSetting"))	# Retrieve saved value from config
   frame = nb.Frame(parent)
   nb.Label(frame, text="Hello").grid()
   nb.Label(frame, text="Commander").grid()
   nb.Checkbutton(frame, text="My Setting", variable=this.mysetting).grid()

   return frame
```

This gets called when the user dismisses the settings dialog:

```python
def prefs_changed(cmdr, is_beta):
   """
   Save settings.
   """
   config.set('MyPluginSetting', this.mysetting.getint())	# Store new value in config
```

## Display

You can also have your plugin add an item to the EDMC main window and update from your event hooks. This works in the same way as `plugin_prefs()`. For a simple one-line item return a tk.Label widget or a pair of widgets as a tuple. For a more complicated item create a tk.Frame widget and populate it with other ttk widgets. Return `None` if you just want to use this as a callback after the main window and all other plugins are initialised.

You can use `stringFromNumber()` from EDMC's `l10n.Locale` object to format numbers in your widgets in a locale-independent way.

```python
this = sys.modules[__name__]	# For holding module globals

def plugin_app(parent):
    """
    Create a pair of TK widgets for the EDMC main window
    """
    label = tk.Label(parent, text="Status:")    # By default widgets inherit the current theme's colors
    this.status = tk.Label(parent, text="", foreground="yellow")    # Override theme's foreground color
    return (label, this.status)
   
# later on your event functions can update the contents of these widgets
    this.status["text"] = "Happy!"
    this.status["foreground"] = "green"
```

You can dynamically add and remove widgets on the main window by returning a tk.Frame from `plugin_app()` and later creating and destroying child widgets of that frame.

```python
from theme import theme

this = sys.modules[__name__]	# For holding module globals

def plugin_app(parent):
    """
    Create a frame for the EDMC main window
    """
    this.frame = tk.Frame(parent)
    return this.frame

# later on your event functions can add or remove widgets
    row = this.frame.grid_size()[1]
    new_widget_1 = tk.Label(this.frame, text="Status:")
    new_widget_1.grid(row=row, column=0, sticky=tk.W)
    new_widget_2 = tk.Label(this.frame, text="Unhappy!", foreground="red")    # Override theme's foreground color
    new_widget_2.grid(row=row, column=1, sticky=tk.W)
    theme.update(this.frame)	# Apply theme colours to the frame and its children, including the new widgets
```

## Events

Once you have created your plugin and EDMC has loaded it there are three other functions you can define to be notified by EDMC when something happens: `journal_entry()`, `dashboard_entry()` and `cmdr_data()`.

Your events all get called on the main Tkinter loop so be sure not to block for very long or the app will appear to freeze. If you have a long running operation such as sending or receiving data from an external server then you should do this in a separate worker Thread. You can send work items to the worker thread over a Queue. Tkinter is not thread-safe so you should not access any Tkinter resources (including widgets and variables) from worker threads - doing so may cause the app to crash intermittently. You can signal back to the main thread using Tkinter's `event_generate()` widget method, generating a user-defined event that you have previously registered with the [`bind_all()`](http://effbot.org/tkinterbook/tkinter-events-and-bindings.htm) widget method. See the [EDSM plugin](https://github.com/Marginal/EDMarketConnector/blob/master/plugins/edsm.py) for an example of these techniques.

### Journal Entry

This gets called when EDMC sees a new entry in the game's journal. `state` is a dictionary containing information about the Cmdr and their ship and cargo (including the effect of the current journal entry).

A special "StartUp" entry is sent if EDMC is started while the game is already running. In this case you won't receive initial events such as "LoadGame", "Rank", "Location", etc. However the `state` dictionary will reflect the cumulative effect of these missed events.

Similarly, a special "ShutDown" entry is sent when the game is quitted while EDMC is running. This event is not sent when EDMC is running on a different machine so you should not *rely* on receiving this event.

```python
def journal_entry(cmdr, is_beta, system, station, entry, state):
    if entry['event'] == 'FSDJump':
        # We arrived at a new system!
        if 'StarPos' in entry:
            sys.stderr.write("Arrived at {} ({},{},{})\n".format(entry['StarSystem'], *tuple(entry['StarPos'])))
        else:
            sys.stderr.write("Arrived at {}\n".format(entry['StarSystem']))
```

### Player Dashboard

This gets called when something on the player's cockpit display changes - typically about once a second when in orbital flight. See the "Status File" section in the Frontier [Journal documentation](https://forums.frontier.co.uk/showthread.php/401661) for the available `entry` properties and for the list of available `"Flags"`. Refer to the source code of [plug.py](./plug.py) for the list of available constants.

```python
import plug

def dashboard_entry(cmdr, is_beta, entry):
    is_deployed = entry['Flags'] & plug.FlagsHardpointsDeployed
    sys.stderr.write("Hardpoints {}\n".format(is_deployed and "deployed" or "stowed"))
```

### Getting Commander Data

This gets called when EDMC has just fetched fresh Cmdr and station data from Frontier's servers.

```python
def cmdr_data(data, is_beta):
    """
    We have new data on our commander
    """
    sys.stderr.write(data.get('commander') and data.get('commander').get('name') or '')
```

The data is a dictionary and full of lots of wonderful stuff!

### Plugin-specific events

If the player has chosen to "Send flight log and Cmdr status to EDSM" this gets called when the player starts the game or enters a new system. It is called some time after the corresponding `journal_entry()` event.

```python
def edsm_notify_system(reply):
    """
    `reply` holds the response from a call to https://www.edsm.net/en/api-journal-v1
    """
    if not reply:
        sys.stderr.write("Error: Can't connect to EDSM\n")
    elif reply['msgnum'] // 100 not in (1,4):
        sys.stderr.write('Error: EDSM {MSG}\n').format(MSG=reply['msg'])
    elif reply.get('systemCreated'):
        sys.stderr.write('New EDSM system!\n')
    else:
        sys.stderr.write('Known EDSM system\n')
```

If the player has chosen to "Send flight log and Cmdr status to Inara" this gets called when the player starts the game, enters a new system, docks or undocks. It is called some time after the corresponding `journal_entry()` event.

```python
def inara_notify_location(eventData):
    """
    `eventData` holds the response to one of the "Commander's Flight Log" events https://inara.cz/inara-api-docs/#event-29
    """
    if eventData.get('starsystemInaraID'):
        sys.stderr.write('Now in Inara system {ID} at {URL}\n'.format(ID=eventData['starsystemInaraID'], URL=eventData['starsystemInaraURL']))
    else:
        sys.stderr.write('System not known to Inara\n')
    if eventData.get('stationInaraID'):
        sys.stderr.write('Docked at Inara station {ID} at {URL}\n'.format(ID=eventData['stationInaraID'], URL=eventData['stationInaraURL']))
    else:
        sys.stderr.write('Undocked or station unknown to Inara\n')
```

If the player has chosen to "Send flight log and Cmdr status to Inara" this gets called when the player starts the game or switches ship. It is called some time after the corresponding `journal_entry()` event.

```python
def inara_notify_ship(eventData):
    """
    `eventData` holds the response to an addCommanderShip or setCommanderShip event https://inara.cz/inara-api-docs/#event-11
    """
    if eventData.get('shipInaraID'):
        sys.stderr.write('Now in Inara ship {ID} at {URL}\n'.format(ID=eventData['shipInaraID'], URL=eventData['shipInaraURL']))
```

## Error messages

You can display an error in EDMC's status area by returning a string from your `journal_entry()`, `dashboard_entry()` or `cmdr_data()` function, or asynchronously (e.g. from a "worker" thread that is performing a long running operation) by calling `plug.show_error()`. Either method will cause the "bad" sound to be played (unless the user has muted sound).

The status area is shared between EDMC itself and all other plugins, so your message won't be displayed for very long. Create a dedicated widget if you need to display routine status information.

## Localisation

You can localise your plugin to one of the languages that EDMC itself supports. Add the following boilerplate near the top of each source file that contains strings that needs translating:

```python
import l10n
import functools
_ = functools.partial(l10n.Translations.translate, context=__file__)
```

Wrap each string that needs translating with the `_()` function, e.g.:

```python
    this.status["text"] = _('Happy!')	# Main window status
```

If you display localized strings in EDMC's main window you should refresh them in your `prefs_changed` function in case the user has changed their preferred language.

Translation files should reside in folder named `L10n` inside your plugin's folder. Files must be in macOS/iOS ".strings" format, encoded as UTF-8. You can generate a starting template file for your translations by invoking `l10n.py` in your plugin's folder. This extracts all the translatable strings from Python files in your plugin's folder and places them in a file named `en.template` in the `L10n` folder. Rename this file as `<language_code>.strings` and edit it.

See EDMC's own [`L10n`](https://github.com/EDCD/EDMarketConnector/tree/master/L10n) folder for the list of supported language codes and for example translation files.


# Python Package Plugins

A _Package Plugin_ is both a standard Python package (i.e. contains an `__init__.py` file) and an EDMC plugin (i.e. contains a `load.py` file providing at minimum a `plugin_start()` function). These plugins are loaded before any non-Package plugins.

Other plugins can access features in a Package Plugin by `import`ing the package by name in the usual way.


# Distributing a Plugin

To package your plugin for distribution simply create a `.zip` archive of your plugin's folder:

* Windows: In Explorer right click on your plugin's folder and choose Send to &rarr; Compressed (zipped) folder.
* Mac: In Finder right click on your plugin's folder and choose Compress.

If there are any external dependencies then include them in the plugin's folder.

Optionally, for tidiness delete any `.pyc` and `.pyo` files in the archive.

# Disable a plugin

EDMC now lets you disable a plugin without deleting it, simply rename the plugin folder to append ".disabled". Eg,
"SuperSpaceHelper" -> "SuperSpaceHelper.disabled"

Disabled and enabled plugins are listed on the "Plugins" Settings tab

# Migration to Python 3.7

In a future release EDMC will use Python **3.7**, instead of the Python 2.7 it has historically used.  As of mid-2020 there is a [python3 branch](https://github.com/EDCD/EDMarketConnector/tree/python3) to test migrated plugins against.  This is a brief outline of the steps required to migrate a plugin from earlier versions of EDMC:

- Rename the function `plugin_start` to `plugin_start3(plugin_dir)`. Plugins without a `plugin_start3` function will be listed as disabled on EDMC's "Plugins" tab and a message like "plugin SuperSpaceHelper needs migrating" will appear in the log.
- Check that callback functions `plugin_prefs`, `prefs_changed`, `journal_entry`, `dashboard_entry` and `cmdr_data` if used are declared with the correct number of arguments. Older versions of this app were tolerant of missing arguments in these function declarations.
- Port the code to Python 3.7. The [2to3](https://docs.python.org/3/library/2to3.html) tool can automate much of this work.

Depending on the complexity of the plugin it may be feasible to make it compatible with both EDMC 3.4 + Python 2.7 and later EDMC + Python 3.7. [Here's](https://python-future.org/compatible_idioms.html) a guide on writing Python 2/3 compatible code and [here's](https://github.com/Marginal/HabZone/commit/3c41cd41d5ad81ef36aab40e967e3baf77b4bd06) an example of the changes required for a simple plugin.

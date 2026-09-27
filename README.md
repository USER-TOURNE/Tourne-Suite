# Tourne'Suite

A work in progress: every way to theme Windows, from Windows 7 to the latest Windows 11 Insider builds, in one place. It's built on [Windhawk](https://windhawk.net/) as one mod made of parts, and every part can be switched on or off on its own.

![Desktop](images/01-desktop.png)

## Goals

| Goal | Status |
|---|---|
| **StartAllBack, rebuilt from the ground up.** A classic taskbar and Windows 7 style Start menu that work with existing StartAllBack themes | In progress: the theme reader works (see below) |
| **Winaero Tweaker, rebuilt from the ground up** | Done: 312 Windows settings as toggles |
| **A SecureUxTheme replacement**, for applying unsigned visual styles | Planned |
| **Various other modifications**: tray icons, flyouts, icon themes, device batteries and more | Working today, shown below |

## What works today

### Taskbar and tray

![Taskbar](images/02-taskbar.png)

- **System Icons.** Pixel-art tray icons for network, volume, VPN, battery, brightness, notifications, Bluetooth, Windows Security and more, each showing live state.
- **Tray Clock.** Restyles the classic clock.
- **Classic Tray Folders.** Groups tray icons into folders.
- **Change Tray Icons.** Swaps any app's tray icon for a themed one.

### Flyouts

Every tray icon opens its own flyout in the same style, in the spirit of the Windows 7 ones:

| Volume and per-app mixer | Network |
|---|---|
| ![Volume mixer](images/04-volume-mixer.png) | ![Network flyout](images/05-network-flyout.png) |

| Hidden icons | Device batteries |
|---|---|
| ![Hidden icons](images/06-hidden-icons.png) | ![Device batteries](images/07-device-batteries.png) |

There's also a power flyout with plans, brightness over DDC/CI, VPN (Windscribe), notifications with a calendar, Windows Security, and Bluetooth with connect and disconnect. Flyouts widen to fit their text automatically, up to a maximum you set.

### Device batteries

Wireless headsets, mice, keyboards and controllers report their battery right in the tray. It reads the level straight from the device, so no vendor app needs to be running. This is a port of HaloBattery (MIT) to C++.

- **Brands:** Alienware, Razer, Logitech, SteelSeries, HyperX, Audeze, MCHOSE, WLmouse, PlayStation and Xbox controllers, plus any Bluetooth device that reports its battery to Windows.
- **Where it shows:** a tray icon per device (the level drawn as a ring around a small picture of the device), inside the existing flyouts, or both.
- **How often:** anywhere from every 5 seconds to every 24 hours (`30sec`, `9min`, `3hr`), and immediately when a device is plugged in or removed.
- **Alerts:** a low battery alert that fires once and waits until the device has been charged before firing again.

### Everything else

- **Icon Redirect.** An icon theme engine written from scratch (MIT), which replaces the Resource Redirect mod. It applies system-wide icon themes and single-app icon swaps.
- **Tweaks.** 312 Windows settings as toggles (Explorer, the desktop, menus, power, privacy and more), applied live where Windows allows it.
- **Text and menus.** A custom system text colour (including Chrome's menus), and context menus without icons and without duplicate entries.
- **Tourne'Table.** The audio visualizer next to the Start orb.

![Start orb and visualizer](images/08-orb-and-visualizer.png)

### A tour of the settings

Every part is a group on one Windhawk settings page. **[Watch the two-minute scroll through all of them](video/tourne-suite-settings-tour.mp4)** (no sound).

## In progress: our own taskbar and Start menu

![Start menu](images/03-start-menu.png)

The taskbar and Start menu above are StartAllBack's, styled with the Bouquet SAB theme. The next big piece is our own classic taskbar and Windows 7 style Start menu inside the suite, **drawn straight from StartAllBack themes**, so the themes people already use keep working without StartAllBack.

Step one works. The suite can now load a StartAllBack `.msstyles` and draw every part of it through Windows' own theme engine. Here are the Bouquet SAB taskbar buttons in all their states, plus the progress bar colours and the separators, drawn without StartAllBack running:

![StartAllBack theme reader](images/09-sab-theme-reader.png)

Coming up, in order:

1. A test-mode taskbar that runs alongside StartAllBack.
2. Grouping, pinned apps, jump lists, thumbnails, the clock and show desktop.
3. Tray icon hosting.
4. The Windows 7 style Start menu.
5. A switch that retires StartAllBack.

## Credits

- [Windhawk](https://windhawk.net/), which everything runs on.
- The **Bouquet** icon and StartAllBack theme by **niivu**.
- **HaloBattery** (MIT), whose device protocols the battery reader is ported from.

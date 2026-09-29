# Devlog: our own taskbar, builds b8 to b19

Two days (2026-09-28 and 09-29) spent turning the test-mode taskbar into a 1:1 stand-in for StartAllBack (SAB). The bar and Start menu are drawn straight from SAB's `.msstyles`, so a theme made for SAB should look the same in both. Every change below was checked against SAB side by side where that was possible.

## How it's built and checked

- **One file per build.** `builds/tourne-suite-bN.wh.cpp` is generated from the parts, with its own mod id (`local@tourne-suite-bN`), so a new build installs next to the old one. Settings are copied from whichever build is running, so nothing has to be set up again.
- **A test bar outside Windhawk.** The same code runs as a plain exe (`bartest`), driven by `TB_*` environment variables, so a change can be photographed and compared without touching the live taskbar. `Compare-Matrix` shoots SAB and ours for four styles (Default, Bouquet SAB, Plain8, Windows 7), idle and hovered, and diffs them.
- **Ground truth is a screenshot from the real screen.** Our own BitBlt and DXGI photos show SAB's acrylic areas as copper, which they are not. Colours were measured from real screenshots only.
- Both x64 and x86 are compile-checked for every build.

![Ours against StartAllBack, Bouquet SAB](images/18-parity-bouquet.png)

Rows one and three are ours (idle, hovered), rows two and four are StartAllBack. The orb differs only because the test bar doesn't load SAB's orb. The Windows 7 style, same layout:

![Ours against StartAllBack, Windows 7](images/19-parity-windows7.png)

## b8 to b10: matching the look

- **Follows StartAllBack live.** Changing SAB's style, orb or colours now reloads our bar at once. Before, the two drifted apart until a restart.
- **Bar frame.** A 3 px frame (#1E2E36) around the inner #121C21, measured from SAB, kept out of the custom colouring pass.
- **Button frame and glow.** SAB draws a button's frame 28 px high, 4 px in from the side facing the screen's centre, and keeps the hover glow inside it. Ours now does the same, with our own dynamic aura kept on top.
- **Start menu.** Video-frame comparison against SAB: row height 22, right column pitch 29 px, separator moved up 4 px, cue text colour, search focus, and the capitals setting.

## b9 to b11: lightning fast

The complaint was that right clicks and the Start menu felt heavy. What changed:

- The apps list, folder contents and jump lists are read on background threads and cached, and the caches are dropped when settings or pins change. Hovering a button reads its jump list ahead of the right click.
- The Start menu keeps its back buffers and repaints only the rows that changed. Its home rows are built once while idle.
- Windows' own fade and menu animations are switched off for our popups (`DWMWA_TRANSITIONS_FORCEDISABLED`, `TPM_NOANIMATION`), and menu icons are filled in lazily.

## b12 to b14: the details you notice

- **Click a group.** Clicking a button with several windows brings up the last one you used (Ctrl+click still lists them).
- **Tooltips keep their lines.** Multi-line tray tooltips are no longer squashed into one, and the wrongly sized ones are fixed.
- **Menu frame.** Right-click menus and every shell menu and submenu they open get SAB's double frame (a 1 px line, a 6 px band, another line, then the interior), with the row and text colours measured from Bouquet SAB.
- **Menu icons** come from the icon theme's own imageres.dll (pin, shredder, X), and there's a new **End task** entry.
- **Stacked windows.** A group is drawn as a stack, its edge peeking out behind the button, using the theme's own stack parts (13 for a button, 14 for the active one; state 2 for two windows, 3 for more), with a little room so it doesn't run into the next button.

| Flyout menu | The stack parts, from Bouquet SAB |
|---|---|
| ![Flyout menu](images/20-flyout-menu.png) | ![Stack edge](images/21-stack-edge.png) |

The style's band class, as far as it's decoded (every state of every part, drawn from the file with no lock on it):

![Band parts](images/22-band-parts.png)

Parts 1 to 4 are the bar background, 5 the button (normal, hot, pressed, active, active hot, active pressed, flashing), 6 to 8 its variants, 9 to 12 the progress bar, 13 and 14 the stack edge.

## What happens with StartAllBack switched off

Our tray already hosts the icons itself (it takes them from the tray notifications directly, not from SAB), and the bar and Start menu are ours. With SAB off, Windows' own taskbar comes back, which "Hide StartAllBack's taskbar" plus "Turn StartAllBack off" handle today. What SAB did to Explorer itself (its restyling) is still open.

## b15: four ideas from studying YASB

YASB (a Windows status bar) has four things worth having. All four are in.

- **A bar per screen.** Every other screen gets a bar with its own windows and the clock. The main bar keeps the Start button, the tray and the watching of windows. Which windows go where follows Windows' "Show taskbar buttons on" (all bars, main plus where open, only where open), and pinned apps stay on the main bar unless Windows shares every button everywhere. New setting: *A bar on every screen* (as Windows is set, always, never). Screens plugged in or removed are picked up live.
- **Hot reload.** Save a theme file, drop one in the Styles or Orbs folder, or change something in StartAllBack, and the bar reloads about 0.4 s later. A burst of changes is one reload.
- **Theme install.** Drop a `.msstyles`, an orb picture, a folder or a `.zip` on the bar, or use *Install a theme or orb...* in its menu. Themes go to `%LOCALAPPDATA%\Tourne\Styles`, orbs to the orbs folder, and a small toast says what was installed.
- **Hover glow fade.** The glow fades in over about 90 ms and out over about 140 ms, from wherever it had got to, like SAB.

![The three screens' bars](images/23-multi-monitor.png)

Main bar on top; then the bar of the second screen (a Discord window and one more) and the third (nothing open there yet). Each has its own clock.

**How it's done.** The drawing code works on globals, so each bar keeps its own copy of the few that differ (buttons, hover state, tooltips, tray, layout) and they're swapped in when a message arrives for that bar and out again after, which keeps menus and previews on one bar from confusing another. The test bar turned three screens on and off live, and the hot reload and zip install were run in it too.

## b16: the other screens, finished

The gaps left after b15 are closed, each one tried in the test bar on the three screens:

- **Each screen has its own scale.** Sizes, fonts and the clock are worked out per bar from its screen's DPI, so a 200% screen next to a 100% one gets its own sizes (the scale is a whole number, so 100% and 125% are the same size, as on the main bar). A bar's popups (previews, menus) follow its scale too. Tested by setting one bar to 2x: its buttons came out twice the size and the main bar didn't change; a reload put it back.
- **Menus, dragging, hiding, dialog and drop** all work on the other bars: the button's jump list and the bar's own menu open on that screen and close cleanly; dragging a button saves its order; auto-hide slides a bar to its 2 px edge and back when the pointer reaches it; the install dialog opens and closes; a `.msstyles` dropped on it is installed.
- **Your settings are the defaults.** The build script now takes its defaults from the build that is switched on right now (the highest numbered enabled one) unless told otherwise, so each new build starts from what you use, button icons included.

## Research: can YASB themes be ported?

A YASB theme is a `styles.css` plus a `config.yaml` (which widgets go left, center and right, and their label formats). To see what a port would need, all 68 themes in [amnweb/yasb-themes](https://github.com/amnweb/yasb-themes) were read.

- **Variables are the norm:** 53 of 68 use `:root` custom properties and `var()`; 78% of themes.
- **Popups are most of the CSS.** 39% of rules are about the bar; the rest style popups (calendar, media, control centre) and Qt sub-controls (`::item`, `::groove`), which a bar doesn't need at first.
- **Rare:** gradients (2 themes), `box-shadow` (2), `transition` (11: `background-color` and `opacity`, 0.08 to 0.25 s).
- **They are status bars.** Clock is in all 68, volume in 65, media 58, power menu 56, workspaces 51, weather 48; only 20 show taskbar buttons.
- **Icons are font glyphs** (Nerd Font, Segoe Fluent Icons), so the look depends on the font installed.

Full numbers in `research/YASB-SURVEY.md` in the working folder.

## The CSS spike

A small CSS engine, a config reader and a software renderer, in the same style as the rest of the bar (no libraries, GDI only, x64 and x86):

- `css-engine.cpp`: parser, selectors (classes, descendants, `>`, `:hover` and friends), the cascade with specificity, `var()`, shorthands, gradients, transitions read.
- `css-draw.cpp`: rounded boxes with borders and gradients, text with icon-font fallback, icons.
- `yaml-lite.cpp`: `config.yaml` (block and flow lists and maps, `\uXXXX` escapes).
- `tools/css-spike`: builds YASB's element tree (`.yasb-bar`, `.container-left`, `.widget`, `.clock-widget`, `.widget-container`, `.icon`, `.label`), lays it out like a Qt row and draws it.

Across the 68 themes: all configs and layouts read, 6,551 of 6,908 rules kept (the rest are pseudo-element ones), 99% of 19,445 declarations recognised. Six themes drawn from their own files:

![Six YASB themes drawn by our engine](images/24-yasb-spike-themes.png)

Detail of one (Windows 11 theme by amnweb): the left of the bar, the right, and a hover on a workspace button:

![Detail](images/25-yasb-spike-detail.png)

Layout, colours, radii, borders and spacing follow the themes' own screenshots closely. What differs: fonts (icon fonts are different widths from the theme's own), the acrylic blur and rounded window, and real data for the widgets (the spike uses stand-in values). It runs beside the suite for now; it isn't in a build yet.

## b18: YASB themes in the bar (first stage)

The engine is in the bar. Pick a theme under **YASB theme** in the Taskbar settings and the bar is drawn from it.

![Two YASB themes drawn in the suite's own bar](images/26-yasb-in-the-bar.png)

- **Where themes live.** A folder under `%LOCALAPPDATA%\Tourne\Yasb\<name>` with `styles.css` and `config.yaml`. Drop a theme's folder or `.zip` on the bar (or its two files), or use *Install a theme or orb...*; the list fills itself. Saving a file reloads the bar.
- **What the theme decides.** The bar's size, position and padding (it floats off the screen's edge as it does in YASB, with the window as the bar's body, so blur and rounded corners match), the acrylic blur, the widgets in the left, center and right lists, and all their styling, `.dark` and `:hover` included.
- **What stays ours.** The Start button, task buttons (with jump lists, previews, drag), the tray and the clock keep working as they do; they are drawn in the boxes the theme gives them (`.app-container`, `.tray-icon`, `.clock-widget`). Also built: volume (the real level), active window, apps lists, power menu, custom buttons with `start_menu`.
- **Themes without buttons or a tray** (most of them: only 20 of 68 have task buttons) get ours: *Add buttons and tray to YASB themes without them*, on by default, adds task buttons and the Start button at the left and the tray at the right.
- **Commands.** A theme's buttons can name commands (`exec ...`). Those run only if *Let YASB themes run commands* is on (off by default: a theme is someone else's file). Start menu, power menu, notification centre and Windows settings pages always work.
- **Speed.** Building and styling the bar takes 0.2 to 0.5 ms even for the biggest themes (rules are looked up by class).
- **More screens.** Every bar on every screen uses the theme, each at its own scale.

Not there yet: transitions (`:hover` fades), gradients on the bar's window, box shadows, the data widgets (cpu, memory, weather, media, workspaces), popups (calendar, control centre), and `--yasb-*` colours from the system accent. Fonts matter: themes lean on Nerd Fonts, so install the one a theme names.

## b19: get YASB themes from the gallery

**Get a YASB theme...** in the bar's right-click menu opens a small window for the community themes of [yasb.dev/themes](https://yasb.dev/themes). They all live in [amnweb/yasb-themes](https://github.com/amnweb/yasb-themes) on GitHub (one folder per theme with `theme.json`, `styles.css`, `config.yaml` and a screenshot), so the window reads that repository directly.

![The gallery window](images/27-yasb-gallery.png)

- **Nothing is fetched until you ask.** The window opens empty; *Get the list* reads the folder listing and every theme's `theme.json` (name, author, description), 68 themes in a few seconds, six at a time. *Install* (or a double-click) downloads that one theme's `styles.css`, `config.yaml` and screenshot, and installs it like a dropped theme folder, under the theme's name. The bar picks it up at once; choose it under **YASB theme** in the Taskbar settings.
- **Paste a link instead.** The box takes a theme's gallery link, a GitHub folder or file link, a raw link, a `yasb-themes://` link or the bare id; the id in it is what counts.
- **Screenshot** opens the theme's picture in the browser before installing.
- **Only that repository.** File addresses named inside a `theme.json` are used only if they point into `amnweb/yasb-themes`.
- **Safe at unload.** A download in progress holds the module loaded until it ends. Tried in the test bar against the real repository: the list (68), an install from the list and one from a pasted GitHub link.

Because a theme is someone else's file, its commands still stay off unless *Let YASB themes run commands* is on.

## Small things

- Discord's taskbar button now uses the same pixel chat icon as its tray icon (`Discord.ico` in `%LOCALAPPDATA%\Tourne\Icons`, one more entry in *Button icons* for `Discord.exe`). Claude, GitHub Desktop, Settings and Snipping Tool already had theirs.
- Every build compiles for x64 and x86 (`__stdcall` callbacks fixed for x86).

## Not done yet

- **The other screens' bars were tried in the test bar** (three screens), not yet for long in Windhawk itself. A real 150% or 200% screen hasn't been seen, only a bar forced to 2x.
- The separator SAB draws before the last window's button: the rule isn't known.
- Progress bars on the buttons (parts 9 to 12).
- SVG and `.orb` orbs.
- Explorer restyling when SAB is off.

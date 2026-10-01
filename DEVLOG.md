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

## b21: smoother dynamic shyness

- **Slides and fades instead of jumping.** The bar used to snap between shown and away. Now Windows' compositor moves and fades the finished bar (no redraw per frame), with frames timed to the screen's refresh: 6.9 ms apart on a 144 Hz screen in the test bar. When it turns back halfway, it continues from where it is.
- **New settings:** Hiding animation (Slide, Fade, Slide and fade, None); Smoothing (Smooth, Soft landing, Gentle, Off); Animation length; Hide delay; Return delay; While hidden (a sliver of the bar like StartAllBack, a soft glow in the accent colour, or nothing), with glow strength and depth.
- **Less flicker.** Windows have to stay clear for the return delay before the bar comes back, and nothing changes while a window is being dragged or resized. Moving the pointer to the screen's edge still brings it back at once.
- **Hidden means out of the way.** While away, the bar ignores clicks, and the blur is turned off under a fading bar or the glow.
- Blurring the bar as it moves (motion blur) was left out. It needs the whole bar redrawn every frame, which costs CPU; moving and fading cost nothing.
- The animation ignores Windows' "Animation effects" setting, since many people turn that off for Windows' own animations. Use None to turn it off.

## b22: progress on buttons, a crash guard, tidier settings

- **Progress bars on task buttons.** Downloads, copies and installers now show their progress on their button, drawn with the style's own progress parts (normal, busy, error, paused). A probe of Windows' own code showed where apps send it: to Explorer's task switcher window, as a state and a value from 0 to 65535. The suite's Explorer side catches it there and passes it to our bar. Busy progress sweeps across the button; that animation runs only while something is busy. Without a style, the colour scheme's accent is used (warning colour for errors).
- **Not yet matched to StartAllBack.** SAB's styles hold thin 42x5 progress strips, and niivu's full Everforest style holds full-button 27x26 ones, so where SAB puts the strip on a button still has to be photographed. The tool for that is ready; it needs an unlocked screen.
- **Crash guard.** If Explorer ends three times within two minutes of starting, the suite stops loading its parts into Explorer and says so on the taskbar. Saving any suite setting lets them back in. Folder windows running in their own Explorer process don't count. It costs one registry read and write when Explorer starts.
- **Safe Mode.** The suite doesn't load at all in Safe Mode.
- **Settings of a part that's off are hidden** (Windhawk 2.0). Each part's options show only while its Enabled switch is on. Windhawk 1.7.3 still installs the build and shows everything.
- The Explorer side now also runs for the taskbar alone (for progress), without System Icons' Explorer work when System Icons is off.
- The test bar can now save the frame it drew without reading the screen, so tests work while the PC is locked.

## Research: what a visual style fills in, and what it leaves out

Every class, part, state and property of 14 styles was dumped with msstyleEditor's own library: Windows' aero and aerolite, Windows 7, Plain8, niivu's Bouquet family (Bouquet, Night, MAC Dark, MAC Night, Bouquet SAB), Everforest Night and Everforest SAB, and the one in use here. Each was compared with aero value by value, with images compared by their decoded pixels. The full map is `research/msstyles/MSSTYLES-MAP.md` in the working folder.

- **Missing usually means inherited.** uxtheme fills a missing value from the part, then the class, then the style's globals, and a missing `App::Class` falls back to the plain class. This was checked against uxtheme itself, not assumed. niivu leaves scroll arrows, combo box dropdown backgrounds and the navigation sizer blank on purpose.
- **What's really left behind** is Windows' accent blue in a few places even finished styles don't repaint: the calendar's cells, dark Explorer's property text and fills, the command bar's split buttons, and dark list expanders. Task Manager's colours are data colours and stay.
- **A style's system colours hold its whole palette.** Bouquet SAB is Plain8 repainted (it kept Plain8's preview and tray arrow images). Everforest SAB is a full style plus the SAB classes. Bouquet and Bouquet MAC differ only in the window frame.

## b24: the style's own tray parts, and its palette

(b23, a SecureUxTheme build, was skipped.)

![Tray end with Bouquet SAB: at rest, arrow hovered, clock hovered, show desktop hovered and pressed](images/28-sab-tray-parts.png)

- **The show-hidden-icons arrow** is the style's own (`TrayNotifyHoriz::Button` and its Vert and Open variants), lit when pointed at or pressed. It flips to the Open arrow while the overflow flyout is up.
- **Clock and tray icon hovers** use `TrayNotify::Clock` and `TrayNotify::Toolbar`, as SAB does. The clock's text colour comes from the style too.
- **Show desktop button** at the far end (`ShowDesktop::Button`), sized by the style (10 px in Bouquet SAB, 14 in Windows 7). Clicking it sends Win+D. New setting *Show desktop button*: as Windows is set (the default, Windows' own "show desktop" switch), always or never. **It will appear on this PC**, since that switch is on here, as it does in SAB.
- **Thumbnail previews** are drawn with `TaskbandExtendedUI`: the popup's background, the hovered and flashing window, and the close button. Which part and state is which was worked out from the images; SAB's own previews haven't been compared side by side yet.

  ![Preview popup with Bouquet SAB's parts](images/29-sab-preview.png)

- **Only when the style has them.** A class that only falls back to Windows' plain one (compared by its pixels) is ignored, so styles without SAB's classes keep the old look. Checked: the style in use here, aero and Bouquet MAC Night draw as before; Bouquet SAB, Everforest SAB and Windows 7 get every part.
- **Palette from the style's system colours.** The flat colour scheme taken from a style now starts from its own menu, text, border and grey text colours before the menu's pictures are measured. For the style in use here and every Night style, the palette is unchanged. The light-menu styles (Bouquet Dark and Medium, aero) get a readable faint-text colour, the style's own grey instead of a near-white mix.
- **Windows' blue left in the style** (Theme Gaps, off by default): swaps the accent blues listed in the research for the style's own highlight colour (a darker mix of it for fills). Off, it costs nothing: the hook isn't even put in.
- Nothing forces title bar colours or changes Windows' system colours.
- The test bar's saved frames were upside down (GetObject reports a top-down picture's height as positive), and its preview test never opened a preview. Both fixed.

## b25: Explorer in your colours, and the Windows logo from a folder

### Explorer Colours (new part, off until switched on)

The idea comes from VitalS's [Explorer Visual Tweaks Dark](https://windhawk.net/mods/explorer-visual-tweaks-dark): catch the few theme parts Explorer draws its drive bars, selections and preview pane with, and draw them in your colours. That mod replaces the drive bar with its own rounded gradient, which throws away a style's shape (like the pixel segments here). This part keeps it.

![Drive bars and selections from the style in use here: its own, recoloured, recoloured with a fade and a recoloured empty part, flat; then selections and the focus pill](images/31-explorer-colours.png)

- **Drive bars, three ways.** The style's own; *the style's shape in your colours*; or flat bars (rounding, border, fade). A recoloured bar is the style's own picture drawn aside and remapped by brightness, so its most common colour becomes exactly yours and every segment, gap and shade stays. Separate colours for the filled part, a nearly full drive and the empty part, each with an optional second colour to fade to, and each left as the style's own when empty.
- **Selections** in the file list and the navigation pane (either or both): pointed at, selected, and (new) selected in a window that isn't active, each with its own fill and border; border width 0 to 3 and corner rounding 0 to 6; the navigation pane's focus pill with its own colour and (new) width.
- **Preview pane** background, and plain-text previews, in your colour, with (new) a text colour of your own or one picked to suit.
- **Also in these programs:** their Open and Save dialogs get the same selections and bars.
- **Any colour can be `accent`**, the visual style's highlight.
- **Cost.** Nothing is hooked unless something is on, and each hook checks a part and state number first, then one flag, before it looks at the class. A recoloured bar is two small draws and a pass over its pixels (a drive bar is a few thousand), only when a drive bar is painted.
- Colours start as the ones set in Explorer Visual Tweaks Dark here. Switch the part and the blocks on to use them.
- Checked outside Explorer against the style in use here (`tools/compare/colortest.cpp`). Not yet seen inside Explorer itself.

### Windows branding pictures (Icon Redirect)

![The logo from BrandingLoadImage, redirected: the three WINDux sizes, then an .svg drawn at 573x90](images/30-branding.png)

- **Folders of pictures.** A redirect's replacement can now be a folder of pictures named by resource id (`123.png`, `1123.bmp`, `2123.svg`). They stand in for that file's IMAGE, PNG and BITMAP resources. PNGs are used as they are; anything else is converted once, the first time it's asked for, and an .svg is drawn by Windows' own SVG renderer at the size of the picture it replaces. Ids the folder doesn't have stay Windows' own.
- **Windows branding pictures**, one setting: a folder for `Branding\Basebrd\basebrd.dll`, whose 123, 1123 and 2123 are the logo in About Windows (winver), the shut down dialog and System at their three sizes, the ids niivu's packs use. No system file is changed.
- **How it was checked.** `tools/basebrd/brandtest.cpp` compiles Icon Redirect into a test program, points winbrand.dll's own resource calls at its hooks, and asks `BrandingLoadImage` for the logo, which is what winver does. PNG, BMP and SVG all came back at the right sizes.
- **Cost.** Nothing new is hooked (Icon Redirect already reads resources); WIC and Direct2D are loaded only if a .bmp or .svg has to be converted, and never while a DLL is loading.


## b26: a quicker start, and a bar that stays out of the way

Measured in the test bar (warm disk, 3 screens) with timings built into the taskbar; `%LOCALAPPDATA%\Tourne\startup.txt` now logs the same steps on every real start.

### Starting up

- **The rest of the suite no longer waits for the taskbar.** It waited for the whole first setup (650 to 1400 ms); now it waits for the bar's window only (about 12 ms), and the bar sets itself up once its hooks are in.
- **Icons are read once at startup**, not twice (the second read came from a message that arrived after setup).
- **The programs list is read once at startup**, not twice (about 190 ms of background work saved).
- **Not in the lock screen.** The suite no longer loads into LockApp and LogonUI at all.

### The Start menu

| | b25 | b26 |
|---|---|---|
| First open after starting | 190 to 216 ms | about 30 ms |
| Every later open | 21 to 26 ms | about 9 ms |
| First open after a screen change | as slow as the first | about 9 ms |

- **Warmed while closed.** The home list, places, their icons and the right-click menus are made on low-priority threads a few seconds after starting (and after any settings change), so the first open is as quick as the rest.
- **A programs list that didn't change no longer throws the home list away.**
- **Open on press** (on): opens the moment the Start button goes down, as a menu does, instead of when it's let go.

### Screens, sleep and wake

- **Screen changes are gathered up** (a second's wait for waking screens to settle) and redo only the bars' placement, keeping the fonts, style and Start caches: 26 ms instead of a full reload.
- **Theme and colour notices that change nothing are ignored.** Windows sends several on wake; each used to reload everything.
- **Choices are written once**, not 60 times per reload.

### Tooltips and the aura

- **Tooltips:** by the pointer (Windows' place) or beside the bar, centred on the button; moved away from the bar or along it by any number of pixels; their own delay; and closed the moment a preview opens so they never sit over it.
- **The dynamic aura follows at the screen's refresh rate** instead of 15 ms steps, with a *trail* (0 sticks to the pointer; more is a smooth catch-up, as a half-life in ms), plus *size* (40 to 300 %) and *strength* (0 to 300 %) for the Aura style.
- **Cost.** A redraw with the aura following is 0.5 ms, the same as without; the pacer stops when the aura has arrived.

### Window previews, your way

- **How they open:** on resting on a button (as before), on resting on groups only, or only on a click. Ctrl+click now opens any running button's previews, not only a group's. While they're up, moving to another button moves them there.
- **Where:** on the button (as before) or on the pointer, then moved away from the bar or along it by any number of pixels.
- **How long they stay:** until the pointer leaves (after a delay you set, 300 ms as before), or **until dismissed**, in the ways you pick: a click elsewhere, another window coming to the front (not one of theirs), Esc (taken, as a menu takes it), any key (passed on), or a time away. Each can be on or off.
- **After picking a window:** close them (as before), or keep them up to flip between windows.
- **Cost.** The click and key watchers are only in while previews kept until dismissed are up; otherwise nothing is added.
- Checked in the test bar: the offsets and pointer placement land where expected, and the watchers go in and out with the previews.

### Frames and stacks, as StartAllBack draws them

![Ours (top) and StartAllBack (bottom) with Bouquet SAB, 8x: square frames with their dark outer rows, and a stack's edges as straight lines](images/32-frames-and-stacks.png)

- **The dark outer layer is back.** The colour scheme swaps its tint in for whatever is still the bar's background, and it found that by colour alone. Bouquet's frame border is exactly the bar's colour there, so the frame's two dark outer rows were taken for background and tinted away, leaving the light line at the edge. Drawn pixels are now told apart by alpha as well.
- **Square frames.** Frames are 4 px in from both sides of the bar (30 px on a 38 px bar), as StartAllBack's are, not 28. With the dark rows back, a frame now matches StartAllBack's pixel for pixel.
- **Stacks.** A group's edge is the style's own picture at its own width, right against the frame, as tall as the frame, and one straight line: StartAllBack leaves out the picture's little top and bottom caps, which showed here as hooks. The next button starts right after it (5 px for two windows, 8 for more, read from the picture). Before, it was squeezed to 6 px, so a stack of three or more ran its two edges together.
- **Cost.** Nothing measurable: a redraw is still about 0.5 ms.
### Shadows

- **Under icons, under button frames, or both** (off by default), in any colour, with strength, softness (0 to 8 px) and offset. Each shape's shadow is blurred once and kept, found again by its pixels, so a changed icon gets its own.
- A shadow falling on the bar's background falls on your colour scheme's tint (it's kept aside and laid over the tint at the end), not on the style's own background.
- About 0.06 ms per redraw.

### Hiding and fading

- **Fade when idle** (off by default): after a set time away, the bar fades to a set opacity. Choose whether that happens always, only while nothing touches the bar, or only while a window does. The opacity rises as the pointer comes within a set distance, and anything that keeps the bar busy wakes it at once: a menu, its previews, a press, the Start menu, or (as set) a window wanting attention. Fade-out and wake times are separate. Only the compositor's opacity changes; the bar isn't drawn again for it.
- **Automatic hiding:**
  - A *reveal zone*: the bar comes back before the pointer reaches the very edge.
  - It can come back for a window wanting attention.
  - It can come back while the Start menu is open.

### Tray

- **Overflow arrow:** System Icons' green pixel arrow again, or the visual style's (b24 always took the style's).
- **Tray icons, one click** (right-click the clock; they're first there, or right-click the bar):
  - A running icon goes into the tray or back to the overflow with one click.
  - *Not running now* lists every icon Windows has seen (the list its own settings page uses). Ticking one puts it in the tray the next time its program starts, without going through Windows' settings.
  - The old per-icon choices are under *Where each goes*.

### Previews your way, jump lists out of the way

![Previews in Bouquet SAB's own look (top) and in colours set by hand, with a 2 px border, bigger thumbnails and more padding (bottom)](images/33-preview-looks.png)

- **Preview look:** the visual style's (as before), or your own colours: background, titles, the pointed-at window and its border, the window in front, a window wanting attention, and the close button pointed at and its cross. Empty colours are the scheme's.
- **Either look:**
  - thumbnail width and height
  - padding in each window
  - padding to the border (also between windows)
  - border colour (or none), and in your own look its width
  - corners: square, slightly rounded or rounded
  - a drop shadow
  - a backdrop (blur, acrylic or Mica) behind a background of any opacity. A backdrop takes the place of the style's background picture.
- **Jump lists and window menus:** at the pointer (as before), or beside the bar, lined up with the button's start or centre, moved away from the bar or along it by any number of pixels. The button's previews and tooltip close first either way.
- **Show desktop with segments.** The button was only laid out on a bar without segments, so with segments on it was missing. It now sits at the end of the tray's segment, inside its island (with no tray shown, in an island of its own). Bouquet SAB draws it empty until it's pointed at, as StartAllBack does.
- **The notification centre** stays on the taskbar's edge when it shrinks (after *Clear all*). A resizing flyout used to guess which edge to keep from whether its top was in the top half of the screen, and the notification centre is tall enough that it always is. It now remembers the edge it was placed against.
### Explorer: navigation pane lines and gaps (Explorer Style)

- **Hide the navigation pane's separator lines** and **Close the navigation pane's gaps**, both off by default. The idea is Languster's Explorer TreeLine Killer.
  - Lines are found by their pixels: after the pane paints, any flat row across a group's row that isn't the background is painted over. So this doesn't depend on where Windows puts the line.
  - Gaps close by making the groups' double-height rows single height; they're given back when switched off.
  - Nothing is subclassed while both are off.
- Not yet seen with Home, Gallery or pinned folders showing (the pane here shows only This PC).

### Windows 11 26H2, and running without StartAllBack

- 26H2 is an enablement package on the same files (this PC's 26200.9550 already has them). Nothing in the suite checks the build number, and its hooks match files that don't change. The only risk is StartAllBack needing an update for build 26300.
- Research for replacing Windows' taskbar and Start without StartAllBack is in `research/disable-w11-shell.md`. It weighs m417z's Taskbar auto-hide fine tuning and Exiled Eye's Block Start Menu and Hosts. The plan: cloak the taskbar from inside Explorer, and take every Start opening through `XamlLauncher::ShowStartView`, which also retires the keyboard hook.
### Settings folded at first

Windhawk 2.0 alpha 6 has no way for a mod to ask for folded groups. It remembers what you fold per mod id, and each build here has its own id, so every new build opens unfolded.
### Does switching a part off unload it?

- A part that's off has no hooks: they're removed when the suite reloads. Where no part is on at all, the suite asks Windhawk to unload it from that program.
- Where it is loaded, it costs about 75 KB of private memory per program, plus about 350 KB shared by all of them (the 3.3 MB DLL is mapped once).
- Programs that can't be unloaded into (sandboxed browsers, suspended apps) keep an old build until they close.

## b27: the Wi-Fi list comes back by itself

- **Nearby networks were missing from the network flyout.** System Icons opened its connection to Windows' Wi-Fi service once, at start. After the service restarts (a Windows update, like the 26H2 one here, or waking from sleep), that connection goes dead, and every scan and list after it failed quietly. The flyout then showed only the current connection, which comes from somewhere else. It now opens the connection again whenever a call fails, and retries.
- **Checked** with a new test program (`tools/sysicons/nettest.cpp`) that runs System Icons' own network code outside Windhawk, started as a plain desktop app: it lists the nearby networks (6 to 8 here).
- **When Windows really does withhold the list:** since Windows 11 24H2, Windows may keep nearby networks from apps without location access. In that case the flyout now says so, with a link to Location settings, instead of *Looking for networks...*. It shows only when Windows actually refuses, or gives back nothing but the connected network while location is off. Here Windows gave the list without location access, so this is a fallback.

## b28: the location question, explained and optional

- **New setting in System Icons: Nearby Wi-Fi networks.** Its description explains why Windows may ask about location access (or list Windhawk under *Recently used* in its location settings) when the network flyout lists nearby networks. Windows counts that list as location data. The flyout itself only gets network names and signal strength.
- **List them** is the default and works as before.
- **List them, without the location note** keeps the list but never shows the "location access" message, for people who keep location off on purpose.
- **Off** never asks Windows for nearby networks, so there's no scan and no location question. The Wi-Fi section then shows your current connection and a *Show available networks* link to Windows' own list.

## b29: Windows' own taskbar and Start, switched off from inside Explorer; All Programs opens at once

### Windows' taskbar and Start

The suite's Explorer relay now turns Windows' own taskbar and Start away itself, the way the best Windhawk mods do it (m417z's Taskbar auto-hide fine tuning, whose symbol names this follows), but only for the suite's own bar. Every symbol was checked against this PC's Windows files (26100.9278, the 26H2 binaries) before building.

- **Taskbar.** With "Hide StartAllBack's taskbar" on, Explorer's taskbars are **cloaked**: DWM draws nothing and they take no input, so there's no flash for a notification and no sliver at the screen's edge. Explorer uncloaks its taskbar by itself as it redraws, so that one call is turned away (`DwmSetWindowAttribute`, only for the taskbar windows). Its unhide (`TrayUI::Unhide`, `CSecondaryTray::_Unhide`) is dropped too, so it doesn't even animate. A new screen's taskbar is cloaked a moment after the screen appears.
- **The old 500 ms check** that re-hid the taskbar now looks in every 5 seconds while the taskbar is cloaked, just to catch an Explorer restart.
- **Safety.** If the suite's bar ends for any reason, a crash included, Explorer's relay notices at once (it waits on the bar's process) and puts Windows' taskbar back, so there's always a taskbar.
- **Start.** Without StartAllBack, Explorer hands every opening of Windows' Start (`XamlLauncher::ShowStartView`) to the suite's Start: the Windows key, Ctrl+Esc, or anything else. Then the low-level keyboard hook switches itself off, so no key press on the system passes through the bar any more. With StartAllBack loaded nothing changes: it keeps the key, and the keyboard hook stays.
- **Cost:** nothing between events. The hooks only run when Explorer tries to show its taskbar or Start.
- **Not tried live yet:** it needs StartAllBack switched off and Explorer restarted.

### All Programs

- **Measured** with a new test-bar check (`TB_perf=all`): pressing All programs cost about 340 ms, and 700 ms the first time. Two causes:
  - The program list was thrown away on every Start open and read again, asking the shell for each entry's display name (about 1.3 ms each, 81 entries here).
  - The icons were made on the menu's own thread as it painted.
- **Now** the idle warm-up that already fetched the home list's icons also fetches All Programs' top-level names and icons, and the apps' icons, on its own thread while the menu is closed. The names are kept from one open to the next.
- **Result:** pressing All programs takes **1.6 ms** after the warm-up, and 0.8 ms on later opens. Start's own open is unchanged (29.8 ms first, 7.2 ms after).

## b30: no more "Initializing..." when Windhawk opens

- **What you saw:** every time Windhawk was opened, a window listed the suite as *Initializing...* in a windhawk.exe process.
- **Why:** each windhawk.exe loads the suite's launcher, both Windhawk's tray process and the one started when you open Windhawk. Each launcher started the suite's tool process (the taskbar and System Icons). With one already running, the second sat in its start-up for up to 5 seconds, waiting for the first to go, and then gave up. Windhawk showed that wait as a mod still initializing.
- **Now** the launcher only starts a tool process when none is running. It waits for one on a thread of its own, since after a settings change the old one is on its way out and gone within moments. One still there after a few seconds is running for good, so it's left alone. The tool process itself now gives up after 1 second instead of 5, for the rare race.

## b31: Wi-Fi connects again; tray menus close on a click away and can open clear of the taskbar

- **CONNECT and DISCONNECT did nothing in the network flyout.** The button sits inside its network's row, and a click went to whatever was listed first under the pointer, which was the row. So each click just folded the row up again. A click now goes to the smallest target under the pointer, so the button wins over its row, whichever order a flyout lists them in. Bluetooth's CONNECT had the same fault and is fixed with it.
- **A tray icon's right-click menu wouldn't close on a click elsewhere.** Windows closes a menu on an outside click only when the menu's window is in front, which a click on the Tourne taskbar (which never takes the focus) doesn't always leave it. While a menu is up, a mouse hook now closes it on any click outside it, and swallows that click as Windows' own menus do, so a click on the same icon doesn't reopen it. The hook exists only while a menu is open.
- **New System Icons settings for where things open:**
  - *Where the icons' right-click menus open*: at the pointer (as before), or clear of the taskbar, next to the icon, on whichever edge the taskbar is.
  - *Menu lined up with the icon*: starting at it, centred on it, or ending at it.
  - Distance from the taskbar and shift along it, in pixels, for both the menus and the flyouts. The flyouts' distance is added to their usual 8 px.
  - All default to how things were.

## b32: flyouts and menus stay on their own screen

- **Flyouts** are kept inside the work area of the screen their icon is on, both when they open and when they grow while open (the notification centre, a network list). They're moved back in, and are never taller or wider than that screen.
- **Every menu in the suite** (the taskbar's, Start's, System Icons') goes through one place that keeps it on one screen. That's the screen of whatever opened it, or else the screen under its point. A point on or past the screen's edge is brought back inside.
- **A menu doesn't cover what opened it.** Windows is told what to keep clear of, and when there's no room on one side it shows the menu on the other:
  - A menu opened from inside a flyout (an icon in the hidden icons flyout) opens beside the flyout, on the side toward the middle of the screen, and on the other side if there's no room.
  - Tray menus set to open clear of the taskbar, and jump lists placed by their button, stay off the taskbar.
- **Other programs' own menus** (an app's tray icon in the hidden icons flyout) are drawn by those programs, so they open where the program puts them.

## b33: the hidden icons flyout's own menu

A right click on the open part of the hidden icons flyout (off its icons) now opens its own menu, beside the flyout:
- **Show on the taskbar**: pick an icon and it moves into the taskbar's tray.
- **Take out of the taskbar and here**: pick an icon and it shows in neither the taskbar nor this flyout.
- **Show them all on the taskbar**: every icon in the flyout moves at once.
- **Bring back here**: icons taken out, listed with *not running* for programs that are closed. Pick one and it's back in the flyout.

Icons are listed by their tooltip and program. The choices are kept where the taskbar reads them too, so it updates at once. The flyout stays open under the menu and takes the focus back afterwards, so a click away still closes it. A right click on an icon still opens that program's own menu.

## b34: new calendar events, through Google Calendar

**Right-click a day to add an event (Google Calendar, online)** is a new System Icons setting, off by default. A right click on a day in the notification centre's calendar opens a small window:
- Title, the day, all day or a time from and to, where, importance (Low, Normal, High) and notes.
- Times can be typed as `9`, `9:30`, `0930`, `9pm`, `9:30 am` or `21:15`. An end before the start runs past midnight.
- Tab, Enter and Escape work as in a dialog.

![The new event window](images/34-new-event.png)

**Add to Google Calendar** opens Google Calendar's own new-event page in the browser with everything filled in, and one click there saves it. The event then reaches every device, and this agenda the next time it reads the calendar.
- The times are sent as UTC, so the event lands at the right time whatever time zone Google's calendar is set to.
- Google has no importance field: High puts "!" before the title, and Low or High is written in the notes. Tasks can't be made this way, because Google Tasks has no such page.
- **Why it's online, as the setting says:** Google Calendar itself saves the event, in a browser signed in to your account. The suite only opens that page. It sends nothing itself and keeps no account, key or password.

Checked outside Windhawk (`tools/sysicons/evtest.cpp`): the time parsing, 14:30 local on 3 October 2026 becoming `20261003T183000Z`, the link's encoding (`Dentist & café` becomes `Dentist%20%26%20caf%C3%A9`), and the window as drawn.

## b35: testing the preview pane's sizer

With StartAllBack off, Explorer's side pane could be dragged wider but never back narrower than the width it opened at. b35 added test-only layouts behind a hidden switch, to rule the layout in or out, and `tools/navpane/Test-PreviewPane.ps1` to drag the pane in a folder window of its own (it posts mouse messages, so the pointer isn't touched, and Explorer's saved pane sizes are put back after).
- Every layout tried behaved the same, ours and StartAllBack's own: StartAllBack's DLL still loads into Explorer while it's switched off, and its details-on-the-bottom layout wins over ours.
- With StartAllBack running in the test window, the pane shrank normally. So the limit isn't the layout but Windows' sizer code in shell32, which StartAllBack patches.

Also from b35: the user's b33 settings were saved as `settings/tourne-suite-b33-settings.reg` for future builds.

## b36: choosing the Google account for new events

**Google accounts for new events** (System Icons) takes your calendar's email, or several separated by commas. The event window gets an Account row, which cycles through them on a click, and Google's page opens as that account (`authuser=`). This matters when the browser's default Google account isn't the calendar's. That account has to be signed in to the browser.

b36 also tried two fixes for the pane, still as tests. Neither worked live, which b37 explains.

## b37: side panes shrink again, and resized windows stay where you put them

**The pane fix.** A small test DLL, loaded only into a test folder window, logged every call to shell32's `CDUISizerElement::_ComputeBoundedSize` (its disassembly came from Windhawk's own LLVM DLL, as this PC has no disassembler). It showed:
- b36's hook was installed and running, but only looked for the old preview pane (ReadingPane). The pane being dragged is Windows 11's combined details and preview pane, DetailsContainer.
- That sizer's MinSize is set to the width the pane opened at (277 px here), and the clamp held every smaller size at 277.
- Letting that floor go in the test DLL let the pane shrink the full 100 px, twice.

Explorer Style now does the same for both panes, only when the minimum is what holds the pane, and never below 100 px. The test layouts are gone. `tools/navpane/Probe-Sizer.ps1` keeps the probe for next time.

**The window manager.** A tiled window you resize by hand now leaves the tiling instead of snapping back, and stays floating where you put it.
- Its place is remembered for that program (file name and window class) and saved, so the program's next window opens there, also after a restart. One window at a time holds a program's place, so a second window is tiled as usual.
- Moving a kept window updates its place. Shift+W+Space puts it back in the tiling and forgets the place.
- It works with automatic tiling off too, so Shift+W+Tab doesn't pull it back either. Dragging a tiled window without resizing it still swaps it with the one it's dropped on.
- New setting under Window tiling: **Leave a window you resize where you put it**, on.

## b38: window layouts, every window's place remembered, and the pane goes narrower

**The pane.** b37's fix works, but it stopped at 100 px. The probe showed one long drag reaching that floor straight away, after which the pane wouldn't go narrower and seemed broken. The floor is now 24 px. Repeated drags in a window of its own process, in one the shell opens, with a picture previewing, and across three windows in a row (each opening at the width the last left) all shrank.

**Remember where every window closed** (Window tiling, on). Every window with a title bar or a resizing border is followed while it's open: programs' windows, dialogs and tool windows. Its place is saved when it closes, or hides, as programs that close to the tray do.
- The next window of its kind opens there, and maximised again if it closed maximised. A kind is the program and window class, and for dialogs the title too, as one program has many.
- The tiling still places the windows it tiles, and a place kept by resizing a window by hand comes first.
- Places are written two seconds after a window closes, once for a burst, and only the 256 most recent are kept.

**Window layouts** (Shift+W+M). A layout is each screen's tiling layout, plus each window's place and whether it's tiled, saved under a name.
- **Save current as new** takes the windows open now. **Update from windows** retakes the chosen one.
- **Apply** puts the open windows back: each line takes the first window of its program not taken yet, and the tiled ones go back in their order.
- A layout is plain text, one line per screen or window, and can be edited, renamed and saved in the window. **Delete** asks first.
- Shift+W and 1 to 9 apply the first nine layouts without opening the window.

![The Window layouts window](images/38-window-layouts.png)

Checked outside Windhawk (`tools/tiling/lytest.cpp`, with two see-through test windows in a separate process):
- A layout taken, the windows moved away and the layout applied: both went back.
- A window closed at one place and a new one of its kind opened elsewhere: it was moved to where the last one closed.
- Two layouts were saved and one renamed.

The first run used resizable test windows, so the live b37 tiler tiled them. The test windows are now fixed size, which it leaves alone.

## b39: one icon for a program, everywhere

The ask: Process Lasso, every part of NVIDIA, and Discord (launched through the Vencord shortcut, from a folder that changes with every update) each with one icon in every place Windows shows one. The NVIDIA Settings tray icon kept going back to NVIDIA's own.

**Why they slipped.**
- The Icon Redirect rows for Discord had a `*` in front of the replacement file, which turned it into a path that led nowhere. The same was true of the two InputSwitch rows.
- Discord's rows named `app-1.0.9260`, and NVIDIA's named one driver store folder. Discord's updates and NVIDIA's driver updates move both.
- The tray swaps other programs' icons as they arrive. Anything it doesn't recognise keeps the program's own icon.

**Use everywhere.** The taskbar's Button icons rows get a "Use everywhere" switch, and their program can hold `*` and `?`, like `*\NVIDIA Corporation\*.exe`. One marked row now drives everything:
- **The taskbar buttons**, as before.
- **The tray:** the row's icon wins over the tray's own designs.
- **Alt+Tab.**
- **Icon Redirect,** which puts the .ico wherever Windows reads the icon from the program's file: Explorer, Start, its windows. A plain file name matches in any folder, so updates can't lose it.

Rows that already existed stay off, so nothing else changes. b39's defaults add rows for:
- **Discord:** `Discord.exe`, `*\Discord\*.exe`, `*\Discord\app.ico` and its app id.
- **NVIDIA:** `*\NVIDIA Corporation\*.exe`, `*\Display.NvContainer\*.exe`, `nvcplui.exe`, and the NVIDIA App and Control Panel app ids.
- **Process Lasso:** `*\Process Lasso\*.exe`.

**The icons.** They're drawn by `Make-PixelIcons.ps1`, exactly as the tray draws its art: the ink green, coral accents, one-pixel shade, and whole-number scales from 16 to 256 px. Redrawing the Discord icon with it matched the original pixel for pixel. NVIDIA is an eye with a coral pupil, and Process Lasso is a lasso with a coral knot.

![NVIDIA and Process Lasso](images/39-program-icons.png)

**Known limits:**
- The Start menu's tile for the NVIDIA Control Panel is a packaged app's own picture, which nothing here replaces. Its taskbar button and Alt+Tab do change.
- Explorer may keep old icons in its cache until it restarts.

## b40: a desktops pager, and tray icons that stay themed

The ask: a way to see and reach the desktops Win+Tab makes, from the taskbar. It should move along the bar, come off it to sit anywhere on the screen, lock in place, and change its look from a right-click menu. Also, NVIDIA Settings was still showing its own tray icon.

**The pager.** It shows a mark for each desktop, and the one you're on is lit in the accent colour.
- **Click** a mark to go to that desktop. **Scroll** over it to step through them. It switches with Ctrl+Win+Left and Right, the same way Windows does.
- **Docking:** it can sit before the tray (the default), after Start, or after the clock. It takes its own room on the bar, so the buttons move over. With segments, it shares an island with the tray or with Start.
- **Floating:** drag it off the bar and it floats anywhere on the screen, on every desktop. While it's over the bar it turns see-through, and letting go there docks it at the nearest of its three places.
- **Right-click** it for:
  - Lock in place.
  - Its look: dots, numbers, or the names Task view gives them.
  - Where it goes, and whether a floating pager stays on top.
  - Hide it when there's only one desktop.
  - Task view, a new desktop, or closing the one you're on.

The desktops come from Explorer's own registry, which the bar already watched for YASB themes' workspace buttons, so changes show at once without polling. Your pager choices are kept beside the tray's, in `HKCU\Software\TourneTray`. It's in the taskbar settings as "Desktops pager", and it's on.

**NVIDIA Settings' tray icon.** The tray swaps a program's icon as it arrives. NVIDIA's arrived before anything could swap it, and nothing ever looked at it again. Now every icon the tray holds remembers whether it's themed. The relay's existing 10 s pass themes any that aren't. An icon with nothing to theme it is tried a few times, then left alone until its program sends a new one.

**Known limits:**
- Clicking a mark several desktops away steps through each one in between, as the keys do.
- On a vertical bar, the names look shows numbers.

## b41: scans from the security flyout, and a pager that really floats

The ask: run a scan from our Windows Security flyout. Also, let the desktops pager truly float, so it doesn't attach itself when it's over the taskbar, and give it options for size, shape, colours, fade and opacity.

**Scanning.** The flyout has **Quick scan** and **Full scan** under the last scan's time.
- It uses Defender's own command line (`MpCmdRun`), with no window. Defender's service does the scanning; we only wait for it to finish.
- The wait runs on its own thread, so you can close the flyout. Reopen it and you'll see how long the scan has been running.
- The tray icon gets a dot while a scan runs, and its tooltip says which kind of scan.
- When it's done, the flyout says "no threats" or "threats found", and the last-scan time updates.
- If Windows asks for an administrator, the flyout offers **Scan as admin**, which runs the same scan through the UAC prompt.
- Unloading the suite only stops the waiting. The scan itself carries on in Defender.

**Floating that stays floating.**
- A floating pager stays where you let go of it, over the taskbar too. It's never pushed off the bar's part of the screen any more.
- It docks only when you pick a place on the bar from its menu, or when you carry it off the bar and drop it back on.
- With "Keep on top" on, the bar owns it, so it always sits above the bar.

**Its look:** a new "Desktops pager look" group in the taskbar settings.
- **Marks:** size, spacing, and shape: pixel dot, square, circle, diamond, pill (the desktop showing is a wide pill) or bar.
- **Outline:** an outer colour ring around each mark, with the inner colour inside.
- **Colours:** from the colour scheme, or custom ones for the marks, the desktop showing, and the floating panel and its edge.
- **Floating only:** the panel's shape (rounded, square, pill or none), the panel's opacity, the whole pager's opacity, a fade when the pointer leaves it (how faint, and after how long), and running top to bottom.
- Shapes are smoothed. Text on a see-through panel keeps its own edges, so names stay crisp with no panel behind them.
- The right-click menu adds "Size, shape and colours...", which opens Windhawk.

The defaults draw it exactly as b40 did.

## b42: Explorer stops crashing, tooltips hold still, classic menus everywhere

The ask: find out why Explorer kept crashing and why the tooltips flickered (from a screen recording), and fix both. Also, a better way to turn off Windows' "immersive" menus.

**The crashes.** Explorer had crashed 11 times since 2026-09-29, and never before that. Every crash was in Windows 11's own taskbar, which is built with XAML.
- Turning StartAllBack off brings that taskbar back to life inside Explorer, and we were keeping it hidden in two ways it can't cope with:
  - We cloaked it, then answered "done" when Explorer tried to uncloak it.
  - Our taskbar process kept hiding its window from outside.
- The crash dumps fit: the failure always came inside the taskbar's own code, on the thread that runs it.

**How it's hidden now.** It's the approach of m417z's "Taskbar auto-hide fine tuning" mod, which hides that same taskbar without trouble:
- Windows' own auto-hide is on, so Explorer itself has the taskbar hidden away.
- Everything that would bring it back is turned away before it starts:
  - notifications and the Windows key asking it to unhide,
  - the timer that slides it in when the pointer rests at the edge,
  - Windows 11's "should it be expanded" check, and the pointer or a swipe at the edge.
- Only once Explorer has hidden it is it cloaked, so not even its sliver shows. That's done on the taskbar's own thread, and Explorer's own cloaking and uncloaking go through untouched.
- Its window is never hidden from outside any more. A taskbar an older build hid gets shown again.

**Tooltips.** Every second, a tray icon's tooltip changing (the CPU temperature, say) made our bar refresh whichever tooltip was showing. Each refresh put it on top of the bar for one frame before it jumped back.
- Now only the tooltip you're looking at refreshes, and only when its own text changes.
- The tooltip can no longer land anywhere but its place above the bar.

**Classic Menus (new part).** Windows draws some of its own menus as "immersive" ones. Each part of the shell carries its own copy of the code that does that.
- I looked through every Windows file for that code. It's in 12 64-bit and 6 32-bit files on this Windows.
- The usual mod's list predates Windows 11 24H2, so it misses:
  - the taskbar (moved into Taskbar.dll),
  - the keyboard language menu,
  - Windows Security's renamed tray module,
  - File Explorer's new parts,
  - Task Manager,
  - Store apps' title bars.
- Ours covers them all, and hooks each file as it loads instead of loading it into every program.
- On 32-bit programs it uses the same calling convention Windows does, which the usual mod gets wrong.
- Switches cover Explorer and file dialogs, the taskbar, tray icons, and Windows' programs. The "Taskbar settings" gear is left off.

## b43: desktop previews

**Previews of your virtual desktops.** Point at a pager mark, on the bar or floating, and after a moment a preview of that desktop opens beside it.
- It shows the desktop's own wallpaper (or the main one), its windows in their real places, and its name.
- The windows are live, as Task view shows them, straight from Windows' compositor. Windows on other desktops keep their pictures there even while hidden.
- Sliding along the marks swaps the preview at once. Clicking, dragging, the menu or leaving the pager closes it.
- Nothing runs while no preview is open.
- Two new settings under "Desktops pager look": the previews themselves (on) and their width (280 px at 100%). With previews on, the mark's tooltip steps aside, since the preview names the desktop.

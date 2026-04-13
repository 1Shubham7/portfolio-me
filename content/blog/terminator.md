---
title: "Terminator Terminal: Every Shortcut and Feature You Need to Know"
description: "A complete cheat sheet for the Terminator terminal emulator covering all keyboard shortcuts, broadcast mode, layouts, plugins and launch options."
dateString: April 2026
draft: false
tags: ["Linux", "Terminal", "Terminator", "Productivity", "Developer Tools"]
weight: 101
cover:
    image: "blog/terminator.avif"
---

If you spend a lot of time in the terminal, you have probably felt the pain of juggling multiple terminal windows or constantly alt-tabbing between sessions. Terminator fixes that. It lets you split a single window into as many panes as you want, group them together, and even type in all of them at once. Once you get used to it, going back to a regular terminal feels genuinely uncomfortable.

This post is a complete reference for everything Terminator can do - shortcuts, launch flags, plugins, and a few tips I find genuinely useful day to day.

## Installation

```bash
sudo apt install terminator        # Debian / Ubuntu / Mint
sudo yum install terminator        # RHEL / CentOS / Fedora
sudo pacman -S terminator          # Arch Linux
sudo zypper install terminator     # OpenSUSE
sudo emerge -a x11-terms/terminator # Gentoo
```

## Splitting and Layout

This is the core of what makes Terminator different. You can split your window both ways and keep splitting from there.

| Shortcut | Action |
|---|---|
| `Ctrl + Shift + E` | Split terminal vertically (side by side) |
| `Ctrl + Shift + O` | Split terminal horizontally (top and bottom) |
| `Ctrl + Shift + W` | Close the current pane |
| `Ctrl + Shift + Q` | Quit Terminator entirely |
| `Alt + L` | Open the layout launcher |

Between any two panes you will see a thin grab handle. You can click and drag it to resize panes however you like, the same way you would in a tiling window manager.

## Navigating Between Panes

Once you have a few panes open, jumping between them with the keyboard is much faster than reaching for the mouse.

| Shortcut | Action |
|---|---|
| `Alt + ↑` | Move focus to the terminal above |
| `Alt + ↓` | Move focus to the terminal below |
| `Alt + ←` | Move focus to the terminal on the left |
| `Alt + →` | Move focus to the terminal on the right |
| `Ctrl + Shift + N` | Focus the next terminal |
| `Ctrl + Shift + P` | Focus the previous terminal |

## Tabs

Terminator supports tabs on top of splits, so you can have multiple independent tab groups each with their own split layouts.

| Shortcut | Action |
|---|---|
| `Ctrl + Shift + T` | Open a new tab |
| `Ctrl + PageDown` | Switch to next tab |
| `Ctrl + PageUp` | Switch to previous tab |
| `Ctrl + Alt + A` | Rename current tab |

## Focus, Zoom and Fullscreen

| Shortcut | Action |
|---|---|
| `F11` | Toggle fullscreen |
| `Ctrl + Shift + X` | Maximize current terminal |
| `Ctrl + Shift + Z` | Zoom into current terminal |

These two are worth explaining properly because they look similar but behave differently.

**Ctrl+Shift+X (Maximize)** hides all your other panes and expands the current one to fill the Terminator window. The font size stays exactly the same. Toggle it again and everything comes back exactly where it was.

**Ctrl+Shift+Z (Zoom)** does the same thing but also scales up the font and content so the text gets bigger. Think of it like a presentation mode - useful when you are screensharing or want to focus on a single terminal without squinting.

So in short: X just clears the clutter, Z clears the clutter and zooms in visually.

## Broadcast Mode (the best feature)

This is the one that makes people go "wait, what?" when they see it for the first time. Broadcast mode lets you type in one terminal and have it appear in all of them simultaneously. If you ever manage multiple servers, this saves an enormous amount of time.

| Shortcut | Action |
|---|---|
| `Super + G` | Group all terminals, input goes to all of them |
| `Super + Shift + G` | Remove grouping from all terminals |
| `Super + T` | Group all terminals in the current tab only |
| `Super + Shift + T` | Remove grouping from the current tab |
| `Alt + A` | Broadcast to all terminals |
| `Alt + G` | Broadcast to grouped terminals only |
| `Alt + O` | Turn broadcasting off |

(Super is the Windows key on most keyboards.)

## Copy, Paste and Search

| Shortcut | Action |
|---|---|
| `Ctrl + Shift + C` | Copy selected text |
| `Ctrl + Shift + V` | Paste from clipboard |
| `Ctrl + Shift + F` | Search within the terminal scrollback |
| `Ctrl + Shift + S` | Toggle the scrollbar |

## Font Size

| Shortcut | Action |
|---|---|
| `Ctrl + +` | Increase font size |
| `Ctrl + -` | Decrease font size |
| `Ctrl + 0` | Reset font size to default |

## Reset and Clear

| Shortcut | Action |
|---|---|
| `Ctrl + Shift + R` | Reset terminal state (fixes a misbehaving terminal) |
| `Ctrl + Shift + G` | Reset terminal state and clear the window |

## Renaming Things

| Shortcut | Action |
|---|---|
| `Ctrl + Alt + W` | Rename the window title |
| `Ctrl + Alt + A` | Rename the tab title |
| `Ctrl + Alt + X` | Rename the terminal pane title |

## Drag and Drop

You can physically move panes around by dragging them. Click and hold on a terminal's titlebar (the colored strip at the top of each pane) and drag it somewhere else in the layout. The drop zone gets highlighted so you know where it will land.

If you prefer using the mouse without the titlebar, hold Ctrl, click and hold the right mouse button, then release Ctrl and drag.

## Launch Flags

These are options you pass when starting Terminator from the command line.

| Command | What it does |
|---|---|
| `terminator -m` | Start maximized |
| `terminator -b` | Start borderless (no window decorations) |
| `terminator -H` | Hide the window on startup |
| `terminator -l <name>` | Launch with a saved layout |
| `terminator -e <command>` | Run a specific command instead of your shell |
| `terminator --new-tab` | Open a new tab in an already running instance |
| `terminator --toggle-visibility` | Toggle visibility of a running instance (Wayland) |
| `terminator -d` | Enable debug output |

## Saving and Loading Layouts

If you always open Terminator with the same split configuration, save it as a layout so you do not have to rebuild it every time.

1. Arrange your panes exactly how you want them
2. Right-click and go to Preferences → Layouts
3. Click Add, give it a name, and save
4. Next time, launch it directly with `terminator -l your-layout-name`

## Profiles and Customization

Everything visual about Terminator is customizable through Profiles. Right-click anywhere and go to Preferences.

Under Profiles you can change colors, fonts, cursor shape, and how many lines of scrollback history to keep. Under Keybindings you can remap literally every shortcut listed in this post to whatever you prefer. All the shortcuts here are just the defaults.

## Plugins

Terminator has a plugin system for extra functionality. You can enable them under Preferences -> Plugins.

Some useful ones are ActivityWatch (notifies you when a terminal goes quiet or active), Logger (saves terminal output to a file), and CustomCommandsMenu (lets you add your own commands to the right-click menu).

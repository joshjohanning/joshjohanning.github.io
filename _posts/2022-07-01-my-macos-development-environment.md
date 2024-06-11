---
title: 'My macOS Development Environment: iTerm2, oh-my-zsh, and VS Code'
author: Josh Johanning
date: 2022-07-01 14:30:00 -0500
description: Detailing out my local macOS development environment with iTerm, oh-my-zsh with the powerlevel10k theme, VS Code, and more.
categories: [macOS, Development Environment]
tags: [VS Code, Codespaces, Development Environment]
media_subpath: /assets/screenshots/2022-07-01-my-macos-development-environment
image:
  path: iterm2.png
  width: 100%
  height: 100%
  alt: iTerm2 with oh-my-zsh and the powerlevel10k theme
---

## Overview

A new team member had just joined my team at GitHub and it was their first time using macOS as the primary work machine. They had asked if I had any tips on setting up your local development environment. Hint: I do! I also came from a Windows background and only first started using macOS for work in late 2019.

I was going to link them to my [Powerlevel10k Zsh Theme in GitHub Codespaces](/posts/github-codespaces-powerlevel10k/), but then I realized: this is for setting up a development environment in _Codespaces_, not so much locally. I wrote up these instructions for my co-worker, but I thought I would re-purpose them into a blog post that I can share with others as well!

## iTerm2, oh-my-zsh, and powerlevel10k theme setup

1. Install iTerm2: `brew install --cask iterm2`
2. Install the [MesloLGS fonts](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) 
3. Download my [iTerm profile](https://github.com/joshjohanning/dotfiles/blob/main/iterm2-profile.json) as a json file and import into iTerm
   - In iTerm, go to: Preferences > Profile, you can use the `+` to import the `iterm2-profile.json`{: .filepath} profile
   - I believe the only other special things that I have in the profile (other than colors) is the ability to use `Option ⌥` + `←` or `→` arrow keys to to go left / right to the end of strings, [`Option ⌥` + `Shift ⇧` + `←` or `→` arrow keys](https://stackoverflow.com/questions/30055402/how-to-select-text-in-iterm-with-shiftarrow) to highlight entire strings, and `Option ⌥` + `Delete` to delete entire strings
4. Install [oh-my-zsh](https://ohmyz.sh/#install) (run the `curl` command)
5. Install plugins like [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md#oh-my-zsh), [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md#in-your-zshrc) (basically you clone the repo and then add the plugin to the list of `plugins` in your `~/.zshrc`{: .filepath} file
6. Install [powerlevel10k zsh theme](https://github.com/romkatv/powerlevel10k#oh-my-zsh) - basically clone the repo and modify the `~/.zshrc`{: .filepath} file to update the `ZSH_THEME`
7. You will be prompted to configure powerlevel10k - but my configuration for `~/.p10k.zsh`{: .filepath} is [here](https://github.com/joshjohanning/dotfiles/blob/main/.p10k.zsh)
8. My `~/.zshrc`{: .filepath} config is [here](https://github.com/joshjohanning/dotfiles/blob/main/.zshrc) 
9. Make iTerm2 the default terminal: Make iTerm default terminal (`Control ^` + `Shift ⇧` + `Command ⌘` + `\`)

That should be all you need to make your terminal look exactly like mine 😀. 

![iTerm terminal](iterm2.png){: .shadow }
_My iTerm terminal_

If you're using the powerlevel10k theme, make sure to set up the [font in VS Code's terminal](#terminal-fonts) as well!

> You can now back up your `~/.zshrc`{: .filepath} file and `~/.p10k.zsh`{: .filepath} files in a dotfiles repository similar to [mine](https://github.com/joshjohanning/dotfiles) by creating symlinks (documentation on how to do this is in my [repo](https://github.com/joshjohanning/dotfiles) also).
{: .prompt-info }

## VS Code

### Terminal Fonts

To allow VS Code's terminal to look similar to the iTerm terminal, there are a few additional things we need. Add/modify these lines to your VS Code `settings.json`{: .filepath} file by opening the command palette (`Cmd ⌘`/`Ctrl` + `Shift ⇧` + `P`) and typing `> Preferences: Open Settings (JSON)`: 

```json
{
    "terminal.integrated.shell.osx": "/bin/zsh",
    "terminal.integrated.defaultProfile.linux": "zsh",
    "terminal.integrated.fontFamily": "MesloLGS NF",
}
```
{: file='settings.json'}

Now the terminal in VS Code looks nice also!

![VS Code terminal](vscode.png){: .shadow }
_The terminal looks good in VS Code too!_

> Pro-tip: Turn on [VS Code settings sync](https://code.visualstudio.com/docs/editor/settings-sync)!
{: .prompt-tip }

### Extensions

I'll just highlight some of my favorite extensions that I use in VS Code:

1. [GitHub Actions](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-github-actions) - native GitHub Actions YAML syntax and Actions workflow visualization in the IDE
2. [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) - because Copilot! 🤖
3. [GitHub Copilot Chat](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat&ssr=false) - also Copilot! 🤖 💬
4. [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) - I love this because I can highlight a piece of text and paste in a link and it will automatically format the markdown for me, similar to [this feature in GitHub](https://github.blog/changelog/2022-02-02-paste-links-directly-in-markdown/)
5. [GitHub Markdown Preview](https://marketplace.visualstudio.com/items?itemName=bierner.github-markdown-preview) - a non-official GitHub extension to make the markdown preview look more like how GitHub renders markdown
6. [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) - to help me from misspelling, and as a bonus if you're using [VS Code settings sync](https://code.visualstudio.com/docs/editor/settings-sync), you can keep a custom dictionary synced across VS Code instances / Codespaces by using the "quick fix" on aka `Cmd ⌘` + `.` on unrecognized words and "add to user settings"
7. [YAML](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) - for YAML syntax highlighting in the editor
8. [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) - for creating charts/architecture diagrams directly in VS Code
9.  [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) - among other things, allows the ability to Git Blame view inline [like you can in GitHub](https://github.blog/2017-01-18-navigate-file-history-faster-with-improved-blame-view/)
10. [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph) - another option for visualizing Git branches in VS Code
11. [Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens) - to tweak how errors and warnings are shown
12. [TODO Highlight](https://marketplace.visualstudio.com/items?itemName=wayou.vscode-todo-highlight) - highlights those `TODO` comments in your code
13. [GitHub Theme](https://marketplace.visualstudio.com/items?itemName=GitHub.github-vscode-theme) - make your VS Code look more like GitHub!

### Key Bindings

Coming from Windows, my brain is wired that `Ctrl` + `Z` is undo and `Ctrl` + `Y` is redo. In macOS, undo is `Cmd ⌘` + `Z` and redo is `⌘` + `Shift ⇧` + `Z`. I added this key binding to allow for both `Cmd ⌘` + `Shift ⇧` + `Z` AND `Cmd ⌘` + `Y` to be used for redo while editing in VS Code.

You can modify VS Code's `keybindings.json`{: .filepath} file by opening the command palette (`Cmd ⌘`/`Ctrl` + `Shift ⇧` + `P`) and typing `> Preferences: Open Keyboard Shortcuts (JSON)`:  

```json
{
    "keybindings": [
        {
            "key": "ctrl+z",
            "command": "undo",
            "when": "editorTextFocus"
        },
        {
            "key": "ctrl+shift+z",
            "command": "redo",
            "when": "editorTextFocus"
        }
    ]
}
```
{: file='keybindings.json'}

### Tabs, Spaces, and Paste Formatting

When editing GitHub Actions workflows, I would get frustrated when it would add 4 spaces for a tab vs. the customary 2 that the UI editor uses by default. Additionally, it would try to format the YAML upon pasting, which would often break the YAML. To fix this, I added the following to my VS Code `settings.json`{: .filepath} file by opening the command palette (`Cmd ⌘`/`Ctrl` + `Shift ⇧` + `P`) and typing `> Preferences: Open Settings (JSON)`:

```json
"[yaml]": {
    "editor.tabSize": 2,
    "editor.autoIndent": "none"
}
```
{: file='settings.json'}

I set this specifically for `YAML`{: .filepath} files, but you can set it for any file type by updating the header (or removing the header so that all file types use the same settings).

### Colorized Bracket Pairs

I used to use [Bracket Pair Colorizer 2](https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer-2), but this is now [built-in to VS Code](https://code.visualstudio.com/blogs/2021/09/29/bracket-pair-colorization) by adding this to your VS Code `settings.json`{: .filepath} file by opening the command palette (`Cmd ⌘`/`Ctrl` + `Shift ⇧` + `P`) and typing `> Preferences: Open Settings (JSON)`:

```json
"editor.guides.bracketPairs": true
```
{: file='settings.json'}

## Brew

My `Brewfile`{: .filepath} of things that I have installed with Brew is [here](https://github.com/joshjohanning/dotfiles/blob/main/Brewfile).

You can install everything in the `Brewfile`{: .filepath} by running:

```bash
brew bundle install --file=./Brewfile
```
{: .nolineno}

I was able to generate the `Brewfile`{: .filepath} by running:

```bash
brew bundle dump --force
```
{: .nolineno}

> `brew bundle dump` also includes installed VS Code extensions!
{: .prompt-tip }

## App Store Apps

These are my must have App Store apps:

1. [Magnet](https://apps.apple.com/us/app/magnet/id441258766?mt=12) ($) - for pinning windows to certain regions of the screen
2. [Copyclip](https://apps.apple.com/us/app/copyclip-clipboard-history/id595191960?mt=12) (free) - for clipboard management
   - I like to go into preferences and remember and display 2,000 clippings and start at system startup!
3. [Get Plain Text](https://apps.apple.com/us/app/get-plain-text/id508368068?mt=12) (free) - paste without formatting
   - I set my keyboard shortcut to `Option ⌥` + `Cmd ⌘` + `v` as well as launching at startup
4. [MeetingBar](https://apps.apple.com/us/app/meetingbar-for-meet-zoom-co/id1532419400?mt=12) (free) - to show upcoming meetings in the menu bar
5. [Gifski](https://apps.apple.com/us/app/gifski/id1351639930?mt=12) (free) - for creating GIFs from videos
6. [Pro Mouse](https://apps.apple.com/us/app/pro-mouse/id1505869474?mt=12) ($) - a better mouse cursor for presentations
7. [Homie](https://apps.apple.com/us/app/homie-menu-bar-app-for-homekit/id1533590432?mt=12) (free) - for controlling HomeKit devices in the menu bar
8. [Bitwarden](https://apps.apple.com/us/app/bitwarden/id1352778147?mt=12) (free) - password management
9. [Netspot: Wifi Analyzer](https://apps.apple.com/us/app/netspot-wifi-analyzer/id514951692?mt=12) (free) - for scanning WiFi networks and seeing signal strength

## System Settings

- Add the sound icon to the menu bar (easily switch sound outputs)
  - System Settings > Control Center > Sound - Always Show in Menu Bar

## Keyboard Shortcuts

### General Keyboard Shortcuts

These are helpful, out of the box keyboard shortcuts that I often use:

| Action                                        | Shortcut                                          |
|-----------------------------------------------|---------------------------------------------------|
| Hide the windows of the front app             | `Cmd ⌘` + `h`                                     |
| Minimize the front window to the dock         | `Cmd ⌘` + `m`                                     |
| Hide all but current app                      | `Option ⌥` + `Cmd ⌘` + `h`                        |
| Use the app in full screen / exit full screen | `Control ^` + `Cmd ⌘` + `f`                       |
| Maximize an app (not full screen) / put back  | `Option ⌥` + click maximize button                |
| Find again (when using Cmd ⌘ f)               | `Cmd ⌘` + `g`                                     |
| Quit app                                      | `Cmd ⌘` + `q`                                     |
| Force quit an app                             | `Option ⌥` + `Cmd ⌘` + `esc`                      |
| Screenshot the entire screen                  | `Cmd ⌘` + `Shift ⇧` + `3`                         |
| Screenshot a selection using a picker         | `Cmd ⌘` + `Shift ⇧` + `4`                         |
| Screenshot or record a selection              | `Cmd ⌘` + `Shift ⇧` + `5`                         |
| Screenshot the touch bar (RIP)                | `Cmd ⌘` + `Shift ⇧` + `6`                         |
| Screenshot entire window                      | `Space bar` when in screenshot mode               |
| Switch between open apps                      | `Cmd ⌘` + `tab`                                   |
| Switch between multiple windows of app        | `Cmd ⌘` + `` ` `` (backtick key)                  |
| Quit app when switching between apps          | When in the `Cmd ⌘` + `tab` interface, press `q`  |
| See all apps                                  | `F3` (may need `fn` + `F3`)                       |
| See desktop                                   | `Cmd ⌘` + `F3` (may need `fn` + `Cmd ⌘` + `F3`)   |
| Switch between desktops/maximized apps        | `Control ^` + `←` or `→` arrow keys               |
| Spotlight Search                              | `Cmd ⌘` + `Space bar`                             |
| Re-order menu bar icons                       | Hold `Cmd ⌘` and drag                             |
| Refresh page (in browser)                     | `Cmd ⌘` + `r`                                     |
| View history (in browser)                     | `Cmd ⌘` + `y`                                     |

### Finder Keyboard Shortcuts

| Action                  | Shortcut                                        |
|-------------------------|-------------------------------------------------|
| Delete a file           | `Cmd ⌘` + `delete`                              |
| Rename a file           | `return`                                        |
| Show/hide hidden files  | `Cmd ⌘` + `Shift ⇧` + `.`                       |
| Cut (move) file         | Copy normally, then `Option ⌥` + `Cmd ⌘` + `v`  |
| Open a file / folder    | `Cmd ⌘` + `o` or `Cmd ⌘` + `↓` arrow key        |
| Enter folder            | `Cmd ⌘` + `↓` arrow key                         |
| Leave folder            | `Cmd ⌘` + `↑` arrow key                         |
| Rename file/folder      | `return`                                        |

### Text Editing Keyboard Shortcuts

| Action                            | Shortcut                                          |
|-----------------------------------|---------------------------------------------------|
| Delete whole line                 | `Cmd ⌘` + `delete`                                |
| Delete just the last word         | `Option ⌥` + `delete`                             |
| Go to the beginning of the line   | `Control ^` + `a`                                 |
| Go to the end of the line         | `Control ^` + `e`                                 |
| Highlight just one word           | `Shift ⇧` + `Option ⌥` + `←` or `→` arrow keys    |
| Highlight the entire line         | `Shift ⇧` + `Cmd ⌘` + `←` or `→` arrow keys       |
| Jump to bottom of document        | `Cmd ⌘` + `↓` arrow key                           |
| Jump to top of document           | `Cmd ⌘` + `↑` arrow key                           |
| Highlight text vertically         | `Shift ⇧` + `Option ⌥` + select with mouse        |

## Other Tips

- Open a new Finder window from the current directory in Terminal: `open .`
- Update the modified time of a file: `touch -mt202303261924 ./file-to-update.xyz`
- Terminate all instances of a process: `killall Finder` (case sensitive)
- Open file in application: drag the file over the app (e.g. VS Code) in the dock to open
- Output the URL and title of each tab in all open Chrome windows: 
    ```bash
    osascript -e{'set o to""','tell app"google chrome"','repeat with t in tabs of windows','set o to o&url of t&" "&title of t&linefeed',end,end}|sed \$d
    ```
    {: .nolineno}

## Troubleshooting

> **Question**: I'm seeing special characters that aren't rendering in my terminal?
> 
> **Answer**: Make sure you install the [MesloLGS fonts](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) and configure it to be used in iTerm and VS Code. Powerlevel10k uses custom glyphs from the font to render the terminal correctly.

## Summary

Before writing this post, I had most of this in OneNote of what to do when I get a new Mac. Most things are automated, but some like the App Store Apps I install, are not. I plan on sharing this with folks who ask how to get started quickly on a new Mac!

Let me know anything I missed or improvements I can make here, or tips for anyone else coming over from the Windows world 🙏

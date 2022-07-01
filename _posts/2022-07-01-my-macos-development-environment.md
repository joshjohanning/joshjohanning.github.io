---
title: 'My macOS Development Environment: iTerm2, oh-my-zsh, and VS Code'
author: Josh Johanning
date: 2022-07-01 14:30:00 -0500
description: Detailing out my local macOS development environment with iTerm, oh-my-zsh with the powerlevel10k theme, VS Code, and more.
categories: []
tags: [VS Code]
img_path: /assets/screenshots/2022-07-01-my-macos-development-environment
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
   - I believe the only other special things that I have in the profile (other than colors) is the ability to use OPTION+arrow keys to to go left / right to the end of strings, [OPTION+SHIFT+arrow keys](https://stackoverflow.com/questions/30055402/how-to-select-text-in-iterm-with-shiftarrow) to highlight entire strings, and OPTION+Backspace to delete an entire strings
4. Install [oh-my-zsh](https://ohmyz.sh/#install) (run the `curl` command)
5. Install plugins like [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md#oh-my-zsh), [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md#in-your-zshrc) (basically you clone the repo and then add the plugin to the list of `plugins` in your `~/.zshrc`{: .filepath} file
6. Install [powerlevel10k zsh theme](https://github.com/romkatv/powerlevel10k#oh-my-zsh) - basically clone the repo and modify the `~/.zshrc`{: .filepath} file to update the `ZSH_THEME`
7. You will be prompted to configure powerlevel10k - but my configuration for `~/.p10k.zsh`{: .filepath} is [here](https://github.com/joshjohanning/dotfiles/blob/main/.p10k.zsh)
8. My `~/.zshrc`{: .filepath} config is [here](https://github.com/joshjohanning/dotfiles/blob/main/.zshrc) 
9. Make iTerm2 the default terminal: Make iTerm default terminal (`^` + `Shift` + `Command` + `\`)

That should be all you need to make your terminal look exactly like mine 😀. 

![iTerm terminal](iterm2.png){: .shadow }
_My iTerm terminal_

If you're using the powerlevel10k theme, make sure to set up the [font in VS Code's terminal](#terminal) as well!

> You can now back up your `~/.zshrc`{: .filepath} file and `~/.p10k.zsh`{: .filepath} files in a dotfiles repository similar to [mine](https://github.com/joshjohanning/dotfiles) by creating symlinks (documentation on how to do this is in my [repo](https://github.com/joshjohanning/dotfiles) also).
{: .prompt-info }

## VS Code

### Terminal 
To allow VS Code's terminal to look similar to the iTerm terminal, there are a few additional things we need. Add/modify these lines to your VS Code `settings.json`{: .filepath} file by opening the command palette (`CMD`/`CTRL` + `Shift` + `P`) and typing `> Preferences: Open Settings (JSON)` : 

1. Adding the font configuration, osx shell, and shell to use when using a Linux environment (ie: in Codespaces): 

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

1. [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) - Because of course
2. [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) - I love this because I can highlight a piece of text and paste in a link and it will automatically format the markdown for me, similar to [this feature in GitHub](https://github.blog/changelog/2022-02-02-paste-links-directly-in-markdown/)
3. [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) - to help me from misspelling, and as a bonus if you're using [VS Code settings sync](https://code.visualstudio.com/docs/editor/settings-sync), you can keep a custom dictionary synced across VS Code instances / Codespaces by using the "quick fix" on aka `⌘` + `.` on unrecognized words and "add to user settings"
4. [YAML](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) - for YAML syntax highlighting in the editor
5. Bracket Pair Colorizer - I used to use [Bracket Pair Colorizer 2](https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer-2), but this is now [built-in to VS Code](https://code.visualstudio.com/blogs/2021/09/29/bracket-pair-colorization) by adding this to your settings: `"editor.guides.bracketPairs": true`
6. [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) - for creating charts/architecture diagrams directly in VS Code
7. [GitLense](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) - it's nice to be able to do things such as opening up a Git Blame view inline [like you can in GitHub](https://github.blog/2017-01-18-navigate-file-history-faster-with-improved-blame-view/)

### Key Bindings

Coming from Windows, my brain is wired that `CTRL` + `Z` is undo and `CTRL` + `Y` is redo. In macOS, undo is `⌘` + `Z` and redo is `⌘` + `SHIFT` + `Z`. I added this key binding to allow for both `⌘` + `SHIFT` + `Z` AND `⌘` + `Y` to be used for redo while editing in VS Code.

You can modify VS Code's `keybindings.json`{: .filepath} file by opening the command palette (`CMD`/`CTRL` + `Shift` + `P`) and typing `> Preferences: Open Keyboard Shortcuts (JSON)`:  

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

## Brew

My `Brewfile`{: .filepath} of things that I have installed with Brew is [here](https://github.com/joshjohanning/dotfiles/blob/main/Brewfile).

You can install everything in the `Brewfile`{: .filepath} by running:

```bash
brew bundle install --file=./Brewfile
```
{: .nolineno}

I was able to generate the `Brewfile`{: .filepath} by running:

```bash
brew bundle dump
```
{: .nolineno}

## App Store Apps

These are my must have App Store apps:

1. [Magnet](https://apps.apple.com/us/app/magnet/id441258766?mt=12) ($$$) - for pinning windows to certain regions of the screen
2. [Copyclip](https://apps.apple.com/us/app/copyclip-clipboard-history/id595191960?mt=12) (free) - for clipboard management
   - I like to go into preferences and remember and display 2,000 clippings and start at system startup!
3. [Get Plain Text](https://apps.apple.com/us/app/get-plain-text/id508368068?mt=12) (free) - paste without formatting
   - I set my keyboard shortcut to `⌥ ⌘ V` as well as launching at startup

## Troubleshooting

> **Question**: I'm seeing special characters that aren't rendering in my terminal?
> 
> **Answer**: Make sure you install the [MesloLGS fonts](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) and configure it to be used in iTerm and VS Code. Powerlevel10k uses custom glyphs from the font to render the terminal correctly.

## Summary

Before writing this post, I had most of this in OneNote of what to do when I get a new Mac. Most things are automated, but some like the App Store Apps I install, are not. I plan on sharing this with folks who ask how to get started quickly on a new Mac!

Let me know anything I missed or improvements I can make here, or tips for anyone else coming over from the Windows world 🙏

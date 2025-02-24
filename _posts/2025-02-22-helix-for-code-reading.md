---
layout: post
title: Helix for Code Reading
category: tools, ide, editor
---

While looking through the Kubernetes codebase to look at [code for kubelet initialization](https://samof76.space/kubelet-initialization.html), I wanted lesser distractions and wanted something lighter, [Helix](https://helix-editor.com/) was the perfect choice. Helix is a lightweight text editor that is designed for code reading and editing. It has a minimalistic interface that is easy to navigate and has a fast startup time. Helix excellent language server support and also helps in debugging(while I have not used it extensively yet).

## Installation

To install Helix, you can use the following command:

```bash
sudo add-apt-repository ppa:maveonair/helix-editor
sudo apt update
sudo apt install helix
```

> NOTE: Helix installation is supported for many Linux Distros, you can check them all out [here](https://docs.helix-editor.com/package-managers.html).

The above command will install Helix into your system path, to be accessible using the `hx` command. Helix is got an interesting command options `--health` that displays language server support.

```bash
hx --health
Config file: default
Language file: default
Log file: /home/samof76/.cache/helix/helix.log
Runtime directories: /home/samof76/.config/helix/runtime;/usr/lib/helix/runtime;/usr/lib/helix/runtime
Runtime directory does not exist: /home/samof76/.config/helix/runtime
Clipboard provider: xclip
System clipboard provider: xclip

Language                   LSP                          DAP                             ...
ada                        ✘ ada_language_server       None                            ...
...
go                         ✓ gopls                     ✓ dlv                          ...
python                     ✓ pylsp                     ✓ ptvsd                        ...
...
rust                       ✓ rust_analyzer             ✓ lldb-dap                     ...
...
zig                        ✓ zls                       ✓ lldb-dap                     ...
...
```

`LSP` - Language Server Protocol | `DAP` - Debug Adapter Protocol

Now `hx` could be used for both code traversal and debugging. There are just my notes on what are the various options I used for code traversal. But first lets look at the modes.

## Modes

Helix essentially has three modes:
1. **Normal Mode**: Normal mode is the default mode when you launch helix. You can return to it from other modes by pressing the `Esc` key.
2. **Insert Mode**: The insert mode is used for editing text. You can enter insert mode by pressing the `i` key in normal mode.
3. **Select Mode**: The select mode is used for selecting text. You can enter select mode by pressing the `v` key in normal mode.

But in this article all that is needed is the *normal* mode, as I hardly did any editing or selecting text. Before I go there there some interactive elements of helix you should be aware of. There are two interactive elements:
1. **Picker**: Is an interactive element that allows that pops up an internal window to type or select an option. Try `space` to open the picker.
2. **Prompt**: Is an interactive element that allows you to enter text. used like vim `Esc` and `:`. This mode has a auto-completion feature.

## Interactions for Traversal

The when you open up Helix, passing is directory as an argument.

```bash
hx ~/github/kubernetes
```

It would present you with a list of files and directories in the specified directory. You can navigate through the list using the arrow keys or type in the name of the file or directory, it does an fzf kind of fuzzy matching. Press `Enter` to open the file. Assuming I type `kubelet.go`, the picker will display all fuzzy matched files, then I would select `cmd/kubelet/kubelet.go`, and `Enter` to open it.

From here you could use arrow keys to navigate through the file. Suppose you wanted to **jump to the definition of the function** `NewKubeletCommand`, you can put your cursor at or on the function call, and do...

```
g + d
```

And suppose you wanted to **go back** to where you came from you could just move backward by pressing

```
ctrl + o
```

and **go forward** by pressing

```
ctrl + i
```

Now suppose you wanted **list all reference** of the function `NewKubeletCommand`, you can put your cursor at or on the function call, and do...

```
g + r
```

Suppose you lost your way come want to **navigate through hops**, you could use the following key combination:

```
space + j
```

This will list all what Helix calls a **jumplist**, as picker. Now suppose you want to go to an implementation of an interface function, you could use the following key combination:

```
g + i
```

Suppose you are aware of symbol, wanted to **search for that symbol** inside the project, you could use the following key combination:

```
space + shift(s)
```

Suppose you wanted **copy something into the system clipboard**, you first enter the _Select Mode_ by pressing `v`, and then use the following key combination:

```
<arrow-keys-to-select> + space + shift(y)
```

I guess with this I could get through most of the stuff for writing a my notes on [kubelet initialization](https://samof76.space/kubelet-initialization.html). And one thing for sure Helix is great IDE, with very sane defaults, mouse support and much of it works out of the box; as a matter of fact I really did not configure anything at all. One last thing befor I wrap this article, one command I use is to wrap text, I rececommand you try this...

```
: + set-option soft...
```

Go figure! I am sure you will be pleseantly surprised.

# ext_editor_snap_config

This repository provides a solution for using Emacs as an external editor with Thunderbird installed as a snap package on Linux. The solution specifically addresses the challenges of running emacsclient within the snap's confined environment.

## Problem Description

When using Thunderbird as a snap package with the External Editor Revived add-on, the default setup fails because:
1. The snap environment cannot access the system's emacsclient directly
2. Libraries required by emacsclient are not available within the snap
3. The Emacs server socket is not accessible from within the snap environment

## Prerequisites

- Thunderbird installed as a snap package in Ubuntu 24.10
- Emacs running in server mode
- [External Editor Revived add-on](https://github.com/Frederick888/external-editor-revived) installed in Thunderbird

## Solution

The solution involves three main components:

### 1. Setup Emacs Server Configuration

Add to your `~/.emacs.d/init.el`:
```lisp
(setq server-socket-dir "/home/user/snap/thunderbird/common/.emacs-server")
(make-directory server-socket-dir t)
(require 'server)
(server-start)
(setq server-kill-new-buffers t)
(add-hook 'server-switch-hook
          (lambda ()
            (when (current-local-map)
              (use-local-map (copy-keymap (current-local-map))))
            (when server-buffer-clients
              (local-set-key (kbd "C-x #") 'server-edit))))
```


### 2. Prepare the Snap Environment

1. Create necessary directories in the snap's common folder:
```bash
mkdir -p ~/snap/thunderbird/common/bin
mkdir -p ~/snap/thunderbird/common/lib64
mkdir -p ~/snap/thunderbird/common/lib/x86_64-linux-gnu
```

2. Copy extended_editor_revived in the snap’s bin folder:

a) Install the Add-on in Thunderbird using the Extension
manager.

b) Download and install the
[Native Messaging Host](https://github.com/Frederick888/external-editor-revived/wiki/Linux#installing-the-native-messaging-host),
(e.g  ubuntu-latest-gnu-native-messaging-host-vXXX.zip)


c) Copy extended_messaging_host to snap: 

```bash



```


3. Copy required emacs files:
```bash
cp /usr/bin/emacsclient.emacs ~/snap/thunderbird/common/bin/emacsclient
cp /lib64/ld-linux-x86-64.so.2 ~/snap/thunderbird/common/lib64/
cp /lib/x86_64-linux-gnu/libc.so.6 ~/snap/thunderbird/common/lib/x86_64-linux-gnu/
```

### 3. Create the Editor Script

Create a script named `em` in `~/snap/thunderbird/common/bin/`:

```bash
#!/bin/bash
exec 1> >(tee -a /home/user/em.log)
exec 2> >(tee -a /home/user/em.log >&2)
echo "=== $(date) ===" >> /home/user/em.log
echo "Starting with args: $@" >> /home/user/em.log

export LD_LIBRARY_PATH="/home/user/snap/thunderbird/common/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH"
export GTK_USE_PORTAL=1

input_file="$1"
if [ -z "$input_file" ]; then
    input_file="/path/to/temp.eml"
fi

/home/user/snap/thunderbird/common/lib64/ld-linux-x86-64.so.2 \
    /home/user/snap/thunderbird/common/bin/emacsclient \
    --socket-name="/home/user/snap/thunderbird/common/.emacs-server/server" \
    "$input_file"
```

Make the script executable:
```bash
chmod +x ~/snap/thunderbird/common/bin/em
```

## Configuration

1. In External Editor Revived's settings, choose "Editor": Custom,
   "Shell": sh, "Command template": 

```
/home/user/snap/thunderbird/common/bin/em "/path/to/temp.eml" 
```

Optionally, activate: "Suppress help headers" and "Meta headers"

2. Restart Thunderbird and Emacs. When composing
   a new message, the Button "External Editor" appears
   in the Thunderbird Message Window (upper right). A click opens the message in an Emacs frame. Finish
   editing, save the draft (C-s C-f in
   emacs) and return to Thunderbird (C-x #). 

## How It Works

The solution works by:
1. Creating a confined environment within the snap's common directory that includes necessary libraries
2. Placing the Emacs server socket in a location accessible to both Emacs and the snap
3. Using a script that properly handles the confined environment and maintains the connection between Thunderbird and Emacs

## Troubleshooting

- Check em.log in your home directory for debugging information
- Verify that Emacs server is running: In Emacs, evaluate `(server-running-p)`
- Ensure all paths in the scripts match your system's username and directory structure

## Credits

This solution was developed with the assistance of Claude (Anthropic) in February 2025. The debugging process and final implementation were the result of an interactive problem-solving session with the AI assistant.

## License

MIT License
MIT Licence

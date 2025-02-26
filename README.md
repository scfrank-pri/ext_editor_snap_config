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


2. Copy required emacs files:
```bash
cp /usr/bin/emacsclient.emacs ~/snap/thunderbird/common/bin/emacsclient
cp /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ~/snap/thunderbird/common/lib64/
cp /lib/x86_64-linux-gnu/libc.so.6 ~/snap/thunderbird/common/lib/x86_64-linux-gnu/
```

### 3. Create the Editor Script

Create a script named `em` in `~/snap/thunderbird/common/bin/` ("user" MUST be replaced by your user id). 

```bash
#!/bin/bash

export LD_LIBRARY_PATH="/home/user/snap/thunderbird/common/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH"

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

### 4. Install the extended_editor_revived in the snap’s bin folder:

##### a) Install the Add-on in Thunderbird using the Extension manager.

##### b) Download and install the [MUSL-Native-Messaging-Host](https://github.com/Frederick888/external-editor-revived/releases)

Example for ubuntu 24.10: (see [wiki](https://github.com/Frederick888/external-editor-revived/wiki/Linux#installing-the-native-messaging-host))

- download ```ubuntu-latest-musl-native-messaging-host-v1.2.0.zip``` to your ```~/Downloads``` folder
- ```unzip ubuntu-latest-musl-native-messaging-host-v1.2.0.zip```
- make it executable: ```chmod a+x external-editor-revived```
- run: ```./external-editor-revived```
- edit the resulting file ```external_editor_revived.json``` and change: ```"path": "/home/user/Download/external-editor-revived",``` into: ```"path": "/home/user/snap/thunderbird/common/bin/external-editor-revived",```

Lastly, copy the files into the snap
```bash
mkdir -p ~/snap/thunderbird/common/.mozilla/native-messaging-hosts/
cp ~/Downloads/external_editor_revived.json
~/snap/thunderbird/common/.mozilla/native-messaging-hosts/
cp ~/Downloads/external-editor-revived ~/snap/thunderbird/common/bin/
```


#### c) Configuration

1. In Thunderbird's Add-on External Editor Revived's settings, choose "Editor": Custom,
   "Shell": sh, "Command template": (change `user` into your user id)

```
/home/user/snap/thunderbird/common/bin/em "/path/to/temp.eml" 
```

Optionally, activate: "Suppress help headers" and "Meta headers"

2. Restart Thunderbird and Emacs. When composing
   a new message, the Button "External Editor" appears
   in the Thunderbird Message Window (upper right). A click opens the message in an Emacs frame. Finish
   editing, save the draft (`C-s C-f` in
   emacs) and return to Thunderbird (`C-x #`). 

## How It Works

The solution works by:
1. Creating a confined environment within the snap's common directory that includes necessary libraries
2. Placing the Emacs server socket in a location accessible to both Emacs and the snap
3. Using a script that properly handles the confined environment and maintains the connection between Thunderbird and Emacs

## Troubleshooting

- Verify that Emacs server is running: In Emacs, evaluate `(server-running-p)`
- Ensure all paths in the scripts match your system's username and directory structure

## Credits

This solution was developed with the assistance of Claude (Anthropic) in February 2025. The debugging process and final implementation were the result of an interactive problem-solving session with the AI assistant.

## License
none

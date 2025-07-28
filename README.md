# UPSilon-2000-installer-patch download
A patch for Ubuntu 24.04 install script for https://www.megatec.com.tw/software-download/
# Fix for [UPSilon 2000 v5.5 software installer](https://www.megatec.com.tw/software-download/) for Ubuntu 24.04 / Debian 12 (for UPSilon 2000 UPS)

# NOTES:

**The patched script simply skips upsilon.eml and upsilon.pgr if they’re missing.** 

>**If you need {**

> **Email updates;** 

>**SMS updates;**

> **}**

> **then {** 

>**put the needed helper scripts into /etc/upsilon/** 

>**}**

# 1. Install the missing library (libtinfo5)

    # grab the last maintained build
    wget http://security.ubuntu.com/ubuntu/pool/universe/n/ncurses/libtinfo5_6.3-2ubuntu0.1_amd64.deb
    
    # install 
    sudo apt install ./libtinfo5_6.3-2ubuntu0.1_amd64.deb

\**32‑bit uses the* `i386` *deb instead.)*

# 2 Replace the install.linux with this patched one

Patched script:

    #!/bin/sh
    # Patched UPSilon 2000 installer – July 2025 by MarsTheProtogen on github
    # - Quotes variables (supports paths with spaces)
    # - Skips optional helper files (upsilon.eml / upsilon.pgr) if absent
    # - Auto‑symlinks libncurses.so.5 & libtinfo.so.5 → *.so.6 when packages are missing
    #   (so the program starts even if only the -6 libs are present)
    
    PROG=rupsd
    INSTALL_DIR="$(pwd)"
    PROGRAM_DIR=/etc/upsilon
    
    echo "Linux 2.x INSTALL FOR UPSilon 2000 (patched)"
    
    [ "$(id -u)" -eq 0 ] || { echo "Run as root."; exit 1; }
    
    echo "UPSilon 2000 will be installed to $PROGRAM_DIR."
    
    # stop any running daemon
    [ -x "$PROGRAM_DIR/upsilon" ] && "$PROGRAM_DIR/upsilon" stop 2>/dev/null
    
    # backup previous install
    [ -d "$PROGRAM_DIR" ] && { rm -rf "$PROGRAM_DIR.old"; mv "$PROGRAM_DIR" "$PROGRAM_DIR.old"; }
    
    mkdir -p "$PROGRAM_DIR"
    
    echo -n "Copying files "
    for f in rupsd upsilon email pager shutdown.ini rups.ini preshut.bat upsilon.eml upsilon.pgr; do
      if [ -s "$INSTALL_DIR/$f" ]; then
         cp "$INSTALL_DIR/$f" "$PROGRAM_DIR" && echo -n "."
      fi
    done
    echo " OK"
    
    chmod 544 "$PROGRAM_DIR/rupsd"
    chmod 555 "$PROGRAM_DIR/upsilon"
    
    # add legacy lib symlinks if packages not installed
    for lib in ncurses tinfo; do
      ldconfig -p | grep -q "lib${lib}.so.5" || {
        [ -e /lib/x86_64-linux-gnu/lib${lib}.so.6 ] && \
        ln -sf /lib/x86_64-linux-gnu/lib${lib}.so.6 /lib/x86_64-linux-gnu/lib${lib}.so.5
      }
    done
    ldconfig
    
    "$PROGRAM_DIR/upsilon" reginit
    "$PROGRAM_DIR/upsilon" start && echo "Installation completed!"

Save it over the existing `install.linux` and:

    # make sure the file is exicuteable 
    chmod +x install.linux

# 2.5 make sure that there are no actively running upsilon processes

>the installer may say `**Please stop the UPSilon 2000 background process**` you will need to list the current upsilon processes **twice** in case the first one you see isn't "actually" doing stuff

    ps aux | grep -i upsilon
    
    # you should see something like:
    $ ps aux | grep -i upsilon
        user      2573  0.0  0.1  15480  5556 ?  Ssl  14:02   0:00 /etc/upsilon/rupsd
        user      2589  0.0  0.0   9212  2168 ?  Ss   14:02   0:00 /etc/upsilon/upsilon 
    
    $ ps aux | grep -i upsilon
        user      2573  0.0  0.1  15480  5556 ?  Ssl  14:02   0:00 /etc/upsilon/rupsd
        user      3690  0.0  0.0   9212  2168 ?  Ss   14:02   0:00 /etc/upsilon/upsilon 

you want to `sudo kill 2573` as it's an process that's doing something

# 3 Run the installer

    sudo ./install.linux

>you may need to try 2.5 again and/ or `sudo /etc/upsilon/upsilon stop`

# 4 Register & configure without the CLI

the CLI doesn't work for me, so I manually changed the .ini file

**THIS MAY NOT WORK**

>there is a warning saying protection will be disabled after 30 days is not registered properly, and as of this post's creation, ***not*** tested by time

    # stop daemon 
    sudo /etc/upsilon/upsilon stop
    
    # edit registration info
    sudo nano /etc/upsilon/rups.ini
    # [REGISTRATION]         
    # CDKEY=AAAAAAA-BBBBBBB  
    # EMAIL=you@example.com  
    # PASSWORD= ****
    
    
    # flush cache & restart
    sudo /etc/upsilon/upsilon reginit
    sudo /etc/upsilon/upsilon start
    sudo /etc/upsilon/upsilon status   # shows voltage, battery, etc.

# extra upsilon commands

|Path (as root)|Purpose / Action|Typical use‑case or note|
|:-|:-|:-|
|**/etc/upsilon/upsilon start**|Start the background daemon (`rupsd`).|Run at boot via `rc.local`; use manually for testing.|
|**/etc/upsilon/upsilon stop**|Gracefully stop the daemon.|Always try this before any `pkill` brute‑force.|
|**/etc/upsilon/upsilon restart**|Convenience wrapper: `stop` → 1 s wait → `start`.|Useful after editing `rups.ini`.|
|**/etc/upsilon/upsilon status**|One‑shot status dump (line‑voltage, battery %).|Quick health check from the shell.|
|**/etc/upsilon/upsilon config**|Launch the text‑mode parameter editor.|Change serial port, shutdown timer, etc.|
|**/etc/upsilon/upsilon reginit**|Flush license cache & reread `rups.ini`.|Run after you edit CD‑Key or e‑mail by hand.|
|**/etc/upsilon/upsilon issuer**|Send direct commands to the UPS (on/off, test).|Advanced / diagnostic only.|
|**/etc/upsilon/upsilon help**|Bare‑bones help screen (same text as README).|Shows key bindings.|
|**/etc/upsilon/upsilon.eml**|*Helper script* for e‑mail alerts (shell script).|Called automatically when you enable e‑mail events.|
|**/etc/upsilon/upsilon.pgr**|*Helper script* for pager/SMS alerts.|Legacy dial‑out; safe to leave empty if unused.|
|**/etc/upsilon/rupsd**|The actual daemon binary UPSilon controls.|Started by `upsilon start`; seldom called directly.|
|**/etc/upsilon/rups.ini**|Main INI file: CD‑Key, serial port, timers, etc.|Edit in a text editor, then run `reginit`.|
|**/etc/upsilon/rupslog**|Rolling event log (plain text).|View with `tail -f` or any log watcher.|

# 

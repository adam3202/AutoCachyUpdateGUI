# AutoCachyUpdateGUI
I vibe coded an auto update script that has a good looking GUI to basically wrap and automate the Cachy-Updater. I decided to do this because I wanted the update process to be as hands free and painless as possible, seeing as how handhelds don't exactly have keyboards, AND I also wanted it to work from WITHIN steam mode. This script can be run on any laptop, desktop, and handheld PC running CachyOS.

# CachyOS GUI Updater

A sleek, automated, dark-mode graphical interface for CachyOS system updates. Built with Python and CustomTkinter, this utility completely automates the terminal-based `cachy-update` process. 

Designed for both traditional desktop environments and handheld gaming PCs (like the Lenovo Legion Go or Steam Deck) running Steam Game Mode. It acts as a smart virtual keyboard, automatically handling package confirmations, service restarts, and AUR reviews in the background while displaying a clean progress bar.

## ✨ Features
* **Zero-Touch Automation:** Uses a Python pseudo-terminal (PTY) to automatically answer `[Y/n]` prompts, bypass Arch News blocks, and hit `0` for service restarts.
* **AUR Pager Bypass:** Automatically injects `PAGER="cat"` to prevent AUR package reviews from freezing the update process.
* **Steam Game Mode Immunity:** Natively strips `LD_PRELOAD` from the environment to prevent Steam's 32-bit overlay injection from causing false-positive errors during execution.
* **Handheld / Gamepad Ready:** Includes built-in hotkeys mapped to standard keyboard inputs (`X` to close, `Y` to toggle terminal) for seamless Steam Input controller integration.
* **Fail-Safe Logging:** Keeps the raw terminal output hidden by default, but automatically drops down the console if any critical `ld.so` or `timeout` errors are detected.

## 📦 Prerequisites
This script requires the `python-customtkinter` UI library. Install it via the AUR:
```bash
paru -S python-customtkinter
```

## 🚀 Installation & Setup

**1. Configure Passwordless Sudo**
To allow the updater to run silently without prompting for your admin password, you must add a rule for the backend package managers. (This allows the script to run as a standard user, preventing Arch Linux from blocking AUR installations). 

Open the sudoers file:
```bash
sudo EDITOR=nano visudo
```
Add the following line at the very bottom (replace `yourusername` with your actual Linux username):
```text
yourusername ALL=(ALL) NOPASSWD: /usr/bin/pacman, /usr/bin/pacman-key, /usr/bin/systemctl, /usr/bin/rm -f /tmp/arch-update.lock
```
Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

**2. Download the Script**
Save the `updater_gui.py` file to your home directory (or any preferred application folder) and grant it executable permissions:
```bash
chmod +x ~/updater_gui.py
```

**3. Launch**
You can now run the app directly from your terminal or create a standard `.desktop` shortcut:
```bash
./updater_gui.py
```

## 🎮 Steam Game Mode Setup (Handhelds)
To use this as a native "1-Click" updater on a Legion Go or Steam Deck UI:

1. Open Steam in Desktop Mode.
2. Click **Games > Add a Non-Steam Game to My Library**.
3. Click **Browse**, change the file filter to **"All Files"**, and select `updater_gui.py`.
4. Right-click the new shortcut in your library, select **Properties**, and rename it to "CachyOS Updater".
5. Return to **Game Mode**.
6. Select the updater, open the **Controller Settings

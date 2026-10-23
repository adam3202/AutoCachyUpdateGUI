# AutoCachyUpdateGUI
I vibe coded an auto update script that has a good looking GUI to basically wrap and automate the Cachy-Updater. I decided to do this because I wanted the update process to be as hands free and painless as possible, seeing as how handhelds don't exactly have keyboards, AND I also wanted it to work from WITHIN steam mode. This script can be run on any laptop, desktop, and handheld PC running CachyOS.

# CachyOS GUI Updater

A sleek, automated, dark-mode graphical interface for CachyOS system updates. Built with Python and CustomTkinter, this utility completely automates the terminal-based `cachy-update` process. 

Designed for both traditional desktop environments and handheld gaming PCs (like the Lenovo Legion Go) running Steam Game Mode. It acts as a smart virtual keyboard, automatically handling package confirmations, service restarts, and AUR reviews in the background while displaying a clean progress bar.

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

> [!WARNING]
> **Security Trade-Off**
> This application requires adding `NOPASSWD` rules for `pacman` and `systemctl` to function without user intervention. While this provides a seamless, console-like experience on personal gaming handhelds, it intentionally bypasses a core Linux security layer. This configuration is **not recommended** for shared workstations or production servers, as it allows background processes on your user account to execute package management commands as root. Use at your own discretion.

**2. Download the Script**
Save the `updater_gui.py` file to your home directory (or any preferred application folder) and grant it executable permissions:
```bash
chmod +x ~/updater_gui.py
```

**3. Launch**
You can now run the app directly from your terminal or create a standard








The Exact script is posted below for review

#!/usr/bin/env python
import customtkinter as ctk
import subprocess
import threading
import pty
import os
import re

# Configure the window theme
ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("blue")

ANSI_ESCAPE = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]|\x1B\].*?(?:\x07|\x1B\\)')
SYSTEMD_TAGS = re.compile(r'\]?\d{4};(?:start|end)=[a-f0-9]+(?:;[a-z]+=[^;\n]+)*')

class CachyUpdaterApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("CachyOS System Updater")
        self.geometry("650x480")
        self.resizable(False, False)

        self.label = ctk.CTkLabel(self, text="System Update Manager", font=ctk.CTkFont(size=22, weight="bold"))
        self.label.pack(pady=(20, 5))

        self.status_label = ctk.CTkLabel(self, text="Preparing to update...", font=ctk.CTkFont(size=14), text_color="gray")
        self.status_label.pack(pady=(0, 10))

        self.progress = ctk.CTkProgressBar(self, width=450, height=15)
        self.progress.pack(pady=(5, 15))
        self.progress.set(0)

        self.btn_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.btn_frame.pack(pady=(5, 10))

        self.action_btn = ctk.CTkButton(self.btn_frame, text="Updating...", state="disabled", height=40, font=ctk.CTkFont(size=14, weight="bold"))
        self.action_btn.grid(row=0, column=0)

        self.console_visible = False
        self.toggle_btn = ctk.CTkButton(self, text="▼ Show Terminal Output", command=self.toggle_terminal, fg_color="transparent", text_color="gray", hover_color="#2b2b2b")
        self.toggle_btn.pack(pady=(10, 5))

        self.console = ctk.CTkTextbox(self, width=600, height=200, state="disabled", font=ctk.CTkFont(family="monospace", size=12), fg_color="#1e1e1e")

        # HOTKEYS (Listens for keyboard presses)
        self.bind("<Key-x>", self.safe_close)
        self.bind("<Key-X>", self.safe_close)
        self.bind("<Escape>", self.safe_close)
        self.bind("<Key-y>", self.safe_toggle)
        self.bind("<Key-Y>", self.safe_toggle)

        # AUTO-START
        self.after(200, self.start_update_thread)

    def toggle_terminal(self):
        if self.console_visible:
            self.console.pack_forget()
            self.toggle_btn.configure(text="▼ Show Terminal Output")
            self.console_visible = False
        else:
            self.console.pack(padx=20, pady=5)
            self.toggle_btn.configure(text="▲ Hide Terminal Output")
            self.console_visible = True

    def safe_close(self, event=None):
        """Allows the hotkey to close the app, but only if the update is completely finished."""
        if self.action_btn.cget("text") == "Close":
            self.destroy()

    def safe_toggle(self, event=None):
        """Allows a hotkey to toggle the terminal drop-down."""
        self.toggle_terminal()

    def write_to_console(self, text):
        clean_text = ANSI_ESCAPE.sub('', text)
        clean_text = SYSTEMD_TAGS.sub('', clean_text)
        clean_text = clean_text.replace('==> Press "enter" to quit', '')
        
        self.console.configure(state="normal")
        self.console.insert("end", clean_text)
        self.console.see("end") 
        self.console.configure(state="disabled")

    def start_update_thread(self):
        self.status_label.configure(text="Updating system... Please wait.", text_color="white")
        self.progress.configure(mode="indeterminate")
        self.progress.start()
        
        self.console.configure(state="normal")
        self.console.delete("0.0", "end")
        self.console.configure(state="disabled")
        self.write_to_console("Initializing cachy-update...\n")
        
        threading.Thread(target=self.run_update_process, daemon=True).start()

    def run_update_process(self):
        clean_env = os.environ.copy()
        clean_env.pop("LD_PRELOAD", None)
        clean_env["PAGER"] = "cat"
        
        subprocess.run("sudo /usr/bin/rm -f /tmp/arch-update.lock", shell=True, env=clean_env)
        master, slave = pty.openpty()
        
        try:
            # LAUNCH WITHOUT SUDO to prevent AUR root-block
            process = subprocess.Popen("cachy-update", shell=True, stdin=slave, stdout=slave, stderr=subprocess.STDOUT, close_fds=True, env=clean_env)
            os.close(slave) 
            
            error_detected = False
            terminal_buffer = ""
            
            while True:
                try:
                    data = os.read(master, 1024).decode('utf-8', errors='replace')
                    if not data:
                        break
                        
                    clean_logic_data = ANSI_ESCAPE.sub('', data)
                    clean_logic_data = SYSTEMD_TAGS.sub('', clean_logic_data)
                    
                    clean_lower = clean_logic_data.lower()
                    clean_stripped = clean_logic_data.strip()
                    
                    terminal_buffer += clean_lower
                    if len(terminal_buffer) > 50000:
                        terminal_buffer = terminal_buffer[-50000:]
                        
                    is_real_error = False
                    if "error has occurred" in clean_lower or "aborted" in clean_lower or "timeout" in clean_lower:
                        is_real_error = True
                    if "error:" in clean_lower and "ld.so" not in clean_lower:
                        is_real_error = True
                        
                    if is_real_error:
                        error_detected = True
                        
                    self.after(0, self.write_to_console, data)
                    
                    if "[y/n]" in clean_lower:
                        os.write(master, b"y\n")
                        
                    elif "default=" in clean_lower and clean_stripped.endswith((":", "?", ">")):
                        os.write(master, b"\n")
                        
                    elif "restart them all" in terminal_buffer and clean_stripped.endswith((":", "?", ">")):
                        os.write(master, b"0\n")
                        terminal_buffer = "" 
                        
                    elif 'press "enter" to proceed' in clean_lower:
                        os.write(master, b"\n")
                        
                    elif 'press "enter" to quit' in clean_lower:
                        os.write(master, b"\n")
                        
                except OSError:
                    break
                    
            process.wait()
            
            if process.returncode == 0 and not error_detected:
                self.after(0, self.finish_update, True)
            else:
                self.after(0, self.finish_update, False)
                
        except Exception as e:
            self.after(0, self.write_to_console, f"\n\n[ERROR] {str(e)}\n")
            self.after(0, self.finish_update, False)
            
        finally:
            subprocess.run("sudo /usr/bin/rm -f /tmp/arch-update.lock", shell=True, env=clean_env)

    def finish_update(self, success):
        self.progress.stop()
        self.progress.configure(mode="determinate")
        self.progress.set(1)
        self.action_btn.configure(text="Close", state="normal", command=self.destroy, fg_color="#4a4a4a", hover_color="#333333")
        
        if success:
            self.status_label.configure(text="System is fully updated! Press X to close.", text_color="#32a852")
            self.write_to_console("\n\n[SUCCESS] System is fully updated!\n")
        else:
            self.status_label.configure(text="Update finished with errors. Check terminal.", text_color="#e63946")
            self.write_to_console("\n\n[WARNING] Process finished, but errors were detected.\n")
            if not self.console_visible:
                self.toggle_terminal()

if __name__ == "__main__":
    app = CachyUpdaterApp()
    app.mainloop()

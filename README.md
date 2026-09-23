# Nokia FastMile 5G ODU Auto-APN Switcher and Changing Network Mode also for OpenWrt

This repository provides a complete, automated solution for Airtel 5G Nokia ODU /  Nokia FastMile 5G ODUs (Models 5G32-A / 5G16-A) to force-open the data path for secondary carrier SIMs (like Jio or BSNL). Tested on Airtel AOD311NK Software Version R1.

On certain firmware versions (like R1), the ODU requires manual APN and routing modifications on every boot to establish an internet connection. This project automates the complex cryptographic login challenge (AES/SHA256 cascading hashes) and database manipulation via OpenWrt.

## 💻 Hardware Compatibility
This script is designed to run on **any OpenWrt router or Linux device** capable of running Python 3. 

**Tested Devices:**
* Dell Wyse 3020 / 3040 (x86_64 OpenWrt) (for 2nd ethernet port i used USB Ethernet Adapter)
* LeMaker Banana Pi M1 (sunxi/cortexa7 OpenWrt) (for 2nd ethernet port i used USB Ethernet Adapter)
* MediaTek MT7986a (Filogic 830) Home Routers

---

## 🔌 Step 1: Hardware Setup (Connecting ODU to IDU)
Before running any scripts, you must physically connect the Nokia Outdoor Unit (ODU) to your OpenWrt router (Indoor Unit / IDU).

1. Connect an Ethernet cable from the **Nokia 5G ODU** to the **POE** port on your PoE Power Injector adapter.
2. Connect a second Ethernet cable from the **LAN** port on the PoE Power Injector to the **WAN port** of your OpenWrt router.
3. Connect a third Ethernet cable from the **LAN port** of your OpenWrt router to your **Computer** (or connect to the OpenWrt Wi-Fi).
4. **IP Addressing Note:** The Nokia ODU usually defaults to `192.168.0.1`. Your OpenWrt router will automatically get an IP from it on its WAN port. Ensure your OpenWrt router's local LAN IP is something different (like `192.168.1.1` or `192.168.250.1`) to avoid IP conflicts.
5. **Configure Management VLAN (VLAN 1999):** Log in to the OpenWrt web interface. Go to **Network > Interfaces** and add a new interface named `wan_mgmt`. Set its device to `eth1.1999` (or `wan.1999` depending on your WAN port) to create VLAN 1999. In the **Firewall Settings** tab, assign it to the `wan` zone. Finally, add manual DNS servers (`8.8.8.8` and `8.8.4.4`). This is necessary to reach the ODU management interface.

---

## 🖥️ Step 2: Connecting to Your Router via SSH (Using PuTTY)
To install the script, you need to access your OpenWrt router's command line. If you are on Windows, the easiest way is using PuTTY.

1. Download and install [PuTTY](https://www.putty.org/).
2. Open PuTTY. In the **Host Name (or IP address)** box, type your OpenWrt router's IP address (e.g., `192.168.1.1`).
3. Ensure the Port is set to **22** and Connection type is **SSH**.
4. Click **Open**. (If a security alert pops up, click "Accept" or "Yes").
5. When prompted with `login as:`, type `root` and press Enter.
6. Type your OpenWrt password (the characters will be invisible as you type) and press Enter.

You are now in the OpenWrt terminal!

---

## ⚙️ Step 3: Prerequisites
Your OpenWrt device must have Python 3 and a few core modules installed. In your SSH terminal, run:

```bash
# For OpenWrt 23.05+ (using apk)
apk update
apk add python3 python3-requests python3-cryptography

# For older OpenWrt versions (using opkg)
opkg update
opkg install python3 python3-requests python3-cryptography
```

---

## 🚀 Installation Method 1: Simple Background Script (No GUI)

If you just want the script to run silently in the background on every boot:

### 1. Create the Script
In your SSH terminal, create the Python file:
```bash
nano /root/switch_apn.py
```

Paste the following code into the file. **Make sure to change `ODU_IP`, `PASSWORD`, `APN1`, and `APN2` to match your setup.**

```python
#!/usr/bin/env python3
import requests
import urllib3
import base64
import hashlib
import os
import json
import time
import subprocess

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ODU_IP = "192.168.0.1"
USERNAME = "admin"
PASSWORD = "ANKODAXXXXXXXXXX"    #Change to Your Password

# Define the dual APN toggle targets
APN1 = "jionet"
APN2 = "bsnlnet"

def nokia_b64url(b64_str):
    return b64_str.replace('+', '-').replace('/', '_').replace('=', '.')

def sha256_b64(str1, str2):
    combined = f"{str1}:{str2}".encode('utf-8')
    digest = hashlib.sha256(combined).digest()
    return base64.b64encode(digest).decode('utf-8')

def sha256url(str1, str2):
    return nokia_b64url(sha256_b64(str1, str2))

def wait_for_odu():
    print("1. Waiting 1 seconds for ODU initial boot sequence...")
    time.sleep(1)
    
    print("2. Checking if ODU is reachable via Ping...")
    while True:
        res = subprocess.run(
            ["ping", "-c", "1", "-W", "2", ODU_IP],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL
        )
        if res.returncode == 0:
            print("=> ODU is online and responding!")
            break
        print("=> ODU not ready yet. Retrying in 5 seconds...")
        time.sleep(5)

def main():
    wait_for_odu()
    
    print("3. Initializing Session...")
    session = requests.Session()
    session.verify = False
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
        "Content-Type": "application/x-www-form-urlencoded",
        "Origin": f"https://{ODU_IP}",
        "Referer": f"https://{ODU_IP}/"
    }
    
    print("4. Fetching Server Nonce...")
    res0 = session.get(f"https://{ODU_IP}/login_web_app.cgi?nonce", headers=headers)
    nonce_data = res0.json()
    
    raw_nonce = nonce_data['nonce']
    random_key = nonce_data['randomKey']
    iterations = nonce_data.get('iterations', 1)
    safe_nonce = nokia_b64url(raw_nonce)
    
    print("5. Fetching Server Challenge (Alati)...")
    userhash = sha256url(USERNAME, raw_nonce)
    
    payload1 = f"userhash={userhash}&nonce={safe_nonce}"
    res1 = session.post(f"https://{ODU_IP}/login_web_app.cgi?salt", data=payload1, headers=headers)
    alati = res1.json().get('alati')
    
    print("6. Calculating Cascading Cryptographic Hashes...")
    k_string = (alati + PASSWORD).encode('utf-8')
    K = hashlib.sha256(k_string).hexdigest()
    
    for _ in range(1, iterations):
        K = hashlib.sha256(bytes.fromhex(K)).hexdigest()
        
    ce = sha256_b64(USERNAME, K.lower())
    response_hash = sha256url(ce, raw_nonce)
    random_key_hash = sha256url(random_key, raw_nonce)
    
    std_enckey = base64.b64encode(os.urandom(16)).decode('utf-8')
    std_enciv = base64.b64encode(os.urandom(16)).decode('utf-8')
    url_enckey = nokia_b64url(std_enckey)
    url_enciv = nokia_b64url(std_enciv)
    
    print("7. Authenticating...")
    payload2 = (
        f"userhash={userhash}&RandomKeyhash={random_key_hash}"
        f"&response={response_hash}&nonce={safe_nonce}"
        f"&enckey={url_enckey}&enciv={url_enciv}"
    )
    res2 = session.post(f"https://{ODU_IP}/login_web_app.cgi", data=payload2, headers=headers)
    
    try:
        login_data = res2.json()
    except Exception:
        print("=> Login Failed! Non-JSON response.")
        print(res2.text)
        return

    sid = login_data.get("sid")
    token = login_data.get("token")
    
    if sid:
        print(f"=> Login Successful! Session ID: {sid}")
        session.cookies.set("sid", sid, path="/", domain=ODU_IP)
    else:
        print("=> Login Failed!")
        print(res2.text)
        return

    print("8. Fetching Live APN Database Context...")
    api_headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
        "Content-Type": "application/json",
        "Origin": f"https://{ODU_IP}",
        "Referer": f"https://{ODU_IP}/web_whw/",
        "Cookie": f"sid={sid}; token={token}"
    }
    
    token_payload = {
        "version": 1,
        "csrf_token": token,
        "id": 1,
        "interface": "Nokia.GenericService",
        "service": "OAM",
        "function": "GetAPN",
        "paralist": []
    }
    res3 = session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json=token_payload, headers=api_headers)
    
    try:
        res3_json = res3.json()
        csrf_token = res3_json.get("csrf_token") or token
        apn_list = res3_json.get("FunctionResult", {}).get("APNList", [])
    except Exception:
        print("=> Failed to parse GetAPN JSON response.")
        print(res3.text)
        return

    if not apn_list:
        print("=> Error: No APN records found in database.")
        return

    # Determine current APN and calculate toggle target
    target_apn_record = apn_list[0]
    current_apn = target_apn_record.get("AccessPointName", "")
    print(f"=> Current Active APN on ODU: {current_apn}")

    if current_apn == APN1:
        target_apn = APN2
    else:
        target_apn = APN1

    print(f"=> Toggling APN configuration to: {target_apn}")
    target_apn_record["AccessPointName"] = target_apn
    target_apn_record["WorkMode"] = "RouteMode"
    target_apn_record["Services"] = "TR069,INTERNET"
    target_apn_record["NatEnable"] = "1"

    print(f"9. Pushing Updated APN Record ({target_apn}) to ODU Database...")
    modify_payload = {
        "version": 1,
        "csrf_token": csrf_token,
        "id": 2,
        "interface": "Nokia.GenericService",
        "service": "OAM",
        "function": "ModifyAPN",
        "paralist": [target_apn_record]
    }

    res4 = session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json=modify_payload, headers=api_headers)
    print("=> APN Push Complete! Router Response:")
    print(json.dumps(res4.json(), indent=2))

    # ---------------------------------------------------------
    # 10. Force 5G SA Network Mode
    # ---------------------------------------------------------
    network_cgi_url = f"https://{ODU_IP}/service_function_web_app.cgi"
    
    sa_payload = {
        "version": 1,
        "csrf_token": csrf_token,
        "id": 3,
        "interface": "Nokia.GenericService",
        "service": "OAM",
        "function": "SetCellularAccessConfig", 
        "paralist": [{"AccessModeNo": 3}]   # 1 .5G SA and 5G NSA Both , 2 .5G NSA Only, 3 .5G SA only, 4 .4G LTE Only -  Change Values Accordingly
    }

    print("10. Pushing 5G SA Network Lock...")
    sa_req = session.post(
        network_cgi_url,
        headers=api_headers,
        json=sa_payload,
        verify=False,
        timeout=10
    )
    
    if sa_req.status_code == 200:
        print("=> 5G SA Mode locked successfully!")
        print(sa_req.text)
    else:
        print(f"=> Failed to lock 5G SA. Status: {sa_req.status_code}")

if __name__ == '__main__':
    main()
```
Save and exit (`Ctrl+X`, `Y`, `Enter`).

### 2. Make it Executable and Enable Auto-Start
Make the script executable, then add it to OpenWrt's local startup file so it runs in the background (`&`) on every reboot:

```bash
chmod +x /root/switch_apn.py
sed -i '/exit 0/d' /etc/rc.local
echo '/usr/bin/python3 /root/switch_apn.py &' >> /etc/rc.local
echo 'exit 0' >> /etc/rc.local
```

for any suggestion / feedback please write me - dexdigi@gmail.com
## ⚠️ Troubleshooting
* **Script crashes on launch (SyntaxError):** Ensure there are no invisible carriage returns (Windows line endings) in the script if you copy-pasted. Run `sed -i 's/\r$//' /root/switch_apn.py` to clean it.
* **Script hangs endlessly:** The script pings the ODU before starting. Verify your OpenWrt router can reach `192.168.0.1` by running `ping 192.168.0.1` in the terminal. If it fails, check your interface firewall zones.

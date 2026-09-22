# Nokia FastMile 5G ODU Auto-APN Switcher for OpenWrt

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
import requests, urllib3, base64, hashlib, os, json, time, subprocess
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# --- CONFIGURATION ---
ODU_IP = "192.168.0.1"
USERNAME = "admin"
PASSWORD = "YOUR_ROUTER_PASSWORD"
APN1 = "jionet"
APN2 = "bsnlnet"
# ---------------------

def nokia_b64url(b64_str):
    return b64_str.replace('+', '-').replace('/', '_').replace('=', '.')

def sha256_b64(str1, str2):
    return base64.b64encode(hashlib.sha256(f"{str1}:{str2}".encode('utf-8')).digest()).decode('utf-8')

def sha256url(str1, str2):
    return nokia_b64url(sha256_b64(str1, str2))

def wait_for_odu():
    print("Waiting 60 seconds for ODU to boot...")
    time.sleep(60)
    while True:
        if subprocess.run(["ping", "-c", "1", "-W", "2", ODU_IP], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL).returncode == 0:
            print("ODU is online!")
            break
        time.sleep(5)

def main():
    wait_for_odu()
    session = requests.Session()
    session.verify = False
    headers = {"User-Agent": "Mozilla/5.0", "Content-Type": "application/x-www-form-urlencoded", "Origin": f"https://{ODU_IP}", "Referer": f"https://{ODU_IP}/"}
    
    try:
        # 1. Fetch Nonce
        res0 = session.get(f"https://{ODU_IP}/login_web_app.cgi?nonce", headers=headers, timeout=5)
        nonce_data = res0.json()
        raw_nonce, random_key, iterations = nonce_data['nonce'], nonce_data['randomKey'], nonce_data.get('iterations', 1)
        safe_nonce = nokia_b64url(raw_nonce)
        userhash = sha256url(USERNAME, raw_nonce)
        
        # 2. Fetch Alati (Salt)
        res1 = session.post(f"https://{ODU_IP}/login_web_app.cgi?salt", data=f"userhash={userhash}&nonce={safe_nonce}", headers=headers, timeout=5)
        alati = res1.json().get('alati')
        
        # 3. Calculate Cascading Hashes
        K = hashlib.sha256((alati + PASSWORD).encode('utf-8')).hexdigest()
        for _ in range(1, iterations):
            K = hashlib.sha256(bytes.fromhex(K)).hexdigest()
        ce = sha256_b64(USERNAME, K.lower())
        response_hash = sha256url(ce, raw_nonce)
        random_key_hash = sha256url(random_key, raw_nonce)
        
        # 4. Authenticate
        enckey = nokia_b64url(base64.b64encode(os.urandom(16)).decode())
        enciv = nokia_b64url(base64.b64encode(os.urandom(16)).decode())
        res2 = session.post(f"https://{ODU_IP}/login_web_app.cgi", data=f"userhash={userhash}&RandomKeyhash={random_key_hash}&response={response_hash}&nonce={safe_nonce}&enckey={enckey}&enciv={enciv}", headers=headers, timeout=5)
        
        sid, token = res2.json().get("sid"), res2.json().get("token")
        if not sid: 
            print("Login failed.")
            return
            
        session.cookies.set("sid", sid, path="/", domain=ODU_IP)
        api_headers = {"User-Agent": "Mozilla/5.0", "Content-Type": "application/json", "Origin": f"https://{ODU_IP}", "Referer": f"https://{ODU_IP}/web_whw/", "Cookie": f"sid={sid}; token={token}"}
        
        # 5. Fetch Live Database APN List
        res3 = session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json={"version": 1, "csrf_token": token, "id": 1, "interface": "Nokia.GenericService", "service": "OAM", "function": "GetAPN", "paralist": []}, headers=api_headers, timeout=5)
        res3_json = res3.json()
        csrf_token = res3_json.get("csrf_token") or token
        apn_list = res3_json.get("FunctionResult", {}).get("APNList", [])
        
        if not apn_list: 
            return
            
        # 6. Toggle APN and Push Update
        target_record = apn_list[0]
        current_apn = target_record.get("AccessPointName", "")
        target_apn = APN2 if current_apn == APN1 else APN1
        
        target_record["AccessPointName"] = target_apn
        target_record["WorkMode"] = "RouteMode"
        target_record["Services"] = "TR069,INTERNET"
        
        res4 = session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json={"version": 1, "csrf_token": csrf_token, "id": 2, "interface": "Nokia.GenericService", "service": "OAM", "function": "ModifyAPN", "paralist": [target_record]}, headers=api_headers, timeout=5)
        print("APN Successfully Switched to:", target_apn)
        
    except Exception as e:
        print(f"Error: {e}")

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

---

## 🌟 Installation Method 2: Native LuCI GUI Application

If you want a native web interface directly inside your OpenWrt Dashboard to manage passwords, IP addresses, and APN names without touching code, run this one-liner in your SSH terminal.

It will automatically generate the OpenWrt UCI configuration, Lua controllers, and a daemonized python script:

```bash
cat << 'EOF' > /tmp/setup_nokia.sh
#!/bin/sh
echo "Creating Nokia ODU Configuration..."
cat << 'CONFIG' > /etc/config/nokia_odu
config settings 'main'
    option enabled '1'
    option ip_address '192.168.0.1'
    option username 'admin'
    option password 'YOUR_PASSWORD'
    option apn1 'jionet'
    option apn2 'bsnlnet'
    option wait_time '60'
CONFIG

echo "Creating LuCI Controller..."
mkdir -p /usr/lib/lua/luci/controller
cat << 'CONTROLLER' > /usr/lib/lua/luci/controller/nokia_odu.lua
module("luci.controller.nokia_odu", package.seeall)
function index()
    if not nixio.fs.access("/etc/config/nokia_odu") then return end
    entry({"admin", "services", "nokia_odu"}, cbi("nokia_odu"), _("Nokia ODU APN"), 60).dependent = true
end
CONTROLLER

echo "Creating LuCI CBI Interface View..."
mkdir -p /usr/lib/lua/luci/model/cbi
cat << 'CBI' > /usr/lib/lua/luci/model/cbi/nokia_odu.lua
m = Map("nokia_odu", translate("Nokia FastMile ODU APN Sync"), translate("Automates authentication and data-path opening for secondary carrier SIMs."))
s = m:section(TypedSection, "settings", translate("General Parameters"))
s.anonymous = true; s.addremove = false
s:option(Flag, "enabled", translate("Enable Boot Sync")).rmempty = false
ip = s:option(Value, "ip_address", translate("ODU IP Address")); ip.datatype = "ip4addr"
s:option(Value, "username", translate("ODU Username"))
pwd = s:option(Value, "password", translate("ODU Password")); pwd.password = true
s:option(Value, "apn1", translate("Primary APN"))
s:option(Value, "apn2", translate("Secondary APN"))
wt = s:option(Value, "wait_time", translate("Boot Wait Time (Seconds)")); wt.datatype = "uinteger"
return m
CBI

echo "Creating Init Boot Script..."
cat << 'INIT' > /etc/init.d/nokia_odu
#!/bin/sh /etc/rc.common
START=99
start() {
    config_load nokia_odu
    config_get_bool enabled main enabled 0
    if [ "$enabled" -eq 1 ]; then /usr/bin/switch_apn.py & fi
}
INIT
chmod +x /etc/init.d/nokia_odu

echo "Writing Python Automation Script..."
cat << 'PYTHON' > /usr/bin/switch_apn.py
#!/usr/bin/env python3
import requests, urllib3, base64, hashlib, os, json, time, subprocess
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_uci(key, default):
    try:
        val = subprocess.run(["uci", "get", f"nokia_odu.main.{key}"], stdout=subprocess.PIPE, text=True, timeout=2).stdout.strip()
        return val if val else default
    except Exception: return default

ODU_IP = get_uci("ip_address", "192.168.0.1")
USERNAME = get_uci("username", "admin")
PASSWORD = get_uci("password", "YOUR_PASSWORD")
APN1 = get_uci("apn1", "jionet")
APN2 = get_uci("apn2", "bsnlnet")
WAIT_TIME = int(get_uci("wait_time", "60"))

def nokia_b64url(s): return s.replace('+', '-').replace('/', '_').replace('=', '.')
def sha256url(s1, s2): return nokia_b64url(base64.b64encode(hashlib.sha256(f"{s1}:{s2}".encode('utf-8')).digest()).decode('utf-8'))

def main():
    if get_uci("enabled", "0") != "1": return
    time.sleep(WAIT_TIME)
    while subprocess.run(["ping", "-c", "1", "-W", "2", ODU_IP], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL).returncode != 0: time.sleep(5)
    
    session = requests.Session(); session.verify = False
    headers = {"User-Agent": "Mozilla/5.0", "Content-Type": "application/x-www-form-urlencoded", "Origin": f"https://{ODU_IP}", "Referer": f"https://{ODU_IP}/"}
    try:
        r0 = session.get(f"https://{ODU_IP}/login_web_app.cgi?nonce", headers=headers, timeout=5).json()
        rn, rk = r0['nonce'], r0['randomKey']
        r1 = session.post(f"https://{ODU_IP}/login_web_app.cgi?salt", data=f"userhash={sha256url(USERNAME, rn)}&nonce={nokia_b64url(rn)}", headers=headers, timeout=5).json()
        
        K = hashlib.sha256((r1['alati'] + PASSWORD).encode()).hexdigest()
        for _ in range(1, r0.get('iterations', 1)): K = hashlib.sha256(bytes.fromhex(K)).hexdigest()
        
        ce = base64.b64encode(hashlib.sha256(f"{USERNAME}:{K.lower()}".encode()).digest()).decode()
        r2 = session.post(f"https://{ODU_IP}/login_web_app.cgi", data=f"userhash={sha256url(USERNAME, rn)}&RandomKeyhash={sha256url(rk, rn)}&response={sha256url(ce, rn)}&nonce={nokia_b64url(rn)}&enckey={nokia_b64url(base64.b64encode(os.urandom(16)).decode())}&enciv={nokia_b64url(base64.b64encode(os.urandom(16)).decode())}", headers=headers, timeout=5).json()
        
        sid, token = r2.get("sid"), r2.get("token")
        if not sid: return
        session.cookies.set("sid", sid, path="/", domain=ODU_IP)
        api = {"User-Agent": "Mozilla/5.0", "Content-Type": "application/json", "Origin": f"https://{ODU_IP}", "Referer": f"https://{ODU_IP}/web_whw/", "Cookie": f"sid={sid}; token={token}"}
        
        r3 = session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json={"version":1, "csrf_token":token, "id":1, "interface":"Nokia.GenericService", "service":"OAM", "function":"GetAPN", "paralist":[]}, headers=api, timeout=5).json()
        apns = r3.get("FunctionResult", {}).get("APNList", [])
        if not apns: return
        
        tr = apns[0]
        tr["AccessPointName"] = APN2 if tr.get("AccessPointName") == APN1 else APN1
        tr["WorkMode"], tr["Services"] = "RouteMode", "TR069,INTERNET"
        session.post(f"https://{ODU_IP}/service_function_web_app.cgi", json={"version":1, "csrf_token":r3.get("csrf_token", token), "id":2, "interface":"Nokia.GenericService", "service":"OAM", "function":"ModifyAPN", "paralist":[tr]}, headers=api, timeout=5)
    except Exception as e: pass

if __name__ == '__main__': main()
PYTHON
chmod +x /usr/bin/switch_apn.py

echo "Cleaning Cache and Restarting Web UI..."
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache/*
/etc/init.d/uhttpd restart
/etc/init.d/nokia_odu enable
EOF
sh /tmp/setup_nokia.sh
```

**To access the GUI:** Refresh your OpenWrt web dashboard. Navigate to **Services > Nokia ODU APN** (or **Network > Nokia ODU APN** depending on your OpenWrt version) to configure your APNs and passwords directly.

## ⚠️ Troubleshooting
* **Script crashes on launch (SyntaxError):** Ensure there are no invisible carriage returns (Windows line endings) in the script if you copy-pasted. Run `sed -i 's/\r$//' /root/switch_apn.py` to clean it.
* **Script hangs endlessly:** The script pings the ODU before starting. Verify your OpenWrt router can reach `192.168.0.1` by running `ping 192.168.0.1` in the terminal. If it fails, check your interface firewall zones.

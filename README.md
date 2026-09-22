# Nokia FastMile 5G ODU Auto-APN Switcher for OpenWrt
This repository provides a complete, automated solution for Nokia FastMile 5G ODUs (Models 5G32-A / 5G16-A) to force-open the data path for secondary carrier SIMs (like Jio or BSNL).

On certain firmware versions (like R1), the ODU requires manual APN and routing modifications on every boot to establish an internet connection. This project automates the complex cryptographic login challenge (AES/SHA256 cascading hashes) and database manipulation via OpenWrt.

# 💻 Hardware Compatibility
This script is designed to run on any OpenWrt router or Linux device capable of running Python 3.

Tested Devices:

Dell Wyse 3020 / 3040 (x86_64 OpenWrt)

LeMaker Banana Pi M1 (sunxi/cortexa7 OpenWrt)

MediaTek MT7986a (Filogic 830) Home Routers

⚙️ Prerequisites
Your OpenWrt device must have Python 3 and a few core modules installed. SSH into your router and run:

# For OpenWrt 23.05+ (using apk)
apk update
apk add python3 python3-requests python3-cryptography

# For older OpenWrt versions (using opkg)
opkg update
opkg install python3 python3-requests python3-cryptography

🚀 Installation Method 1: Simple Background Script (No GUI)
If you just want the script to run silently in the background on every boot:

### 1. Create the Script
SSH into your router and create the Python file:
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

Save and exit (Ctrl+X, Y, Enter).


2. Make it Executable and Enable Auto-Start
Make the script executable, then add it to OpenWrt's local startup file so it runs in the background (&) on every reboot:

```bash
chmod +x /root/switch_apn.py
sed -i '/exit 0/d' /etc/rc.local
echo '/usr/bin/python3 /root/switch_apn.py &' >> /etc/rc.local
echo 'exit 0' >> /etc/rc.local
```

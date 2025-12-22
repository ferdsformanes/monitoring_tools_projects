# Cisco SD-WAN Monitoring with Prometheus & Grafana (Windows)

Monitor Cisco SD-WAN devices on a **Windows computer** using a Python Flask exporter, Prometheus, and Grafana.

---

## Step 1: Run the SD-WAN Exporter (Windows)

Save as `sdwan_exporter_flask.py`:

```python
from flask import Flask, Response
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Cisco SD-WAN Sandbox credentials
HOST = "https://sandbox-sdwan-2.cisco.com"
USERNAME = "devnetuser"
PASSWORD = "RG!_Yw919_83"

app = Flask(__name__)

# Create session and login once
session = requests.Session()
login_url = f"{HOST}/j_security_check"
payload = {"j_username": USERNAME, "j_password": PASSWORD}
resp = session.post(login_url, data=payload, verify=False)

if resp.status_code != 200 or "JSESSIONID" not in session.cookies:
    raise Exception("Login failed!")

print("Logged in successfully")

@app.route("/metrics")
def metrics():
    try:
        # Retrieve device list
        devices_url = f"{HOST}/dataservice/device"
        resp = session.get(devices_url, verify=False)
        if resp.status_code != 200:
            raise Exception(f"Failed to retrieve devices: {resp.status_code}, {resp.text}")
        devices = resp.json().get("data", [])
        
        # Count all devices
        up_count = len(devices)
        
        # Prometheus metric format
        output = f"# HELP sdwan_devices_up Number of SD-WAN devices\n"
        output += f"# TYPE sdwan_devices_up gauge\n"
        output += f"sdwan_devices_up {up_count}\n"
    except Exception as e:
        output = f"# Error: {str(e)}"
    return Response(output, mimetype="text/plain")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

Run the exporter (Command Prompt or PowerShell):
```powershell
python sdwan_exporter_flask.py
```

Test in browser:
```
http://localhost:8000/metrics
```

---

## Step 2: Configure Prometheus (Windows)

Edit `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "sdwan"
    static_configs:
      - targets: ["localhost:8000"]
```

Validate config:
```powershell
promtool.exe check config prometheus.yml
```

Start Prometheus:
```powershell
prometheus.exe --config.file=prometheus.yml
```

Verify:
- Open `http://localhost:9090/targets`
- Both jobs should be **UP**

---

## Step 3: Configure Grafana (Windows)

1. Open `http://localhost:3000`
2. Login: `admin / admin`
3. Go to **Connections → Data Sources**
4. Add **Prometheus**
5. URL: `http://localhost:9090`
6. Click **Save & Test**

---

## Step 4: Create Grafana Dashboard

1. Go to **Dashboards → New Dashboard**
2. Add a **Stat Panel**
   ```
   sdwan_devices_up
   ```
3. Add a **Time Series Panel**
   ```
   sdwan_devices_up
   ```
4. Save dashboard as **SD-WAN Monitoring**

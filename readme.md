# Cài đặt Traffic Live Monitor & Network Rules

## 1. Tạo Backend (FastAPI)
```bash
cd /www/wwwroot/m.mywep.site
nano 1.py
```

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import HTMLResponse, FileResponse
import datetime
import uvicorn
import time
import os
from threading import Thread, Lock
from collections import defaultdict

app = FastAPI()

# Mở CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Biến toàn cục để đếm request và IP
total_requests = 0
requests_this_second_by_ip = defaultdict(int)
stats_lock = Lock()

# Lịch sử 60 giây gần nhất
traffic_history = []
MAX_HISTORY = 60

@app.middleware("http")
async def count_requests(request: Request, call_next):
    global total_requests
    if not request.url.path.startswith("/api/traffic_data"):
        client_ip = request.client.host
        with stats_lock:
            total_requests += 1
            requests_this_second_by_ip[client_ip] += 1
    
    response = await call_next(request)
    return response

# Thread cập nhật lịch sử mỗi giây
def history_updater():
    global traffic_history, requests_this_second_by_ip
    while True:
        time.sleep(1)
        with stats_lock:
            current_ips = dict(requests_this_second_by_ip)
            total_in_sec = sum(current_ips.values())
            requests_this_second_by_ip.clear()
            
        now = datetime.datetime.now().strftime("%H:%M:%S")
        traffic_history.append({"time": now, "total": total_in_sec, "ips": current_ips})
        if len(traffic_history) > MAX_HISTORY:
            traffic_history.pop(0)

Thread(target=history_updater, daemon=True).start()

@app.get("/api/traffic_data")
def get_traffic_data():
    return {
        "categories": [item["time"] for item in traffic_history], 
        "data_reqs": [item["total"] for item in traffic_history],
        "ip_details": [item["ips"] for item in traffic_history],
        "total_requests": total_requests
    }

# Route chính: Phục vụ luôn file index.html để chạy trên VPS dễ dàng
@app.get("/", response_class=HTMLResponse)
def root():
    if os.path.exists("index.html"):
        return FileResponse("index.html")
    return "<h1>Server running. index.html not found.</h1>"

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 2. Tạo Frontend (Giao diện)
```bash
nano index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Request Monitor</title>
    <script src="[https://cdn.jsdelivr.net/npm/apexcharts](https://cdn.jsdelivr.net/npm/apexcharts)"></script>
    <style>
        body { background-color: #ffffff; color: #333; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; margin: 0; padding: 20px; display: flex; flex-direction: column; align-items: center; }
        .container { max-width: 1000px; width: 100%; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        .header { background-color: #f8f9fa; border-bottom: 1px solid #ddd; padding: 15px 25px; display: flex; justify-content: space-between; align-items: center; }
        h2 { margin: 0; font-size: 1.25rem; }
        .total-requests { font-weight: bold; color: #d63031; font-size: 1.1rem; }
        #chart { padding: 10px; }
        .ip-list { margin-top: 5px; padding-left: 10px; }
        .ip-item { font-size: 12px; color: #fff; list-style-type: disc; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>Traffic Live Monitor (Requests/s)</h2>
            <div class="total-requests">Total Requests: <span id="totalCount">0</span></div>
        </div>
        <div id="chart"></div>
    </div>

    <script>
        let globalIpDetails = [];
        var options = {
            series: [{ name: 'Requests', data: [] }],
            chart: { type: 'area', height: 380, animations: { enabled: false }, toolbar: { show: false }, zoom: { enabled: false } },
            dataLabels: { enabled: false },
            stroke: { curve: 'straight', width: 2, colors: ['#0984e3'] },
            fill: { type: 'solid', opacity: 0.1, colors: ['#74b9ff'] },
            xaxis: { categories: [], labels: { show: false }, axisBorder: { show: false }, axisTicks: { show: false } },
            yaxis: { min: 0, forceNiceScale: false, labels: { formatter: function(val) { return Math.floor(val); } } },
            grid: { borderColor: '#eee', xaxis: { lines: { show: true } } },
            tooltip: {
                enabled: true, theme: 'dark',
                custom: function({ series, seriesIndex, dataPointIndex, w }) {
                    const reqCount = series[seriesIndex][dataPointIndex];
                    const time = w.globals.labels[dataPointIndex];
                    const ips = globalIpDetails[dataPointIndex] || {};
                    let ipHtml = '';
                    for (const [ip, count] of Object.entries(ips)) { ipHtml += `<div class="ip-item">${ip}: <b>${count}</b></div>`; }
                    if (!ipHtml) ipHtml = '<div>Không có request</div>';
                    return `<div style="padding: 10px; background: #2d3436; border-radius: 4px;">
                                <div style="font-weight: bold; border-bottom: 1px solid #636e72; padding-bottom: 5px; margin-bottom: 5px;">${time} - Total: ${reqCount}</div>
                                <div class="ip-list">${ipHtml}</div>
                            </div>`;
                }
            }
        };

        var chart = new ApexCharts(document.querySelector("#chart"), options);
        chart.render();

        async function refresh() {
            try {
                const res = await fetch('/api/traffic_data');
                const result = await res.json();
                if (result.categories) {
                    globalIpDetails = result.ip_details; 
                    chart.updateOptions({ xaxis: { categories: result.categories }, yaxis: { max: Math.max(5, ...result.data_reqs) + 1 } });
                    chart.updateSeries([{ data: result.data_reqs }]);
                    document.getElementById('totalCount').innerText = result.total_requests;
                }
            } catch (err) { console.error("Lỗi khi lấy dữ liệu:", err); }
        }
        setInterval(refresh, 1000);
        refresh();
    </script>
</body>
</html>
```

## 3. Quản lý Service chạy ngầm
```bash
nano /etc/systemd/system/traffic.service
```

```ini
[Unit]
Description=Traffic Dashboard
After=network.target

[Service]
User=root
WorkingDirectory=/www/wwwroot/m.mywep.site
ExecStart=/usr/bin/python3 1.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
# Xem trạng thái Service
systemctl status traffic
```

## 4. Giám sát Network & Iptables
```bash
# Xem cổng mạng vật lý (vd: eth0)
ip link show

# Giám sát DNS & Ping
sudo tcpdump -i any udp port 53 -n
sudo tcpdump -i eth0 'icmp[icmptype] = icmp-echoreply' -n

# Iptables: Chặn Spam Ping (Drop nếu >10 request/60s)
sudo iptables -I INPUT 1 -p icmp --icmp-type echo-request -m recent --name PING_LIMIT --update --seconds 60 --hitcount 10 -j DROP
sudo iptables -I INPUT 2 -p icmp --icmp-type echo-request -m recent --name PING_LIMIT --set -j ACCEPT

# Quản lý Rules
sudo iptables -L INPUT --line-numbers
sudo iptables -D INPUT 1
```

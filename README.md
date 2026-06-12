# B5-APP-MONITOR-ALERT-DATA-REALTIME
# 
# Lý thuyết
## 1. Docker
Docker là một nền tảng mã nguồn mở cho phép bạn đóng gói ứng dụng và tất cả các thành phần phụ thuộc của nó (libraries, môi trường runtime, cấu hình hệ thống...) vào trong một đơn vị duy nhất gọi là Container.

Để dễ hình dung:

- Trước khi có Docker: Bạn chạy ứng dụng trực tiếp trên hệ điều hành (hoặc Máy ảo - VM). Việc này dễ dẫn đến lỗi "Trên máy tôi chạy được nhưng lên server lại lỗi" do khác biệt về phiên bản thư viện hoặc môi trường.

- Khi có Docker: Ứng dụng chạy trong một môi trường cô lập (Container). Nó hoạt động y hệt nhau dù ở trên laptop của bạn, trên máy của đồng nghiệp, hay trên server production.

## 2. Các keyword trong docker-compose.yml
File docker-compose.yml dùng để định nghĩa và quản lý một ứng dụng gồm nhiều container (multi-container). Dưới đây là các từ khóa phổ biến chia theo từng thành phần:

### Định nghĩa Service (Dịch vụ / Container)
Mỗi service đại diện cho một container sẽ được khởi tạo.

- image: Chỉ định Docker Image được sử dụng để tạo container.

  - Ý nghĩa: Tải từ Docker Hub hoặc local registry nếu có sẵn.
  - Ví dụ: image: postgres:15-alpine

- build: Chỉ định đường dẫn đến thư mục chứa Dockerfile để Docker Compose tự build image tại chỗ thay vì tải image có sẵn.
  - Ví dụ: build: ./backend

- ports: Ánh xạ (mapping) cổng từ Máy host (máy thật) vào trong Container. Cấu trúc là HOST:CONTAINER.
  - Ví dụ: ports: - "8080:80" (Truy cập từ bên ngoài qua cổng 8080 sẽ dẫn vào cổng 80 của container).

- environment: Khai báo các biến môi trường (Environment Variables) bên trong container.
  - Ví dụ: ```yaml

    environment:
    - DB_HOST=db
    - DB_USER=root

- volumes: Gắn một thư mục/file từ máy host vào container, hoặc gắn một Docker Volume vào container để lưu trữ dữ liệu bền vững.
  - Ví dụ: - ./data:/var/lib/mysql (Lưu dữ liệu MySQL ra máy host để không bị mất khi xóa container).

- networks: Chỉ định container này tham gia vào mạng nội bộ nào để giao tiếp với các container khác.
  - Ví dụ: ```yaml
  
    networks:
    - app-network

- depends_on: Thiết lập thứ tự khởi động giữa các service.
  - Ví dụ: depends_on: - db (Ứng dụng web sẽ đợi container database khởi động trước rồi mới khởi động sau).

### Định nghĩa Network (Mạng)
Dùng để tạo một không gian mạng cô lập cho các container giao tiếp với nhau bằng tên service thay vì địa chỉ IP.

- networks (cấp cao nhất): Khai báo các mạng sẽ dùng trong file compose.

- driver: Chỉ định kiểu mạng (thường mặc định là bridge trên một máy đơn lẻ).

  - Ví dụ:
  ```
    networks:
      my-net:
        driver: bridge
  ```
### Định nghĩa Volume (Phần lưu trữ dữ liệu)
Dùng để quản lý vòng đời của dữ liệu một cách độc lập với container. Khi container bị xóa, dữ liệu trong volume vẫn an toàn.

- volumes (cấp cao nhất): Khai báo các volume được Docker quản lý.

  - Ví dụ:
  ```
  volumes:
    db_data: # Tạo một vùng lưu trữ tên là db_data
  ```
### 📝 Ví dụ minh họa một file docker-compose.yml hoàn chỉnh:
```
version: "3.9"

services:
  mariadb:
    image: mariadb:11
    container_name: mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: monitor_db
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - monitor_net

  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - monitor_net

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./nodered:/data
    depends_on:
      - mariadb
      - influxdb
    networks:
      - monitor_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - influxdb
    environment:
      GF_SECURITY_ALLOW_EMBEDDING: "true"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitor_net

  flask-api:
    build:
      context: ./flask-api
    container_name: flask-api
    restart: unless-stopped
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - monitor_net

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/html:/usr/share/nginx/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - flask-api
      - grafana
    networks:
      - monitor_net

volumes:
  mariadb_data:
  influxdb_data:
  grafana_data:

networks:
  monitor_net:
    driver: bridge

```
### 3. Ưu điểm khi triển khai ứng dụng sử dụng Docker
- Nhất quán môi trường (Consistency): Loại bỏ hoàn toàn lỗi "chạy được trên máy em nhưng lỗi trên server". Dev, test, production dùng chung một môi trường 100%.
- Triển khai nhanh chóng và gọn nhẹ (Speed & Light): Container khởi động chỉ trong vài giây vì chúng chia sẻ chung nhân (kernel) của hệ điều hành host, không cần khởi động lại toàn bộ OS như máy ảo (VM).
- Cô lập an toàn (Isolation): Mỗi container hoạt động độc lập. Một container bị lỗi hay sập nguồn không làm ảnh hưởng đến các container khác trên cùng một máy chủ.
- Dễ dàng mở rộng (Scalability): Rất dễ dàng tăng số lượng container (scale up) khi lượng traffic tăng đột biến và giảm xuống (scale down) khi hết tải.
- Tiết kiệm tài nguyên: Sử dụng RAM và CPU hiệu quả hơn nhiều so với việc chạy nhiều Máy ảo (Virtual Machines).

## 4. Các bước triển khai ứng dụng bằng Docker lên Server KHÔNG CÓ INTERNET (Offline Deployment)
Khi server thật không có internet (môi trường Air-gapped), bạn không thể dùng lệnh docker pull hay docker-compose up để tải image từ Docker Hub trực tiếp trên server đó. Quy trình chuẩn sẽ bao gồm các bước sau:

### Bước 1: Chuẩn bị tại Laptop cá nhân (Có Internet)
Bạn cần chuẩn bị sẵn các Docker Image và bộ cài đặt Docker.

1. Build ứng dụng thành Image: Chạy lệnh build ứng dụng của bạn thành Docker Image hoàn chỉnh.

2. Đóng gói các Image thành file .tar: Sử dụng lệnh docker save để xuất các image (bao gồm cả image ứng dụng của bạn và các image phụ thuộc như Nginx, Postgres, MySQL...) ra thành file nén.

- Lệnh: docker save -o my-app-image.tar my-app:latest

- Lệnh cho DB: docker save -o postgres-image.tar postgres:15

3. Tải file cài đặt Docker Offline cho Server: Tải file cài đặt Docker dạng .rpm hoặc .deb (tùy thuộc OS của server là CentOS hay Ubuntu) cùng với công cụ docker-compose dạng binary về laptop.

### Bước 2: Chuyển dữ liệu sang Server thật (Offline)
- Sử dụng USB, ổ cứng di động hoặc mạng LAN nội bộ (nếu có kết nối an toàn với server) để copy các file sau sang server:
  - Các file .tar (chứa Docker Images).
  - Bộ cài đặt Docker Offline và Docker Compose binary.
  - Thư mục mã nguồn chứa file docker-compose.yml (đã chỉnh sửa đường dẫn image để trỏ đúng vào các image local).

### Bước 3: Cài đặt và Triển khai trên Server thật
1. Cài đặt Docker Offline: Cài đặt Docker từ các file package (.rpm / .deb) đã copy sang. Di chuyển file docker-compose vào thư mục /usr/local/bin/ và cấp quyền thực thi (chmod +x).

2. Nạp (Load) các Image vào Docker: Sử dụng lệnh docker load để giải nén các file .tar ngược trở lại vào bộ quản lý image của server.

- Lệnh: docker load -i my-app-image.tar

- Lệnh: docker load -i postgres-image.tar

3. Khởi chạy ứng dụng: Di chuyển vào thư mục chứa file docker-compose.yml và chạy lệnh như bình thường:

- Lệnh: docker-compose up -d
  Docker Compose sẽ phát hiện các image đã có sẵn ở local (do bạn vừa load xong) và tiến hành dựng container ngay lập tức mà không cần kết nối Internet để tải gì thêm.



# Thực hành 
## 1. Tổng quan hệ thống
### 1.1. Giới thiệu
Dự án xây dựng một hệ thống giám sát và cảnh báo tự động theo thời gian thực (App Monitor + Alert Data Realtime) chạy trên nền tảng Docker Container. Đối tượng giám sát thực tế được lựa chọn ở đây là dữ liệu thời tiết thu thập trực tiếp thông qua API.

Các thành phần cốt lõi trong hệ thống:

- Node-RED: Đóng vai trò là trung tâm điều phối dữ liệu (ETL Workflow). Thực hiện tác vụ lấy dữ liệu động liên tục, phân tích dị thường, ghi đồng thời vào 2 cơ sở dữ liệu và kích hoạt bot Telegram gửi cảnh báo khi có sự cố về giá.
- MariaDB: Cơ sở dữ liệu quan hệ (RDBMS) dùng để lưu trữ trạng thái tức thời (giá trị mới nhất) nhằm phục vụ các truy vấn nhanh của ứng dụng Web Client.
- InfluxDB (v1.8): Cơ sở dữ liệu chuỗi thời gian (Time-series Database) tối ưu cho việc lưu trữ dữ liệu lịch sử, phục vụ vẽ biểu đồ phân tích xu hướng theo thời gian.
- Flask API (Python): Xây dựng dịch vụ API nội bộ kết nối trực tiếp với MariaDB, cung cấp endpoint endpoint trả về dữ liệu giá mới nhất dạng JSON cho giao diện người dùng.
- Nginx: Web Server phân phối giao diện Frontend tĩnh và đồng thời đóng vai trò làm Reverse Proxy điều hướng luồng request từ trình duyệt client sang Flask API một cách an toàn thông qua cấu hình mạng nội bộ Docker.
- Grafana: Nền tảng trực quan hóa dữ liệu mạnh mẽ, kết nối với InfluxDB để vẽ biểu đồ kỹ thuật và cho phép nhúng trực tiếp vào giao diện Frontend qua thẻ Iframe không cần đăng nhập.

### 1.2. Cấu trúc thư mục dự án
```
bt5-monitor/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── html/
│       ├──index.html
│       ├──script.js
│       ├──style.css
├── flask_api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── nodered/          
├── influxdb/         
└── mariadb/
```
### 1.3. Danh sách service và cổng
## Cấu hình Port Mapping các Service Docker

| Tên Service (Docker) | Cổng Nội bộ (Container) | Cổng Công khai (Host) | Mục đích sử dụng |
|----------------------|-------------------------|-----------------------|------------------|
| `nginx` | `80` | `80` | Giao diện Frontend (HTML/JS) & Reverse Proxy |
| `flask` | `5000` | Không mở (Ẩn) | Cung cấp API lấy dữ liệu tức thời từ MariaDB |
| `nodered` | `1880` | `1880` | Thu thập dữ liệu thời tiết, lưu DB, gửi Telegram |
| `mariadb` | `3306` | `3306` | Lưu trữ dữ liệu thời tiết tức thời (Latest) |
| `influxdb` | `8086` | `8086` | Lưu trữ dữ liệu lịch sử (Time-series) |
| `grafana` | `3000` | `3000` | Trực quan hóa biểu đồ lịch sử (Nhúng Iframe) |

> **Lưu ý mạng nội bộ:** Các service giao tiếp với nhau bằng tên service bên trong mạng Docker chung `weather_network`.

## 2. Cấu hình
### 2.1. Cấu hình file docker-compose.yml
```
version: "3.9"

services:
  mariadb:
    image: mariadb:11
    container_name: mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: monitor_db
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - monitor_net

  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - monitor_net

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./nodered:/data
    depends_on:
      - mariadb
      - influxdb
    networks:
      - monitor_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - influxdb
    environment:
      GF_SECURITY_ALLOW_EMBEDDING: "true"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitor_net

  flask-api:
    build:
      context: ./flask-api
    container_name: flask-api
    restart: unless-stopped
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - monitor_net

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/html:/usr/share/nginx/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - flask-api
      - grafana
    networks:
      - monitor_net

volumes:
  mariadb_data:
  influxdb_data:
  grafana_data:

networks:
  monitor_net:
    driver: bridge

```
<img width="1919" height="1026" alt="Screenshot 2026-06-12 112538" src="https://github.com/user-attachments/assets/31167581-441b-4c53-82ac-9e931e2e707e" />

### 2.2. Xây dựng Flask API
#### Bước 1: Khai báo các thư viện cần thiết trong flask_api/requirements.txt
Khai báo các thư viện Python cần thiết để kết nối MariaDB và chạy API.
```
flask
pymysql
flask-cors
```
<img width="1919" height="1026" alt="Screenshot 2026-06-12 112729" src="https://github.com/user-attachments/assets/24cc8007-f790-4818-a0aa-ef30b793962b" />

#### Bước 2: Edit file app.py
Đoạn code này sẽ tạo một API endpoint /api/weather để kết nối vào MariaDB (sử dụng tên service weather_mariadb làm host) và lấy ra bản ghi thời tiết mới nhất.
```
from flask import Flask, jsonify
import pymysql
from datetime import datetime

app = Flask(__name__)

db_config = {
    'host': 'mariadb',
    'user': 'root',
    'password': '123456',
    'database': 'monitor_db',
    'cursorclass': pymysql.cursors.DictCursor
}

@app.route("/api/weather")
def get_weather():
    try:
        connection = pymysql.connect(**db_config)
        with connection.cursor() as cursor:
            sql = "SELECT temperature, humidity, windspeed, update_time FROM weather_realtime ORDER BY id DESC LIMIT 1;"
            cursor.execute(sql)
            result = cursor.fetchone()
        connection.close()

        if result:
            if isinstance(result['update_time'], datetime):
                result['update_time'] = result['update_time'].strftime('%Y-%m-%d %H:%M:%S')
            return jsonify(result)
        return jsonify({"message": "No data available yet"}), 404
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
<img width="1919" height="1031" alt="Screenshot 2026-06-12 112828" src="https://github.com/user-attachments/assets/da61168a-ef90-4142-a551-90c667bd80d7" />

### Bước 3: Edit file Dockerfile
File này dùng để Docker đóng gói ứng dụng Flask thành một Image riêng.
```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```
<img width="1919" height="1036" alt="Screenshot 2026-06-12 112920" src="https://github.com/user-attachments/assets/7c8c4a32-00d4-4b98-a11d-69e87013dbcb" />

## 2.3. Cấu hình Nginx làm Web Server & Reverse Proxy
- File: nginx/nginx.conf
- File này cấu hình Nginx lắng nghe ở cổng 80 (nội bộ container). Nếu user vào trang chủ / thì trả về file HTML, nếu vào tuyến đường /api/ thì Nginx sẽ "bắn" request đó sang cho container Flask xử lý.
```
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /api/ {
        proxy_pass http://flask-api:5000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
<img width="1919" height="1032" alt="Screenshot 2026-06-12 113022" src="https://github.com/user-attachments/assets/28978193-0b5e-48f6-b1ea-413e7878eac1" />

## 2.4. Tạo file giao diện web
> Viết file index.html làm frontend cho hệ thống.
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>Realtime Weather Monitor | Hệ thống giám sát thời tiết</title>
    <link rel="stylesheet" href="style.css">
    <!-- Font Awesome 6 cho icon đẹp mắt -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
</head>
<body>
    <div class="wrapper">
        <h2>HỆ THỐNG GIÁM SÁT THỜI TIẾT REALTIME</h2>
        <div class="monitor-grid">
            <div class="card temp-card">
                <h3><i class="fas fa-thermometer-half"></i> Nhiệt độ</h3>
                <div class="value"><span id="temp">--</span>°C</div>
            </div>
            <div class="card humidity-card">
                <h3><i class="fas fa-tint"></i> Độ ẩm</h3>
                <div class="value"><span id="humidity">--</span>%</div>
            </div>
            <div class="card wind-card">
                <h3><i class="fas fa-wind"></i> Tốc độ gió</h3>
                <div class="value"><span id="wind">--</span> km/h</div>
            </div>
        </div>
        <p class="timestamp">
            <i class="far fa-clock"></i> Cập nhật lúc: <span id="time">--</span>
        </p>

        <div class="grafana-section">
            <h3><i class="fas fa-chart-line"></i> Biểu đồ lịch sử (Grafana)</h3>
            <!-- Nhúng iframe từ Grafana (thay đổi URL sau khi cấu hình Grafana) -->
            <div class="iframe-container">
                <iframe src="http://192.168.183.130:3000/d-solo/adfjdsq/new-dashboard?orgId=1&from=1781218829580&to=1781240429580&timezone=browser&panelId=panel-1" width="100%" height="400" frameborder="0" title="Grafana Chart 1"></iframe>
            </div>
            <div class="iframe-container" style="margin-top: 20px;">
                <iframe id="grafana-frame" src="" width="100%" height="450" frameborder="0" title="Grafana Dashboard"></iframe>
            </div>
        </div>
    </div>
    <script src="script.js"></script>
</body>
</html>
```
<img width="1919" height="1027" alt="image" src="https://github.com/user-attachments/assets/a3dacff2-ba85-4200-b058-b9b6cdf5a259" />

> Viết file script.js
```
function updateDashboard() {
    fetch('/api/weather')
        .then(response => {
            if (!response.ok) throw new Error("Chưa có dữ liệu mới");
            return response.json();
        })
        .then(data => {
            document.getElementById('temp').innerText = data.temperature;
            document.getElementById('humidity').innerText = data.humidity;
            document.getElementById('wind').innerText = data.windspeed;
            document.getElementById('time').innerText = data.update_time;
        })
        .catch(err => console.log(err));
}

// Gọi API định kỳ mỗi 3 giây
setInterval(updateDashboard, 3000);
updateDashboard();

function updateWeatherData() {
    // Gọi API thông qua proxy Nginx (/api/)
    fetch("/api/")
        .then(response => response.json())
        .then(data => {
            // Kiểm tra xem dữ liệu trả về có hợp lệ hay không
            if (data && data.temperature !== undefined) {
                // Đổ dữ liệu vào các thẻ ID tương ứng trên HTML
                document.getElementById("temp").innerText = data.temperature;
                document.getElementById("humidity").innerText = data.humidity;
                document.getElementById("wind").innerText = data.windspeed;

                // Định dạng thời gian hiển thị nội bộ
                const now = new Date();
                document.getElementById("time").innerText = now.toLocaleTimeString('vi-VN') + ' ' + now.toLocaleDateString('vi-VN');
            }
        })
        .catch(error => {
            console.error("Lỗi khi kết nối tới Flask API:", error);
        });
}

// Chạy hàm cập nhật ngay khi vừa tải xong trang web
updateWeatherData();

// Thiết lập tự động gọi API lấy dữ liệu mới sau mỗi 5 giây (5000ms) để đồng bộ với Node-RED
setInterval(updateWeatherData, 5000);
```
<img width="1919" height="1030" alt="image" src="https://github.com/user-attachments/assets/bfa65f6f-1e17-4651-8cd4-61ee82b9ebf6" />

> Viết file style.css
```
/* Reset & global styles */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: #333;
    text-align: center;
    margin: 0;
    padding: 40px 20px;
    min-height: 100vh;
    position: relative;
}

/* Hiệu ứng nền động nhẹ */
body::before {
    content: '';
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1440 320"><path fill="rgba(255,255,255,0.08)" fill-opacity="0.4" d="M0,96L48,112C96,128,192,160,288,160C384,160,480,128,576,122.7C672,117,768,139,864,154.7C960,171,1056,181,1152,165.3C1248,149,1344,107,1392,85.3L1440,64L1440,320L1392,320C1344,320,1248,320,1152,320C1056,320,960,320,864,320C768,320,672,320,576,320C480,320,384,320,288,320C192,320,96,320,48,320L0,320Z"></path></svg>') repeat-x bottom;
    background-size: cover;
    opacity: 0.2;
    pointer-events: none;
    z-index: 0;
}

/* Wrapper chính */
.wrapper {
    max-width: 1100px;
    margin: 0 auto;
    background: rgba(255, 255, 255, 0.95);
    padding: 35px 30px;
    border-radius: 48px;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.35), 0 0 0 1px rgba(255, 255, 255, 0.2) inset;
    position: relative;
    z-index: 2;
    transition: all 0.3s ease;
    animation: fadeSlide 0.7s ease-out;
}

@keyframes fadeSlide {
    0% { opacity: 0; transform: translateY(15px);}
    100% { opacity: 1; transform: translateY(0);}
}

/* Header đẹp mắt */
h2 {
    font-family: 'Segoe UI', 'Poppins', sans-serif;
    font-size: 1.9rem;
    font-weight: 700;
    background: linear-gradient(135deg, #1e2b6e 0%, #2c3e8f 50%, #1a237e 100%);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
    letter-spacing: -0.3px;
    margin-bottom: 15px;
    display: inline-block;
    padding: 0 20px 12px;
    border-bottom: 4px solid;
    border-image: linear-gradient(90deg, #f093fb, #f5576c, #4facfe) 1;
    border-bottom-style: solid;
    border-bottom-width: 4px;
    position: relative;
}

h2::before {
    content: '🌤️';
    font-size: 2rem;
    margin-right: 12px;
    background: none;
    color: #f5b042;
}

/* Grid các card */
.monitor-grid {
    display: flex;
    justify-content: space-between;
    gap: 25px;
    margin: 35px 0 25px;
    flex-wrap: wrap;
}

/* Card styling hoa mĩ */
.card {
    flex: 1;
    min-width: 180px;
    padding: 28px 18px;
    border-radius: 32px;
    color: white;
    position: relative;
    overflow: hidden;
    transition: all 0.4s cubic-bezier(0.2, 0.9, 0.4, 1.1);
    box-shadow: 0 15px 35px -12px rgba(0, 0, 0, 0.3);
    backdrop-filter: blur(2px);
    cursor: pointer;
}

.card:hover {
    transform: translateY(-8px) scale(1.02);
    box-shadow: 0 25px 40px -12px rgba(0, 0, 0, 0.45);
}

/* Icon trang trí cho từng card */
.card::before {
    font-family: "Font Awesome 6 Free";
    font-weight: 900;
    position: absolute;
    bottom: 12px;
    right: 18px;
    font-size: 4rem;
    opacity: 0.2;
    color: white;
    transition: all 0.3s;
}

.card:hover::before {
    opacity: 0.35;
    transform: scale(1.05);
}

.temp-card::before {
    content: "\f2c7";
}
.humidity-card::before {
    content: "\f750";
}
.wind-card::before {
    content: "\f72e";
}

.temp-card {
    background: linear-gradient(145deg, #ff6b6b, #ee5a24);
    background: radial-gradient(circle at 20% 30%, #ff7e5e, #f14a2e);
}
.humidity-card {
    background: linear-gradient(145deg, #4facfe, #00f2fe);
    background: radial-gradient(circle at 70% 20%, #5dade2, #2c7ab1);
}
.wind-card {
    background: linear-gradient(145deg, #43e97b, #38f9a5);
    background: radial-gradient(circle at 30% 70%, #2ecc71, #239b56);
}

.card h3 {
    font-size: 1.5rem;
    font-weight: 600;
    margin-bottom: 20px;
    letter-spacing: 1px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
}

.card h3 i {
    font-size: 1.8rem;
    filter: drop-shadow(2px 2px 4px rgba(0,0,0,0.2));
}

.value {
    font-size: 3rem;
    font-weight: 800;
    margin-top: 10px;
    font-family: 'Segoe UI', monospace;
    letter-spacing: 2px;
    text-shadow: 2px 2px 8px rgba(0,0,0,0.2);
    display: flex;
    align-items: baseline;
    justify-content: center;
    gap: 5px;
}

.value span {
    font-size: 3.2rem;
    line-height: 1;
}

/* Timestamp cập nhật */
.timestamp {
    margin: 20px 0 25px;
    font-size: 1rem;
    font-weight: 500;
    color: #2c3e66;
    background: rgba(100, 108, 255, 0.1);
    display: inline-block;
    padding: 10px 24px;
    border-radius: 60px;
    backdrop-filter: blur(4px);
    font-family: monospace;
    letter-spacing: 0.5px;
}

.timestamp i {
    margin-right: 8px;
    color: #5f6caf;
}

.timestamp span {
    font-weight: 700;
    color: #1e2b6e;
    background: rgba(255,255,240,0.8);
    padding: 3px 12px;
    border-radius: 30px;
    margin-left: 6px;
}

/* Grafana Section */
.grafana-section {
    margin-top: 40px;
    border-top: 2px dashed rgba(0, 0, 0, 0.1);
    padding-top: 30px;
    background: linear-gradient(to bottom, rgba(255,255,255,0.5), rgba(245,248,255,0.9));
    border-radius: 36px;
    padding: 20px 20px 25px;
}

.grafana-section h3 {
    font-size: 1.7rem;
    font-weight: 700;
    background: linear-gradient(120deg, #2b3b6e, #1f2b4e);
    background-clip: text;
    -webkit-background-clip: text;
    color: transparent;
    margin-bottom: 20px;
    display: inline-flex;
    align-items: center;
    gap: 12px;
}

.grafana-section h3 i {
    font-size: 1.8rem;
    color: #f39c12;
    background: none;
    -webkit-background-clip: unset;
    background-clip: unset;
    color: #e67e22;
}

/* Container cho iframe đẹp mắt */
.iframe-container {
    position: relative;
    width: 100%;
    border-radius: 24px;
    overflow: hidden;
    box-shadow: 0 12px 28px rgba(0, 0, 0, 0.2);
    background: #00000008;
    transition: all 0.3s;
    margin-top: 15px;
}

.iframe-container:hover {
    transform: scale(0.99);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
}

iframe {
    border: none;
    width: 100%;
    display: block;
    background: #f8faff;
    border-radius: 20px;
}

#grafana-frame {
    width: 100%;
    height: 450px;
    border-radius: 20px;
}

.grafana-section iframe:first-of-type {
    width: 100%;
    height: 400px;
    margin-bottom: 0;
    border-radius: 20px;
}

/* Responsive trên mobile */
@media (max-width: 780px) {
    body {
        padding: 20px 12px;
    }
    .wrapper {
        padding: 20px 18px;
    }
    h2 {
        font-size: 1.3rem;
    }
    .monitor-grid {
        gap: 15px;
    }
    .card {
        padding: 18px 12px;
        min-width: 140px;
    }
    .value {
        font-size: 2.2rem;
    }
    .value span {
        font-size: 2.4rem;
    }
    .card h3 {
        font-size: 1.2rem;
    }
    .grafana-section h3 {
        font-size: 1.3rem;
    }
    #grafana-frame, .grafana-section iframe:first-of-type {
        height: 280px;
    }
}

@media (max-width: 560px) {
    .monitor-grid {
        flex-direction: column;
    }
    .card {
        width: 100%;
    }
}
```
<img width="1919" height="1042" alt="image" src="https://github.com/user-attachments/assets/fc5333c6-94ba-44c2-8654-55fa6494a844" />

# B5-APP-MONITOR-ALERT-DATA-REALTIME
- Môn học: Phát triển ứng dụng với mã nguồn mở - TEE0421
- Họ và tên: Ngụy Đình Tuấn Hà
- MSSV: K225480106011
# YÊU CẦU BÀI TẬP
# LÝ THUYẾT
```
+ docker là gì? 
+ các keyword được sử dụng trong docker-compose.yml
  để mô tả 1 service, network, volume,...
  liệt kê + ý nghĩa của từ khoá đó + ví dụ minh hoạ
+ ưu điểm khi triển app sử dụng docker là gì?
+ dùng docker: tạo app, test app OK trên laptop cá nhân
  giờ muốn triển khai app này trên máy chủ thật ko có internet
  thì các bước cần làm là?
```
# THỰC HÀNH ÁP DỤNG
```
sử dụng docker compose có nhiều serivce 
và các thành phần cần thiết để tạo thành ứng dụng:
 + nodered liên tục lấy dữ liệu từ nguồn nào đó (chứng khoán, thời tiết, giá vàng,...)
   nguồn thực tế, số liệu luôn động sau thời gian ngắn
 + nodered lưu trữ dữ liệu vào 2 database: mariadb để lưu giá trị tức thời
   lưu lịch sử vào influxdb
 + sử dụng grafana để trực quan hoá dữ liệu: vẽ biểu đồ
 + sử dụng nginx để làm webserver
   chạy 1 trang web html+js+css làm front-end
   js: lấy dữ liệu tức thời trong mariadb qua (ajax | socket) 
       gọi api (api tự build bằng Flask giống bt1)
       api trả về giá trị tức thời trong mariadb
       hiển thị lên web, auto hiển thị số mới khi thay đổi
   sử dụng iframe để gọi grafana
   hiển thị biểu đồ dữ liệu lịch sử của thông số đã lưu
 + QUAN SÁT DỮ LIỆU LỊCH SỬ => GIÁ TRỊ BẤT THƯỜNG
   (VD MIỀN A..B: OK, DƯỚI A: ALERT LOW, TRÊN B: ALERT HIGH)
 + nodered: kết hợp bot Telegram
   khi dữ liệu not OK, thì gửi tin nhắn từ bot => group trên telegram
   group đã add bot vào: (nhóm đã có 2 người), add thêm 1875746636 thành 3 người
   mỗi khi bot gửi dữ liệu vào nhóm: mọi member of group đều nhận đc
   nội dung alert: tường minh, có value gây alert

 xuất tất cả các container ra file nén.
 xoá mọi container đang chạy
 load lại các container  từ file nén để khôi phục các container đã xoá
```
# BÀI LÀM
# Phần 1: Lý thuyết
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



# Phần 2: Thực hành 
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
    <title>Realtime Weather Monitor</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="wrapper">
        <h2>HỆ THỐNG GIÁM SÁT THỜI TIẾT REALTIME</h2>
        <div class="monitor-grid">
            <div class="card temp-card">
                <h3>Nhiệt độ</h3>
                <div class="value"><span id="temp">--</span>°C</div>
            </div>
            <div class="card humidity-card">
                <h3>Độ ẩm</h3>
                <div class="value"><span id="humidity">--</span>%</div>
            </div>
            <div class="card wind-card">
                <h3>Tốc độ gió</h3>
                <div class="value"><span id="wind">--</span> km/h</div>
            </div>
        </div>
        <p class="timestamp">Cập nhật lúc: <span id="time">--</span></p>

        <div class="grafana-section">
            <h3>Biểu đồ lịch sử (Grafana)</h3>
            <!-- Nhúng iframe từ Grafana (thay đổi URL sau khi cấu hình Grafana) -->
            <iframe src="http://192.168.183.130:3000/d-solo/adch2pt/1?orgId=1&from=1781225551640&to=1781247151640&timezone=browser&panelId=panel-1" width="900" height="400" frameborder="0"></iframe>
            <iframe id="grafana-frame" src="" width="100%" height="450" frameborder="0"></iframe>
        </div>
    </div>
    <script src="script.js"></script>
</body>
</html>
```
<img width="1919" height="1034" alt="image" src="https://github.com/user-attachments/assets/12d337ca-3513-4f4f-add5-d931561d1cd8" />


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
<img width="1919" height="1031" alt="image" src="https://github.com/user-attachments/assets/aafc3866-5c25-4a6a-ab88-b5cc08a3276f" />

> Viết file style.css
```
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: #333;
    text-align: center;
    margin: 0;
    padding: 20px;
    min-height: 100vh;
    position: relative;
}

/* Thêm hiệu ứng nền nhẹ */
body::before {
    content: '';
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: radial-gradient(circle at 20% 50%, rgba(255,255,255,0.1) 0%, transparent 50%);
    pointer-events: none;
}

.wrapper {
    max-width: 1000px;
    margin: 0 auto;
    background: rgba(255, 255, 255, 0.96);
    padding: 30px 25px;
    border-radius: 32px;
    box-shadow: 0 25px 45px -12px rgba(0, 0, 0, 0.35), 0 1px 2px rgba(0,0,0,0.05);
    position: relative;
    backdrop-filter: blur(2px);
    transition: transform 0.2s ease;
}

/* Header đẹp hơn */
h2 {
    font-size: 1.8rem;
    font-weight: 700;
    background: linear-gradient(120deg, #1e2b6e, #2c3e8f, #1a237e);
    background-clip: text;
    -webkit-background-clip: text;
    color: transparent;
    margin-bottom: 25px;
    letter-spacing: -0.3px;
    position: relative;
    display: inline-block;
    padding-bottom: 10px;
}

h2::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 20%;
    width: 60%;
    height: 3px;
    background: linear-gradient(90deg, #f093fb, #f5576c);
    border-radius: 3px;
}

.monitor-grid {
    display: flex;
    justify-content: space-between;
    gap: 25px;
    margin: 30px 0 25px;
    flex-wrap: wrap;
}

.card {
    flex: 1;
    min-width: 170px;
    padding: 28px 18px;
    border-radius: 28px;
    color: white;
    transition: all 0.35s cubic-bezier(0.2, 0.9, 0.4, 1.1);
    box-shadow: 0 15px 30px -10px rgba(0, 0, 0, 0.25);
    position: relative;
    overflow: hidden;
    cursor: pointer;
}

.card:hover {
    transform: translateY(-8px);
    box-shadow: 0 25px 40px -12px rgba(0, 0, 0, 0.4);
}

/* Giữ nguyên màu nền card nhưng thêm gradient đẹp hơn */
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

/* Thêm icon trang trí nhẹ cho từng card */
.temp-card::after {
    content: "🌡️";
    position: absolute;
    bottom: 12px;
    right: 15px;
    font-size: 3rem;
    opacity: 0.2;
}

.humidity-card::after {
    content: "💧";
    position: absolute;
    bottom: 12px;
    right: 15px;
    font-size: 3rem;
    opacity: 0.2;
}

.wind-card::after {
    content: "🍃";
    position: absolute;
    bottom: 12px;
    right: 15px;
    font-size: 3rem;
    opacity: 0.2;
}

.card h3 {
    font-size: 1.5rem;
    font-weight: 600;
    margin-bottom: 18px;
    letter-spacing: 0.5px;
    text-shadow: 1px 1px 3px rgba(0,0,0,0.2);
}

.value {
    font-size: 3rem;
    font-weight: 800;
    margin-top: 8px;
    text-shadow: 2px 2px 6px rgba(0,0,0,0.2);
    display: flex;
    align-items: baseline;
    justify-content: center;
    gap: 5px;
}

.value span {
    font-size: 3.2rem;
    line-height: 1;
}

.timestamp {
    color: #2c3e66;
    font-style: normal;
    font-weight: 500;
    background: rgba(100, 108, 255, 0.12);
    display: inline-block;
    padding: 8px 24px;
    border-radius: 50px;
    margin: 15px 0 10px;
    backdrop-filter: blur(2px);
    font-size: 0.95rem;
}

.timestamp span {
    font-weight: 700;
    color: #1e2b6e;
    background: rgba(255,248,225,0.9);
    padding: 3px 10px;
    border-radius: 30px;
    margin-left: 6px;
}

.grafana-section {
    margin-top: 35px;
    border-top: 2px dashed rgba(0, 0, 0, 0.12);
    padding-top: 25px;
    border-radius: 0;
}

.grafana-section h3 {
    font-size: 1.5rem;
    font-weight: 600;
    color: #1e2b6e;
    margin-bottom: 20px;
    display: inline-flex;
    align-items: center;
    gap: 10px;
    background: rgba(30,43,110,0.05);
    padding: 6px 20px;
    border-radius: 40px;
}

/* Làm đẹp iframe */
iframe {
    border-radius: 20px;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
    transition: all 0.3s ease;
    background: #f8faff;
}

iframe:hover {
    transform: scale(0.99);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

/* Responsive mượt mà */
@media (max-width: 750px) {
    body {
        padding: 15px;
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
        min-width: 130px;
    }
    .card h3 {
        font-size: 1.2rem;
    }
    .value {
        font-size: 2.2rem;
    }
    .value span {
        font-size: 2.4rem;
    }
    .grafana-section h3 {
        font-size: 1.2rem;
    }
    iframe {
        height: 300px;
    }
}

@media (max-width: 550px) {
    .monitor-grid {
        flex-direction: column;
    }
    .card {
        width: 100%;
    }
    iframe {
        height: 250px;
    }
}

/* Thêm hiệu ứng nhẹ khi load trang */
@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.wrapper {
    animation: fadeInUp 0.5s ease-out;
}

.card {
    animation: fadeInUp 0.4s ease-out backwards;
}

.card:nth-child(1) { animation-delay: 0.05s; }
.card:nth-child(2) { animation-delay: 0.1s; }
.card:nth-child(3) { animation-delay: 0.15s; }
```
<img width="1919" height="1026" alt="image" src="https://github.com/user-attachments/assets/cf6c544c-7efa-4ce4-a569-1c6520d9a2db" />

### 2.5. Khởi động hệ thống
Sau khi hoàn tất cấu hình, build và khởi động toàn bộ hệ thống:
```
docker compose up -d --build
```
<img width="1919" height="1030" alt="Screenshot 2026-06-12 113951" src="https://github.com/user-attachments/assets/3562f03b-8e93-4151-8f15-8d5bc40bb283" />

```
docker ps
```
<img width="1919" height="1033" alt="Screenshot 2026-06-12 114009" src="https://github.com/user-attachments/assets/450100bf-7a6a-412d-a363-2216ef4dca48" />

Khởi tạo Table trên MariaDB:
- Truy cập trực tiếp vào container MariaDB để tạo bảng weather_realtime
```
docker exec -it mariadb mariadb -uroot -p123456 -e "
USE monitor_db;
CREATE TABLE IF NOT EXISTS weather_realtime (
    id INT AUTO_INCREMENT PRIMARY KEY,
    temperature FLOAT,
    humidity FLOAT,
    windspeed FLOAT,
    update_time DATETIME
);
"
```
Cấu hình InfluxDB Bucket và Token:
- Truy cập bằng trình duyệt tại: http://192.168.183.130:8086.
- Đăng ký nhanh tài khoản quản trị (VD: User admin, Pass admin123).
- Đặt tên Organization là monitor_org và Bucket đầu tiên là weather_bucket.
- Vào mục Load Data -> API Tokens -> Chọn Generate API Token (All Access) để lấy mã Token.
<img width="1918" height="990" alt="Screenshot 2026-06-12 114216" src="https://github.com/user-attachments/assets/bb4a1890-083a-446e-87d2-73f436b11736" />

<img width="1918" height="994" alt="Screenshot 2026-06-12 114305" src="https://github.com/user-attachments/assets/77b57201-08f1-4c93-b669-a278a0ecb238" />

## 2.6. Cấu hình NODERED để tự động hóa luồng dữ liệu
Trong giai đoạn này, Node-RED sẽ đóng vai trò đầu não thực hiện 4 nhiệm vụ liên tục:

- Cào dữ liệu thời tiết thực tế từ API công khai.
- Lưu trạng thái mới nhất vào MariaDB.
- Lưu lịch sử vào InfluxDB (để Grafana vẽ biểu đồ).
- Phân tích dị thường và kích hoạt Telegram Bot gửi tin nhắn cảnh báo vào Group.
### Bước 1: Chuẩn bị thư viện (nodes) trong Node-RED
Truy cập Node-RED qua địa chỉ http://<IP_máy_chủ_Ubuntu>:1880
```
http://192.168.183.130:1880
```
Click vào Menu (3 dấu gạch ngang góc trên bên phải) -> Chọn Manage palette.Chuyển sang thẻ Install, tìm kiếm và nhấn Install lần lượt 3 thư viện sau:
```
node-red-node-mysql (Kết nối MariaDB)

node-red-contrib-mysql-config

node-red-contrib-influxdb (Kết nối InfluxDB)
```
<img width="1918" height="990" alt="Screenshot 2026-06-12 114451" src="https://github.com/user-attachments/assets/45523e22-37ad-46b1-b0f6-cb4f29086a1e" />

### Bước 2: Chuẩn bị thông tin Telegram Bot
Trước khi viết Flow, cần chuẩn bị thông tin từ Telegram:

Bot Token: Chat với @BotFather trên Telegram, gõ lệnh /newbot, đặt tên cho bot. Sau khi tạo xong, @BotFather sẽ cấp một chuỗi Token. Copy token này để bước sau dán vào Nodered.
<img width="1440" height="941" alt="image" src="https://github.com/user-attachments/assets/de11b713-b269-434d-83e8-d28333f16b65" />

Tạo nhóm chat có bot để cảnh báo:

- Tạo một Group mới trên Telegram, thêm các thành viên vào (bao gồm cả tài khoản ID 1875746636 theo yêu cầu bài tập).
- Thêm cả con Bot vừa tạo ở trên vào nhóm này với quyền Admin (để nó có quyền gửi tin nhắn).
<img width="1917" height="946" alt="image" src="https://github.com/user-attachments/assets/28891cae-01d3-46fb-8424-05641dcac2b6" />

### Bước 3: Thiết kế luồng dữ liệu
kéo các node và điền các thông tin:

> Node inject để thiết lập mỗi 5s lấy dữ liệu một lần
<img width="1917" height="990" alt="Screenshot 2026-06-12 115252" src="https://github.com/user-attachments/assets/94445d62-7ab9-4437-9f76-8577bee2ba7f" />

> Node http request để lấy dữ liệu thực
<img width="1918" height="993" alt="Screenshot 2026-06-12 115300" src="https://github.com/user-attachments/assets/7c320886-24b1-4625-8973-77143fb7271f" />

> chuyển dữ liệu JSON dạng chuỗi (string) thành Object JavaScript
<img width="1917" height="991" alt="Screenshot 2026-06-12 115307" src="https://github.com/user-attachments/assets/7ce5a2ad-f8cc-42c4-957a-fd570caf265f" />

> Lấy dữ liệu Open-Meteo
<img width="1916" height="993" alt="Screenshot 2026-06-12 115314" src="https://github.com/user-attachments/assets/c0c46a61-5cbe-4378-87d8-e7c89f2c32db" />

> SQL Query Builde
<img width="1918" height="997" alt="Screenshot 2026-06-12 115323" src="https://github.com/user-attachments/assets/b729c795-9858-46a5-aca9-b83476208717" />

> Query InfluxDB
<img width="1917" height="995" alt="Screenshot 2026-06-12 115401" src="https://github.com/user-attachments/assets/5c4016e4-c059-40d9-af66-3980a40ca40e" />

> Node switch kiểm tra ngưỡng và cảnh báo nhiệt độ
<img width="1919" height="996" alt="Screenshot 2026-06-12 115233" src="https://github.com/user-attachments/assets/11b46fc4-4256-4fa1-b6b8-e29a0e15069a" />

> Node SQL
<img width="1915" height="998" alt="Screenshot 2026-06-12 115009" src="https://github.com/user-attachments/assets/5bfd87d4-18ff-41a2-b8a0-07e391d1eeaa" />

> Lưu kết quả vào weather_bucket
<img width="1918" height="996" alt="Screenshot 2026-06-12 115123" src="https://github.com/user-attachments/assets/83fc4ed3-a29d-42a1-84ab-93f969a7c3d9" />

<img width="1919" height="998" alt="Screenshot 2026-06-12 115110" src="https://github.com/user-attachments/assets/ac3c56f0-a1bd-4909-b837-6f22602b6d5e" />

> Node http request để bắn cảnh báo tới telegram
<img width="1917" height="994" alt="Screenshot 2026-06-12 115207" src="https://github.com/user-attachments/assets/d69e2919-321c-4b79-a455-85a015f87064" />

### Bước 4: Deploy và kiểm tra
Bấm nút Deploy màu đỏ trên góc phải màn hình để lưu và chạy.
<img width="1917" height="992" alt="Screenshot 2026-06-12 115434" src="https://github.com/user-attachments/assets/aea7a53a-2a42-4a5d-bbe5-bb329ce1eea7" />

Truy cập vào giao diện web để xem kết quả http://192.168.183.130
<img width="1895" height="987" alt="image" src="https://github.com/user-attachments/assets/c325f90c-667d-424d-9aeb-947c4067f69c" />

> Chú ý: Vì nhiệt độ đang ở mức bình thường, nên để có cảnh báo đẩy về telegram, em sẽ sửa lại ngưỡng bất thường high từ 35 độ còn 20 độ.

Kết quả cảnh báo khi nhiệt độ vượt ngưỡng 20 độ:
<img width="1437" height="950" alt="Screenshot 2026-06-12 120752" src="https://github.com/user-attachments/assets/d6d5b256-59f8-42d9-b57f-5a0aae75f667" />

### 2.7. Cấu hình Grafana kết nối InfluxDB
### Bước 1: Đăng nhập grafana
- Truy cập http://192.168.183.130:3000 để vào Grafana
- Đăng nhập và đổi mật khẩu (nếu cần)
<img width="1917" height="995" alt="Screenshot 2026-06-12 115526" src="https://github.com/user-attachments/assets/0887fa9b-160a-4c0b-b8b3-6bde67ceccdf" />

### Bước 2: Thêm datasource
- Tại thanh menu bên trái, chọn Connections -> Data sources -> Add data source.
<img width="1913" height="989" alt="Screenshot 2026-06-12 115604" src="https://github.com/user-attachments/assets/b12cf01c-0bdd-4675-9e3f-4d2996a5e46e" />

<img width="1916" height="989" alt="Screenshot 2026-06-12 115700" src="https://github.com/user-attachments/assets/3a2ca396-4538-4482-a91b-cb0f73ac73e6" />

- Kéo xuống dưới cùng ấn Save & test. Nếu hiện thông báo màu xanh "Data source is working" là thành công!
<img width="1916" height="984" alt="Screenshot 2026-06-12 115709" src="https://github.com/user-attachments/assets/b92dc61d-0afb-4f13-9d8e-167d64bd1f08" />
### Bước 3: Lấy code tạo biểu đồ bên influxdb
- Đăng nhập theo đường link: http://192.168.182.130:8086
- Sau khi tao xong tài khoản thì ta chọn Dashboards -> Create dashboard -> New dashboard -> ALL CELL
- Rồi tích các thông số đã được lưu
<img width="1914" height="956" alt="image" src="https://github.com/user-attachments/assets/5f6d4861-fc84-4e94-81a3-960e6425e29c" />

- Sau khi tích chọn xong ta ấn sang Script Editor rồi copy đoạn code đó
<img width="1919" height="992" alt="image" src="https://github.com/user-attachments/assets/e489dae6-3412-4d12-a30e-b00d05b2819b" />

### Bước 4: Tạo biểu đồ
- Nhấn vào biểu tượng + ở phía trên bên phải, chọn New Dashboard -> add Panel -> Configure Visualization
<img width="1919" height="987" alt="image" src="https://github.com/user-attachments/assets/eb3196a0-59a1-480a-8e87-589e0d00a415" />

- Sau đó add code đã lấy được ở [hàn biểu đồ ở bước 3 vào
<img width="1918" height="995" alt="Screenshot 2026-06-12 115941" src="https://github.com/user-attachments/assets/ae1de100-aa89-470e-870b-3b370cdf6fa0" />

rồi chọn save -> apply -> hiển thị ra kết quả
<img width="1612" height="897" alt="image" src="https://github.com/user-attachments/assets/e9bf2891-215c-4a57-a383-787c14e14c2f" />

### Bước 5: Lấy link nhúng Iframe
- Tại ô biểu đồ vừa vẽ, góc trên cùng bên phải của khung biểu đồ đó -> Xuất hiện dấu 3 chấm -> Chọn Share -> Chọn thẻ Embed.
- Copy đoạn link trong thuộc tính src="..." (đổi localhost thành 192.168.164.129) rồi dán vào file index.html của Nginx.
<img width="1913" height="987" alt="image" src="https://github.com/user-attachments/assets/f48493ef-b087-4e8c-a27c-c32f0f6a39cb" />

<img width="1915" height="987" alt="image" src="https://github.com/user-attachments/assets/b021dfee-2faa-4443-a2e6-e4d62e7d92e3" />

<img width="1919" height="1031" alt="image" src="https://github.com/user-attachments/assets/8a82f9c5-8802-48ab-b8cf-cd7db7608d98" />

## 3. Kết quả
<img width="1895" height="987" alt="Screenshot 2026-06-12 135435" src="https://github.com/user-attachments/assets/eaf63bf4-b61c-4def-98a6-a9dd32e5179f" />

## 4. Đóng gói và khôi phục hệ thống
### Bước 1: Xuất và đóng gói toàn bộ Container hiện tại thành file nén
```
# Trích xuất filesystem của từng container ra file tar độc lập
docker export nodered > nodered.tar
docker export mariadb > mariadb.tar
docker export influxdb > influxdb.tar
docker export grafana > grafana.tar
docker export flask-api > flask-api.tar
docker export nginx > nginx.tar

# Gộp và nén lại thành một file duy nhất
tar -czvf bt5_backup.tar.gz *.tar

# Dọn dẹp các file tar lẻ sau khi nén xong
rm *.tar
```

### Bước 2: Giả lập xóa sạch sẽ toàn bộ môi trường
```
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
```
<img width="1915" height="581" alt="Screenshot 2026-06-12 121833" src="https://github.com/user-attachments/assets/19a4ba75-74cc-4650-a1bd-a6d02fcece10" />

> Lúc này vào trình duyệt sẽ sập hoàn toàn, chứng minh hệ thống đã được dọn dẹp sạch
<img width="1914" height="986" alt="Screenshot 2026-06-12 121914" src="https://github.com/user-attachments/assets/6ee1966a-bd94-4883-9ae3-37bb1455085c" />

### Bước 3: Quy trình giải nén và Khôi phục (Restore)
```
# Giải nén gói backup lớn thu được ban đầu
tar -xzvf bt5_backup.tar.gz

# Import ngược các file tar thành các Image sẵn sàng hoạt động
docker import nodered.tar nodered_restore
docker import mariadb.tar mariadb_restore
docker import influxdb.tar influxdb_restore
docker import grafana.tar grafana_restore
docker import flask-api.tar flask_restore
docker import nginx.tar nginx_restore
```
<img width="1913" height="174" alt="Screenshot 2026-06-12 122027" src="https://github.com/user-attachments/assets/a086438c-f419-43bc-b8ca-7f78e31fa7ad" />

<img width="1916" height="264" alt="Screenshot 2026-06-12 123455" src="https://github.com/user-attachments/assets/0fadeaf5-9e8b-49f9-8239-94e00b5bcdfa" />

Hệ thống sẽ tự động khôi phục lại trạng thái đỉnh cao ban đầu, giữ nguyên lịch sử dữ liệu cũ trong DB mà không cần cấu hình lại từ đầu!

### Bước 4: Kết quả khôi phục
<img width="1904" height="989" alt="image" src="https://github.com/user-attachments/assets/13464bb9-81e0-4df7-9f0c-a76e4093d370" />

# Phần 3: Kết luận
Hệ thống đã đạt được các kết quả cốt lõi sau:

Kiến trúc container hóa tối ưu: Thao tác đóng gói toàn bộ các dịch vụ (Nginx, Flask API, MariaDB, InfluxDB, Node-RED, Grafana) bằng Docker Compose giúp hệ thống vận hành cô lập, ổn định, giải quyết triệt để bài toán xung đột tài nguyên cổng và dễ dàng di trú, khôi phục bằng các tệp nén sao lưu.

Trực quan hóa và Tự động hóa luồng ETL: Luồng xử lý dữ liệu của Node-RED hoạt động mượt mà, tự động phân tách dữ liệu để phục vụ lưu trữ tức thời (MariaDB) lẫn phân tích xu hướng lịch sử (InfluxDB). Giao diện Web Dashboard được thiết kế hiện đại, đồng bộ hóa dữ liệu thời gian thực và tích hợp thành công biểu đồ động từ Grafana.

Hệ thống cảnh báo thông minh: Tích hợp thành công cơ chế giám sát ngưỡng an toàn của các thông số thời tiết, tự động kích hoạt và gửi tin nhắn cảnh báo tức thời tới nhóm Telegram của các thành viên quản trị, đáp ứng trọn vẹn bài toán giám sát chủ động trong thực tế.

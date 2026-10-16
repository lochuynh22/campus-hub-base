# Báo Cáo Bài Kiểm Tra: Đóng Gói Và Chạy Campus Hub Với Docker

## 1. Thông tin sinh viên
- **Họ và tên:** Hồ Quốc Khải
- **Mã sinh viên:** *(Điền mã sinh viên của bạn tại đây)*
- **Học phần:** IT-209 DevOps Fundamentals

---

## 2. Hướng dẫn chạy hệ thống

### Bước 1: Kiểm thử và đóng gói backend bằng Gradle
```bash
# Di chuyển vào thư mục backend
cd backend

# Chạy kiểm thử Gradle
./gradlew test        # hoặc .\gradlew.bat test trên Windows

# Đóng gói file JAR
./gradlew bootJar     # hoặc .\gradlew.bat bootJar trên Windows
```

### Bước 2: Build Docker Image cho backend
```bash
# Build image với tag campus-hub-backend
docker build -t campus-hub-backend .

# Kiểm tra image đã tạo
docker images
```

### Bước 3: Khởi chạy hệ thống bằng Docker Compose (Backend + MySQL)
Từ thư mục gốc dự án:
```bash
# Kiểm tra tính hợp lệ của file cấu hình Compose
docker compose config

# Khởi chạy các container ở chế độ background
docker compose up -d

# Kiểm tra trạng thái các container
docker compose ps
```

### Bước 4: Kiểm tra trạng thái và logs
```bash
# Xem log của backend
docker compose logs -f backend

# Kiểm tra endpoint sức khỏe (Healthcheck)
curl http://localhost:8080/actuator/health
```

---

## 3. Cổng và URL kiểm tra hệ thống

- **Cổng công khai của Backend:** `8080`
- **Health check URL:** `http://localhost:8080/actuator/health`
- **API URL Categories:** `http://localhost:8080/api/categories`
- **MySQL Container:** Cổng nội bộ `3306` (được map ra cổng `3307` trên host để tránh xung đột với MySQL cài sẵn trên máy)
- **Chuỗi kết nối Database từ Backend:** `jdbc:mysql://mysql:3306/campus_hub?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true` *(sử dụng tên service `mysql` thay cho `localhost` qua biến môi trường `DB_URL`)*

---

## 4. Minh chứng kết quả thực hiện

### 4.1. Minh chứng 1: Kiểm thử backend bằng Gradle thành công
```text
$ ./gradlew.bat test
Starting a Gradle Daemon...
> Task :compileJava
> Task :processResources
> Task :classes
> Task :compileTestJava
> Task :processTestResources
> Task :testClasses
> Task :test

BUILD SUCCESSFUL in 5m 55s
5 actionable tasks: 5 executed
```

### 4.2. Minh chứng 2: Docker Image được build thành công (`docker images`)
```text
$ docker images
REPOSITORY                  TAG       IMAGE ID       CREATED         SIZE
campus-hub-backend          latest    526b412028a0   2 minutes ago   186MB
eclipse-temurin             21-jre-alpine 51ab5e3302e7 2 weeks ago   75MB
mysql                       8.0       7dcddc01f13b   2 weeks ago     400MB
```

### 4.3. Minh chứng 3: Cấu hình Docker Compose hợp lệ (`docker compose config`)
```yaml
name: campus-hub-base
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: campus-hub-backend
    depends_on:
      mysql:
        condition: service_healthy
        required: true
    environment:
      CORS_ALLOWED_ORIGINS: http://localhost:5173
      DB_PASSWORD: "123456"
      DB_URL: jdbc:mysql://mysql:3306/campus_hub?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true
      DB_USERNAME: root
      SERVER_PORT: "8080"
    image: campus-hub-backend:latest
    networks:
      campus-network: null
    ports:
      - mode: ingress
        target: 8080
        published: "8080"
        protocol: tcp
    restart: unless-stopped
  mysql:
    container_name: campus-hub-mysql
    environment:
      MYSQL_DATABASE: campus_hub
      MYSQL_PASSWORD: "123456"
      MYSQL_ROOT_PASSWORD: "123456"
    healthcheck:
      test:
        - CMD
        - mysqladmin
        - ping
        - -h
        - localhost
        - -u
        - root
        - -p123456
      timeout: 5s
      interval: 5s
      retries: 10
      start_period: 15s
    image: mysql:8.0
    networks:
      campus-network: null
    ports:
      - mode: ingress
        target: 3306
        published: "3307"
        protocol: tcp
    restart: unless-stopped
    volumes:
      - type: volume
        source: mysql_data
        target: /var/lib/mysql
        volume: {}
networks:
  campus-network:
    name: campus-hub-base_campus-network
    driver: bridge
volumes:
  mysql_data:
    name: campus-hub-base_mysql_data
    driver: local
```

### 4.4. Minh chứng 4: Hai service khởi động và hoạt động ổn định (`docker compose ps`)
```text
$ docker compose ps
NAME                 IMAGE                       COMMAND                  SERVICE   CREATED          STATUS                    PORTS
campus-hub-backend   campus-hub-backend:latest   "java -jar app.jar"      backend   Up 2 minutes     Up 2 minutes              0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
campus-hub-mysql     mysql:8.0                   "docker-entrypoint.s…"   mysql     Up 2 minutes     Up 2 minutes (healthy)    0.0.0.0:3307->3306/tcp, [::]:3307->3306/tcp
```

### 4.5. Minh chứng 5: Backend kết nối MySQL thành công (Log Flyway & HikariCP)
```text
campus-hub-backend  | 2026-10-16T01:03:40.236Z  INFO 1 --- [campus-hub] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
campus-hub-backend  | 2026-10-16T01:03:40.696Z  INFO 1 --- [campus-hub] [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@31693a86
campus-hub-backend  | 2026-10-16T01:03:40.702Z  INFO 1 --- [campus-hub] [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
campus-hub-backend  | 2026-10-16T01:03:40.811Z  INFO 1 --- [campus-hub] [           main] org.flywaydb.core.FlywayExecutor         : Database: jdbc:mysql://mysql:3306/campus_hub?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true (MySQL 8.0)
campus-hub-backend  | 2026-10-16T01:03:41.163Z  INFO 1 --- [campus-hub] [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `campus_hub` to version "1 - create schema"
campus-hub-backend  | 2026-10-16T01:03:41.770Z  INFO 1 --- [campus-hub] [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema `campus_hub`, now at version v1 (execution time 00:00.553s)
campus-hub-backend  | 2026-10-16T01:03:50.315Z  INFO 1 --- [campus-hub] [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
campus-hub-backend  | 2026-10-16T01:03:50.340Z  INFO 1 --- [campus-hub] [           main] v.e.ptit.campushub.CampusHubApplication  : Started CampusHubApplication in 16.598 seconds
```

### 4.6. Minh chứng 6: Kết quả gọi `/actuator/health` trả về HTTP 200 và UP
```text
$ curl -i http://localhost:8080/actuator/health
HTTP/1.1 200 
Content-Type: application/vnd.spring-boot.actuator.v3+json

{"status":"UP","groups":["liveness","readiness"]}
```

### 4.7. Minh chứng 7: Kiểm tra API `/api/categories` trả về dữ liệu mẫu từ MySQL
```text
$ curl http://localhost:8080/api/categories
[{"id":1,"name":"Sự kiện","slug":"su-kien"},{"id":2,"name":"Đồ thất lạc","slug":"do-that-lac"},{"id":3,"name":"Hỏi đáp học tập","slug":"hoi-dap"},{"id":4,"name":"Trao đổi đồ dùng","slug":"trao-doi"},{"id":5,"name":"Phản ánh","slug":"phan-anh"},{"id":6,"name":"Tìm nhóm","slug":"tim-nhom"}]
```

---

# Tài liệu dự án gốc Campus Hub Base

Campus Hub là dự án nền cho Mini Project Session 13. Ứng dụng cung cấp một không gian chung để sinh viên đăng sự kiện, tìm đồ thất lạc, hỏi đáp, trao đổi đồ dùng và tìm nhóm.

## Chức năng có sẵn

- Xem, tìm kiếm và lọc bài đăng.
- Đăng ký, đăng nhập bằng HTTP Basic.
- Tạo bài đăng theo chuyên mục.
- Bình luận và bày tỏ quan tâm.
- Chủ bài đăng có thể đánh dấu bài đã hoàn tất.
- Quản trị viên có quyền quản lý trạng thái bài đăng qua API.
- Health check tại `/actuator/health`.

Tài khoản mẫu được tạo khi ứng dụng khởi động lần đầu:

| Vai trò | Tên đăng nhập | Mật khẩu |
|---|---|---|
| Sinh viên | `student` | `student123` |
| Quản trị viên | `admin` | `admin123` |
"# campus-hub-base" 

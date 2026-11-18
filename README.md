# GoatSports Discovery Server

Service registry (Netflix Eureka) của GoatSports. Các microservice Spring Boot đăng ký vào đây khi khởi động,
API Gateway và các Feign client tra cứu địa chỉ service qua Eureka thay vì cấu hình cứng host/port.

Bản thân server chạy ở chế độ standalone: không tự đăng ký (`register-with-eureka=false`) và không kéo registry
từ node khác (`fetch-registry=false`).

## Công nghệ

- Java 21, Spring Boot 3.3.0, Spring Cloud 2023.0.2 (`spring-cloud-starter-netflix-eureka-server`)
- Build image bằng Jib, đẩy lên Docker Hub `minhdat1312/goatsports-eureka-server`

## Cấu hình

`application.properties` chỉ đọc biến môi trường, không có giá trị mặc định:

| Biến | Ý nghĩa | Ví dụ (Docker Compose) |
|---|---|---|
| `SERVER_PORT` | Cổng HTTP của server | `8761` |
| `EUREKA_INSTANCE_HOSTNAME` | Hostname server tự quảng bá | `eureka-server` |

Các service còn lại trỏ về server qua `EUREKA_SERVICE_URL`, ví dụ `http://eureka-server:8761/eureka/`.

Actuator chỉ mở `health` và `info`; probe `/actuator/health/readiness` trả `200` khi UP, `503` khi DOWN.

## Chạy local

Tạo `src/main/resources/application-dev.properties` (file này bị `.gitignore`), tối thiểu:

```properties
server.port=8761
eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

Rồi chạy:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

Dashboard: http://localhost:8761. Health: http://localhost:8761/actuator/health

## Chạy bằng Docker Compose

Service `eureka-server` nằm trong `infrastructure/compose/docker-compose-infra.yml` (giới hạn 256 MB RAM,
healthcheck theo readiness probe). Khởi động server này trước các microservice khác.

## Kiểm thử

```bash
./mvnw test
```

Test dùng profile `test` (`src/test/resources/application-test.properties`, cổng ngẫu nhiên) và chỉ kiểm tra
Spring context khởi động được.

## CI/CD

`.gitlab-ci.yml` chạy trên nhánh mặc định:

1. **test**: `mvn -B test`
2. **build**: `mvn compile jib:build`, đẩy image lên Docker Hub (cần biến `DOCKER_USERNAME`, `DOCKER_PASSWORD`)
3. **deploy**: gọi Jenkins job `GoatSports-CD` với `SERVICE_NAME=eureka-server` để khởi động lại container trên VPS
   (cần `JENKINS_URL`, `JENKINS_USER`, `JENKINS_API_TOKEN`)
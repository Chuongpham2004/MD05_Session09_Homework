# Bài tập 5: Tối ưu hoá CI/CD với Multi-stage Build (Gradle)

## 1. Bối cảnh và nguyên nhân

Docker image của Notification Service nặng khoảng **850 MB**, trong khi code thực tế chỉ vài KB. Nguyên nhân là Dockerfile đang dùng mô hình **Single-stage (Fat Container)**: build và chạy ứng dụng trong **cùng một image**. Vì vậy image chạy thực tế vẫn mang theo mọi thứ chỉ cần cho lúc build:

| Thành phần thừa trong image chạy | Vì sao thừa |
|---|---|
| **JDK** (trình biên dịch `javac`, công cụ phát triển) | Lúc chạy chỉ cần **JRE** |
| **Gradle** cài sẵn (`/opt/gradle`) và Gradle Wrapper tải về | Chỉ dùng để build |
| **Cache Gradle** (`~/.gradle`, chứa toàn bộ dependencies đã tải) | Chỉ dùng để build |
| **Source code** (`src/`, `build.gradle`, ...) | Chỉ cần file `.jar` đã đóng gói |
| Base image Ubuntu đầy đủ | Alpine nhỏ hơn rất nhiều |

## 2. Giải pháp: Multi-stage build

Chia Dockerfile thành **2 stage**. Image cuối cùng **chỉ lấy kết quả** của stage build:

```
┌──────────── Stage 1: builder ────────────┐      ┌──────── Stage 2: runtime ────────┐
│ gradle:7.6-jdk17                         │      │ eclipse-temurin:17-jre-alpine    │
│  - JDK 17 + Gradle                       │      │  - Chỉ JRE 17                    │
│  - Copy source code                      │ jar  │  - app.jar                       │
│  - chmod +x gradlew                      │ ───► │  - User không phải root          │
│  - ./gradlew clean build -x test         │      │  - ENTRYPOINT java -jar app.jar  │
│ ➜ BỊ BỎ ĐI, không có trong image cuối    │      │ ➜ IMAGE ĐƯỢC PUSH / DEPLOY        │
└──────────────────────────────────────────┘      └──────────────────────────────────┘
```

## 3. Bảng so sánh

| Tiêu chí | Single-stage (Fat Container) | Multi-stage build |
|---|---|---|
| **Base image khi chạy** | `gradle:7.6-jdk17` (Ubuntu + JDK + Gradle) | `eclipse-temurin:17-jre-alpine` (Alpine + JRE) |
| **Nội dung image cuối** | JDK, Gradle, cache Gradle, toàn bộ source code, thư mục `build/` | **Chỉ** JRE và `app.jar` |
| **Dung lượng dự kiến** | 700–900 MB | 150–200 MB |
| **Dung lượng đo thực tế** (project này) | **762 MB** (nén khi push: ~540 MB) | **188 MB** (nén khi push: ~73 MB) |
| **Bảo mật mã nguồn** | ❌ Thư mục `src/` và `build.gradle` nằm trong image. Ai pull được image là đọc được source code | ✅ Không có `src/`. Image chỉ chứa bytecode trong file `.jar` |
| **Bề mặt tấn công** | ❌ Lớn: có `javac`, `gradle`, nhiều package của Ubuntu, nhiều lỗ hổng (CVE) tiềm ẩn | ✅ Nhỏ: Alpine tối giản, không có công cụ build |
| **User chạy ứng dụng** | ❌ `root` (mặc định) | ✅ User thường `spring` |
| **Tốc độ push/pull, deploy** | ❌ Chậm, tốn băng thông và dung lượng registry | ✅ Nhanh hơn khoảng 7 lần (73 MB so với 540 MB) |
| **Độ phức tạp Dockerfile** | ✅ Đơn giản, 1 khối lệnh | ⚠️ Dài hơn một chút (2 stage) |
| **Môi trường build** | ✅ Build ngay trong Docker, không cần cài JDK trên máy | ✅ Giống vậy: stage 1 lo việc build |
| **Tận dụng cache** | ❌ `COPY . .` nên sửa 1 dòng code là tải lại dependencies | ✅ Copy file Gradle trước, tách layer dependencies, chỉ build lại phần code |

**Kết luận:** Multi-stage build **giảm dung lượng khoảng 75%** (762 MB → 188 MB), nằm trong khoảng 150–200 MB mà đề bài đặt ra. Image cũng không còn chứa source code, công cụ build và quyền root.

## 4. Kết quả đo thực tế

Môi trường đo: Docker 29.9.3, build trên chính project này (Spring Boot 4.1.2, file jar 8,7 MB).

| Image | Dung lượng filesystem (`du -sx /`) | Dung lượng nén khi push lên registry |
|---|---|---|
| `payment-service:single-stage` | 762 MB | ~540 MB |
| `payment-service:multi-stage` | **188 MB** | **~73 MB** |
| *(tham chiếu)* `eclipse-temurin:17-jre-alpine` | 180 MB | ~66 MB |

Kiểm tra nội dung bên trong từng image:

| Kiểm tra | Single-stage | Multi-stage |
|---|---|---|
| Có thư mục `src/`? | Có (`/home/gradle/project/src`) | **Không** (`ls: cannot access '/app/src'`) |
| Có `javac`, `gradle`? | Có (`/opt/java/openjdk/bin/javac`, `/usr/bin/gradle`) | **Không** |
| Cache Gradle | 216 MB (`/home/gradle/.gradle`) | Không có |
| Gradle cài sẵn | 137 MB (`/opt/gradle`) | Không có |
| User chạy | `root` | `spring` |
| App khởi động được? | Có | Có: `Started Md05Session09HomeworkApplication in 1.382 seconds` |

Image multi-stage chỉ lớn hơn base JRE khoảng 8 MB, gần bằng đúng dung lượng file `app.jar`.

## 5. Dockerfile Multi-stage (file `bai5/Dockerfile`)

```dockerfile
# ---------- STAGE 1: BUILDER ----------
FROM gradle:7.6-jdk17 AS builder
WORKDIR /home/gradle/project

COPY --chown=gradle:gradle gradlew settings.gradle build.gradle ./
COPY --chown=gradle:gradle gradle ./gradle
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon > /dev/null

COPY --chown=gradle:gradle src ./src
RUN ./gradlew clean build -x test --no-daemon

# ---------- STAGE 2: RUNTIME ----------
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring

COPY --from=builder --chown=spring:spring /home/gradle/project/build/libs/payment-service-1.0.0.jar app.jar

USER spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Giải thích các điểm chính:**

| Dòng | Ý nghĩa |
|---|---|
| `FROM gradle:7.6-jdk17 AS builder` | Đặt tên `builder` cho stage 1 để stage 2 tham chiếu |
| `COPY` file Gradle trước, rồi `./gradlew dependencies` | Tách layer dependencies: chỉ sửa code thì Docker dùng lại cache, không tải lại thư viện |
| `chmod +x gradlew` | Đảm bảo Gradle Wrapper có quyền thực thi trên Linux |
| `./gradlew clean build -x test` | Build ra jar, bỏ qua test (stage `test` trong CI đã chạy) |
| `FROM eclipse-temurin:17-jre-alpine` | Bắt đầu image mới tinh. Mọi thứ của stage 1 bị bỏ lại |
| `COPY --from=builder ... payment-service-1.0.0.jar app.jar` | Chỉ copy **đúng 1 file jar**, không lấy `*-plain.jar` và không lấy `src/` |
| `adduser` + `USER spring` | Không chạy ứng dụng bằng root |

File `bai5/Dockerfile.dockerignore` loại `.git`, `.gradle`, `build`, `.idea` khỏi build context để build nhanh hơn và không lọt file thừa vào stage builder.

## 6. Cách build và kiểm chứng

Chạy từ **thư mục gốc** project:

```bash
# Build image multi-stage
docker build -f bai5/Dockerfile -t payment-service:multi-stage .

# (Đối chứng) build image single-stage
docker build -f bai5/Dockerfile.single-stage -t payment-service:single-stage .

# So sánh dung lượng filesystem
docker run --rm --entrypoint sh payment-service:single-stage -c 'du -sxm /'
docker run --rm --entrypoint sh payment-service:multi-stage  -c 'du -sxm /'

# Kiểm tra không còn source code
docker run --rm --entrypoint sh payment-service:multi-stage -c 'ls /app'

# Chạy ứng dụng
docker run --rm payment-service:multi-stage
```

## 7. Áp dụng vào pipeline GitLab CI

Với multi-stage, việc build jar diễn ra **bên trong** `docker build`. Job dockerize không cần lấy artifact jar từ stage build nữa:

```yaml
docker_job:
  stage: dockerize
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
  script:
    - docker build -f bai5/Dockerfile -t $DOCKER_USERNAME/payment-service:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_USERNAME/payment-service:$CI_COMMIT_SHORT_SHA
```

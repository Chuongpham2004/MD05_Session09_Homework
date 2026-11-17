# Bài tập 1: Khắc phục sự cố Pipeline CI/CD cơ bản

## 1. File `.gitlab-ci.yml` ban đầu (bị lỗi)

```yaml
stages:
  build_app

build_job:
  stage: build_app
  script:
    - ./gradlew clean build -x test
```

## 2. Phân tích các lỗi

### Lỗi 1: Sai cú pháp khai báo mảng `stages`

GitLab CI yêu cầu `stages` là một **mảng (list)** YAML. Trong file gốc:

```yaml
stages:
  build_app
```

YAML hiểu `build_app` là một **chuỗi (string)** đơn, không phải phần tử của mảng, vì thiếu dấu gạch đầu dòng `-`.
GitLab sẽ báo lỗi kiểu `stages config should be an array of strings` và pipeline không được tạo.

**Cách sửa:**

```yaml
stages:
  - build_app
```

(hoặc viết dạng inline: `stages: [build_app]`)

### Lỗi 2: Thiếu `image` chứa môi trường Java/Gradle

File gốc không khai báo `image`, nên job chạy trên image mặc định của GitLab Runner
(thường là `ruby`/`alpine`/`ubuntu` tùy cấu hình runner). Các image này **không có JDK**,
trong khi `./gradlew` cần Java để khởi động Gradle Wrapper. Kết quả là lỗi:

```
java: command not found
ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.
```

Dự án dùng Spring Boot với `toolchain` Java **17** (`build.gradle`), vì vậy cần image có JDK 17.

**Cách sửa:**

```yaml
image: eclipse-temurin:17-jdk
```

> Có thể dùng `gradle:8-jdk17` thay thế, nhưng vì dự án đã có Gradle Wrapper (`gradlew`)
> nên chỉ cần image có JDK 17 là đủ; Wrapper sẽ tự tải đúng phiên bản Gradle của dự án.

### Lỗi 3 (bổ sung): `gradlew` không có quyền thực thi

File `gradlew` trong repository được lưu với mode `100644` (không có quyền execute) do commit từ Windows.
Khi runner Linux chạy `./gradlew` sẽ gặp lỗi:

```
/bin/bash: line 1: ./gradlew: Permission denied
```

**Cách sửa:** thêm `chmod +x ./gradlew` vào `before_script` (và/hoặc set quyền trong Git bằng
`git update-index --chmod=+x gradlew`).

## 3. File `.gitlab-ci.yml` sau khi sửa

```yaml
image: eclipse-temurin:17-jdk

stages:
  - build_app

variables:
  GRADLE_USER_HOME: "$CI_PROJECT_DIR/.gradle"
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .gradle/wrapper
    - .gradle/caches

before_script:
  - chmod +x ./gradlew
  - java -version

build_job:
  stage: build_app
  script:
    - ./gradlew clean build -x test
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 week
```

## 4. Giải thích các keyword chính

| Keyword | Vai trò |
|---|---|
| `image` | Docker image dùng để chạy job — cung cấp môi trường JDK 17 |
| `stages` | Danh sách (mảng) các giai đoạn của pipeline, chạy theo thứ tự |
| `stage` | Gán job vào một stage đã khai báo trong `stages` |
| `before_script` | Lệnh chạy trước `script` của mọi job (cấp quyền cho `gradlew`) |
| `script` | Lệnh chính của job: `./gradlew clean build -x test` (build, bỏ qua test) |
| `variables` | Biến môi trường; `GRADLE_USER_HOME` để cache Gradle trong thư mục project |
| `cache` | Lưu lại dependency Gradle giữa các lần chạy để build nhanh hơn |
| `artifacts` | Lưu file `.jar` sau khi build để tải về / dùng cho bước deploy |

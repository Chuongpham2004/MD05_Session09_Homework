# =====================================================================
# Multi-stage Dockerfile cho Spring Boot + Gradle
# Build (chạy từ thư mục gốc project):
#   docker build -f bai5/Dockerfile -t payment-service:multi-stage .
# =====================================================================

# ---------- STAGE 1: BUILDER ----------
# Môi trường build đầy đủ: JDK 17 + Gradle. Stage này KHÔNG có mặt trong image cuối.
FROM gradle:7.6-jdk17 AS builder

WORKDIR /home/gradle/project

# Copy file cấu hình Gradle trước để tận dụng cache layer:
# chỉ khi các file này đổi thì mới tải lại dependencies.
COPY --chown=gradle:gradle gradlew settings.gradle build.gradle ./
COPY --chown=gradle:gradle gradle ./gradle
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon > /dev/null

# Copy source code và build ra file .jar (bỏ qua test, đã chạy ở stage test của CI)
COPY --chown=gradle:gradle src ./src
RUN ./gradlew clean build -x test --no-daemon

# ---------- STAGE 2: RUNTIME ----------
# Chỉ chứa JRE 17 trên Alpine: không có JDK, Gradle, source code.
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Chạy bằng user thường thay vì root để tăng bảo mật
RUN addgroup -S spring && adduser -S spring -G spring

# Chỉ copy DUY NHẤT file .jar từ stage builder sang
# (không lấy file *-plain.jar và không lấy thư mục src)
COPY --from=builder --chown=spring:spring /home/gradle/project/build/libs/payment-service-1.0.0.jar app.jar

USER spring

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]

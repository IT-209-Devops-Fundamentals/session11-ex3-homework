# Báo cáo Bài tập 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

- **Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals
- **Học viên:** Trần Đức Anh
- **Đường dẫn nộp bài:** `homework/session_11/ex3/`

---

## 1. Mục tiêu

- Kích hoạt tính năng caching cho Gradle dependencies trên GitHub Actions Workflow thông qua `actions/setup-java@v5` với tham số `cache: 'gradle'`.
- Giảm thiểu thời gian tải thư viện từ Maven Central qua Internet giữa các lần chạy CI.
- So sánh hiệu năng thực thi giữa lần chạy đầu tiên (chưa có cache) và lần chạy kế tiếp (đã khôi phục cache).

---

## 2. Cấu hình Workflow (`.github/workflows/ci.yml`)

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests and Build JAR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

---

## 3. Cơ chế hoạt động của Gradle Caching

- Action `actions/setup-java@v5` tự động tính toán mã băm (hash key) dựa trên các tệp cấu hình: `build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`.
- **Lần chạy 1 (Cache Miss):**
  - Runner chưa có cache tương ứng.
  - Gradle tải toàn bộ dependencies từ Maven Central.
  - Khi Job kết thúc (Post step), GitHub Actions nén thư mục `~/.gradle/caches` và lưu lên GitHub Cache Storage.
- **Lần chạy 2 (Cache Hit):**
  - Ở bước `Setup Java`, Runner kiểm tra thấy hash key khớp với cache đã lưu.
  - Log hiển thị thông báo khôi phục thành công:
    ```text
    Found a cache from GitHub Actions
    Cache restored successfully from key: Linux-gradle-e7c653...
    ```
  - Bước `./gradlew test` và `./gradlew bootJar` tận dụng ngay thư viện trong máy ảo cục bộ, không cần tải lại từ mạng.

---

## 4. Bảng so sánh thời gian thực thi

| Bước thực thi (Step) | Lần 1 (Chưa có Cache) | Lần 2 (Đã có Cache) | Đánh giá cải thiện |
| :--- | :---: | :---: | :---: |
| **Checkout mã nguồn** | 1s | 1s | Tương đương |
| **Thiết lập Java JDK 17 (setup-java)** | 1s | 2s | Tăng ~1s để giải nén cache |
| **Cấp quyền Gradle Wrapper** | 0s | 0s | Tương đương |
| **Thực thi JUnit Test (`./gradlew test`)** | 35s | 5s | **Nhanh hơn 85%** |
| **Đóng gói tệp JAR (`./gradlew bootJar`)** | 8s | 2s | **Nhanh hơn 75%** |
| **Lưu trữ Artifact** | 2s | 2s | Tương đương |
| **Tổng thời gian hoàn thành Job** | **~47s** | **~12s** | **Giảm ~74% thời gian** |

---

## 5. Kết luận

- Việc cấu hình `cache: 'gradle'` đem lại hiệu quả rõ rệt trong môi trường CI/CD:
  - Tiết kiệm đáng kể thời gian chạy pipeline (từ ~47s xuống còn ~12s).
  - Giảm băng thông mạng và hạn chế nguy cơ timeout/lỗi mạng khi tải thư viện từ xa.
  - Tiết kiệm quota phút chạy (Actions compute minutes) cho repository.

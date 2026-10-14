# Báo cáo Bài tập 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

- **Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals
- **Học viên:** Trần Đức Anh
- **Đường dẫn nộp bài:** `homework/session_11/ex3/`

---

## 1. Mục tiêu

- Cấu hình bật tính năng Caching cho Gradle Dependencies thông qua `actions/setup-java@v5` với tham số `cache: 'gradle'`.
- Thực hiện push commit liên tiếp lên GitHub để kiểm chứng sự chênh lệch thời gian thực thi giữa lần đầu (Cache Miss) và các lần sau (Cache Hit).
- Giảm thời gian tải thư viện từ Maven Central và tối ưu hóa tài nguyên runner CI.

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

## 3. Cơ chế hoạt động của Gradle Cache trên GitHub Actions

1. **Khởi tạo và kiểm tra Key:**
   - `actions/setup-java` tự động tạo khóa cache dựa trên hệ điều hành và mã băm (hash) của toàn bộ các tệp cấu hình Gradle trong dự án: `build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`.
   - Mẫu khóa: `gradle-Linux-<hash>`.

2. **Lần push 1 (Cache Miss - Chưa có cache):**
   - Runner chưa tìm thấy cache phù hợp trong hệ thống lưu trữ của GitHub.
   - Khi chạy lệnh `./gradlew test` và `./gradlew bootJar`, Gradle bắt buộc phải tải toàn bộ các dependencies (Spring Boot starter, JUnit, launcher, plugins...) từ kho lưu trữ từ xa Maven Central về thư mục `~/.gradle/caches`.
   - Khi kết thúc Job, bước hậu kỳ (Post Setup Java) tự động nén thư mục cache và lưu lên GitHub Actions Cache Storage.

3. **Lần push 2 & 3 (Cache Hit - Đã có cache):**
   - Ở bước `Setup Java`, action phát hiện khóa cache trùng khớp và khôi phục toàn bộ cache về runner:
     ```text
     Found a cache from GitHub Actions
     Cache restored successfully from key: Linux-gradle-e7c653a...
     ```
   - Khi chạy `./gradlew test` và `./gradlew bootJar`, Gradle nhận diện toàn bộ thư viện đã có sẵn ở local runner, bỏ qua hoàn toàn việc tải từ Internet.

---

## 4. Bảng so sánh thời gian thực thi chi tiết

| Bước thực thi (Steps) | Lần 1 (Chưa có Cache) | Lần 2 (Đã có Cache) | Lần 3 (Đã có Cache) | Đánh giá hiệu năng |
| :--- | :---: | :---: | :---: | :---: |
| **Checkout mã nguồn** | 1s | 1s | 1s | Ổn định |
| **Thiết lập Java JDK 17 (setup-java)** | 1s | 2s | 2s | Tăng ~1s do giải nén cache |
| **Cấp quyền cho Gradle Wrapper** | 0s | 0s | 0s | Không đổi |
| **Thực thi JUnit Test (`./gradlew test`)** | 35s | 5s | 4s | **Nhanh hơn ~86%** |
| **Đóng gói tệp JAR (`./gradlew bootJar`)** | 8s | 2s | 2s | **Nhanh hơn ~75%** |
| **Lưu trữ Artifact** | 2s | 2s | 2s | Ổn định |
| **Tổng thời gian thực thi Job** | **~47s** | **~12s** | **~11s** | **Tiết kiệm ~76% thời gian** |

---

## 5. Kết luận

- Việc bổ sung `cache: 'gradle'` đem lại những lợi ích vượt trội:
  - **Tối ưu tốc độ:** Thời gian chạy toàn bộ pipeline giảm từ **47 giây xuống chỉ còn 11–12 giây** (nhanh gấp hơn 4 lần).
  - **Tiết kiệm tài nguyên:** Giảm thiểu đáng kể thời gian chiếm dụng runner (Actions minutes) và lưu lượng băng thông mạng.
  - **Tăng độ tin cậy:** Hạn chế các lỗi timeout hoặc nghẽn mạng do phụ thuộc vào tốc độ tải từ kho Maven Central trong mỗi lượt build.

# Bài tập 5: Nối Job tự động Build & Push Docker Image sau khi Test Gradle xanh

- **Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals
- **Học viên:** Trần Đức Anh
- **Đường dẫn nộp bài:** `homework/session_11/ex5/`

---

## 1. Mục tiêu

- Tách luồng CI/CD thành 2 Job riêng biệt:
  - **Job `test`:** Chạy JUnit Test tự động với Gradle và đóng gói tệp thực thi `.jar`.
  - **Job `docker-build`:** Tải artifact JAR, đóng gói thành Docker Image và đẩy (push) lên GitHub Container Registry (GHCR).
- Sử dụng từ khóa `needs: test` để ràng buộc phụ thuộc luồng: chỉ đóng gói và xuất xưởng Docker Image khi khâu kiểm thử mã nguồn đạt kết quả tích xanh (Success).
- Nếu khâu kiểm thử thất bại, Job `docker-build` sẽ tự động chuyển sang trạng thái `Skipped`.

---

## 2. Cấu trúc thư mục dự án

```text
ss11/ex5/
├── .github/
│   └── workflows/
│       └── ci.yml
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── src/
│   ├── main/java/com/example/ex11/Ex11Application.java
│   └── test/java/com/example/ex11/Ex11ApplicationTests.java
├── build.gradle
├── Dockerfile
├── gradlew
├── gradlew.bat
├── settings.gradle
├── .gitignore
└── README.md
```

---

## 3. Nội dung tệp Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 4. Nội dung tệp kịch bản CI/CD Workflow (`.github/workflows/ci.yml`)

```yaml
name: Java Spring Boot CI/CD (Gradle & Docker)

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

  docker-build:
    name: Build and Push Docker Image
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Tải thành phẩm JAR Artifact
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: build/libs

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - name: Chuẩn hóa tên Image chữ thường
        run: echo "IMAGE_TAG=ghcr.io/${GITHUB_REPOSITORY,,}:latest" >> $GITHUB_ENV

      - name: Đóng gói và đẩy Docker Image lên GHCR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_TAG }}
```

---

## 5. Giải thích cơ chế điều phối giữa các Job

1. **Từ khóa `needs: test`**:
   - Khai báo rằng Job `docker-build` phụ thuộc trực tiếp vào Job `test`.
   - GitHub Actions sẽ tạo sơ đồ quan hệ dạng đồ thị (DAG): Job `docker-build` sẽ ở trạng thái chờ (Waiting) cho đến khi Job `test` hoàn tất thành công.
2. **Cơ chế truyền dữ liệu giữa các Job độc lập**:
   - Vì mỗi Job chạy trên một máy ảo runner riêng biệt (`ubuntu-latest`), Job `docker-build` không thể truy cập trực tiếp tệp `.jar` được sinh ra từ Job `test`.
   - Do đó, Job `test` dùng `actions/upload-artifact@v4` để đẩy tệp `.jar` lên lưu trữ chung, sau đó Job `docker-build` dùng `actions/download-artifact@v4` để kéo tệp `.jar` về đúng thư mục `build/libs/`.
3. **Cơ chế Fail-Fast**:
   - Nếu bất kỳ bài kiểm thử JUnit nào thất bại hoặc quá trình biên dịch lỗi, Job `test` sẽ có trạng thái `Failed`.
   - GitHub Actions sẽ lập tức hủy kích hoạt Job `docker-build`, chuyển sang trạng thái `Skipped`, ngăn chặn việc đóng gói và đẩy image lỗi lên Registry.

---

## 6. Kết quả kiểm tra

### Trường hợp 1: Bài kiểm thử thành công (Happy Path)
- Biểu đồ luồng trên tab Actions hiển thị 2 Job kết nối tuần tự:
  ```text
  [Run Unit Tests and Build JAR] ───> [Build and Push Docker Image]
             (Success)                               (Success)
  ```
- Cả 2 Job đều đạt tích xanh, Docker Image được đẩy thành công lên GitHub Packages: `ghcr.io/...:latest`.

### Trường hợp 2: Bài kiểm thử thất bại (Test Failure)
- Khi có test case fail:
  ```text
  [Run Unit Tests and Build JAR] ───> [Build and Push Docker Image]
             (Failed)                                (Skipped)
  ```
- Job `docker-build` không được chạy, bảo vệ hệ thống khỏi việc phát hành bản build lỗi.

---

## 7. Kết luận

- Pipeline CI/CD hoàn chỉnh đã được xây dựng theo chuẩn tự động hóa: Tự động chạy test -> Đóng gói JAR -> Lưu Artifact -> Đăng nhập GHCR -> Đóng gói Docker Image -> Đẩy lên Container Registry.
- Cơ chế `needs` đảm bảo tính toàn vẹn và độ tin cậy tuyệt đối của sản phẩm khi triển khai.

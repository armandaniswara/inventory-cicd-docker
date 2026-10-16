# Inventory App — Laporan Pembangunan Sistem Stok Barang Toko

Laporan ini merangkum seluruh perkembangan proyek dari kommit awal hingga versi terbaru.

---

## 1. Komit Awal (52e0dc8 — Wed Sep 23 2026)

Proyek dimulai dengan struktur lengkap untuk manajemen produk, kategori, dan riwayat pergerakan stok:

- **27 file dibuat** dengan total 1004 insertions
- **Core Entities**: `Category`, `Product`, `StockMovement`, `MovementType`
- **REST Controllers**: `CategoryController`, `ProductController`, `StockController`
- **DTOs**: `CategoryRequest`, `ProductRequest`, `ProductResponse`, `StockMovementRequest`, `StockMovementResponse`
- **Repositories**: `CategoryRepository`, `ProductRepository`, `StockMovementRepository` (Spring Data JPA)
- **Services**: `CategoryService`, `ProductService`, `StockService` (logika bisnis termasuk transaksi stok)
- **Exceptions**: `GlobalExceptionHandler`, `InsufficientStockException`, `ResourceNotFoundException`
- **Configuration**: `application.yml`, `.gitignore`
- **Docker**: `docker-compose.yml` (PostgreSQL siap pakai)

---

## 2. CI/CD Pipeline (e767037 — Wed Sep 23 2026)

Menambahkan otomasi pengembangan dan deployed:

- `.github/workflows/ci-cd.yml` — GitHub Actions pipeline
- `Dockerfile` — Konfigurasi Docker untuk build aplikasi
- `deployment/deployment.yml` — Konfigurasi produksi (compose app + postgres)
- `.dockerignore` — Penomoran file untuk Docker build

---

## 3. Perbaikan Maven & Lombok (5776cd8 — Wed Sep 23 2026)

- Menambahkan **Maven Wrapper** (`mvnw`, `mvnw.cmd`)
- Mengupdate Lombok ke versi **1.18.38**
- Konfigurasi annotation processor di `pom.xml`

---

## 4. Konfigurasi Deployment (9b504c1, 24d90f2, 54c0f22, 8fabf4f)

- **SSH Port**: Diatur ke port **20257** untuk deployment
- **SCP Transfer**: Diganti dari GHCR pull ke SCP transfer untuk gambar Docker
- **GitHub Actions workflow**: Diperbaiki dan diperbarui untuk alur CI/CD produksi

---

## 5. Struktur Proyek Saat Ini

```
inventory-app/
├── docker-compose.yml           # PostgreSQL siap pakai
├── deployment/deployment.yml    # Konfigurasi produksi (compose app + postgres)
├── docker-compose.yml           # PostgreSQL siap pakai
├── pom.xml
├── Dockerfile
├── .dockerignore
├── .gitignore
└── src/main/java/com/example/inventory/
    ├── InventoryApplication.java
    ├── entity/          # Category, Product, StockMovement, MovementType
    ├── repository/      # Spring Data JPA repositories
    ├── service/         # Logika bisnis (termasuk transaksi stok)
    ├── controller/      # REST endpoints (Category, Product, Stock)
    ├── dto/             # Request/Response objects
    └── exception/       # Custom exception + global handler
```

---

## 6. Cara Menjalankan

### 1. Jalankan PostgreSQL lewat Docker

```bash
docker compose up -d
```

Membuat database `inventory_db` dengan user `inventory_user` / password `inventory_pass` di port 5432.

### 2. Jalankan aplikasi

`src/main/resources/application.yml` tidak di-commit (lihat `.gitignore`). Sumber config ada di `deployment/config/application-dev.yml`. Untuk development lokal, salin dulu:

```bash
cp deployment/config/application-dev.yml src/main/resources/application.yml
```

```bash
./mvnw spring-boot:run
```

Aplikasi jalan di `http://localhost:8080`.

Dokumentasi API otomatis (Swagger UI): `http://localhost:8080/swagger-ui.html`

Tabel akan dibuat otomatis oleh Hibernate (`ddl-auto: update`) saat pertama kali start.

---

## 7. Contoh Penggunaan API

### Buat kategori

```http
POST /api/categories
Content-Type: application/json

{ "name": "Minuman", "description": "Produk minuman kemasan" }
```

### Buat produk

```http
POST /api/products
Content-Type: application/json

{
  "name": "Teh Botol 450ml",
  "sku": "TB-450",
  "price": 5000,
  "stock": 100,
  "minStock": 10,
  "categoryId": 1
}
```

### Catat barang masuk (restock)

```http
POST /api/stock-movements
Content-Type: application/json

{ "productId": 1, "type": "IN", "quantity": 50, "note": "Restock dari supplier A" }
```

### Catat barang keluar (terjual)

```http
POST /api/stock-movements
Content-Type: application/json

{ "productId": 1, "type": "OUT", "quantity": 20, "note": "Penjualan harian" }
```

Jika `quantity` OUT melebihi stok yang tersedia, API akan menolak dengan status `400` dan pesan error yang jelas.

### Cek produk dengan stok menipis

```http
GET /api/products/low-stock
```

### Lihat riwayat pergerakan stok suatu produk

```http
GET /api/stock-movements/product/1
```

---

## 9. Ide Pengembangan Lanjutan

- Tambah Spring Security (login admin/staff, role-based access)
- Tambah endpoint laporan (total nilai stok, produk paling laris per periode)
- Tambah pagination & sorting pada `GET /api/products`
- Tambah unit test untuk `StockService` (kasus stok cukup vs tidak cukup)
- Ganti `ddl-auto: update` dengan Flyway/Liquibase untuk migrasi schema yang lebih terkontrol saat production
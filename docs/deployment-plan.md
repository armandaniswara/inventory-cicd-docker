# Rencana Deployment Inventory App via Docker + GitHub Actions

Tujuan: aplikasi Spring Boot berjalan di Docker container, dideploy ke server sendiri
lewat SSH + docker compose, dan otomasi berjalan di GitHub Actions.

Semantik trigger (sesuai pilihan):

- **Pull Request ke `main`** → jalankan CI: build + test. Tidak deploy.
- **PR di-merge ke `main`** (yaitu `push` ke `main`) → build image, push ke GHCR, lalu deploy ke server.

---

## 1. Kondisi Saat Ini

Yang sudah ada:

- `Dockerfile` — multi-stage (Maven build → JRE alpine), sudah jalan sebagai user non-root.
- `docker-compose.yml` — PostgreSQL untuk development.
- `deployment/deployment.yml` — compose produksi (app + PostgreSQL + healthcheck), dipakai workflow.
- `deployment/config/application-dev.yml` — config Spring eksternal, di-mount ke container.
- `deployment/.env.example` — contoh variabel untuk menjalankan compose manual.
- `.github/workflows/ci-cd.yml` — pipeline CI/CD.
- `.dockerignore`, Maven Wrapper.

### Temuan yang harus diperbaiki sebelum deploy berjalan

| # | Masalah | Status |
|---|---------|--------|
| 1 | `server.port` 5000 vs Docker/`EXPOSE` 8080 | Selesai → 8080 |
| 2 | `buildkit: true` bukan key valid di GitHub Actions | Selesai → `docker/build-push-action` |
| 3 | `scp-action` dipakai dengan input `script:` | Selesai → `scp-action` (kirim file) + `ssh-action` (jalankan) |
| 4 | Tag image dobel owner | Selesai → `ghcr.io/${{ github.repository }}` |
| 5 | Image compose tidak cocok tag GHCR | Selesai → `ghcr.io/armandaniswara/inventory-cicd-docker:latest` |
| 6 | Tidak ada `permissions: packages: write` | Selesai → ditambahkan |
| 7 | `ddl-auto: validate` tanpa Flyway | Selesai → `update` (utang teknis) |
| 8 | `deployment/*` kosong | Selesai → `deployment.yml` + `.env.example` |

---

## 2. Keputusan Desain

1. **Port dalam container = 8080**, seragam di app, Dockerfile, dan compose. Di server, host port digeser ke **8085** (`8085:8080`) karena 8080 dipakai Jenkins.
2. **Deploy target = server sendiri.** Docker + docker compose sudah terpasang di server. Server `docker login ghcr.io` sekali, lalu `docker compose pull && up -d`.
3. **Image di GHCR.** Nama image: `ghcr.io/armandaniswara/inventory-cicd-docker`. Tag `latest` dan tag commit SHA (`${{ github.sha }}`) untuk rollback.
4. **PR = verifikasi saja.** Deploy hanya dari `push` ke `main`.
5. **Rahasia di GitHub Secrets**, tidak ada kredensial di repo.

---

## 3. Langkah Implementasi

### Fase 1 — Perbaiki aplikasi agar konsisten

- [ ] `application.yml`: `server.port: 8080`. Kalau ingin tetap configurable, biarkan `SPRING_SERVER_PORT`/`SERVER_PORT` dioverride env.
- [ ] Pastikan `spring.jpa.hibernate.ddl-auto` = `update` untuk sementara (validate baru aman setelah ada Flyway). Catat sebagai utang teknis.
- [ ] (Opsional) Hapus `spring-boot-devtools` dari image production, atau pastikan tidak ikut aktif.
- [x] Isi `deployment/deployment.yml` & `deployment/config/application-dev.yml` (compose + config eksternal).

Kriteria selesai: `docker build` sukses, `curl localhost:8080` mengembalikan respons app.

### Fase 2 — Rapikan Docker

- [ ] `Dockerfile`: pastikan `EXPOSE 8080` selaras `server.port`. Pertahankan multi-stage & non-root user.
- [x] `deployment/deployment.yml`:
  - Image `ghcr.io/armandaniswara/inventory-app:latest` (bisa dioverride `APP_IMAGE`).
  - `ports: "8085:8080"` (host 8085 → container 8080; 8080 di server dipakai Jenkins).
  - `healthcheck` untuk postgres + `depends_on: condition: service_healthy`.
  - `SPRING_JPA_HIBERNATE_DDL_AUTO: update`.
  - Kredensial dari env (`DB_PASSWORD`), bukan hardcode.
  - Mount `./config` → `/app/config` untuk config Spring eksternal.
- [x] Tambah `deployment/.env.example`.
- [x] `docker-compose.prod.yml` dihapus (digantikan `deployment/deployment.yml` agar satu sumber).

Kriteria selesai: `docker compose -f deployment/deployment.yml up -d` di server lokal jalan dan app bisa diakses di `:8085`.

### Fase 3 — Perbaiki GitHub Actions

`ci-cd.yml` punya dua job:

**Job `build`** — jalan untuk `pull_request` dan `push` ke `main`:
- [x] Checkout, setup JDK 17 (temurin), cache Maven.
- [x] `./mvnw clean verify` (build + test sekaligus).
- [x] `docker/setup-buildx-action` + `docker/build-push-action@v6`.
- [x] Login GHCR + push image berkondisi `push: ${{ github.event_name == 'push' }}` (di PR hanya build, tanpa push).
  - tags: `ghcr.io/${{ github.repository }}:latest` dan `:<sha>`.

**Job `deploy`** — hanya `push` ke `main` (`needs: build`):
- [x] `permissions: { contents: read, packages: write }`.
- [x] `appleboy/scp-action` mengirim `deployment/deployment.yml` + `deployment/config/application-dev.yml` ke `/opt/inventory-app` (`strip_components: 1`).
- [x] `appleboy/ssh-action` menjalankan:
  ```
  cd /opt/inventory-app
  echo $GHCR_TOKEN | docker login ghcr.io -u <user> --password-stdin
  docker compose -f deployment.yml pull
  docker compose -f deployment.yml up -d
  docker image prune -f
  ```
  Variabel `DB_PASSWORD` dan `APP_IMAGE` dikirim lewat env action.
- [ ] Rollback: deploy ulang tag `:<sha>` sebelumnya bila gagal.

Kriteria selesai: PR gagal kalau test gagal; merge ke `main` menghasilkan image di GHCR dan container ter-update di server.

### Fase 4 — Siapkan server

- [x] Docker Engine + Compose plugin: **sudah terinstall di server (akses via SSH)**. Cukup pastikan `ssh <user>@<host> docker compose version` berhasil.
- [ ] Buat direktori `/opt/inventory-app` (workflow akan mengisi `deployment.yml` + `config/`).
- [ ] `docker login ghcr.io` dengan token yang punya scope `read:packages`.
- [ ] Buka port 22 (atau SSH port khusus, repo menyebut **20257**) dan port **8085** di firewall.
- [ ] Siapkan PostgreSQL (pakai service `postgres` di compose, atau DB eksternal). Backup volume `inventory_pg_data`.

### Fase 5 — Secrets & konfigurasi

Secrets GitHub yang dibutuhkan:

| Secret | Keterangan |
|--------|------------|
| `DEPLOY_HOST` | IP/domain server |
| `DEPLOY_USER` | user SSH (idealnya bukan root) |
| `DEPLOY_KEY` | private key SSH |
| `DEPLOY_PORT` | port SSH (20257) |
| `GHCR_TOKEN` | token untuk `docker login` di server |
| `DB_PASSWORD` | password PostgreSQL |

Nilai non-rahasia (path project di server, user GHCR) taruh di `env:` workflow.

---

## 4. Verifikasi End-to-End

1. Buka PR berisi perubahan kecil → cek job `ci` hijau/merah sesuai test.
2. Merge PR ke `main` → cek job `deploy` push image dan update container.
3. Di server: `docker ps`, `docker logs inventory-app`, akses `http://<host>:8085/swagger-ui.html`.
4. Cek `/api/products` mengembalikan data, dan data PostgreSQL tetap ada setelah redeploy.

---

## 5. Rollback

- `docker compose pull` ke tag SHA sebelumnya lalu `up -d`.
- Simpan minimal 2 tag terakhir di GHCR; jangan hapus langsung.
- Backup volume PostgreSQL sebelum migrasi schema besar.

---

## 6. Risiko & Catatan

- `ddl-auto: update` di production berisiko (perubahan schema tak terkontrol). Pindah ke Flyway/Liquibase setelah schema stabil.
- Deploy lewat SSH butuh key aman & user non-root; batasi pemakaian `sudo`.
- Deploy otomatis tiap merge = tidak ada approval gate. Tambahkan environment protection di GitHub kalau perlu.

---

## 7. Urutan Pengerjaan

1. Fase 1 (port & config) — wajib, tanpa ini container tidak bisa diakses.
2. Fase 2 (Docker) — wajib untuk compose.
3. Fase 3 (workflow) — inti permintaan.
4. Fase 4 & 5 (server & secrets) — prasyarat deploy nyata.
5. Fase 6 diverifikasi lewat Fase 4 di Verifikasi.

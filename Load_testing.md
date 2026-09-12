 
# DOCUMENTATION & PORTFOLIO: API PERFORMANCE & LOAD TESTING REPORT

**Project Target:** API Service Authentication & User Management (`[https://belajar-bareng.onrender.com](https://belajar-bareng.onrender.com)`)

**QA / Performance Engineer:** Dicky Ade Mahendra

**Tools Used:** Apache JMeter 5.6.3 (Non-GUI/CLI Mode), PowerShell, HTML Dashboard Generator

**Date:** September 2026

---

## 1. Executive Summary

Laporan pengujian beban (Load Testing) ini disusun untuk mengevaluasi ketahanan, latensi, dan batas kapasitas maksimum (*breaking point*) pada layanan API Authentication & User Management. Pengujian dirancang menggunakan pola pembagian trafik **Read-Heavy** (30% Login vs 70% Get Users) untuk memprediksi performa sistem saat menerima lonjakan beban pengguna di lingkungan produksi.

---

## 2. Test Scenario & Workload Distribution

Pengujian mensimulasikan alur transaksi pengguna dari proses autentikasi hingga pembacaan data profil menggunakan fitur *Throughput Controller* dan *JSON Extractor* di JMeter.

| Endpoint API | HTTP Method | Alokasi Trafik (%) | Mekanisme Teknis & Assertion |
| --- | --- | --- | --- |
| `/api/login` | POST | 30.0% | Mengirim kredensial JSON, memvalidasi HTTP 200 OK, dan mengekstrak `token` ke variabel `${bearer_token}` via JSON Extractor. |
| `/api/users` | GET | 70.0% | Mengirim permintaan data profil menggunakan header `Authorization: Bearer ${bearer_token}` dan memvalidasi HTTP 200 OK. |

### Konfigurasi Pengujian Beban:

* **Stress Test Skenario (Peak Load):** 5.000 Virtual Users, Ramp-up Time 1 detik.
* **Normal Load Test Skenario (Baseline):** 100 Virtual Users, Ramp-up Time 10 detik.

---

## 3. Key Findings & Performance Metrics

Berikut adalah tabel perbandingan hasil pengujian dari dua skenario beban yang dieksekusi:

| Metric Pengujian | Stress Test (5.000 Users) | Normal Load Test (100 Users) | SLA / Standard Target |
| --- | --- | --- | --- |
| **Total Samples / Requests** | 5.000 samples | 100 samples | - |
| **Error Rate (%)** | **84.22%** *(Failed)* | **0.00%** *(Passed)*<br> | < 1.00% |
| **Average Response Time** | 16.330 ms (16,3s) | 753,4 ms (0,75s)

 | < 2.000 ms |
| **Max Response Time** | 55.606 ms (55,6s) | 4.778 ms (4,7s)

 | < 5.000 ms |
| **Throughput (RPS)** | 79,2 req/sec | 138,8 req/sec | Target System Capacity |

### Analisis Temuan Bottleneck:

1. **Degradasi Performa Ekstrem pada Stress Test:** Pada beban 5.000 user simultan dengan ramp-up 1 detik, server mengalami *overload* parah dengan tingkat kegagalan 84.22% (Error HTTP 500/504 Timeout) akibat kemacetan antrean koneksi (*connection queue*).
2. **Stabilitas Pengujian Baseline:** Pada beban 100 user dengan ramp-up 10 detik, sistem terbukti sangat stabil (0.00% Error Rate) dengan rata-rata latensi di bawah 1 detik.


3. **Spike Latensi Inisialisasi:** Ditemukan puncak latensi maksimum (4,7s) pada transaksi login awal akibat proses *SSL Handshake* dan *cold-start* server saat pertama kali menerima trafik.



---

## 4. CLI Execution Log & Documentation Setup

### Perintah Eksekusi Non-GUI (PowerShell / CMD):

```powershell
jmeter -n -t "C:\Users\ASUS\Documents\apache-jmeter-5.6.3\apache-jmeter-5.6.3\backups\LoginAndGetUserr-000005.jmx" -l "C:\Users\ASUS\results.jtl"

```

### Perintah Generate HTML Dashboard Report:

```powershell
Remove-Item -Path "C:\Users\ASUS\Documents\apache-jmeter-5.6.3\apache-jmeter-5.6.3\html-report\*" -Recurse -Force
jmeter -g "C:\Users\ASUS\results.jtl" -o "C:\Users\ASUS\Documents\apache-jmeter-5.6.3\apache-jmeter-5.6.3\html-report"

```

---

## 5. Technical Recommendations for Development Team

* **Penerapan Rate Limiting / Throttling:** Memasang pembatasan kuota request pada endpoint `POST /api/login` untuk mencegah server *down* total saat diserang lonjakan trafik mendadak.
* **Implementasi Caching (Redis):** Menerapkan mekanisme *caching* pada endpoint `GET /api/users` guna mengurangi beban kueri langsung ke basis data pada trafik pembacaan tinggi.
* **Penyesuaian Autoscaling:** Konfigurasi ulang aturan *Horizontal Pod Autoscaler* (HPA) agar penambahan pod/instance server dipemicu lebih awal sebelum kapasitas CPU/Memori mencapai 80%.
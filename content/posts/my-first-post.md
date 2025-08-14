---
title: "Grafana Alloy Introduction"
date: 2024-08-14T17:45:00+07:00
draft: false
---

Selamat datang di pengenalan Grafana Alloy! Jika Anda berkecimpung di dunia observability, Anda mungkin sudah familiar dengan tantangan mengelola berbagai *agent* untuk mengumpulkan metrik, log, dan *trace*. Seringkali, kita harus menjalankan Prometheus agent untuk metrik, Promtail untuk log, dan OpenTelemetry Collector untuk *trace*.

Grafana Alloy hadir sebagai solusi untuk menyederhanakan semua ini.

### Apa itu Grafana Alloy?

Grafana Alloy adalah pipeline telemetri open-source dari Grafana yang dirancang untuk menjadi satu-satunya *agent* yang Anda perlukan. Dibangun di atas OpenTelemetry (OTel) Collector, Alloy menggabungkan fungsionalitas dari berbagai *agent* populer ke dalam satu biner yang terpadu.

### Fitur Utama

-   **Satu Agent untuk Semua**: Mengumpulkan metrik (Prometheus), log (Loki), *trace* (Tempo/OTel), dan profil (Pyroscope) dengan satu aplikasi.
-   **Konfigurasi Terpadu**: Menggunakan bahasa konfigurasi yang kuat dan fleksibel bernama "River" yang terinspirasi dari Terraform. Ini memungkinkan Anda mendefinisikan alur kerja telemetri yang kompleks dengan mudah.
-   **Kompatibilitas Luas**: Kompatibel dengan ekosistem Prometheus, Loki, Tempo, dan standar OpenTelemetry.
-   **Vendor-Neutral**: Meskipun dikembangkan oleh Grafana, Alloy bersifat *vendor-neutral* dan dapat mengirim data ke berbagai *backend* observability.

Di postingan selanjutnya, kita akan mencoba melakukan instalasi dan konfigurasi dasar Grafana Alloy untuk mulai mengumpulkan metrik dari sebuah aplikasi.
# *Dockerization Standard Operating Procedure*

Daftar isi:
- [Untuk Tim Pengembang](#tim-pengembang)
- [Untuk Tim Infrastruktur](#tim-infrastruktur)
---
## Tim Pengembang

Jika pengembang membutuhkan bantuan tim infrastruktur untuk melakukan dokerisasi aplikasi, maka pengembang:
1. **Memberikan keterangan mengenai arsitektur perangkat lunak** yang dikembangkan
2. **Memberikan daftar aplikasi** yang dibutuhkan untuk menjalankan perangkat lunak
3. **Memberikan langkah-langkah** untuk menjalankan program dengan asumsi program akan dijalankan di sebuah mesin kosong

Jika pengembang melakukan dokerisasi secara mandiri, maka pengembang perlu:
1. Meminta akses *portainer* pada tim infrastruktur
---
## Tim Infrastruktur

Berdasarkan keterangan yang diberikan pengembang, tim infrastruktur perlu:
1. **Membuat *image*** yang sesuai
2. **Menyiapkan *container***
3. **Menyiapkan akses** untuk pengembang
4. **Menyiapkan *domain***
5. **Melakukan tes**
6. **Memberikan akses** ke pengembang
7. **Memberikan pendampingan** ( *CI/CD*, *auto-deploy*)

Jika dokerisasi sudah *self-service*, maka tim infrastruktur perlu:
1. Memastikan *environment docker* siap pakai dengan kriteria sebagai berikut:
    - *Registry* cukup
    - Pengembang memiliki akses
    - Ter-*backup* dengan baik
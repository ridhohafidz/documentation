Sharing Dokerisasi

-> Intro: Container vs Visual Machine (VM)
	- Penggunaan VM selama ini belum maksimal (jika dialokasikan 1 GB, maka meskipun tidak ada aktivitas, 1 GB resource tidak bisa dialokasikan untuk hal lain) dan memungkinkan adanya konflik versi dependensi jika ada lebih 1 aplikasi yang diinstall dalam VM tersebut
	- Tiap aplikasi terisolasi dalam tiap container sehingga tidak ada masalah versi dependensi aplikasi yang berbeda, serta penggunaannya efisien (jika tidak ada aktivitas pada container tersebut, maka resource dapat dialokasikan untuk hal lain)

-> Definisi
	- Image: snapshot isi sistem
	- Dockerfile: "Resep" untuk membuat dan menyajikan *image*
	- .dockerignore: File berisi daftar file yang tidak perlu disalin ke dalam *image*

-> Membuat Dockerfile
	- Susun dockerfile dengan mindset bahwa source code akan dijalankan pada OS kosong
	-> Sintaks Dockerfile
		- FROM - image sebagai landasan
		- WORKDIR
		- COPY
		- RUN
		- EXPOSE
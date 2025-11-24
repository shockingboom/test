
# Dokumentasi Hall — Penjelasan sederhana untuk orang awam

Dokumentasi ini menjelaskan bagaimana fitur utama aplikasi bekerja, ditulis dengan bahasa sederhana agar mudah dimengerti.

Garis besar yang penting:

- Pengunjung scan QR di meja.
- QR membuka halaman menu di ponsel pengunjung.
- Pengunjung memilih menu dan menambahkan ke keranjang (di browser mereka).
- Saat selesai, pengunjung melakukan checkout. Pesanan dikirim ke server.
- Server menyimpan pesanan dan memberi tahu admin lewat WhatsApp.
- Admin melihat pesanan di dashboard, memprosesnya, dan saat pembayaran dikonfirmasi server akan mengirim invoice ke customer lewat WhatsApp.

---

## Peran berkas-berkas utama (singkat dan jelas)

- `resources/views/customer/menu.blade.php` — halaman menu yang dilihat customer. Di sini terdapat JavaScript untuk menyimpan keranjang di browser dan mengirim pesanan.
- `app/Http/Controllers/OrderController.php` — bagian server yang menerima pesanan dari customer dan menyimpannya.
- `app/Models/Pesanan.php` — model yang menyimpan data pesanan ke database.
- `app/Services/WhatsappService.php` — bagian yang mengirim pesan WhatsApp ke admin atau customer.
- `app/Http/Controllers/Admin/PesananController.php` — halaman admin untuk melihat pesanan dan mengubah status (diproses / selesai).
- `app/Http/Controllers/TableController.php` dan `resources/views/admin/table/*` — untuk membuat meja, melihat QR, dan mengganti token QR.

---

## Alur sederhana langkah demi langkah (untuk orang awam)

1) Scan QR dan buka menu

- Pengunjung memindai QR yang ada di meja.
- Browser membuka halaman menu. Server mengirim daftar menu dan informasi meja.

2) Pilih menu dan tambahkan ke keranjang

- Pengunjung klik "Tambah ke Keranjang".
- Data keranjang disimpan di browser (localStorage). Tidak langsung ke server.

3) Checkout (kirim pesanan ke server)

- Pengunjung klik Checkout, mengisi nama (opsional), nomor WhatsApp, dan metode pembayaran.
- Browser mengirimkan data pesanan ke server.

4) Server menyimpan pesanan dan memberi tahu admin

- Server memeriksa data yang dikirim (validasi).
- Jika valid, server menyimpan pesanan (model `Pesanan`) dan menghitung total harga.
- Server merangkai pesan dan memanggil `WhatsappService` untuk mengirim pemberitahuan ke admin.

5) Admin memproses pesanan

- Admin membuka dashboard pesanan dan mengubah status pesanan saat membayar atau menyelesaikan pesanan.
- Jika admin mengubah status dari "menunggu pembayaran" ke "diproses", server akan mengirim invoice ke WhatsApp customer.

---

## Diagram pemanggilan singkat (teks)

customer browser (menu page)
  -> simpan keranjang di localStorage
  -> kirim POST ke `/order/store`
    -> `OrderController@store` (server)
      -> simpan `Pesanan` ke database
      -> `WhatsappService->sendMessageToAdmin` (kirim notifikasi)

admin dashboard
  -> lihat daftar pesanan
  -> ubah status pesanan
    -> kalau konfirmasi pembayaran -> `WhatsappService->sendMessageToCustomer` (kirim invoice)

---

## Rute penting (yang perlu diketahui)

- Buka menu: `GET /` (menjalankan `OrderController@index`)
- Kirim pesanan: `POST /order/store` (menjalankan `OrderController@store`)
- Halaman admin pesanan: `GET /admin/pesanan` (menjalankan `Admin\PesananController@index`)
- Ubah status pesanan: `PATCH /admin/pesanan/{pesanan}/status` (menjalankan `Admin\PesananController@updateStatus`)
- Lihat QR meja: `GET /admin/table/{table}/qrcode` (menjalankan `TableController@showQrCode`)

---

## Hal-hal teknis yang perlu diperhatikan (disederhanakan)

- Pastikan nomer WhatsApp customer diisi dengan benar — server akan menambahkan kode negara jika perlu.
- Pastikan pengaturan WhatsApp API ada di file lingkungan (env): `WHATSAPP_API_KEY`, `WHATSAPP_API_URL`, `ADMIN_NUMBER`.
- Jika ada perubahan pada nama tabel database (mis. produk), sesuaikan validasi di server.

---

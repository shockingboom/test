# CALL_GRAPH — Diagram pemanggilan lengkap (ASCII) dan penjelasan

---

## Legend singkat
- [view] : Blade view (halaman yang dilihat user)
- (JS)   : JavaScript di view (client-side)
- [route]: endpoint (URL)
- (ctrl) : Controller method (server-side)
- {model}: Model Eloquent (akses DB)
- <svc>  : Service (WhatsappService)

---

## Diagram ASCII (tingkat 1 — high level)

```text
[customer phone]
    |
    | scans QR
    v
[view] customer.menu (resources/views/customer/menu.blade.php)
    |-- (JS) save cart -> localStorage
    |-- (JS) checkout -> POST /order/store
    v
[route] POST /order/store
    -> (ctrl) OrderController@store
         -> validate data
         -> {model} Pesanan::create(items JSON, total, ...)
         -> <svc> WhatsappService->sendMessageToAdmin(message)
             -> external WhatsApp API

Admin flow:
[admin browser]
    |
    v
[route] GET /admin/pesanan
    -> (ctrl) Admin\PesananController@index -> [view] admin.pesanan.index
    -> admin change status -> PATCH /admin/pesanan/{pesanan}/status
        -> (ctrl) Admin\PesananController@updateStatus
            -> update {model} Pesanan
            -> if (waiting_payment -> processing) -> <svc> WhatsappService->sendMessageToCustomer(phone, message)
```

---

## Diagram ASCII (tingkat 2 — lengkap dengan file pemanggil)

```text
+--------------------------------------------------------------------------------+
|                             Customer (browser)                                 |
+--------------------------------------------------------------------------------+
| [resources/views/customer/menu.blade.php]                                      |
|   - JS: addToCart(), viewCart(), checkout()                                    |
|   - localStorage key: cart_{TOKEN}                                             |
|   - Checkout => POST /order/store (JSON body: token, items, customer_phone...) |
+--------------------------------------------------------------------------------+
                 | POST /order/store (fetch)
                 v
+--------------------------------------------------------------------------------+
|                             Server (HTTP)                                      |
+--------------------------------------------------------------------------------+
| Route: POST /order/store -> app/Http/Controllers/OrderController.php@store     |
|   - Validates request payload                                                   |
|   - Table::where('token', $token) -> verify table                              |
|   - calculate total from items                                                 |
|   - Pesanan::create([...])  (saves to DB, items as JSON)                        |
|   - Calls: app/Services/WhatsappService::sendMessageToAdmin(message)            |
+--------------------------------------------------------------------------------+
                 |
                 v
+--------------------------------------------------------------------------------+
|                         External: WhatsApp API (via Http::post)                |
+--------------------------------------------------------------------------------+
```


---

## Penjelasan per node (file yang memanggil dan dipanggil)

1) `resources/views/customer/menu.blade.php` (view)
   - Fungsi: tampilkan daftar menu, sediakan tombol "Tambah ke Keranjang" dan tombol "Buka Keranjang".
   - JS penting:
     - `addToCart(id, name, price)` : menambah item ke localStorage
     - `checkout()` : menampilkan modal detail customer lalu POST ke `/order/store`
   - Tidak langsung menyimpan ke server sampai checkout.

2) Route `/order/store` -> `app/Http/Controllers/OrderController@store`
   - Fungsi: menerima request order dari client.
   - Proses:
     - Validasi payload (token, items, phone, payment_method)
     - Cari meja berdasarkan token
     - Hitung total
     - Simpan Pesanan
     - Panggil `WhatsappService::sendMessageToAdmin`
   - Hasil: JSON success / error

3) `app/Services/WhatsappService.php` (service)
   - Fungsi: mengirim pesan ke admin/customer.
   - Menggunakan `Http::post(WHATSAPP_API_URL/api/send-message)` dan header `x-api-key`.
   - Methods: `sendMessageToAdmin($message)`, `sendMessageToCustomer($phone, $message)`.

4) Admin flow: `Admin\PesananController`
   - `index()` -> menampilkan daftar pesanan (view `resources/views/admin/pesanan/index.blade.php`).
   - `updateStatus(Request $request, Pesanan $pesanan)`:
     - Update status pesanan
     - Jika status berubah dari `waiting_payment` -> `processing` : panggil `sendInvoiceToCustomer($pesanan)` yang kemudian memanggil `WhatsappService->sendMessageToCustomer`

5) `app/Http/Controllers/TableController.php` dan views admin/table
   - Fungsi: CRUD meja, generate & download QR code
   - `showQrCode(Table $table)` mengenerate QR yang berisi URL `/?t={token}`

---

## Mapping singkat route->file (ulang singkat)

- GET `/` -> `OrderController@index` -> view `customer.menu`
- POST `/order/store` -> `OrderController@store`
- GET `/order/history` -> `OrderController@history` -> view `customer.history`
- GET `/admin/pesanan` -> `Admin\PesananController@index`
- PATCH `/admin/pesanan/{pesanan}/status` -> `Admin\PesananController@updateStatus`
- Resource `admin/table` -> `TableController` (CRUD + QR)

---

## Catatan & tips untuk presentasi singkat
- Tunjukkan demo live: scan QR -> tambah ke keranjang -> checkout -> cek admin dashboard.
- Tekankan: keranjang disimpan di browser (localStorage) sampai checkout.
- Jelaskan: WhatsappService hanya memanggil API pihak ketiga — konfigurasinya di .env.

---

File dibuat: `CALL_GRAPH.md`.
Saya akan cek ulang jika mau ditambah diagram Mermaid atau contoh pesan WA. Jika cocok, saya tandai tugas selesai.

---

## Files for presentation (mudah dicari)

Berikut file yang sering dipakai untuk membuat slide atau demo. Buka path ini jika ingin screenshot atau menjelaskan kode:

- `resources/views/customer/menu.blade.php` — halaman menu customer (JS: cart & checkout).
- `resources/views/admin/pesanan/index.blade.php` — dashboard admin untuk melihat dan mengubah status pesanan.
- `app/Http/Controllers/OrderController.php` — method `index`, `store`, `history` (lihat alur simpan pesanan).
- `app/Http/Controllers/Admin/PesananController.php` — method `index`, `updateStatus` (kirim invoice ke customer).
- `app/Services/WhatsappService.php` — fungsi `sendMessageToAdmin` & `sendMessageToCustomer` (template WA).
- `app/Http/Controllers/TableController.php` — lihat `showQrCode`, `downloadQrCode`, `regenerateToken`.
- `resources/views/admin/table/qrcode.blade.php` — halaman tampil/unduh/print QR code.
- `routes/web.php` — sumber mapping route yang dipakai di demo.

---

Jika mau, saya bisa juga buat folder `presentation_assets/` dengan screenshot dan teks presentasi otomatis.

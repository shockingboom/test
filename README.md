# Presentasi: Penjelasan kode untuk presentasi

Dokumen ini memberikan penjelasan baris-fungsional dan talking points untuk presentasi. Target:

- `resources/views/admin/table/*`
- `resources/views/admin/pesanan/*`
- `app/Http/Controllers/OrderController.php`
- `app/Http/Controllers/TableController.php`

## Daftar file yang dijelaskan

1. `resources/views/admin/table/index.blade.php` — daftar meja + aksi CRUD
2. `resources/views/admin/table/create.blade.php` — form tambah meja
3. `resources/views/admin/table/edit.blade.php` — form edit meja
4. `resources/views/admin/table/qrcode.blade.php` — lihat/download/print QR Code
5. `resources/views/admin/pesanan/index.blade.php` — dashboard pesanan & kontrol status
6. `app/Http/Controllers/OrderController.php` — endpoint customer: tampil menu, simpan order, history, WA
7. `app/Http/Controllers/TableController.php` — CRUD meja dan QR code generation

---

## Penjelasan kode per-file (detail untuk presentasi)

Catatan: tiap bagian di bawah menjelaskan potongan penting dan "talking points" yang bisa dibacakan saat presentasi.

### `resources/views/admin/table/index.blade.php`

- Purpose: menampilkan daftar `tables` yang dikirim dari `TableController@index`.
- Struktur penting:
- `@forelse ($tables as $table)` -> loop menampilkan setiap baris meja.
- `route('admin.table.edit', $table->id')` -> URL untuk edit (menunjukkan RESTful routing).
- Form delete dan regenerate token menggunakan `@csrf` dan `@method('DELETE'|'PATCH')`.
- Menampilkan `{{ $table->token }}` di dalam `<code>` untuk memudahkan copy.

Talking points:
- Tekankan penggunaan CSRF dan method spoofing untuk aksi non-GET.
- Jelaskan bagaimana controller mengembalikan `tables` (Eloquent) dan blade merendernya.

### `resources/views/admin/table/create.blade.php`

- Purpose: form untuk menambahkan meja baru.
- Struktur penting:
- `<form action="{{ route('admin.table.store') }}" method="POST">` dan `@csrf`.
- Field `nomer_meja` dengan `old('nomer_meja')` untuk memulihkan input saat validasi gagal.

Talking points:
- Validasi dilakukan di controller; pada form kita menampilkan error dengan `@error('nomer_meja')`.
- Setelah submit, `TableController@store` membuat token secara otomatis.

### `resources/views/admin/table/edit.blade.php`

- Purpose: edit nomor meja.
- Struktur penting:
- Method form `@method('PUT')` dan route `admin.table.update`.
- Value pre-filled: `value="{{ old('nomer_meja', $table->nomer_meja) }}"`.

Talking points:
- Menjelaskan strategi update: unique validation yang mengecualikan id saat edit.

### `resources/views/admin/table/qrcode.blade.php`

- Purpose: menampilkan QR code SVG (dihasilakn di controller) dan UI download/print.
- Struktur penting:
- `{!! $qrCode !!}` -> menyuntikkan HTML/SVG langsung (pastikan aman karena di-generate internal).
- Button `route('admin.table.qrcode.download', $table->id)` untuk mengunduh (controller mengatur header attachment).
- `form action="{{ route('admin.table.regenerate', $table->id) }}" method="POST"` dengan `@method('PATCH')` untuk regenerate token.
- `printQRCode()` JS: membuka window baru, menuliskan HTML, lalu panggil `window.print()` — jelaskan alasan membuka window baru agar styling print terkontrol.

Talking points:
- Jelaskan keamanan: token yang di-generate harus unik dan regen membuat QR lama tidak berlaku.
- Tampilkan demo: klik download, lalu scan hasilnya untuk buka `/?t=TOKEN`.

### `resources/views/admin/pesanan/index.blade.php`

- Purpose: dashboard admin untuk melihat dan mengubah status pesanan.
- Struktur penting:
- Statistik ringkasan: `{{ $pesanan->where('status','...')->count() }}` menampilkan jumlah per status.
- Tabel pesanan memakai `@forelse($pesanan as $order)`; tiap baris menyertakan `data-status` untuk filter client-side.
- Daftar items ditampilkan di dalam `<details><summary>` dan `@foreach($order->items as $item)`.
- Form untuk mengubah status: `<form method="POST"> @csrf @method('PATCH') <select name="status"> ...</select>` dan JS `handleStatusChange` untuk submit otomatis dengan konfirmasi.

Talking points:
- Tekankan UX: konfirmasi khusus saat berpindah dari `waiting_payment` ke `processing`.
- Jelaskan kenapa invoice ke customer dikirim setelah admin konfirmasi pembayaran (untuk menghindari spam/resiko).

### `app/Http/Controllers/OrderController.php` (detail kode)

1) `__construct(WhatsappService $whatsappService)`
   - Dependency injection service WA; menunjukkan desain terpisah untuk notifikasi.

2) `index(Request $request)`
   - Mengambil query param `t` (token meja): `$token = $request->query('t');`
   - Validasi token: `Table::where('token', $token)->first()` dan jika tidak ada kembalikan view `customer.menu` dengan `error`.
   - Ambil `Item::all()` dan kirim ke view.

   Talking point: alur aman — jika token invalid customer melihat pesan yang ramah.

3) `store(Request $request)`
   - Validasi payload dengan aturan ketat termasuk `items.*.id` dan `items.*.quantity`.
   - Cari `Table` dari token; jika tidak valid kembalikan JSON 400.
   - Hitung total: loop `foreach ($validated['items'] as $item)` -> `$total += $item['price'] * $item['quantity'];`.
   - Simpan `Pesanan::create([...])` dengan status `'waiting_payment'`.
   - Panggil `$this->sendWhatsAppNotification($order, $table)` — notifikasi admin.
   - Response: JSON success atau error dengan kode HTTP.

   Talking points:
   - Tunjukkan validasi sebagai lapisan proteksi pada client-supplied data.
   - Jelaskan mengapa server tidak langsung kirim invoice ke customer (admin harus konfirmasi pembayaran dulu).

4) `history(Request $request)`
   - Mirip `index` tetapi mengambil `Pesanan::where('table_id', $table->id)->orderBy('created_at','desc')->get()`.

5) `sendWhatsAppNotification($order, $table)` dan `sendInvoiceToCustomer($order, $table)`
   - Keduanya membuat string message yang diformat rapi (loop item -> subtotal) lalu memanggil `WhatsappService`.
   - Error handling: `try/catch` dan `Log::error` — tidak menggagalkan penyimpanan order.

   Talking points:
   - Tekankan pemisahan tanggung jawab: controller mem-format pesan, service yang bertanggung jawab mengirim.

Catatan teknis:

- Validasi `items.*.id` menggunakan `exists:products,id` — pastikan nama tabel/kolom sesuai migration.

### `app/Http/Controllers/TableController.php` (detail kode)

1) CRUD standar (`index`, `create`, `store`, `edit`, `update`, `destroy`)
   - `store` validasi `nomer_meja` dan memanggil `Table::create(['nomer_meja' => ..., 'token' => Table::generateToken()])`.
   - `update` menggunakan rule unique yang mengecualikan id saat edit: `'unique:tables,nomer_meja,' . $table->id`.

2) Token & QR
   - `generateQrCode(Table $table)` dan `showQrCode(Table $table)` membuat URL `url('/?t=' . $table->token)`.
   - Menggunakan `QrCode::size(...)->margin(...)->generate($url)` untuk membuat SVG.
   - `downloadQrCode` menambahkan header `Content-Disposition: attachment` agar browser mengunduh file.

Talking points:
  - Jelaskan flow generate token -> QR -> customer scan.
  - Tekankan header download untuk UX dan alasan menggunakan SVG dengan ukuran berbeda untuk print/scan.

---

## Alur data (sederhana) — slide diagram text
1. Admin membuat meja (`TableController@store`) -> token di-generate.
2. Admin lihat QR (`TableController@showQrCode`) -> QR berisi `/?t=TOKEN`.
3. Customer scan QR -> membuka `/?t=TOKEN` -> `OrderController@index` menampilkan menu.
4. Customer pesan -> kirim POST ke `OrderController@store` dengan token & items.
5. Server menyimpan `Pesanan` status `waiting_payment` dan mengirim notifikasi WA ke admin.
6. Admin melihat dashboard `admin/pesanan` -> ubah status (processing/completed) via patch.
7. Saat admin konfirmasi pembayaran -> (opsional) sistem kirim invoice ke customer via WA (`sendInvoiceToCustomer`).

## Catatan teknis & pertanyaan untuk presentasi
- Pastikan model `Table::generateToken()` ada.
- Cek migration `create_table.php` dan `create_order.php` untuk nama kolom `products` vs `items`.
- `OrderController@store` validasi menggunakan `exists:products,id` — periksa apakah tabel produk bernama `products`.
- `WhatsappService` ter-inject; jelaskan konfigurasi / env var yang diperlukan (API key / nomor admin).

## Pointers untuk slide presentasi (3-4 slide)
- Slide 1: Tujuan & overview fitur (manage meja + QR + order flow)
- Slide 2: Tampil & CRUD Meja (screenshot `index`, `create`, `edit`)
- Slide 3: QR Code flow (generate, show, download, scan -> menu)
- Slide 4: Dashboard Pesanan & notifikasi WA (tabel, change status, send invoice)

---

## Appendix: Files quick summary
- resources/views/admin/table: index, create, edit, qrcode (UI untuk meja & QR)
- resources/views/admin/pesanan: index (dashboard & kontrol status)
- Controllers: OrderController (customer endpoints + WA), TableController (admin meja + QR)


---

Jika perlu, saya bisa:
- Buat versi presentasi slide (Markdown + gambar placeholder) atau
- Tambah contoh skrip presentasi per-slide (narasi singkat per slide)

Tandai permintaan tambahan yang Anda mau supaya saya lanjutkan.

# Cara mencopot Midtrans (template Laravel)

> Template ini memuat integrasi Midtrans (mode mock + jalur Snap asli).
> Ikuti berurutan — jangan loncat. Estimasi: ±20 menit (termasuk bedah file shared).
> Konvensi Scaffdev: file 100% milik Midtrans DIHAPUS utuh; file shared DIBEDAH
> barisnya (jangan hapus file shared).

## B.1. Hapus file milik Midtrans (aman — tidak dipakai kode lain)

- `app/Services/MidtransService.php` — service mock-kompatibel + jalur Snap.
- `config/midtrans.php` — konfigurasi `MIDTRANS_*`.
- `app/Http/Controllers/PaymentController.php` — webhook notification + simulasi mock.
- `resources/views/checkout/pay.blade.php` — halaman bayar mock.

```bash
rm "app/Services/MidtransService.php" "config/midtrans.php" \
  "app/Http/Controllers/PaymentController.php" "resources/views/checkout/pay.blade.php"
```

## B.2. Bedah file shared (JANGAN hapus file-nya)

### 1. `app/Http/Controllers/CheckoutController.php`

- Hapus import: `use App\Services\MidtransService;` (baris ~7).
- Ubah signature `store()`: hapus injeksi `MidtransService $midtrans` (baris ~29).
- Hapus pemanggilan `$midtrans->createTransaction($order->fresh());` (baris ~87).
- Ubah redirect `checkout.pay` (baris ~89) → ke halaman orders, mis.
  `redirect()->route('orders.show', $order->order_number)`.
- Hapus method `pay()` seluruhnya (menampilkan view `checkout.pay` yang sudah dihapus).
- NOTED: tanpa Midtrans, order berhenti di status `pending` — ganti dengan alur
  pembayaran manual (transfer/COD + konfirmasi admin) sesuai kebutuhanmu.

### 2. `routes/web.php`

- Hapus import `PaymentController` (baris 13).
- Hapus route webhook (baris 38-39): `POST /payment/notification`.
- Hapus route halaman bayar (baris 50): `GET /checkout/pay/{orderNumber}`.
- Hapus route simulasi (baris 52): `POST /checkout/pay/{orderNumber}/simulate`.
- Biarkan route checkout (`index`/`store`) dan orders apa adanya.

### 3. Kolom database (pilih salah satu)

- Project BELUM production (migrasi boleh diedit): hapus 3 kolom di
  `database/migrations/2026_09_19_000005_create_orders_table.php`
  (`payment_status`, `payment_reference`, `snap_token`), lalu `php artisan migrate:fresh`.
- Project SUDAH production: buat migrasi baru penghapus kolom, JANGAN edit
  migrasi lama:
  ```bash
  php artisan make:migration drop_midtrans_columns_from_orders --table=orders
  ```
  Isi `up()` dengan `$table->dropColumn(['payment_status', 'payment_reference', 'snap_token']);`.

### 4. `app/Models/Order.php`

- Hapus `'payment_status', 'payment_reference', 'snap_token'` dari `$fillable`
  (baris ~13). Biarkan `order_status` bila masih dipakai alur manualmu.

### 5. `.env` + `.env.example`

- Hapus blok `MIDTRANS_MODE/MIDTRANS_SERVER_KEY/MIDTRANS_CLIENT_KEY/MIDTRANS_IS_PRODUCTION/MIDTRANS_SNAP_URL`.
  Hapus barisnya, jangan dikosongkan saja.

### 6. `resources/views/orders/show.blade.php`

- Hapus tombol "Bayar Sekarang" (`route('checkout.pay', ...)` — route sudah tidak ada,
  tombol mati bila dibiarkan). Ganti dengan instruksi pembayaran manual bila perlu.

## B.3. Bersihkan dependency

- Bawaan template BELUM me-require SDK (`composer.json` tanpa `midtrans/midtrans-php`).
  Bila kamu sempat menginstal manual, cabut:
  ```bash
  composer remove midtrans/midtrans-php
  ```

## B.4. Verifikasi (wajib lolos semua)

```bash
composer install --no-dev
php artisan config:clear
```

```bash
grep -ri "midtrans\|snap_token\|payment.simulate\|checkout.pay\|payment.notification" app routes resources config database .env.example
```

- Install harus sukses tanpa error.
- Grep harus menghasilkan **0 baris**.
- Buka halaman checkout → buat order → pastikan tidak ada error route/view hilang.
- Hapus juga blok `midtrans` di `config/services.php` bila kamu sempat menambahkannya.

## B.5. Yang JANGAN dihapus

- `CheckoutController.php` itu sendiri (hanya bedah, alur order tetap dipakai).
- `routes/web.php` itu sendiri (hanya cabut 3 route + 1 import).
- Migrasi orders (hanya kolomnya, via langkah B.2.3).
- `config/services.php` itu sendiri (cukup hapus blok terkait bila ada),
  `composer.json`, `.env` (cukup hapus baris env-nya).

### Ringkasan Tujuan
Dokumentasi ini menjelaskan alur teknis (flow) dan formulasi perhitungan pembayaran (calculation) untuk method `bulkInvoiceCreate` di `src/services/invoice/index.js`. Fitur ini membuat satu "bulk invoice" (invoice induk) untuk pembelian multiperson/multibuyer, sekaligus membuat beberapa "child invoice" (invoice anak) untuk masing‑masing pembeli.

---

### Alur Teknis (Flow) `bulkInvoiceCreate`
1. Validasi & Normalisasi Requester
   - Memanggil `__adjustAndValidateRequester(params, requester)` untuk menyesuaikan dan memvalidasi akses (mendukung non‑login pada fitur multibuyer).

2. Mulai Transaksi Database
   - Membuka session MongoDB dan memulai transaksi: semua operasi simpan invoice dilakukan dalam satu transaksi (`session.startTransaction`).

3. Inisialisasi Bulk Invoice
   - Membuat instance `new BulkInvoice()` yang akan menjadi invoice induk.

4. Custom Form Submissions (jika ada)
   - Setelah validasi umum `__validateBeforeCreate`, jika ada `submissions`, disimpan via `customFormSubmissionService.saveSubmissions({ submissions, productId, invoiceId })` menggunakan `bulkInvoice._id` sebagai `invoiceId`.
   - Menandai `shouldRollbackCustomForm = true` agar submission dihapus bila transaksi invoice gagal.

5. Validasi Khusus Multipurchase
   - Pastikan `product.maxRegistrantPerTransaction > 1` (jika tidak, multipurchase ditolak).
   - Pastikan `params.users` tidak kosong.
   - Cek duplikasi email pada daftar user tambahan + requester: menggunakan `uniqBy([{ email: user.email }, ...params.users], 'email')`. Jika jumlah berubah, berarti ada duplikasi.
   - Pastikan jumlah `params.users.length` tidak melebihi `product.maxRegistrantPerTransaction`.
   - Jika `vendor` bukan `system`, pastikan `paymentMethod.isAllowMultiBuyer === true`.

6. Hitung Kuantitas & Buat Order ID
   - `quantity = appliedUsers.length` (requester + seluruh tambahan pembeli).
   - `orderId` bulk dibuat dari: `CC-<vendorCode>-<YYYYMMDD>-<invoiceIdentifier>`.

7. Hitung Field Pembayaran (Bulk)
   - Memanggil `__calculatePaymentFields({ paymentMethod, appliedPrice: appliedPrice * quantity, product, contentPage, user, invoice: bulkInvoice, fixedFeeMultiplier: quantity })`.
   - Menghasilkan objek `payment` dan `expiredAt` untuk invoice induk.
   - Status bulk: `paid` jika `payment.vendor === 'system'`, selain itu `pending`. `paidAt` diisi jika `system`.

8. Fail‑safe Validasi User & Registrasi Guest
   - Untuk setiap email pada `appliedUsers`:
     - Jika user sudah ada di DB, gunakan instance tersebut.
     - Jika belum, validasi email via `millionverifier.verify(email)`. Bila tidak valid → throw error (gagal total, tidak ada side‑effect parsial).
     - User yang tidak ada namun email valid akan didaftarkan sebagai guest via `userService.createGuestUser({ email, phone, name })`. Gagal registrasi guest di‑log (tidak menggagalkan transaksi, namun sebelumnya validasi email sudah memastikan email valid agar fail‑safe terpenuhi).

9. Pembuatan Child Invoices (Per User)
   - Untuk tiap user yang sudah final:
     - Menyusun `childPayment` dengan pembagian nilai bulk per `quantity` (lihat bagian formulasi di bawah). `method` dipaksa `system/multibuyer`, `vendor: 'system'` untuk konsistensi child invoice.
     - Membuat `new Invoice({... parent: bulkInvoice._id })` dengan `status`, `paidAt`, `expiredAt` mengikuti bulk invoice.
     - Memberi `orderId` child dengan format sama namun `identifier` berdasarkan `childInv._id`.
     - Mengumpulkan referensi anak (id, email, phone, name) untuk disimpan di field `children` milik bulk invoice.

10. Simpan ke Database (Dalam Transaksi)
    - Set `children` pada bulk dan `save(opts)`.
    - Simpan seluruh child invoices via `Invoice.insertMany(childInvoiceInstances, opts)`.

11. Side‑effects Setelah Penyimpanan
    - Menjalankan `__queueInvoiceInternalCallback(bulkInvoice)` bila diperlukan.
    - Mengirim task showcase update untuk tiap child: `showcaseService._sendTaskQueue('invoice-update', { invoiceId })`.

12. Commit / Rollback
    - Jika semua langkah berhasil → `session.commitTransaction()`.
    - Bila terjadi error:
      - `session.abortTransaction()`.
      - Jika sempat menyimpan custom form submissions (`shouldRollbackCustomForm === true`) → `CustomFormSubmission.deleteMany({ invoiceId: bulkInvoice._id })`.
      - Lempar ulang error.
    - `session.endSession()` pada blok `finally`.

13. Publish Event (di luar transaksi)
    - Setelah commit, publish event pub/sub: `__publishInvoicePubSubEvent(bulkInvoice)`.

14. Return
    - Mengembalikan instance `bulkInvoice` (invoice induk dengan relasi ke child invoices).

---

### Formulasi Perhitungan Pembayaran (Level Bulk)
Perhitungan utama dilakukan di `__calculatePaymentFields` dan menerima `appliedPrice` yang telah dikalikan `quantity`. Parameter `fixedFeeMultiplier` juga diisi `quantity` agar biaya bertipe "fixed" diskalakan per pembeli.

Variabel pembayaran utama:
- `vendor` = `paymentMethod.vendor`
- `method` = `paymentMethod.method`
- `vendorFee`: biaya yang dibayarkan ke vendor
- `baseAmount`: nilai dasar transaksi (diisi `appliedPrice`)
- `handlingFee`: biaya yang dibebankan ke user (Clicky fee + addons)
- `extraFee`: biaya tambahan berdasarkan konten/fitur produk (ditambahkan ke `handlingFee`)
- `uniqueCode`: angka acak 901–989
- `amountWithoutTax` = `baseAmount + uniqueCode + handlingFee`
- `tax` = PPN 11% dari `handlingFee` (dibulatkan ke atas)
- `amount` = `ceil(amountWithoutTax + tax)`
- `expiredAt`: 15 menit bila `product.maxRegistrant` ada; jika tidak, 1 hari

Detail rumus:
1) Vendor System
```
payment.vendor === 'system'
→ baseAmount = 0
→ handlingFee = 0
→ uniqueCode = 0
→ amountWithoutTax = 0
→ tax = 0
→ amount = 0
```

2) Vendor Non‑System
- Vendor fee
```
if paymentMethod.vendorFee.type === 'fixed':
  vendorFee = paymentMethod.vendorFee.amount
else: // percentage
  vendorFee = ceil(appliedPrice * (paymentMethod.vendorFee.amount / 100))
```
Catatan: `appliedPrice` di sini sudah dikalikan `quantity`, sehingga vendor fee persentase mengikuti total bulk. Untuk vendor fee bertipe fixed, nilainya TIDAK dikalikan `quantity` (sesuai implementasi).

- Base amount
```
baseAmount = appliedPrice // (appliedPrice sudah = price * quantity)
```

- Handling fee
```
if paymentMethod.handlingFee.type === 'fixed':
  handlingFee = paymentMethod.handlingFee.amount * fixedFeeMultiplier
else: // percentage
  handlingFee = ceil(appliedPrice * (paymentMethod.handlingFee.amount / 100))
```
`fixedFeeMultiplier = quantity` sehingga biaya fixed terakumulasi per pembeli.

- Reminder email (opsional)
```
if product.isReminderEmail && product.features?.reminder?.sendDate && now <= sendDate:
  payment.isReminderEmail = true
  payment.reminderEmailFee = 100 * fixedFeeMultiplier
  handlingFee += reminderEmailFee
```
Biaya reminder email berskala per pembeli.

- Extra fee berdasarkan konten/fitur
  - Deteksi konten: `__isSectionIncludeVideoFileAudio(contentPage.sections)`
  - Rumus tambahan (berdasar `baseAmount`):
```
if include file/audio: extra += 2% * baseAmount; payment.isIncludeFileOrAudio = true
if include video:     extra += 5% * baseAmount; payment.isIncludeVideo = true
if watermark:         extra += 1% * baseAmount; payment.isIncludeExtraFeeForWatermark = true

if extra > 0:
  payment.isIncludeExtraFee = true
  payment.extraFee = ceil(extra)
  handlingFee += payment.extraFee
```

- Unique code, subtotal, pajak, total
```
uniqueCode = randomInt(901, 989)
amountWithoutTax = baseAmount + uniqueCode + handlingFee
tax = ceil(handlingFee * 0.11)
amount = ceil(amountWithoutTax + tax)
```

- Validasi batas minimum/maksimum metode pembayaran
```
if amount < paymentMethod.minimumPayment or amount > paymentMethod.maximumPayment:
  throw ServiceException('invoice/payment-method-not-support-amount')
```

- Expiration (untuk bulk dan diwariskan ke child)
```
expiredAt = product.maxRegistrant ? now + 15 minutes : now + 1 day
```

---

### Pembentukan Pembayaran (Level Child Invoice)
Setiap child invoice memperoleh `payment` yang dibagi rata dari nilai bulk, agar agregasi laporan tetap konsisten:
```
childPayment = {
  isIncludeExtraFeeForWatermark: payment.isIncludeExtraFeeForWatermark,
  isIncludeExtraFee: payment.isIncludeExtraFee,
  extraFee:         payment.extraFee / quantity,
  vendorFee:        payment.vendorFee / quantity,
  baseAmount:       payment.baseAmount / quantity,
  amount:           payment.amount / quantity,

  handlingFee:      payment.handlingFee / quantity,
  uniqueCode:       payment.uniqueCode / quantity,
  amountWithoutTax: payment.amountWithoutTax / quantity,
  tax:              payment.tax / quantity,

  method: 'system/multibuyer',
  vendor: 'system',

  isReminderEmail:  payment.isReminderEmail,
  reminderEmailFee: payment.reminderEmailFee,
}
```
Catatan:
- Status child mengikuti status bulk (`paid` bila vendor bulk `system`, selain itu `pending`).
- `paidAt` dan `expiredAt` pada child mengikuti bulk.
- `orderId` child dibentuk serupa bulk tetapi menggunakan ID child.

---

### Contoh Perhitungan Sederhana
Misalkan:
- `quantity = 3`
- `appliedPrice (single)` = 100.000 → `appliedPrice (bulk)` = 300.000
- `paymentMethod.handlingFee` tipe fixed 2.000 → dengan `fixedFeeMultiplier = 3` → `handlingFee` awal = 6.000
- Produk menyertakan video (5%) dan file/audio (2%), watermark aktif (1%) → extra fee = 8% dari `baseAmount` = 0,08 × 300.000 = 24.000 → `handlingFee` = 6.000 + 24.000 = 30.000
- Unique code (misal) = 950
- `amountWithoutTax` = 300.000 + 950 + 30.000 = 330.950
- `tax` = ceil(11% × 30.000) = 3.300
- `amount` = ceil(330.950 + 3.300) = 334.250
- Maka per child (dibagi 3):
  - `baseAmount` 100.000, `handlingFee` 10.000, `uniqueCode` ~ 316,67, `tax` 1.100, `amountWithoutTax` ~ 110.316,67, `amount` ~ 111.416,67
  - Nilai pecahan tetap disimpan apa adanya di kode (pembagian langsung). Agregasi 3 child ≈ nilai bulk.

---

### Penentuan Status Pembayaran
- Bulk dan child akan `paid` segera hanya bila `payment.vendor === 'system'`.
- Untuk vendor non‑system (mis. Flip/Xendit), bulk berstatus `pending` dan kode memproses pembuatan tagihan vendor (mis. `__flipCreate`, `__xenditCreate`) serta menyimpan `payment.URL`, `payment.id`, `payment.vendorTransactionId`, `payment.payload` sesuai respons vendor.

---

### Penanganan Error & Konsistensi Data
- Semua operasi DB utama berada dalam satu transaksi.
- Bila gagal di tengah, transaksi di‑rollback.
- Submission custom form yang sudah terlanjur disimpan akan dihapus manual setelah abort transaksi (best‑effort logging saat gagal hapus).
- Validasi email calon pembeli dilakukan sebelum pembuatan user guest agar tidak ada efek samping ketika input tidak valid.

---

### Titik Ekstensi & Catatan Implementasi
- `fixedFeeMultiplier` saat ini dipakai untuk mengeskalasi biaya bertipe fixed (handling fee fixed, reminder email) sesuai jumlah pembeli.
- `vendorFee` bertipe fixed tidak diskalakan; bertipe persentase mengikuti `appliedPrice` (yang sudah dikalikan `quantity`). Pastikan kebijakan bisnis ini memang diinginkan untuk seluruh vendor.
- Pembagian nilai ke child dilakukan secara aritmetika sederhana (`/ quantity`). Jika diperlukan akurasi pembulatan per child, dapat dipertimbangkan strategi alokasi pembulatan (mis. last‑bucket adjustment).
- Expiry mengikuti kebijakan kapasitas produk: `maxRegistrant` → 15 menit; lainnya → 1 hari.
- Pub/Sub event (`__publishInvoicePubSubEvent`) dikirim setelah commit agar konsumen event tidak menerima data setengah jadi.

---

### Titik Kode Utama (Referensi Cepat)
- Validasi & hitung pembayaran bulk: `__validateBeforeCreate`, `__calculatePaymentFields`
- Pembuatan bulk: set `orderId`, `payment`, `status`, `expiredAt`, link `children`
- Pembuatan child: loop `appliedUserInstances`, `childPayment` hasil pembagian, `method: 'system/multibuyer'`
- Transaksi DB: `session.startTransaction` → `bulkInvoice.save(opts)` → `Invoice.insertMany(childInvoiceInstances, opts)` → commit/abort
- Side‑effects: `__queueInvoiceInternalCallback`, `showcaseService._sendTaskQueue`, `__publishInvoicePubSubEvent`

---

### Kesimpulan
`bulkInvoiceCreate` menyatukan proses multipurchase dalam satu invoice induk dengan perhitungan biaya pada level bulk (mengalikan harga dengan jumlah pembeli dan menskalakan biaya fixed), kemudian membagi seluruh komponen biaya secara proporsional ke setiap child invoice. Mekanisme transaksi, validasi email, dan rollback submission memastikan konsistensi data dan pengalaman pengguna yang terprediksi.

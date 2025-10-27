### Rangkuman detail perhitungan parent dan semua child (berdasarkan hasil eksekusi Anda)

Di transaksi ini, sistem mendeteksi 2 buyer: requester yang sedang login (`kevin@myskill.id`) dan 1 buyer dari payload (`holakevinromario@gmail.com`). Harga per orang: 50.000 (produk) + 50.000 + 100.000 + 50.000 (add-ons) = 250.000. Karena 2 orang, total baseAmount parent = 500.000.

Konfigurasi biaya yang terimplikasi dari hasil:
- handlingFee: 5% dari baseAmount
- PPN atas handlingFee: 11% (dibulatkan ke atas/ceil)
- vendorFee: 0,7% dari baseAmount (metadata, tidak menambah amount)
- extraFee: 0
- reminderEmailFee: 0

---

### Parent (BulkInvoice)
```json
{
  "extraFee": 0,
  "vendorFee": 3500,
  "baseAmount": 500000,
  "amount": 527750,
  "vendor": "flip",
  "handlingFee": 25000,
  "uniqueCode": 926,
  "amountWithoutTax": 525000,
  "tax": 2750,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
Penjelasan angka parent:
- baseAmount = 500.000 (2 × 250.000)
- handlingFee = 5% × 500.000 = 25.000
- tax = ceil(11% × 25.000) = 2.750
- amountWithoutTax = baseAmount + handlingFee = 525.000
- amount = amountWithoutTax + tax = 525.000 + 2.750 = 527.750
- vendorFee = 0,7% × 500.000 = 3.500 (hanya metadata)
- uniqueCode = 926 (acak, untuk parent/bulk)

---

### Child invoices per buyer dan per produk
Catatan:
- Setiap child dihitung per-produk (bukan dari total), dengan rumus yang sama: handlingFee = 5% × baseAmount_item, tax = ceil(11% × handlingFee_item), amount = baseAmount_item + handlingFee_item + tax_item.
- vendorFee per-child = 0 (metadata vendorFee hanya dihitung di level agregat/total).
- uniqueCode per-child pada 2 buyer ini kebetulan sama karena template perhitungan di-share antar user sebelum dibuatkan invoice; ini bisa diubah jika ingin unique di setiap anak per user.

#### Buyer 1: kevin@myskill.id
1) Parent product (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 968,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
2) Add-on (100.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 100000,
  "amount": 105550,
  "vendor": "flip",
  "handlingFee": 5000,
  "uniqueCode": 972,
  "amountWithoutTax": 105000,
  "tax": 550,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
3) Add-on (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 923,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
4) Add-on (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 930,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
Subtotal buyer 1 = 52.775 + 105.550 + 52.775 + 52.775 = 263.875

#### Buyer 2: holakevinromario@gmail.com
1) Parent product (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 968,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
2) Add-on (100.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 100000,
  "amount": 105550,
  "vendor": "flip",
  "handlingFee": 5000,
  "uniqueCode": 972,
  "amountWithoutTax": 105000,
  "tax": 550,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
3) Add-on (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 923,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
4) Add-on (50.000)
```json
{
  "extraFee": 0,
  "vendorFee": 0,
  "baseAmount": 50000,
  "amount": 52775,
  "vendor": "flip",
  "handlingFee": 2500,
  "uniqueCode": 930,
  "amountWithoutTax": 52500,
  "tax": 275,
  "isReminderEmail": false,
  "reminderEmailFee": 0
}
```
Subtotal buyer 2 = 263.875

---

### Validasi total akhir
- Total semua child = 263.875 × 2 = 527.750
- Sama persis dengan parent.amount = 527.750

---

### Catatan tambahan penting
- vendorFee hanya disajikan pada parent (3.500) sebagai metadata dari total baseAmount; di child diset 0 agar tidak terduplikasi.
- uniqueCode pada child terlihat sama antar 2 buyer (968, 972, 923, 930) karena template per-user dibagikan saat kalkulasi. Jika ingin unik benar-benar per-child-per-user, regenerasi `uniqueCode` saat membuat setiap child invoice atau lakukan deep-clone template per-user dan regenerasi field tersebut sebelum build child.

# PRD — Kain Nusantara ERP (repo jakjsbdndj/KN) · lanjutan sesi 2026-09-16

## Problem statement (verbatim)
saya ingin anda lanjutkan development dari repo ini https://github.com/jakjsbdndj/KN , sebelumnya saya ingin anda lanjutkan fitur penjualan sampel, sekarang penjualan sampel bisa di akses di tab menu pesanan, pilih productnya dll, nah agar mempermudah user di pos saya ingin tambahkan jual sebagai sampel yang langsung redirect ke page penjualan sampel namun datanya sudah otomatis terisi, tapi masalahnya seharusnya ini logicnya mirip dengan penjualan roll juga, bisa checkout bisa multiple item, bisa dikirimkan dll (identik flownya hanya saja untuk fullfillmentnya adalah permintaan potong ke admin gudang) coba pikirkan kembali best design implementationya bagaimana saya hanya ingin mempersimple dengan menggabungkan menu yang identik agar tidak terlalu banyak menu

## Keputusan pemilik (ask_human)
- Sampel dibedakan PER ITEM di cart (toggle "Jual sebagai sampel"); campur roll + sampel boleh.
- Tidak boleh duplikasi: menu lama "Jual Sampel" dihapus; sampel = baris SO.
- Fulfillment = permintaan potong ke admin gudang: cari roll dengan RFID handheld, potong, catat roll asal + panjang aktual; yard roll induk & qty inventory diperbarui.
- Harga: master harga sampel per induk (fallback harga daftar) DAN harga manual per baris.

## Desain yang diimplementasikan (2026-09-16)
- Backend: `SalesOrderItemIn.is_sample/sample_price`; `create_order` baris sampel → tanpa reservasi FEFO, saran roll FIFO, `has_sample`, status awal `reserved`.
- `create_outbound_tasks_for_order` (saat confirm) → tugas outbound `task_subtype=sample_cut` per baris sampel.
- `POST /api/outbound/tasks/{id}/cut-sample` {epc|roll_id, actual_length, reason}: resolve EPC→rfid_tags→roll (atau nomor roll), validasi produk/status/panjang, alasan wajib bila bukan saran FIFO, klaim atomik, split induk (guard), roll anak `reserved` untuk SO (tanpa tag, P-1), rebuild balance, reprice SO bila panjang aktual ≠ permintaan, task → packing (picked_qty=aktual) → dispatch/deliver identik roll.
- `GET /api/sample-quote` untuk POS; `/api/sample-prices` master tetap (tampil di view Harga per Badan Usaha).
- Dihapus: `/api/sample-requests*`, koleksi `sample_requests` tidak dipakai lagi, `SampleSalesView`, `SampleRequestForm`, menu hub "Jual Sampel", item "Jual Sampel" di HP sales.
- Frontend: tombol "Sampel" di ProductQuickView; CheckoutItemCard toggle + `SampleLineFields` (quote, saran FIFO, harga manual); badge Sampel di step 1/3, daftar & detail pesanan; WMS Barang Keluar `SampleCutPanel`; HP gudang tab Sampel (subtype) + lanjut Berangkatkan.
- Meja peran: antrean sampel sales/gudang dibaca dari sales_orders.has_sample / wms_tasks sample_cut.
- Uji: `backend/tests/test_sample_pos_flow.py` (E2E lolos manual via API).

## Backlog / P1
- Mobile sales cart (MobileCart) belum punya toggle sampel (hanya POS desktop).
- Label potongan: tempel tag RFID pada roll anak lewat /rfid/untagged-rolls (sudah ada).
- Batas panjang sampel & sampel gratis (Rp0) — belum diminta.

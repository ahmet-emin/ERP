# Proje Geliştirme Günlüğü

---

## 1. Veri Katmanı (DB) Kurulumu 🧱

- SQLite tabanlı `sappp.db` veritabanı oluşturuldu.
- Northwind ürün verileri içe aktarıldı.
- Aşağıdaki çekirdek tablolar kuruldu:
  - `products` → Ürün kartları
  - `bom` → Malzeme listesi (Bill of Materials)
  - `production_orders` → Üretim emirleri
  - `inventory_movements` → Stok hareketleri
- `pp_schema.sql` ve `seed_movements.sql` betikleriyle:
  - 101 (GR), 261 (GI), 601 (FG_GR) hareketleri işlendi.
  - RM1 ≈ 60, RM2 ≈ 30, FG ≈ 20 seviyelerinde örnek stok profili oluşturuldu.
- `v_stock` görünümüyle gerçek zamanlı stok hesapları aktif hale getirildi ✅

---

## 2. Backend (Flask API) 🧠

- Flask + SQLite altyapısı kuruldu.
- Çekirdek uçlar tanımlandı:
  - `/api/products`, `/api/stock`, `/api/movements`
  - `/api/production-orders` (create, issue, confirm)
- Üretim emri akışı:
  - **Issue** → bileşen stokları 261 hareketiyle düşülüyor
  - **Confirm** → mamul stoğu 601 hareketiyle artıyor
- End-to-end üretim senaryosu sorunsuz çalıştı ✅

---

## 3. Frontend (React + Vite) 🌐

- Dashboard, Materials, Production Orders, Movements sayfaları geliştirildi.
- REST API bağlantıları kuruldu, tablo & buton bileşenleri eklendi.
- Production Orders ekranından Issue/Confirm işlemleri UI üzerinden yapılabilir hale getirildi.
- Movements sayfasına filtrelenebilir listeleme eklendi.

---

## 4. Planlama & Analitik Katmanı (MRP Light + KPI) 📊

- `mrp_views.sql` ile MRP görünümü oluşturuldu:
  - Açık emirlerden bileşen gereksinimleri hesaplandı.
  - Net ihtiyaç = stok – gereksinim formülü uygulandı.
- `/api/analytics/kpi` ve `/api/analytics/mrp` uçları eklendi.
- Dashboard’a:
  - KPI kutuları (stok toplamı, açık emir sayısı, 24h hareket sayısı vb.)
  - MRP tablosu (net_after, req_qty) yerleştirildi.
- RM1/RM2 örneğinde `net_after` hesapları başarıyla doğrulandı ✅

---

## 5. Reversal Hareketleri & Audit Log 🔁

- `production_orders` statüsüne `REVERSED` / `CANCELLED` eklendi.
- `inventory_movements` tablosuna `reverse_ref` sütunu eklendi.
- 102 / 262 / 602 reversal uçları yazıldı.
- Audit log sistemi kuruldu:
  - Reversal işlemleri loglandı (`CONFIRM_REVERSAL`, `ISSUE_REVERSAL`).
- Frontend’e:
  - Reversal butonları yalnızca ilgili statülerde aktif olacak şekilde eklendi.
  - `/audit-log` ekranı audit kayıtlarını listeliyor.

---

## 6. Material Master Genişletmesi 🧱

- `products` tablosuna SAP Material Master alanları eklendi:
  - `material_type`, `reorder_point`, `lot_size`, `lead_time_days`, `procurement_type`
- `v_mrp` görünümü `mrp_view_extended.sql` ile geliştirildi:
  - `planned_order_qty` (lot size & reorder point bazlı öneri)
  - `planned_receipt_date` (lead time’a göre tarih hesaplama)
- `/api/material-master` & `/api/analytics/mrp-extended` uçları eklendi.
- Frontend’de Materials tablosu ve Dashboard planlama tablosu güncellendi (bg-amber-50 vurgulu satırlar).

---

## 7. Routing + Work Centers 🧭

- Routing ve Work Centers yapısı için şema ve API planları hazırlandı:
  - `work_centers` tablosu: kapasite ve iş merkezi bilgileri
  - `routing` tablosu: ürün başına operasyon adımları (`operation_no`, `std_time_min` vs.)
  - `v_capacity_load` görünümü: üretim emirlerine göre iş merkezi bazlı yük hesaplama
- Routing + Work Centers aşaması tamamlanmıştır.

---

## 8. Materials Management (MM)

### 🎯 Amaç

Üretim planlama (PP) modülünü tamamlayacak şekilde, malzeme tedarik ve satın alma süreçlerini sisteme entegre etmek. Bu modül ile tedarikçi yönetimi, satın alma emri (PO) akışı, mal kabul (GR 101), ters kayıt (Reversal 102) ve stok güncellemeleri gerçek ERP mantığında çalışır hale getirildi.

### 🔧 Yapılanlar

#### 8.1 Vendors (Tedarikçi Yönetimi)
- `vendors` tablosu oluşturuldu: `vendor_id`, `name`, `contact_email`, `lead_time_days_default`.
- `/api/vendors` uçları eklendi (GET / POST).
- Audit log kayıtları (`VENDOR_CREATE`) aktif hale getirildi.

#### 8.2 Purchase Orders (PO)
- `purchase_orders` ve `purchase_order_items` tabloları oluşturuldu.
- PO yaşam döngüsü: **DRAFT → APPROVED → PARTIAL_RECEIVED → RECEIVED → CANCELLED**.
- `/api/purchase-orders` uçları geliştirildi (create / list / detail).
- `approve` / `cancel` işlemleri duruma bağlı aktif hale getirildi.
- Audit log: `PO_APPROVE`, `PO_CANCEL`.

#### 8.3 Goods Receipt (GR — 101)
- `/api/purchase-orders/:po_id/receive` uç noktası eklendi.
- Kısmi ve tam mal alımı desteklendi.
- Over-receipt koruması: `receive_qty ≤ remaining_qty`.
- PO statüsü otomatik güncelleniyor (`APPROVED → PARTIAL_RECEIVED → RECEIVED`).
- Audit log: `MM_GR`.

#### 8.4 Reversal (GR Reversal — 102)
- `/api/purchase-orders/:po_id/reverse` uç noktası geliştirildi.
- 102 hareketleri `inventory_movements` tablosuna ekleniyor, `reverse_ref` ile ilişkilendiriliyor.
- `received_qty` ve PO statüsü doğru şekilde güncelleniyor.
- Audit log: `MM_GR_REVERSAL`.

#### 8.5 Görünümler ve Analitik (v_po_status & v_mm_kpi)
- Eksik SQL görünümleri (`v_po_status`, `v_mm_kpi`) oluşturuldu.
- `/api/analytics/po-status` ve `/api/analytics/mm-kpi` uçları aktif hale getirildi.
- Dashboard KPI’ları (Pending POs, Partial POs, GR last 24 h, Avg Lead Time) artık doğru veri çekiyor.
- `ensure_analytics_views()` fonksiyonu eklendi; app start’ta görünümleri otomatik doğruluyor.

#### 8.6 Frontend Entegrasyonu
- **Vendors** sayfası: listeleme / ekleme modali.
- **PO Listesi**: durum renkleri (DRAFT gray, APPROVED blue, PARTIAL amber, RECEIVED green, CANCELLED red).
- **PO Detail** paneli: header + kalem tablosu (Remaining, Toplam Qty’ler).
- **Receive / Reverse** modalları eklendi; toast bildirimleri ve status yenileme akışı tamamlandı.
- **Movement sekmesi** ile 101/102 geçmişi görselleştirildi.

### 🧪 Smoke Test Sonucu
- **Vendor akışı:** `V-ACME` başarıyla oluşturuldu ve audit log’a işlendi.
- **PO yaşam döngüsü:** DRAFT → APPROVED → RECEIVED akışı başarıyla geçti.
- **GR işlemleri:** RM1 için 25 + 15, RM2 için 20 adet alım; statü güncellemeleri doğru.
- **Reversal (102):** RM2 − 5; status geri PARTIAL_RECEIVED, stok dengesi sağlandı.
- **Negatif guard’lar:** Over-receipt (400) ve over-reversal (409) doğru yakalandı.
- **Analytics:** Görünümler eklendikten sonra `/api/analytics/*` uçları 200 dönüyor, metrikler senaryoyla uyumlu.
- **Stok durumu:** `/api/stock` sonucu, GR + Reversal sonrası tutarlı (+40 RM1, +15 RM2).

### 📘 Sonuç
🟢 MM modülü tamamlandı, tüm ana akışlar (Vendor → PO → GR → Reversal → Analytics) stabil çalışıyor.

---

## 9. Sales and Distribution (SD)

### 🎯 Amaç

Müşteri taleplerini sisteme dahil ederek Tedarik (MM) ve Üretim (PP) modülleriyle entegre, uçtan uca bir sipariş karşılama döngüsü oluşturmak. Bu modül, müşteri yönetimi, satış siparişi yaşam döngüsü, stok rezervasyonu ve teslimat süreçlerini kapsar.

### 🔧 Yapılanlar

#### 9.1 Veri Katmanı ve Mantık
- **Şema:** `customers`, `sales_orders`, `sales_order_items` ve `stock_reservations` tabloları `sd_schema.sql` içinde oluşturuldu.
- **Stok Rezervasyonu:** Bir sipariş onaylandığında (`CONFIRMED`), ilgili ürün miktarının `stock_reservations` tablosuna kaydedilmesi sağlandı.
- **Analitik Görünüm:** `v_available_stock` adında yeni bir SQL görünümü (`view`) oluşturuldu. Bu görünüm, `Fiziksel Stok - Rezerve Stok = Satılabilir Stok (ATP)` formülünü anlık olarak hesaplar.
- **Stok Hareketi:** Müşteriye teslimat için `701 - Customer Delivery` hareket kodu sisteme eklendi.

#### 9.2 Backend (Flask API)
- **Müşteri Uçları:** `/api/customers` endpoint'leri (GET / POST) ile müşteri listeleme ve oluşturma işlemleri aktif hale getirildi.
- **Satış Siparişi Yaşam Döngüsü:**
    - `POST /api/sales-orders`: `OPEN` statüsünde yeni sipariş oluşturur.
    - `POST /api/sales-orders/<id>/confirm`: Siparişi `CONFIRMED` statüsüne geçirir. Stok kontrolü yapar.
    - `POST /api/sales-orders/<id>/deliver`: Siparişi `DELIVERED` statüsüne geçirir, `701` hareketiyle fiziksel stoğu düşürür ve rezervasyonu kaldırır.
- **Analitik Uç Nokta:** `/api/analytics/available-stock` ile anlık satılabilir stok verisi sunuldu.

#### 9.3 Frontend Entegrasyonu
- **Yeni Sayfalar:** "Customers" ve "Sales Orders" sayfaları oluşturuldu.
- **Müşteri Yönetimi:** Müşteri listeleme ve ekleme modalı tamamlandı.
- **Satış Siparişi Arayüzü:**
    - Siparişler, statülerine göre renklendirilmiş bir tabloda listeleniyor.
    - "Create Order" modalı ile dinamik sipariş oluşturma imkanı sağlandı.
    - Sipariş detaylarını gösteren bir yan panel (Drawer) eklendi.
    - Aksiyon butonları (`Confirm`, `Deliver`) dinamik olarak gösteriliyor.

### 🧪 Smoke Test Sonucu
- **Müşteri Akışı:** UI üzerinden yeni müşteri başarıyla oluşturuldu.
- **Sipariş Yaşam Döngüsü:** `OPEN` → `CONFIRMED` → `DELIVERED` akışı UI üzerinden sorunsuz tamamlandı.
- **Stok Rezervasyonu:** Sipariş onaylandığında ATP değeri doğru bir şekilde azaldı.
- **Stok Düşüşü:** Teslimat sonrası fiziksel stok `701` hareketiyle doğru güncellendi.
- **Negatif Senaryo:** Yetersiz stok durumunda sipariş onayı engellendi ve kullanıcıya doğru hata mesajı gösterildi.

### 📘 Sonuç
🟢 SD modülü tamamlandı, tüm ana akışlar stabil çalışıyor.

---

## 10. Logistics Execution (LE)

### 🎯 Amaç

Stok yönetimini **Depo Yeri (Location)** bazına taşıyan, **Sevkiyat Belgesi (Delivery)** ve **Toplama (Picking)** süreçlerini entegre ederek lojistik akışı tamamlamak.

### 🔧 Yapılanlar

#### 10.1 Mimari Değişiklik
- **Veritabanı:** `inventory_movements` tablosuna zorunlu `location_id` sütunu eklendi.
- **MM/PP Adaptasyonu:** Tüm stok hareket API'ları artık zorunlu olarak depo yeri parametresi alacak şekilde güncellendi.
- **Stok Görünümü:** `v_stock` görünümü, stokları **malzeme/lokasyon çiftleri** bazında hesaplayacak şekilde güncellendi.
- **Frontend Adaptasyonu:** MM ve PP ekranlarındaki ilgili modallara zorunlu **Depo Yeri Seçimi** alanları eklendi.

#### 10.2 LE Çekirdek Yapısı (Delivery & Picking)
- **Şema:** `warehouse_locations`, `deliveries` ve `picking_list` tabloları oluşturuldu.
- **Delivery Oluşturma:** Onaylanmış SD siparişleri için `Delivery` belgesi oluşturma ucu (`POST /api/sales-orders/<id>/create-delivery`) eklendi.
- **Toplama (Picking) Yönetimi:**
    - `Start Picking` ile sistem kaynak lokasyonu otomatik olarak önerir.
    - `Confirm Picking` ile miktar girişi yapılır ve Delivery statüsü `PACKED`'e geçer.
- **Sevkiyat ve SD Entegrasyonu:** `Ship Delivery` aksiyonu Delivery statüsünü `SHIPPED` yapar, `701` hareketiyle stoğu düşürür ve rezervasyonu kaldırır.

#### 10.3 Frontend Entegrasyonu
- **Yeni Sayfa:** "Logistics" sayfası oluşturuldu.
- **Aksiyon Yönetimi:** Delivery statüsüne göre (`OPEN`, `PICKING`, `PACKED`, `SHIPPED`) ilgili butonlar dinamik olarak yönetiliyor.
- **Akış Başlatma:** `Sales Orders` ekranındaki `CONFIRMED` siparişler için **"Create Delivery"** butonu aktif hale getirildi.

---

## 11. Financial Accounting (FI)

### 🎯 Amaç

Tedarik (MM) ve Satış (SD) süreçlerinin finansal yansımalarını otomatik olarak kaydeden merkezi bir muhasebe sistemi kurmak.

### 🔧 Yapılanlar

#### 11.1 Veri Katmanı ve Mimari
- **Şema:** `fi_schema.sql` ile `accounts` (Hesap Planı), `journal_entries` (Yevmiye Defteri) tabloları oluşturuldu.
- **Maliyet Altyapısı:** `products` tablosuna `standard_cost` sütunu eklendi.
- **Ana Veri:** `seed_accounts.sql` ile temel muhasebe hesapları yüklendi.

#### 11.2 Backend ve Otomatik Entegrasyon
- **Merkezi FI Servisi:** `app/services/fi_service.py` oluşturuldu.
- **Hesap Tayini (Account Determination):** Kurallar `fi_service` içinde tanımlandı.
- **MM-FI Entegrasyonu (AP):** Mal Kabulü (101) yapıldığında **"153 Ticari Mallar (Borç) / 320 Satıcılar (Alacak)"** kaydı otomatik oluşturulur.
- **SD-FI Entegrasyonu (AR):** Sevkiyat (701) yapıldığında iki kayıt tetiklenir:
    1. **Maliyet:** **"621 SMM (Borç) / 153 Ticari Mallar (Alacak)"**
    2. **Gelir:** **"120 Alıcılar (Borç) / 600 Yurtiçi Satışlar (Alacak)"**

#### 11.3 Raporlama ve Frontend
- **Rapor Görünümü:** `v_trial_balance` (Mizan Raporu) SQL `view`'i oluşturuldu.
- **API Uç Noktaları:** `/api/fi/accounts`, `/api/fi/journal-entries`, `/api/fi/reports/trial-balance`.
- **Frontend Sayfaları:** "Chart of Accounts" ve "Journal Entries" sayfaları oluşturuldu.

---

## 12. Controlling (CO) - Maliyet Merkezi Muhasebesi

### 🎯 Amaç

Masrafların hangi departmanda veya fonksiyonel alanda oluştuğunu takip ederek temel bir maliyet kontrolü altyapısı kurmak.

### 🔧 Yapılanlar

#### 12.1 Veri Katmanı ve Altyapı
- **Şema:** `cost_centers` tablosu oluşturuldu. `accounts` tablosuna `is_cost_element` sütunu, `journal_entry_lines` tablosuna `cost_center_id` sütunu eklendi.
- **Ana Veri:** Örnek masraf yerleri (`PROD_CC`, `SALES_CC`, `ADMIN_CC`) eklendi.

#### 12.2 FI Entegrasyonu ve Çekirdek Mantık
- **Servis Güncellemesi:** `fi_service.py` güncellendi. Masraf hesabına kayıt atılırken `cost_center_id` gönderilmesi zorunlu hale getirildi.

#### 12.3 Operasyonel Süreç Entegrasyonu
- **PP-CO:** Hammadde Tüketim masrafı otomatik olarak `PROD_CC` ile ilişkilendirildi.
- **SD-CO:** Satılan Malın Maliyeti (SMM) masrafı otomatik olarak `PROD_CC` ile ilişkilendirildi.

#### 12.4 Raporlama (Backend & Frontend)
- **Backend:** `v_cost_center_report` SQL `view`'i ve ilgili API uçları oluşturuldu.
- **Frontend:** "Cost Centers" ve "Cost Center Report" sayfaları oluşturuldu.

---

## 13. Controlling (CO) - Ürün Maliyetlemesi & Kârlılık Analizi

### 🎯 Amaç

Ürün maliyetini BOM ve Rotalara dayalı dinamik olarak hesaplamak ve her satışın gerçek kârlılığını analiz etmek (CO-PA).

### 🔧 Yapılanlar

#### 13.1 Veri Katmanı Genişletmesi
- **Yeni Tablo:** `product_cost_breakdown` tablosu oluşturuldu.
- **Güncellemeler:** `work_centers` tablosuna `cost_rate_per_hour`, `products` tablosuna `calculated_cost` sütunları eklendi.

#### 13.2 Backend - Maliyet Hesaplama Motoru (`co_service.py`)
- **Yeni Servis:** `app/services/co_service.py` oluşturuldu.
- **Dinamik Hesaplama:** `calculate_product_cost` fonksiyonu çok seviyeli BOM ve aktivite maliyetlerini (işçilik) destekleyecek şekilde yazıldı.
- **Kayıt:** Hesaplama sonucunu `product_cost_breakdown` ve `products.calculated_cost` alanlarına kaydeder.

#### 13.3 Backend - Entegrasyon ve Raporlama (CO-PA)
- **Maliyetlendirme Koşusu:** `POST /api/co/run-cost-calculation` API ucu oluşturuldu.
- **SD-CO Entegrasyonu:** SMM kaydı atılırken artık `calculated_cost` değeri baz alınıyor.
- **Kârlılık Raporu:** `v_profitability_analysis` SQL `view`'i ve ilgili API ucu oluşturuldu.

#### 13.4 Frontend
- **Yeni Sayfa:** `ProfitabilityAnalysis.jsx` oluşturuldu.
- **Görselleştirme:** Sayfa, her satışın gelir, maliyet, brüt kâr ve kâr marjını gösterir.

---

## 14. Dashboard Geliştirmesi: Etkileşimli KPI Kartları

### 🎯 Amaç

Dashboard'u statik bir rapordan, sistemin genel sağlığını gösteren ve ilgili modüllere hızlı erişim sağlayan etkileşimli bir kontrol paneline dönüştürmek.

### 🔧 Yapılanlar

#### 14.1 Backend: Merkezi KPI Uç Noktası
- `GET /api/analytics/dashboard-kpis` adında merkezi bir uç nokta oluşturuldu.
- Metrikler iş mantığına göre hesaplandı (Açık Üretim Emirleri, Bekleyen Satın Almalar, Kritik Stok).

#### 14.2 Frontend: Gelişmiş Arayüz Bileşenleri
- **`KPICard.jsx`:** Yeniden kullanılabilir, tıklanabilir kart bileşeni oluşturuldu.
- **`KPICardSkeleton.jsx`:** Animasyonlu bir yükleme bileşeni tasarlandı.

#### 14.3 Frontend: Dashboard Entegrasyonu
- **Koşullu Render:** `loading`, `error`, `success` durumları yönetildi.
- **Etkileşim:** Her KPI kartı, tıklandığında kullanıcıyı ilgili sayfaya önceden filtrelenmiş bir şekilde yönlendirir.

---

## 15. MM - Fatura Kontrolü ve GR/IR Entegrasyonu

### 🎯 Amaç

"Procure-to-Pay" döngüsünü, 3-yönlü eşleştirme (PO-GR-Fatura) mantığı ile uçtan uca işler hale getirmek.

### 🔧 Yapılanlar

#### 15.1 Mimari Düzeltme: GR/IR Muhasebesi
- **Hesap Planı:** `190 - GR/IR Mutabakat Hesabı` eklendi.
- **FI Entegrasyonu:** Mal Kabulü (101) kaydı `153 Ticari Mallar (Borç) / 190 GR/IR (Alacak)` olarak değiştirildi.

#### 15.2 Fatura Kontrolü (Invoice Verification)
- **Veritabanı:** `vendor_invoices` ve `vendor_invoice_items` tabloları oluşturuldu.
- **Backend:** `POST /api/purchase-orders/<po_id>/post-invoice` API'ı geliştirildi.
- **Uyuşmazlık Kontrolü:** Fatura fiyat/miktar bilgisi PO ile karşılaştırılır. Uyuşmazlık varsa fatura `BLOCKED` olur.
- **FI Entegrasyonu:** Uyuşmazlık yoksa, fatura `POSTED` olur ve **`190 GR/IR (Borç) / 191 KDV (Borç) / 320 Satıcılar (Alacak)`** kaydı atılır.

#### 15.3 Frontend Entegrasyonu
- PO Detay sayfasına **"Post Invoice"** butonu ve **"Invoices"** sekmesi eklendi.
- Fatura giriş modalı ve fatura listesinde statüye göre renklendirme yapıldı.

---

## 16. Kullanıcı ve Rol Yönetimi (Güvenlik ve Yetkilendirme)

### 🎯 Amaç

Uygulamayı güvenli, denetlenebilir ve çok kullanıcılı bir ERP prototipine dönüştürmek.

### 🔧 Yapılanlar

#### 16.1 Veritabanı ve Mimari
- **Şema:** `users`, `roles`, `user_roles` tabloları oluşturuldu.
- **Denetim:** Tüm ana işlem tablolarına ve `audit_log`'a `user_id` sütunları eklendi.
- **Ana Veri:** `ADMIN`, `PP_PLANNER` gibi başlangıç rolleri eklendi.

#### 16.2 Backend (Kimlik Doğrulama ve Yetkilendirme)
- **JWT Entegrasyonu:** `Flask-JWT-Extended` ile güvenli token üretimi sağlandı.
- **API Rotaları:** Public (`/login`) ve private (`/admin/users`) uç noktalar oluşturuldu.
- **Güvenlik Katmanı:** `@jwt_required()` ve `@roles_required(...)` decorator'ları ile API güvenliği sağlandı.

#### 16.3 Frontend (Kullanıcı Arayüzü)
- **Global Durum Yönetimi:** React Context API ile kullanıcı bilgileri yönetildi.
- **Güvenli Rotalar:** Giriş yapmamış kullanıcılar login sayfasına yönlendirildi.
- **Dinamik Arayüz:** Kullanıcı rolüne göre menü elemanları gizlendi/gösterildi.
- **Yönetim Paneli:** `/admin/users` sayfası oluşturuldu.

#### 16.4 Operasyonel Entegrasyon
- **Denetim İzi:** Tüm işlemleri yapan `current_user` bilgisi artık kayıtlara ve audit log'a işleniyor.

---

## 17. Quality Management (QM) – Denetim, Kullanım Kararı ve FI/CO Entegrasyonu ✅

### 🎯 Amaç
Stok hareketleri sırasında kalite kontrol gerektiren malzemeler için inspection lot üretmek, kullanım kararı (Usage Decision) sürecini yönetmek ve hurda durumlarında FI/CO kayıtlarını otomatikleştirmek.

### 🔧 Yapılanlar

#### 17.1 Veri Katmanı ve Şema Güncellemeleri
- `products` tablosuna `quality_inspection_required` sütunu eklendi; QM gerektiren malzemeler işaretlenebilir hale geldi.
- `inventory_movements` tablosuna `stock_type` alanı eklendi ve `v_stock` görünümü stok tipine göre gruplanacak şekilde yeniden yazıldı.
- Yeni QM şeması (`qm_schema.sql`) ile:
  - `quality_inspection_lots`
  - `quality_usage_decisions`
  tabloları sisteme eklendi.

#### 17.2 Backend Servis ve API’lar
- `qm_service.py` ile inspection lot oluşturma ve usage decision mantığı merkezi olarak toplandı.
- Lot kararlarında otomatik stok hareketleri (321 / 551) tetiklendi, audit log’lara `QM_LOT_CREATED` ve `QM_UD_POSTED` kayıtları düşürüldü.
- Hurdaya ayrılan miktarlar için `QM_SCRAP` FI olayı ile 650 (Hurda Giderleri) / 153 (Stok) yevmiye kaydı oluşturulur; masraf merkezi varsayılan olarak `SCRAP_CC`.
- Yeni QM rotaları (`qm_routes.py`):
  - `GET /api/qm/inspection-lots`
  - `GET /api/qm/inspection-lots/<id>`
  - `POST /api/qm/inspection-lots/<id>/make-decision`
  eklendi ve role-based security ile korundu.

#### 17.3 MM & PP Entegrasyonu
- Satın alma mal kabullerinde (101) ve üretim teyitlerinde (601) QM bayrağına göre stok `QUALITY_INSPECTION` tipine alınıp inspection lot otomatik yaratılıyor.
- Usage decision sonrası stoklar `UNRESTRICTED` veya hurdaya (551) aktarılıyor; ATP (v_available_stock) yalnızca serbest stokları dikkate alıyor.

#### 17.4 Frontend – Quality Management Arayüzü
- `/quality-management` rotasında yeni sayfa: açık lot listesini gösteren `QualityLotTable` ve karar modali `UsageDecisionModal`.
- Modal doğrulamaları (accepted + rejected = lot qty) ve masraf merkezi seçimi sağlandı; karar sonrası toast bildirimleri ve liste yenileme uygulanıyor.
- Materials ve Dashboard sayfaları stok tiplerini (Serbest, Kalite, Bloke) ayrı sütunlarda gösterecek şekilde güncellendi.

#### 17.5 Test & Otomasyon
- `backend/tests/qm_smoke_test.py` ile MM → PP → QM → FI/CO sürecini uçtan uca doğrulayan otomatik smoke testi yazıldı.
- Script login olup QM bayraklarını set ediyor, PO/üretim emri oluşturuyor, kullanım kararlarını veriyor ve ilgili stok, lot, hareket, journal entry, cost center çıktılarının doğru olduğunu kontrol ediyor.

---
## 18. MRP Kokpiti ve Dinamik MRP Run

- Yeni MRP veri katmani tanimlandi: `backend/app/sql/mrp_schema.sql` (purchase_requisitions ve planned_orders tablolarini olusturur) ve `backend/app/sql/mrp_calculation_view.sql` (delta mantikli `v_mrp_calculation` gorunumu) eklendi, `db.create_tables()` / `ensure_analytics_views()` sureclerine baglandi.
- `backend/app/services/mrp_service.py` icinde `run_mrp()` fonksiyonu gelistirildi; `final_net_qty < 0` satirlar icin PR / Planned Order ureten ve kayitlari planlamacinin `user_id` bilgisiyle olusturan MRP motoru artik uygulama i�inde calisiyor.
- `frontend/src/pages/MRPWorkbench.jsx` kokpit ekrani yeniden tasarlandi: MRP kosusu butonu, toast bildirimleri, tab tabanli Acik PR / Acik Planli Siparis listeleri ve vendor/delta kontrolleri UI uzerinde dogrulaniyor. Yeni API cagri yardimcilari `frontend/src/services/api.js`e eklendi.
- Stage 18 UI smoke testi (`ui_smoke_stage18.py`) ile tablo baslangic durumunun bos oldugu, ilk MRP kosusunun kayit olusturdugu ve ikinci kosuda delta mantigiyla ek kayit uretilmedigi otomatik sekilde dogrulaniyor.

## 19. MRP Oneri Donusturme (PR -> PO, PLO -> Production Order)

- `mrp_service.convert_pr_to_po()` ve `mrp_service.convert_plo_to_prod_order()` fonksiyonlari delta onerilerini gercek belgelere donusturuyor; status kontrolleri, audit log kayitlari (`MRP_PR_CONVERTED`, `MRP_PLO_CONVERTED`) ve olusan belge kimlikleri saglaniyor.
- `backend/app/routes/mrp_routes.py` genisletildi: `/mrp/purchase-requisitions/<pr_id>/convert` ve `/mrp/planned-orders/<planned_order_id>/convert` uc noktalarina vendor secimi ve rol kontrolleri eklendi.
- Frontend kokpitte aksiyon butonlari eklendi: `frontend/src/components/ConvertToPOModal.jsx` vendor secimi ve hata gostermeyi yonetiyor, planned order donusumu icin onay modali ve toast mesajlariyla kayitlar listeden kaldiriliyor.
- Stage 19 smoke otomasyonlari tamamlandi:
  - API smoke (`test_smoke.py`) PR/PLO donusumlerini, yeni PO / Production Order detaylarini, status ve audit log kontrollerini yapar.
  - UI smoke (`ui_smoke_stage19.py`) kokpitteki ilk PR ve PLO satirlarini UI uzerinden donusturup toast ve tablo guncellemelerini dogrular.
---

## 20. Parti Yönetimi (Batch Management) Faz 3  🧪

### 🎯 Amaç
PP bileşen tüketimi (261) ve LE sevkiyat (701) akışlarının parti bazlı çalışmasını sağlamak; FEFO (expiry önceliği) önerilerini otomatik sunmak ve FI/CO kayıtlarında partinin gerçek maliyetini kullanmak.

### 🔧 Yapılanlar

#### Backend
- `stock_service.list_available_batches` ve `get_batch_snapshot` FEFO sıralaması ile parti/maliyet bilgisi dönecek şekilde genişletildi; `routes/stock.py` üzerinden `/api/stock/available-batches` ucu expose edildi.
- PP issue (`routes/production.py`) ve LE picking/ship (`routes/logistics.py`) uçlarında batch validate mantığı; batch-managed olmayan malzemeler için esnek doğrulama.
- `seed_phase3.py` script’i ile P-BATCH-FEFO bileşeni, FG-BATCH-FEFO mamulü, BATCH-A/B/C partileri ve stok girişleri kolayca seed edilebilir; eski veritabanlarında eksik `picked_batch_id` kolonunu otomatik ekliyor.
- `test_frontend_api.py` adlı API smoke betiği login → üretim emri → FEFO default → manuel override → issue/reversal → satış siparişi → confirm → teslimat → picking override akışını uçtan uca doğruluyor.

#### Frontend
- Production Orders “Issue” modalında FEFO önerisini otomatik seçen `Select Batch` dropdown’ı, kullanıcıya manuel override imkânı, gönderimde seçilen `from_batch_id` UI’den API’ye taşındı.
- Logistics “Confirm Picking” modalında FEFO önerisi, batch seçim doğrulamaları ve override desteklendi.
- `frontend/tests/batch-management.spec.ts` Playwright senaryosu ile login → üretim emri oluşturma → FEFO doğrulama → batch override → toast/assert akışı otomatik test ediliyor; `npm run test:ui` komutuna eklendi.

#### Test / Destek Scriptleri
- `python seed_phase3.py` ile demo dataset’i tekrar yüklenebilir.
- `python test_frontend_api.py` API smoke sonuçlarını verir.
- Frontend için `npm run test:ui` (Chromium) komutu yeni Playwright testini çalıştırır.

### ✅ Sonuç
FEFO önerileri ve gerçek maliyet muhasebe kayıtları hem backend hem de UI seviyesinde doğrulandı; batch-managed olmayan malzemeler için akış esnekliğini koruyor.

---

## 21. Sistem Sağlamlaştırma ve Portföy Hazırlığı

### Güvenlik ve yapılandırma

- Public kullanıcı kaydı varsayılan olarak kapatıldı; kontrollü ortamlar için `ALLOW_PUBLIC_REGISTRATION` ayarına bağlandı.
- Pasif kullanıcıların yeni giriş yapması ve daha önce alınmış token ile işlem yapması engellendi.
- Rol kontrolleri token içindeki eski claim yerine güncel veritabanı rolleriyle doğrulanacak şekilde güçlendirildi.
- JWT anahtarı zorunlu `JWT_SECRET_KEY` ortam değişkenine taşındı.
- CORS varsayılanı yalnızca yerel frontend adresiyle sınırlandı; debug modu ortam değişkenine bağlandı.
- Demo kullanıcılarının hazırlanması açık `SEED_DEMO_USERS` seçeneğine bağlandı.

### Veri bağlantısı ve test güvenilirliği

- SQLite context manager bağlantılarının commit/rollback sonrasında mutlaka kapanması sağlandı.
- Batch/FEFO smoke testi benzersiz ürün ve belge kimlikleri kullanacak şekilde tekrar çalıştırılabilir hale getirildi.
- Test veritabanı geçici klasörde oluşturuluyor ve hata durumunda da otomatik temizleniyor.
- Auth güvenliği ve Purchase Order temel akışı için izole SQLite kopyalarında çalışan 6 regresyon testi eklendi.

### Kod organizasyonu ve frontend performansı

- `purchase_orders.py` içindeki doğrulama, filtreleme, durum ve detay sorguları `purchase_order_service.py` servisine taşındı.
- `PurchaseOrders.jsx` içindeki Drawer, Invoice, Receive ve Reverse bileşenleri ayrı dosyalara bölündü.
- Sayfalar React `lazy` ve `Suspense` ile route bazında yüklenerek başlangıç JavaScript paketi yaklaşık 503 KB'dan 303 KB'a indirildi.
- Lojistik ekranındaki eksik hook bağımlılığı giderildi ve frontend lint uyarısız hale getirildi.

### Doğrulama sonucu

- Backend regresyon testleri: **6/6 başarılı**.
- Batch Management Phase 3 smoke testi: **başarılı**.
- Python AST kontrolü: **39/39 başarılı**.
- Frontend ESLint: **başarılı, 0 hata / 0 uyarı**.
- Vite production build: **başarılı**.

---

## 22. Deterministik Demo Veri Üreticisi

- `backend/seed_demo.py` ile Faker `tr_TR` tabanlı, sabit seed destekleyen demo veri üreticisi hazırlandı.
- `showcase` ve `load` presetleri tanımlandı; referans tarih `--as-of`, rastgelelik `--seed` ile kontrol ediliyor.
- Tedarikçi ve müşteri metinleri sentetik üretilirken malzeme, BOM, routing, batch, PO, üretim, satış, QM ve FI/CO ilişkileri kontrollü senaryolarla kuruluyor.
- Ana `sappp.db` yalnızca şablon olarak okunuyor; yazımlar ayrı staging kopyasında yapılıyor ve doğrulama başarılı olduğunda `demo_erp.db` atomik olarak oluşturuluyor.
- Template dosyasını output olarak seçme ve `--force` olmadan mevcut çıktıyı ezme engellendi.
- SQLite integrity, foreign key, negatif demo stoğu ve yevmiye dengesi kontrolleri üreticiye dahil edildi.
- Faker sürümü tekrarlanabilir çıktı için `Faker==40.38.0` olarak sabitlendi.
- İlk kullanıcı çalıştırmasında Northwind tabanlı `customers.company_name` zorunluluğu tespit edildi; generator hibrit Northwind/ERP müşteri kolonlarını birlikte dolduracak şekilde düzeltildi.
- İkinci kullanıcı çalıştırmasında veri üretimi tamamlandıktan sonra şablondan gelen 2.155 adet `order_details → products_old` legacy foreign key ihlali doğrulama kapısında yakalandı; hatalı çıktı yayımlanmadı.
- Generator, `order_details` tablosunu yalnızca staging kopyasında aynı 2.155 satırı koruyarak güncel `products(product_id)` ilişkisine taşıyacak şekilde güncellendi. Bütün satırların güncel ürün ve sipariş karşılıkları statik olarak doğrulandı.
- `clean showcase` varsayılanı eklendi; eski Northwind ve tarihsel ERP satırları yalnızca staging kopyasından temizleniyor. Eski davranış `--keep-template-data` ile isteğe bağlı korunabiliyor.
- Generator `showcase / seed=20260902 / as-of=2026-09-01` ile runtime çalıştırıldı: 7.569 eski satır çıkarıldı ve temiz `demo_erp.db` başarıyla üretildi.
- Üretilen çıktı: 25 malzeme, 6 tedarikçi, 12 müşteri, 20 PO, 10 üretim emri, 20 satış siparişi, 13 kalite lotu, 44 yevmiye kaydı ve 114 stok hareketi.
- Bağımsız doğrulama sonucu: SQLite integrity `ok`, foreign-key hatası `0`, legacy ürün `0`, negatif stoklu demo ürünü `0`; ana `sappp.db` SHA-256 değeri değişmedi.
- Temiz DB üzerinde admin login ve Dashboard, Stock, Movements, MRP, PO, PP, SD, LE, QM, FI ve CO dahil 11 API smoke isteği `200` döndü.
- Git tarafından yanlışlıkla takip edilen 35 adet Python `.pyc` cache dosyası kaldırıldı; `.gitignore` yeni cache dosyalarının tekrar eklenmesini engelliyor.
- Browser bağlantısı etkinleştirildikten sonra Dashboard, PO, PP, MRP, QM, LE ve CO ekranları temiz demo DB ile gerçek tarayıcıda doğrulandı.
- Görsel QA sırasında MRP listelerine malzeme adı ve güncel kullanıcı adı eklendi; MRP ekranındaki Türkçe karakterler düzeltildi.
- Kârlılık raporuna müşteri adı eklendi ve tablo başlıkları Türkçeleştirildi.
- Tarayıcı konsolunda hata/uyarı görülmedi; yedi doğrulanmış ekran görüntüsü `docs/screenshots/` altında kaydedilip README galerisine bağlandı.

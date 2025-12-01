Haier Europe Talep Tahmini Yarışması - 24. Sıra Çözümü
Bu depo, Haier Europe tarafından düzenlenen Demand Forecasting Datathon yarışması için geliştirilen ve yarışmayı 24. sırada (Skor: 0.98973) tamamlayan çözümü içermektedir.

🏆 Yarışma Özeti
Amaç: Haier Avrupa ürün portföyü için SKU (Stock Keeping Unit) bazında 12 aylık talep tahmini yapmak.

Zorluklar:

Yüksek Seyreklik (Sparsity): Verinin büyük kısmı sıfır satışlardan oluşuyordu.

Ürün Yaşam Döngüsü (Phase-Out): Üretimi biten ürünlerin satışının sert bir şekilde düşmesi gerekiyordu.

Hiyerarşik Yapı: Ürünler; Kategori, Yapı (Structure) ve İş Kolu hiyerarşisine sahipti.

Değerlendirme Metriği: Regularized Weighted Mean Absolute Percentage Error (rWMAPE).

🛠️ Çözüm Mimarisi: "Triple Ensemble Recursive Pipeline"
Final çözümümüz, üç güçlü Gradient Boosting algoritmasının (LightGBM, XGBoost, CatBoost) ağırlıklı ortalamasına dayanan, Özyineli (Recursive) bir tahminleme stratejisidir.

1. Veri Ön İşleme (Preprocessing)
Grid Oluşturma (Densification): Ham veride satış olmayan aylar eksikti. Tüm Market x Product x Date kombinasyonları için bir iskelet (skeleton) oluşturulup eksik aylar 0 ile dolduruldu.

Akıllı Tarih Atama (Smart Imputation): start_production_date verisi eksik olan ürünler için, o ürünün ilk satış yaptığı tarih başlangıç tarihi olarak kabul edildi. Hiç satışı olmayanlar için 1980 tarihi atanarak "Eski Ürün" (Mature) muamelesi yapıldı.

Gürültü Temizliği: İade kaynaklı negatif satışlar 0'a çekildi (Clipping).

2. Özellik Mühendisliği (Feature Engineering)
Modelin başarısının anahtarı, hiyerarşik yapıyı ve trend değişimlerini yakalayan 300+ özellik üretmekti.

Hiyerarşik Özellikler (Family Wisdom):

Bir SKU'nun davranışı, ait olduğu ailenin (Structure -> Category -> Sector) davranışına benzer.

Her hiyerarşik seviye için (Örn: Buzdolabı Kategorisi) kayan ortalamalar hesaplandı ve SKU'nun bu ortalamaya oranı (ratio_sku_structure) bir "Pazar Payı" sinyali olarak eklendi.

Gelişmiş Gecikmeler (High-Definition Lags):

Lag_1'den Lag_12'ye kadar kesintisiz geçmiş (Son 1 yılın her ayı).

Yıllık referans için Lag_24.

Hareketli İstatistikler & Momentum:

3, 6 ve 12 aylık Mean, Std, Max, Min, Skew.

Momentum: Kısa vadeli ortalamanın (3 ay), uzun vadeli ortalamaya (12 ay) oranı. (Trend yukarı mı aşağı mı?)

Volatilite: Standart Sapma / Ortalama. (Ürün ne kadar riskli?)

Zaman ve Döngüsellik:

Ay ve Çeyrek bilgileri için Sinüs/Kosinüs dönüşümleri.

3. Modelleme: "Triple Threat Ensemble"
Tek bir model yerine, üç farklı algoritmanın gücü birleştirildi. Her model, yarışmanın resmi metriği olan rWMAPE skorunu maximize edecek şekilde Optuna ile ayrı ayrı optimize edildi.

LightGBM: Hızlı eğitim ve yüksek doğruluk. (Objective: regression_l1)

XGBoost: Histogram tabanlı, kararlı tahminler. (Objective: reg:absoluteerror)

CatBoost: Kategorik değişkenlerde (Structure, Market) üstün performans. (Loss: MAE)

Ağırlıklandırma: Modellerin validasyon skorlarına göre dinamik ağırlıklar atandı (Örn: %40 LGBM, %35 XGB, %25 CAT).

4. Özyineli Tahmin (Recursive Forecasting Loop)
Direct Multi-Step yerine, geleceği adım adım inşa eden Recursive yöntem kullanıldı:

Ay T: Ensemble model Kasım ayını tahmin eder.

Feature Update: Kasım ayı tahmini, veri setine "Gerçekleşmiş Satış" gibi eklenir. Lag_1, Rolling Mean, Momentum gibi tüm dinamik özellikler bu yeni tahmine göre güncellenir.

Ay T+1: Model güncellenmiş özelliklerle Aralık ayını tahmin eder.

Bu döngü 12 ay boyunca devam eder.

5. Son İşlemler (Smart Post-Processing)
Phase-Out Kuralı: end_production_date geçmiş ürünlerin tahminleri sert bir kural ile 0'a eşitlendi.

Cold Start (Yeni Ürünler): Yeni piyasaya sürülen ürünler için lineer bir artış (Ramp-up) katsayısı uygulandı.

Anomaly Capping: Geçmiş 12 ayın maksimum satışının 3 katını aşan ekstrem tahminler baskılandı.

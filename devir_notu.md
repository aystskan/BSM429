# Devir notu — Adım 2 → Adım 3

Diğer Adımlar:
Adım	Konu
1	    İş problemi ve başarı ölçütü (H1)
4	    Denetimsiz analiz: birliktelik kuralı veya OLAP/pivot özeti (H4–H5)
5	    Sınıflandırma: en az iki model, biri açıklanabilir (H6–H8)
6	    Değerlendirme: karışıklık matrisi, çapraz doğrulama, ROC (H9)
7   	Kümeleme (H10–H11)
8	    Anomali tespiti, varyantınızın asıl işi (H11)
9	    Etik ve mahremiyet (H13)
10   	Sunum (H13)

**Kimden:** Adım 2 (Veri anlayışı) · **Kime:** Adım 3 (Ön işleme ve modelleme hazırlığı)
**Proje:** BSM 429 · l3 — Olay tespiti · Chicago Traffic Tracker, Temmuz 2024

Bu not, 3. adıma başlarken bilmen gereken her şeyi tek sayfada topluyor. Ayrıntılar ve kanıtlar
[`reports/02_veri_anlayisi.md`](reports/02_veri_anlayisi.md) ve [`notebooks/02_veri_anlayisi.ipynb`](notebooks/02_veri_anlayisi.ipynb) içinde.
Bütün sayılar [`outputs/bulgular.json`](outputs/bulgular.json) dosyasından okunabilir.

> ⚠️ 2. adım **hiçbir karar vermedi**: hiçbir satır silinmedi, hiçbir değer doldurulmadı, temiz veri dosyası yazılmadı.
> Aşağıdaki "seçenekler" öneri değil, kanıtla birlikte önüne konan seçeneklerdir — kararı ve gerekçesini 3. adım yazacak (H03).

---

## 1 · Veriyi yükleme

```python
import pandas as pd
df = pd.read_csv("data/raw/kf7e-cur8_2024-07.csv.gz")   # 129.311 satır × 17 sütun
df["time"] = pd.to_datetime(df["time"])                    # yerel saat (CDT); record_id ise UTC
df["dilim"] = df["time"].dt.floor("10min")                 # birleştirme anahtarı bu olmalı, ham time değil
```

Ham CSV'nin MD5'i: `2e14c914cc5ce10d1d4ce42c039ee41c` (gzip açılmış hâli).

## 2 · Bilmen gereken 12 gerçek

| # | Gerçek | Neden önemli |
|---|---|---|
| 1 | **`speed = 0` eksik demektir.** Kural kusursuz: `speed > 0` ⟺ `bus_count ≥ 5` ve `num_reads ≥ 30`. 10.214 satır (%7,9) | `isna()` bunları görmez. 0'ı hız sanmak ortalamayı 2 mph düşürür |
| 2 | Sıfırların **%90'ı gece 00–05'te**; saat 03'te satırların %58,6'sı sıfır | Eksik değerler sistematik olarak **hızlı** anlarda — şehir geneli medyanla doldurmak geceleri yapay yavaşlatır |
| 3 | **145 satır hiç yok:** 5 zaman dilimi × 29 bölge (18 Tem 17:50 · 20 Tem 13:00–13:10 · 29 Tem 17:00–17:10) | Örtük eksik. Saatlik birleştirmede bu saatler 6 değil 4–5 dilimle hesaplanır |
| 4 | Bölge başına sıfır oranı %0 ile %43 arası; Riverdale-Hegewisch'te 65 dilimlik (~11 saat) kesintisiz sıfır serisi | Uzun boşluklarda enterpolasyon savunulamaz |
| 5 | Saatlik veride 21.576 bölge-saat hücresinin **782'sinde (%3,6)** hiç geçerli ölçüm yok | Sıfırlar dahil ortalama → 0; hariç → NaN. İkisi aynı şey değil, seçimi yaz |
| 6 | **60 mph üstü 24 değer** (maks 155,8): hepsi az okumalı (`bus_count` ≤ 19), çoğu gece | Ölçüm gürültüsü — l3'te ilk yanlış alarmlar bunlar olur |
| 7 | **8 mph altı 51 değer**, 49'u çok otobüslü (≥ 40): 3 Tem ve 6 Tem akşamı Downtown Lakefront (havai fişek/NASCAR), 5 ve 31 Tem akşamı Loop | **Gerçek olay adayları** (31 Tem henüz açıklanmadı). Temizlikte **silinmemeli** — l3'ün isabet adayları |
| 8 | `bus_count` / `num_reads` = ölçüm güvenilirliği; olay günlerinde otobüs sayısının kendisi değişiyor (Downtown Lakefront 5 Tem: normalin %66'sı) | Modelden atma; en azından güvenilirlik filtresi/ağırlığı olarak tut |
| 9 | `hour`, `day_of_week`, `month` `time`'dan türetilmiş; `day_of_week` **Pazar = 1** (pandas: Pazartesi = 0); `month` sabit | Yeni zaman öznitelikleri türetirken kodlamayı karıştırma |
| 10 | `region_id` **nominal**; `west/east/south/north/nw_location/se_location` bölge başına sabit; `record_id` tanımlayıcı | Sayı olarak modele girerse sahte sıra/uzaklık öğrenilir; tanımlayıcı ve sabit sütunlar ölçüm değil |
| 11 | `num_reads` hızla da ilişkili (r = −0,54) ve ilişki aynı saat + aynı bölge içinde bile sürüyor (≈ −0,44). Hipotez: yavaş trafikte otobüs bölgede uzun kalır → daha çok okuma | Aynı dilimin hızını tahmin ederken `num_reads`/`bus_count`'u öznitelik yapmak hedefin yansımasını modele vermek olabilir (H03 §7.2) — gecikmeli (önceki dilim) hâlleri daha güvenli |
| 12 | Temmuz 2024'te **Pzt, Sal, Çar 5'er kez**, diğer günler 4'er kez; 4 Temmuz bir perşembeyi tatile çeviriyor | Gün bazlı ortalamalar ve zamana göre bölme noktası bu dengesizlikten etkilenir |

## 3 · Senin adımında verilecek kararlar ve elindeki kanıt

Yönerge 3. adım için şunları istiyor (H03): eksik değer envanteri + her sütun için karar **ve gerekçe**, aykırı değer kararı,
ölçekleme ve kodlama, 🔴 **sızıntı önlemi** (adımların sırası: hangisi bölmeden önce, hangisi sonra). Envanter hazır:
[`outputs/eksik_envanteri.csv`](outputs/eksik_envanteri.csv) · [`outputs/eksik_bolge.csv`](outputs/eksik_bolge.csv).

### 3.1 Sıfır hızlar

- **Mekanizma kanıtı:** eleme testi → MCAR değil; eksiklik gözlenen sütunlara (bus_count, num_reads → saat, bölge, gün tipi) bağlı → MAR.
- **Denenebilecek yollar (sen seçeceksin, gerekçesiyle):** hiç doldurmadan NaN olarak bırakıp saatlik ortalamayı yalnızca geçerli dilimlerden almak ·
  bölge × saat × gün tipi bağlamında doldurmak (diğer sütunlardan yararlanan yöntem — H03 §2.8) · kısa boşluklarda zaman içi enterpolasyon ·
  bir `hiz_eksik` gösterge sütunu (H03 §2.7: eksikliğin kendisi bilgi taşıyor — gece/seyrek hat).
- **Dikkat:** şehir geneli tek değerle doldurma, Şekil 2'deki gece yoğunlaşması yüzünden 8. adımda sahte "yavaş gece" alarmı üretir.

### 3.2 Aykırı değerler

- `> 60 mph` (24) ve `45–60 mph` (167): `bus_count`/`num_reads` ile birlikte karar ver; liste [`outputs/aykiri_60mph_ustu.csv`](outputs/aykiri_60mph_ustu.csv).
- `< 8 mph` çok otobüslü bölümler: [`outputs/aykiri_8mph_alti_bolumler.csv`](outputs/aykiri_8mph_alti_bolumler.csv) — **koru**; l3'ün aradığı şey bu.
- Sildiysen kaç satır ve neden; sakladıysan neden — sayıyla yaz (H03 §3.4: "silmeden önce ve sonra temel istatistikleri kaydedin").

### 3.3 Saatlik veri

- Birleştirme anahtarı: `time.dt.floor("h")` (ya da önce `dilim`, sonra saat). Ham `time` ile gruplama yapma — bölgeler 3–107 sn farkla yazılıyor.
- Saat başına kaç geçerli dilim kaldığını bir sütun olarak tut (0–6); güvenilirlik bilgisi kaybolmasın.
- Saatlik `bus_count`/`num_reads` toplamlarını da taşı.

### 3.4 Zaman öznitelikleri

Hazır: `hour`, `day_of_week`. Türetilecek adaylar: tarih, hafta sonu bayrağı, **tatil bayrağı** (4 Temmuz), döngüsel saat (sin/cos — 23 ile 0 komşu),
gecikmeli (lag) değerler — ama 8. adımda takvim doğrulaması yapılacağı için **takvim olaylarını öznitelik olarak modele koyarsan** doğrulama döngüsel olur;
bunu raporda tartış.

### 3.5 Bölme ve sızıntı — bu veriye özgü noktalar

- Veri zaman sıralı → **zamana göre bölme** (rastgele değil). Ay ortasında bir kesim tarihi seç ve gerekçesini yaz.
- **Bölmeden önce güvenli (satır bazlı, istatistik öğrenmeyen):** `speed = 0` → NaN işaretleme, `dilim`/saat sütunu üretme, metin düzeltmeleri, imkânsız değeri işaretleme.
- **Bölmeden sonra, yalnızca eğitimden öğrenilecek:** doldurma değerleri (medyan, bölge × saat profilleri), ölçekleyici, aykırı eşikleri,
  sınıflandırma hedefinin sınır değerleri (ayrıklaştırma), hedef kodlama (yaparsan).
- 🔎 Bölge × saat **profilinden** doldurma ya da "normal desen" çıkarma test dönemini görürse sızıntıdır — profil yalnızca eğitim penceresinden.

### 3.6 Sınıflandırma hedefi

`speed` sürekli; 5. adım sınıflandırma istiyor. Ayrıklaştırma seçenekleri (H03 §5.8): eşit genişlik · eşit frekans · alan bilgisiyle eşik
(ör. bölgenin kendi normaline göre "yavaş / normal / hızlı"). Hangisi olursa olsun **sınırlar eğitim verisinden** hesaplanır ve teste olduğu gibi uygulanır.
Bölge × saat × gün tipine göre göreli bir hedef, Şekil 4'teki tekrar eden deseni hedeften temizler — l3 ile de uyumlu olur.

## 4 · 8. adım için ileriye not

- Takvim ön listesi: [`outputs/takvim_on_liste.csv`](outputs/takvim_on_liste.csv) — 15 olay, bölge eşleştirmesi ve kaynaklarıyla (✓ / 🟡).
- 2. adımda bulunan ve takvimle **açıklanamayan** bölüm: 31 Temmuz 17:10–19:20 Chicago Loop durması.
- Eksik dilimler (18, 20, 29 Temmuz) veri kesintisi — ne isabet ne yanlış alarm.
- Temmuz'da kar yok: varyantın "kar fırtınası" türü bu veriyle test edilemez — sınırlılık olarak yaz.

## 5 · Repo düzeni

```
data/raw/            ham veri (gzip) — dokunma
notebooks/           02_veri_anlayisi.ipynb (çalıştırılmış) — senin defterin 03_on_isleme.ipynb olabilir
reports/             rapor bölümleri (markdown) — 03_on_isleme.md
figures/             sekil01…sekil09 (2. adım)
outputs/             tablolar (CSV, UTF-8 BOM — Excel'de Türkçe karakterler düzgün açılır) + bulgular.json
```

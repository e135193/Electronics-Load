# E-LOAD 200 — LVGL UI Uygulama Spesifikasyonu

Oct 26, 2026 · @Engin

## 1. Amaç ve kullanım

Bu belge, E-LOAD 200 elektronik yük ve batarya test cihazının 240×240 LVGL arayüzünü bir yapay zekâ kod ajanına yazdırmak için hazırlandı. Ajan, buradaki ekran kimliklerini, geçiş tablosunu ve widget kurallarını doğrudan koda çevirmelidir.

- Öncelik sırası: bu belge > ekran görselleri. Görsellerdeki yazılar bulanık olduğundan ekran metinleri yalnızca bu belgedeki İngilizce karakter dizilerinden alınır.
- Tüm menü geçişleri tek dönen encoder ile yapılır; dokunmatik, ikinci buton veya klavye yoktur.
- Yedi segmentli (dijital font) tüm rakamlar `lv_buttonmatrix` ile üretilir (bölüm 7).
- `[EK]` ile işaretli öğeler görsellerde yoktur; çalışması için eklenmiştir (bölüm 11).

## 2. Donanım ve yazılım varsayımları

Arayüz, RP2354A üzerinde çalışan LVGL 9.x ile ST7789 tabanlı 240×240 renkli ekrana uygulanır.

| Konu | Değer |
| --- | --- |
| MCU / ekran | RP2354A, ST7789 240×240, RGB565, SPI |
| Giriş | Tek döner encoder (A/B + buton); başka giriş yok |
| LVGL | 9.2 veya üstü. v8'de `lv_buttonmatrix` yerine `lv_btnmatrix` kullanılır |
| Çizim tamponu | Kısmi mod, 240×24 piksellik iki tampon, DMA ile SPI flush |
| Yenileme | `LV_DEF_REFR_PERIOD` 33 ms; ölçüm verisi arayüze en çok 10 Hz verilir |
| Heap | `LV_MEM_SIZE` en az 64 kB |
| Dil | Yalnız İngilizce, UTF-8 |
| UI fontu | Montserrat benzeri; 10, 12, 14 ve 20 pt. Gerekli glifler: ° Ω µ · › ✓ −; yerleşik fontta yoksa `lv_font_conv` ile üretilir |
| Dijital font | DSEG7 Classic Bold (SIL OFL), `lv_font_conv` ile 4 bpp; yalnız `0-9 . - :` ve boşluk; boyutlar 18, 26 ve 32 px |
| Uzun basma süresi | 600 ms (`lv_indev_set_long_press_time`) |

UI katmanı donanıma doğrudan dokunmaz. Ölçümleri `ui_model_t` yapısından okur, komutları (`ui_cmd_*`) geri çağrılarla gönderir; böylece masaüstünde SDL simülatörüyle test edilebilir (fare tekerleği = encoder dönüşü, orta tuş = encoder butonu).

## 3. Encoder kullanım kuralları

Her ekranda iki mod vardır, odak modu ve düzenleme modu; kurallar tüm ekranlarda aynıdır. Kısa basma onaylar veya ilerletir, uzun basma geri döner veya iptal eder.

| Mod | Döndür | Kısa bas | Uzun bas (≥ 600 ms) |
| --- | --- | --- | --- |
| Odak modu | Odağı sonraki / önceki öğeye taşır (sarma açık) | Odaktaki öğeyi etkinleştirir: buton basılır, satır düzenlemeye girer, menü öğesi ekranı açar | Geri: geçiş tablosundaki üst ekrana döner. Kök ekranda (SCR\_MAIN\_MENU) etkisiz |
| Düzenleme modu | Değeri veya seçeneği değiştirir | Onaylar ve düzenlemeden çıkar | İptal: eski değeri geri yükler ve çıkar |
| Rakam editörü (bölüm 7) | Faz A: imleci rakamlar ve ✓ hücresi arasında taşır. Faz B: seçili rakamı artırır / azaltır | Faz A: rakamı seçip Faz B'ye girer; ✓ hücresinde onaylar. Faz B: Faz A'ya döner | İptal |
| Diyalog (modal) | Düğmeler arasında odağı taşır | Düğmeyi çalıştırır | Diyaloğu kapatır (iptal düğmesi gibi) |

- Her ekranın bir `lv_group_t` grubu vardır; ekran yüklenince `lv_indev_set_group()` ile encoder'a bağlanır. Odak sırası bölüm 6 ve 8'de yazılıdır.
- Odaktaki öğe `LV_STATE_FOCUSED` stiliyle 2 px cyan çerçeve alır. Düzenlemedeki öğe `LV_STATE_EDITED` stiliyle dolu cyan olur.
- Çıkış açıkken (SCR\_LOAD, ON) ekrandan çıkılırsa çıkış önce kapatılır.
- Hızlı dönüşte (iki adım arası 30 ms'den kısa) değer düzenlemede adım ×10 olur; rakam editöründe ivme yoktur.
- Ekran geçişi animasyonsuzdur (`LV_SCR_LOAD_ANIM_NONE`); 120 ms fade yapılandırma bayrağıyla açılabilir.
- Her ekranda en az bir odaklanabilir öğe bulunmalıdır. Öğesi olmayan ekranlarda (SCR\_BRANCH, SCR\_SYSINFO) tüm ekranı kaplayan şeffaf bir `lv_obj` gruba eklenir ve basma olaylarını o alır.

## 4. Ekran listesi ve kimlikler

Uygulama 14 ekran ve 1 diyalog içerir; dört yük modu (CC, CV, CR, CP) görsellerde ayrı ekran gibi görünse de tek `SCR_LOAD` ekranıdır ve moda göre değişir.

| Kimlik | Başlık (header) | Rozet | Görsel no |
| --- | --- | --- | --- |
| `SCR_SPLASH` | — | — | 01 |
| `SCR_MAIN_MENU` | MAIN MENU | — | 02 |
| `SCR_LOAD` | Load · CC mode / CV / CR / CP | ON (yeşil) / OFF (gri) | 03–06 |
| `SCR_BATT_TYPE` | BATTERY TYPE | — | 07 |
| `SCR_BATT_SETUP` | BATTERY TEST | SETUP (cyan) | 08 |
| `SCR_BATT_RUN` | DISCHARGING | RUN (turuncu) / PAUSE (sarı) | 09 |
| `SCR_BATT_RESULT` | TEST RESULT | DONE (yeşil) / STOPPED (turuncu) | 10 |
| `SCR_BATT_GRAPH` | DISCHARGE CURVE | Test akımı, örn. 2.0 A (turuncu) | 11 |
| `SCR_BRANCH` | BRANCH STATUS | 4 CH (cyan) | 12 |
| `SCR_SETTINGS` | SETTINGS | — | 13 |
| `SCR_PROTECTION` | PROTECTION | ARMED (yeşil) / TRIPPED (kırmızı) | 14 |
| `SCR_CALIB` | CALIBRATION | n / 8 (cyan) | 15 |
| `SCR_SYSINFO` | SYSTEM INFO | — | 16 |
| `SCR_FAULT` \[EK\] | FAULT | TRIP (kırmızı) | — |
| `DLG_STOP` \[EK\] | Stop test? (modal) | — | — |

Ana menüdeki "Data Graph" döşemesi `SCR_BATT_GRAPH` ekranını açar; "Thermal Status" döşemesi `SCR_BRANCH` ekranını açar.

## 5. Menü ağacı

&#91;embedded content: menu tree · 14 screens, 1 dialog\]

Main menu altı koldan oluşur; batarya testi en uzun zincirdir (Battery type → Battery setup → Discharging → Test result → Discharge curve). Uzun basma her ekranda bir üst ekrana döner, ayrıntılar bölüm 6'daki tablodadır.

## 6. Menü geçiş tablosu

Aşağıdaki tablo, hangi ekranda hangi öğeye kısa basınca ne olduğunu ve uzun basınca nereye dönüldüğünü verir. Odak sırası, her ekranda satırların yukarıdan aşağıya sırasıdır. Ajan bu tabloyu `ui_trans_t` veri yapısı olarak kodlamalıdır (bölüm 10).

| Ekran | Odak sırası ve öğe | Kısa bas | Uzun bas |
| --- | --- | --- | --- |
| `SCR_SPLASH` | Öğe yok | Atla → `SCR_MAIN_MENU`. Otomatik geçiş: ilerleme %100 olunca, en az 1.5 s sonra | Etkisiz. Açılış öz-testi hatalıysa → `SCR_FAULT` |
| `SCR_MAIN_MENU` | 1 Electronic Load | → `SCR_LOAD` (son kullanılan mod; ilk açılışta CC) | Etkisiz (kök ekran) |
|  | 2 Battery Test | → `SCR_BATT_TYPE` |  |
|  | 3 Data Graph | → `SCR_BATT_GRAPH` |  |
|  | 4 Thermal Status | → `SCR_BRANCH` |  |
|  | 5 Protection | → `SCR_PROTECTION` |  |
|  | 6 Settings | → `SCR_SETTINGS` |  |
| `SCR_LOAD` | 1 Mod sekmeleri CC · CV · CR · CP | Düzenleme moduna girer; döndürmek sekmeyi seçer, kısa bas modu uygular. Çıkış ON iken sekmeler odak dışıdır | → `SCR_MAIN_MENU` (çıkış önce kapatılır) |
|  | 2 SET satırı \[EK\] | Rakam editörü açılır; ✓ ile onaylanınca setpoint uygulanır |  |
|  | 3 START / STOP düğmesi | Çıkışı açar / kapatır; rozet ON / OFF olur, düğme START (yeşil) ↔ STOP (kırmızı) |  |
| `SCR_BATT_TYPE` | 1–5 Li-ion / Li-Po, LiFePO4, NiMH / NiCd, Lead-acid, Custom | Türü seçer, varsayılan değerleri yükler → `SCR_BATT_SETUP` | → `SCR_MAIN_MENU` |
| `SCR_BATT_SETUP` | 1 Battery type | → `SCR_BATT_TYPE` | → `SCR_BATT_TYPE` |
|  | 2 Mode | Düzenleme: CC Discharge / CP Discharge / CR Discharge |  |
|  | 3 Current | Rakam editörü (0.100–20.000 A) |  |
|  | 4 Cut-off voltage | Rakam editörü (0.500–40.000 V) |  |
|  | 5 Max. time | Düzenleme: Off, sonra 00:01–99:59 (dk adımı) |  |
|  | 6 START TEST | Testi başlatır → `SCR_BATT_RUN` |  |
| `SCR_BATT_RUN` | 1 PAUSE / RESUME | Testi duraklatır / sürdürür; rozet RUN ↔ PAUSE | → `DLG_STOP` |
|  | 2 STOP | → `DLG_STOP` |  |
|  | Otomatik | Gerilim ≤ cut-off veya süre doldu → `SCR_BATT_RESULT` (DONE). Koruma tetiklenirse → `SCR_FAULT` |  |
| `DLG_STOP` \[EK\] | 1 Cancel (varsayılan odak) | Diyaloğu kapatır, test sürer | Cancel gibi |
|  | 2 Stop | Testi bitirir → `SCR_BATT_RESULT` (STOPPED) |  |
| `SCR_BATT_RESULT` | 1 GRAPH | → `SCR_BATT_GRAPH` (dönüş: `SCR_BATT_RESULT`) | → `SCR_MAIN_MENU` |
|  | 2 SAVE | Sonucu flash'a kaydeder, 1.5 s "Saved" gösterir, düğme pasif olur |  |
|  | 3 BACK | → `SCR_MAIN_MENU` |  |
| `SCR_BATT_GRAPH` | 1 İmleç (tek odak) | Döndürmek imleci kaydırır; kısa bas → çağıran ekran (`SCR_MAIN_MENU` veya `SCR_BATT_RESULT`) | Kısa basma ile aynı |
| `SCR_BRANCH` | Tüm ekran | → `SCR_MAIN_MENU` | → `SCR_MAIN_MENU` |
| `SCR_SETTINGS` | 1 Brightness | Düzenleme: ±5 %, 10–100 % (anlık uygulanır) | → `SCR_MAIN_MENU` |
|  | 2 Buzzer | Açar / kapatır |  |
|  | 3 Fan control | Düzenleme: Auto, 30 %, 50 %, 70 %, 100 % |  |
|  | 4 Temp. unit | °C ↔ °F |  |
|  | 5 Calibration › | → `SCR_CALIB` (adım 1). Çıkış ON ise reddedilir, "Turn output OFF" uyarısı gösterilir |  |
|  | 6 System info › | → `SCR_SYSINFO` |  |
| `SCR_PROTECTION` | 1 OVP | Rakam editörü (1.0–40.5 V) | → `SCR_MAIN_MENU` |
|  | 2 OCP | Rakam editörü (0.1–20.5 A) |  |
|  | 3 OPP | Rakam editörü (1–205 W) |  |
|  | 4 OTP | Rakam editörü (40–90 °C) |  |
|  | 5 LVP | Rakam editörü (0.00–40.00 V) |  |
|  | 6 ACT (Trip action) | Düzenleme: Latch off ↔ Auto retry |  |
| `SCR_CALIB` | 1 DMM reading | Rakam editörü | Kaydetmeden → `SCR_SETTINGS` |
|  | 2 PREV | Önceki adım (adım 1'de pasif) |  |
|  | 3 ACTION (START / STOP) | Adımın ölçümünü başlatır / durdurur |  |
|  | 4 NEXT | Sonraki adım. Adım 8'de etiket FINISH olur: kalibrasyonu flash'a yazar → `SCR_SETTINGS` |  |
| `SCR_SYSINFO` | Tüm ekran | → `SCR_SETTINGS` | → `SCR_SETTINGS` |
| `SCR_FAULT` \[EK\] | 1 ACKNOWLEDGE | Arıza kalktıysa → `SCR_MAIN_MENU`. Sürüyorsa düğme pasif, etiket "Cooling down..." | Etkisiz |

Her ekrandan `SCR_FAULT`'a geçiş, koruma tetiklendiğinde (OVP, OCP, OPP, OTP, LVP) otomatik olur; çıkış anında kapatılır ve hiçbir encoder eylemi bunu geri almaz.

## 7. Dijital rakam bileşeni (lv\_buttonmatrix)

Yedi segment fontla yazılan her rakam alanı, hücre başına bir karakter içeren tek satırlık bir `lv_buttonmatrix` ile üretilir; bu alanlarda `lv_label` kullanılmaz. Bileşen `ui_digits.c/.h` dosyasında bir kez yazılır, tüm ekranlar onu çağırır.

### 7.1 Hangi alanlar bu bileşeni kullanır

| Ekran | Alan | Font | Biçim (tam + ondalık) | Renk | Düzenlenebilir |
| --- | --- | --- | --- | --- | --- |
| `SCR_LOAD` | VOLTAGE (ölçülen) | DIG\_32 | 2 + 3, örn. `12.034` | Cyan | Hayır |
| `SCR_LOAD` | CURRENT (ölçülen) | DIG\_32 | 2 + 3, örn. `  5.000 ` | Turuncu | Hayır |
| `SCR_LOAD` | SET \[EK\] | DIG\_18 | Moda göre (aşağıda) | Beyaz, imleç cyan | Evet |
| `SCR_BATT_RUN` | Hücre gerilimi (yay göstergenin ortası) | DIG\_26 | 1 + 3, örn. `3.742` | Beyaz | Hayır |
| `SCR_BATT_RESULT` | Kapasite (mAh) | DIG\_32 | 5 + 0, örn. `  2486 ` | Yeşil | Hayır |
| `SCR_BATT_SETUP`, `SCR_PROTECTION`, `SCR_CALIB` | Düzenlenebilir sayı alanları (Current, Cut-off voltage, OVP…, DMM reading) | UI 12 pt kalın | Alana göre | Cyan | Evet. Aynı bileşen, hayalet katman kapalı |

POWER, ENERGY, TIME, sıcaklık, akım özeti gibi diğer sayılar düz `lv_label` olarak kalır (görsellerdeki gibi).

SET alanının biçimi moda göre değişir:

| Mod | Birim | Biçim | Aralık |
| --- | --- | --- | --- |
| CC | A | 2 + 3 | 0.000–20.000 |
| CV | V | 2 + 3 | 0.000–40.000 |
| CR | Ω | 3 + 2 | 0.05–500.00 |
| CP | W | 3 + 2 | 0.00–200.00 |

### 7.2 Yapı

Her örnek, üst üste duran iki matristen oluşur:

- `box`: şeffaf `lv_obj`, kenarlık ve dolgu yok, boyutu hücre genişlikleri toplamı × hücre yüksekliği.
- `bg` (hayalet katman): tüm hücreleri `8` olan matris, nokta hücresi `.`. Metin rengi ana rengin %15 opaklığıdır (`LV_OPA_15`), böylece sönük segmentler görünür. Tüm hücreler `LV_BUTTONMATRIX_CTRL_DISABLED`, nesne tıklanamaz, gruba eklenmez.
- `fg` (ön katman): gerçek rakamları taşıyan matris, `bg` ile aynı harita ve geometri. Düzenlenebilir alanlarda gruba eklenen nesne budur.
- Rakam hücresi göreli genişlik 3, nokta hücresi 1 (`lv_buttonmatrix_set_button_width`). Ondalık noktası ayrı, dar ve seçilemez bir hücredir.
- Birim harfi (V, A, mAh) matrisin dışında, yanında bir `lv_label`'dir.
- `lv_buttonmatrix` haritayı kopyalamaz; harita ve hücre metinleri örnek yapısının içinde, örnek yaşadığı sürece geçerli kalmalıdır (yığında veya geçici bellekte tutulmaz).

```c
struct ui_digits {
    lv_obj_t *box, *bg, *fg, *unit;
    const char *map[UI_DIGITS_MAX + 2];   // N hücre + "" sonlandırıcı
    char cell[UI_DIGITS_MAX + 1][2];      // her hücre tek karakter + '\0'
    uint8_t n, dot_idx, cursor, phase;    // phase: IDLE, CURSOR, ADJUST
    double value, backup, min, max;
};
// değer değişince: hücre karakterlerini yaz, sonra
lv_buttonmatrix_set_map(d->fg, d->map);   // yeniden çizimi tetikler
```

Stiller:

- `LV_PART_MAIN`: `bg_opa` 0, kenarlık 0, tüm `pad` 0, `pad_column` 0, `radius` 0, `outline` 0.
- `LV_PART_ITEMS`: `bg_opa` 0, kenarlık 0, gölge 0, `pad` 0, `radius` 2, `text_font` = dijital font, `text_color` = ana renk.
- `LV_PART_ITEMS | LV_STATE_FOCUSED`: imleç. Faz A'da dolgu cyan, metin arka plan rengi. Faz B'ye geçince aynı stilin `bg_color` değeri turuncuya çevrilir.

### 7.3 Arayüz (API)

```c
typedef struct {
    uint8_t int_digits, frac_digits;
    const lv_font_t *font;
    lv_color_t on_color;
    uint16_t cell_w, cell_h;   // piksel; dijital font için cell_w = yaklaşık 0.62 × font boyu
    bool ghost;                // sönük 8 katmanı
    bool editable;
    double min, max;
} ui_digits_cfg_t;

ui_digits_t *ui_digits_create(lv_obj_t *parent, const ui_digits_cfg_t *cfg);
void         ui_digits_set_value(ui_digits_t *d, double v);   // yuvarlar, baştaki sıfırları boşaltır
double       ui_digits_get_value(const ui_digits_t *d);
void         ui_digits_set_unit(ui_digits_t *d, const char *unit);
lv_obj_t    *ui_digits_focus_obj(ui_digits_t *d);            // gruba eklenecek nesne (fg)
// Commit'te fg üzerinde LV_EVENT_VALUE_CHANGED gönderilir; iptalde gönderilmez.
```

### 7.4 Düzenleme durum makinesi

Düzenlenebilir alanlar bölüm 3'teki rakam editörü kuralıyla çalışır:

| Durum | Döndür | Kısa bas | Uzun bas |
| --- | --- | --- | --- |
| IDLE (odak modu) | Odak başka öğeye geçer | CURSOR'a girer: `lv_group_set_editing(g, true)`, imleç en soldaki rakamda, `backup = value` | Geri |
| CURSOR (Faz A) | İmleç ±1; nokta hücresi atlanır; son hücre ✓ | Rakamdaysa ADJUST'a geçer. ✓ ise commit yapıp IDLE'a döner | İptal: `value = backup`, IDLE |
| ADJUST (Faz B) | Seçili rakam ±1, 0–9 arası sarar, taşıma yok | CURSOR'a döner | İptal |

- ✓ hücresi yalnız düzenleme sırasında görünür; yeri baştan ayrılır, böylece düzen kaymaz.
- Her değişiklikte değer `[min, max]` aralığına kıstırılır ve ekran hemen güncellenir. Donanıma yalnız commit anında gidilir.
- Düzenleme dışında baştaki sıfır hücreleri boş (`' '`), düzenleme sırasında `'0'` gösterilir.
- Tuş işleme: fg matrisinde `LV_EVENT_KEY` (`LV_KEY_LEFT` / `LV_KEY_RIGHT` = düzenleme modunda dönüş, `LV_KEY_ENTER` = kısa bas) ve `LV_EVENT_LONG_PRESSED` işlenir. Matrisin kendi seçim mantığı araya girmesin diye geri çağrı `LV_EVENT_PREPROCESS` ile eklenip `lv_event_stop_processing()` çağrılır. Kullanılan LVGL sürümünde bu çalışmazsa `ui_input` katmanı encoder olaylarını (ROT, PRESS, LONG) uygulama düzeyinde yayınlar ve editör onları kullanır; hangi yol seçilirse seçilsin davranış yukarıdaki tabloyla aynı olmalıdır.

### 7.5 Doğrulama

Simülatörde bir `ui_digits_selftest` ekranı oluşturulur: `12.034`, `  5.000 `, `3.742`, `  2486 ` değerleri doğru hücrelerde, hayalet segmentlerle birlikte görünmeli; SET alanında CURSOR → ADJUST → ✓ akışı, kıstırma ve iptal (uzun bas) elle denenmelidir.

## 8. Ekran ekran uygulama notları

Her ekran ortak iskeleti kullanır: üstte 26 px yüksekliğinde başlık çubuğu (sola 11 pt kalın başlık, sağa rozet, altında 1 px vurgu çizgisi), altta 20 px yüksekliğinde ipucu çubuğu (y = 220, yalnız liste ekranlarında). Boyutlar 240×240 koordinatındadır; kesin yerleşim için ekran görselleri esas alınır.

### 8.1 SCR\_SPLASH

- Ortada logo: 44 px yarıçaplı, 6 px kalınlıklı `lv_arc` halkası (cyan) ve içinde turuncu şimşek (`lv_image`, A8 ikon).
- "E-LOAD 200" (24 pt kalın), "Electronic Load & Battery Tester" (10 pt, soluk), "40 V · 20 A · 200 W" (11 pt kalın, cyan).
- 160×6 px `lv_bar` ilerleme çubuğu; değer açılış adımlarından (0–100) beslenir. Altında "Initializing..." ve "FW v1.0.0".

### 8.2 SCR\_MAIN\_MENU

- 2 sütun × 3 satır `lv_button` döşeme, her biri 108×56, aralık 8 px, yarıçap 9. Sol tarafta 24×24 ikon, sağda iki satırlık etiket.
- Odak sırası satır satır: Electronic Load, Battery Test, Data Graph, Thermal Status, Protection, Settings.
- Odaklı döşeme: dolgu `PANEL2` + 2 px cyan çerçeve. İpucu çubuğu: "Rotate: select" / "Press: open".

### 8.3 SCR\_LOAD

- Başlık metni "Load · CC mode" biçimindedir (CC, CV, CR, CP). Rozet: ON yeşil, OFF gri.
- Mod sekmeleri: tek satırlı `lv_buttonmatrix` (CC, CV, CR, CP), `lv_buttonmatrix_set_one_checked(true)`, seçili sekme cyan dolgu.
- Ölçüm paneli (`PANEL`, yarıçap 9): VOLTAGE ve CURRENT satırları dijital rakamlarla (bölüm 7), POWER satırı düz etiket (yeşil, örn. "60.17 W") ve altında `lv_bar` (güç / 200 W).
- SET satırı \[EK\]: etiket (SET CURRENT / SET VOLTAGE / SET RESISTANCE / SET POWER) + SET rakam alanı + birim. Panelin altına 22 px yer açmak için satır aralıkları sıkıştırılır.
- START / STOP düğmesi tam genişlik, 28 px yüksek: START yeşil, STOP kırmızı.
- Ölçümler `ui_model` üzerinden 10 Hz yenilenir. CC, CV, CR, CP ayrı ekran değil, aynı ekranın modudur; mod değişince yalnız SET satırı (etiket, birim, biçim, aralık) ve başlık metni değişir.
- Çıkış ON iken sekmeler gruptan çıkarılır; SET alanı düzenlenebilir ve onaylanınca yeni setpoint anında uygulanır.

### 8.4 SCR\_BATT\_TYPE

- Beş satır, her biri 224×33: ad (11 pt kalın) + alt satır (8 pt soluk) + sağda radyo işareti. Seçili tür dolu cyan nokta ve cyan çerçeve taşır.
- Varsayılan değerler (Battery setup ve yay göstergesi bunları kullanır):

| Tür | Nominal | Cut-off | Dolu (gösterge üst sınırı) |
| --- | --- | --- | --- |
| Li-ion / Li-Po | 3.7 V | 3.0 V | 4.2 V |
| LiFePO4 | 3.2 V | 2.5 V | 3.65 V |
| NiMH / NiCd | 1.2 V | 1.0 V | 1.45 V |
| Lead-acid | 12 V | 10.5 V | 12.9 V |
| Custom | Li-ion değerleri | Kullanıcı | Kullanıcı |

### 8.5 SCR\_BATT\_SETUP

- Beş değer satırı (Battery type, Mode, Current, Cut-off voltage, Max. time), her biri 224×28; etiket solda, değer sağda cyan. Düzenlenen satırda değer cyan dolgulu kutu içindedir.
- Altta tam genişlik yeşil "START TEST" düğmesi. Varsayılan akım 2.000 A; Current × Cut-off güç sınırını (200 W) aşarsa START TEST pasif olur.

### 8.6 SCR\_BATT\_RUN

- Merkezde `lv_arc`: 135°–45° (270° süpürme), kalınlık 9, bilgi çubuğu yeşil, tutamaç kaldırılır (`LV_PART_KNOB` stilsiz, nesne tıklanamaz). Değer, hücre geriliminin cut-off ile dolu gerilim arasındaki konumudur.
- Yayın ortasında "CELL VOLTAGE" etiketi, DIG\_26 hücre gerilimi ve "V". Yay uçlarının altında cut-off ve dolu gerilim değerleri (8 pt).
- Üç istatistik döşemesi 72×34: CAPACITY (mAh), ENERGY (Wh), TIME (hh:mm:ss). Altında "2.000 A · 7.48 W · 38 °C" özeti (10 pt kalın, cyan).
- Alt düğmeler: PAUSE (turuncu çerçeve), STOP (kırmızı dolgu), her biri 108×26. Veri 5 Hz yenilenir.

### 8.7 SCR\_BATT\_RESULT

- Üstte "DISCHARGE CAPACITY" etiketi, altında yeşil DIG\_32 kapasite + "mAh".
- 2×3 döşeme (108×32): ENERGY, TIME, AVG. VOLTAGE, START VOLTAGE, END VOLTAGE, INT. RESISTANCE (mΩ).
- Altta üç düğme (70×26): GRAPH (varsayılan odak, cyan), SAVE, BACK.

### 8.8 SCR\_BATT\_GRAPH

- `lv_chart` (tür LINE), iki seri: gerilim (cyan, sol eksen, cut-off ile dolu gerilim arası) ve akım (turuncu, ince, 0 – 2 × test akımı). Kayıt RAM'de en çok 2048 örnektir; grafik 120 noktaya seyreltilir. Yatay eksen dakika.
- İmleç: `lv_chart_add_cursor`, yanında "mm:ss · x.xx V" etiketi. Alt satırda gösterge (Voltage, Current) ve yeşil toplam kapasite.
- Veri yoksa grafik yerine ortalanmış "No data" etiketi gösterilir.

### 8.9 SCR\_BRANCH

- Q1–Q4 için dört satır (224×34): sol ad, orta akım (örn. "1.25 A"), sağ sıcaklık, altta `lv_bar` (akım / 5.00 A). Sıcaklık rengi: 60 °C altı yeşil, 75 °C altı sarı, üstü kırmızı.
- Özet kutusu: "Total 5.00 A · 60.2 W" ve "Fan 2850 rpm". İpucu çubuğu: "Heatsink 48 °C" / "Press: back". Bu ekran yalnız gösterim içindir; kol başına setpoint burada düzenlenmez.

### 8.10 SCR\_SETTINGS

- Altı satır (224×28): Brightness (`lv_slider` 72 px + yüzde), Buzzer (`lv_switch`), Fan control (değer + ›), Temp. unit, Calibration ›, System info ›.
- Fan control satırındaki › inline seçenek düzenlemesidir, alt ekran açmaz. Ayarlar 2 s gecikmeyle flash'a yazılır; Temp. unit değişince tüm ekranların sıcaklık etiketleri güncellenir.

### 8.11 SCR\_PROTECTION

- Altı satır: solda renkli etiket (OVP kırmızı, OCP turuncu, OPP sarı, OTP kırmızı, LVP cyan, ACT soluk) + açıklama (Over voltage, Over current, Over power, Over temp., Low-V cut-off, Trip action) + sağda cyan değer.
- Varsayılanlar: OVP 40.5 V, OCP 20.5 A, OPP 205 W, OTP 85 °C, LVP 0.50 V, ACT Latch off. Rozet: ARMED yeşil, tetiklenince TRIPPED kırmızı.

### 8.12 SCR\_CALIB

- Üstte 8 segmentli adım çubuğu (tamamlanan segmentler cyan), altında adım adı (12.5 pt kalın) ve tek satır yönerge.
- Okuma paneli: ADC raw, Measured, Gain, Offset. Altında DMM reading girişi (düzenlenebilir rakam alanı). Altta PREV, ACTION (START / STOP), NEXT düğmeleri.
- Adımlar (donanıma göre ayarlanacak öneridir): 1 Voltage offset (0 V), 2 Voltage span (20.000 V), 3 Current offset (0 A), 4 Current span low (5.000 A), 5 Current span high (15.000 A), 6 Setpoint DAC offset, 7 Setpoint DAC gain, 8 Save & finish.

### 8.13 SCR\_SYSINFO

- Sekiz satır (224×21, zebra dolgu): Model "E-LOAD 200", Ratings "40 V · 20 A · 200 W", Firmware "v1.0.0", Hardware "Rev A", MCU "RP2354A", Display "ST7789 240×240", Serial no. "EL200-0001", Uptime "hh:mm:ss" (canlı).

### 8.14 SCR\_FAULT \[EK\] ve DLG\_STOP \[EK\]

- SCR\_FAULT: koyu kırmızı zemin, kırmızı başlık (FAULT, rozet TRIP), sarı uyarı üçgeni, arıza adı (örn. "OVER TEMPERATURE"), açıklama ("Q3 heatsink reached 86 °C"), "Load input has been disabled.", "ERR 0x04 · OTP" ve tam genişlik kırmızı ACKNOWLEDGE düğmesi.
- DLG\_STOP: arka ekran %62 karartılır, ortada 200×116 panel; başlık "Stop test?", metin "The discharge test is still running. Stop now and show the result?", düğmeler Cancel (varsayılan odak) ve Stop (kırmızı).

## 9. Ortak stil, renk ve ölçüler

Tüm renkler ve ölçüler `ui_theme.h` içinde sabit olarak tanımlanır; ekran dosyalarında sabit sayı veya renk yazılmaz.

| Ad | Değer | Kullanım |
| --- | --- | --- |
| `C_BG` | `#0A0E14` | Ekran zemini |
| `C_PANEL` | `#151C26` | Panel, satır, döşeme |
| `C_PANEL2` | `#202B3A` | Odaklı / seçili satır dolgusu |
| `C_LINE` | `#2E3C4E` | Ayırıcı çizgi, boş çubuk |
| `C_TEXT` | `#E6EDF3` | Ana metin |
| `C_MUTED` | `#7D8A99` | İkincil metin, etiket |
| `C_CYAN` | `#00BEE1` | Vurgu, odak, gerilim |
| `C_GREEN` | `#2ECC71` | ON, START, güç, başarı |
| `C_ORANGE` | `#FF9F1C` | Akım, RUN, düzenleme fazı B |
| `C_RED` | `#E5484D` | STOP, hata, kritik sıcaklık |
| `C_YELLOW` | `#FACC15` | Uyarı, PAUSE, orta sıcaklık |
| Hayalet segment | `C_CYAN` veya ilgili renk, `LV_OPA_15` | Dijital alanların sönük `8` katmanı |

| Ölçü | Değer |
| --- | --- |
| Sayfa kenar boşluğu | 8 px |
| Başlık çubuğu | 26 px + 1 px vurgu çizgisi |
| İpucu çubuğu | y = 220, 20 px |
| Satır yüksekliği | 28 px (liste), aralık 3 px |
| Köşe yarıçapı | Panel / döşeme 8–9 px, düğme 7 px, rozet 7 px |
| Odak çerçevesi | 2 px `C_CYAN`, yarıçap 8 |
| UI font boyları | 8, 10, 11, 12, 14, 20 pt (kalın ve normal) |
| Dijital font boyları | DIG\_18, DIG\_26, DIG\_32 |

Rozet: sağ üstte, 14 px yüksekliğinde hap şekli, 8 pt kalın metin, metin rengi `C_BG`, dolgu rozetin anlamına göre (bölüm 4).

## 10. Modül yapısı, uygulama sırası ve kabul kriterleri

Geçişler kod içine dağılmaz; tek bir tabloda toplanır ve bölüm 6'daki tabloyla satır satır eşleşir.

```c
typedef enum { SCR_SPLASH, SCR_MAIN_MENU, SCR_LOAD, SCR_BATT_TYPE, SCR_BATT_SETUP,
               SCR_BATT_RUN, SCR_BATT_RESULT, SCR_BATT_GRAPH, SCR_BRANCH, SCR_SETTINGS,
               SCR_PROTECTION, SCR_CALIB, SCR_SYSINFO, SCR_FAULT, SCR_COUNT } ui_screen_id_t;

typedef struct {
    ui_screen_id_t from;
    uint8_t        item;        // odak sırası, 0 tabanlı
    ui_screen_id_t to_short;    // kısa basma hedefi, SCR_NONE = ekran içi eylem
    ui_screen_id_t to_long;     // uzun basma hedefi (geri)
} ui_trans_t;

void ui_nav_goto(ui_screen_id_t id);   // eski ekran silinir (lv_screen_load_anim, auto_del = true)
void ui_nav_back(void);                // yalnız SCR_BATT_GRAPH için çağıran ekran saklanır
```

Dosya düzeni:

| Dosya | İçerik |
| --- | --- |
| `ui.c/.h` | `ui_init()`, ekran kaydı, ana döngü bağlantısı |
| `ui_theme.c/.h` | Renk ve ölçü sabitleri, ortak `lv_style_t` nesneleri |
| `ui_input.c/.h` | Encoder `lv_indev` sürücüsü, uzun basma, hızlanma |
| `ui_nav.c/.h` | `ui_trans_t` tablosu, geçiş fonksiyonları |
| `ui_digits.c/.h` | Dijital rakam bileşeni (bölüm 7) |
| `ui_widgets.c/.h` | Başlık, ipucu çubuğu, rozet, satır, döşeme, düğme üreticileri |
| `ui_model.h`, `ui_cmd.h` | Ölçüm verisi ve komut geri çağrıları (donanım arayüzü) |
| `screens/scr_*.c` | Her ekran için bir dosya: `create()`, `update()`, `destroy()` |
| `assets/` | `font_dseg7_*.c`, UI font glifleri, A8 ikonlar |

Uygulama sırası:

1. Tema, fontlar ve `ui_input` (encoder yalnız LVGL grubuna olay verir). Simülatörde odak hareketini doğrula.
2. `ui_digits` ve `ui_digits_selftest` ekranı (bölüm 7.5).
3. Ortak widget'lar ve `ui_nav` tablosu.
4. Ekranlar: SPLASH, MAIN\_MENU, LOAD, BATT\_TYPE, BATT\_SETUP, BATT\_RUN, DLG\_STOP, BATT\_RESULT, BATT\_GRAPH, BRANCH, SETTINGS, PROTECTION, CALIB, SYSINFO, FAULT.
5. `ui_model` bağlantısı: simülatörde sahte veri üretici, sonra gerçek ölçümler.

Kabul kriterleri:

- Bölüm 6'daki her satır yalnız encoder ile (döndür, kısa bas, uzun bas) gerçekleştirilebilir; dokunma veya başka giriş gerekmez.
- Hiçbir ekranda odaklanabilir öğesiz durum yoktur; hiçbir yerde encoder ile çıkılamayan durum kalmaz.
- Bölüm 7.1'deki tüm dijital alanlar `lv_buttonmatrix` ile üretilmiştir; bu alanlarda `lv_label` yoktur.
- Ekran metinleri bu belgedeki İngilizce karakter dizileriyle birebir aynıdır.
- Ekran değişiminde silinen ekranların belleği geri alınır; 10 dakikalık gezinmeden sonra `lv_mem_monitor` boşluk kaybı göstermez.
- Ölçüm yenilemesi sırasında encoder dönüşü kare atlamadan işlenir (simülatörde 30 FPS altına düşmez).
- Ekran görüntüleri, verilen 16 görselle yerleşim ve renk bakımından karşılaştırıldığında farklar bölüm 11'de listelenenlerle sınırlıdır.

## 11. Açık noktalar ve varsayımlar

Görsellerde bulunmayan veya okunamayan yerler için aşağıdaki kararlar verildi; her biri değiştirilebilir ve değişirse ilgili bölüm güncellenmelidir.

| No | Konu | Alınan karar |
| --- | --- | --- |
| 1 | Yük ekranlarında setpoint girişi görselde yok | Ekrana SET satırı \[EK\] eklendi (8.3). Alternatif: setpoint'i START / STOP düğmesine uzun basarak açılan ayrı bir ekranda girmek |
| 2 | `SCR_FAULT` ve `DLG_STOP` güncel görsellerde yok | Önceki sürümden alınıp \[EK\] olarak eklendi; koruma tetiklenmesi ve test durdurma için gereklidir |
| 3 | Görsellerdeki bazı yazılar bozuk (örn. düğme etiketleri, cut-off değeri 2.000 V) | Metinler bu belgedeki karakter dizileriyle alınır; cut-off varsayılanı Li-ion için 3.000 V |
| 4 | CR görselinde düğme STOP, diğerlerinde START | Dört mod aynı ekrandır; STOP yalnız çıkış ON iken gösterilir |
| 5 | Calibration adımları ve aralıkları | 8.12'deki liste öneridir; gerçek ölçüm zincirine (INA226 / INA228, op-amp) göre değiştirilmelidir |
| 6 | Koruma sınırları | OVP 40.5 V gibi değerler 40 V / 20 A / 200 W anma değerlerinin üstündedir; donanım dayanımına göre doğrulanmalıdır |
| 7 | Fan control satırındaki › | Alt ekran değil, inline seçenek düzenlemesi olarak yorumlandı |
| 8 | Dört MOSFET kolunun ayrı setpoint kontrolü | Arayüz yalnız toplam setpoint ister; kolların dağıtımı ve Q1–Q4 verisi `ui_model` üzerinden gelir, kol başına düzenleme yoktur |
| 9 | DSEG7 fontu | Lisansı SIL OFL'dir; görsellerdeki rakam stiline en yakın açık font olarak seçildi. Farklı bir yedi segment font kullanılırsa `cell_w` oranı yeniden ayarlanır |
| 10 | Data Graph ana menüden açıldığında | Son testin kaydını gösterir; kayıt yoksa "No data" yazar |

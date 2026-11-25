# Ev enerji sayacı → Home Assistant

A whole-home energy meter for Home Assistant: an SCT-013 current clamp on an ESP32 running ESPHome
(current, estimated apparent power, energy). Circuit after the Instructables project "Simple Arduino Home
Energy Meter", adapted from a 5 V Arduino Uno to a 3.3 V ESP32. **Not yet tested on hardware**; the YAML
passes `esphome config` only. The rest of this README is in Turkish.

Instructables "Simple Arduino Home Energy Meter" projesindeki CT devresinin aynısı, ancak
Arduino Uno yerine **ESP32 + ESPHome** ile. Uno'da Wi-Fi yok; ESP32 aynı ölçümü yapar ve
HA'ya şifreli native API ile kendiliğinden bağlanır (MQTT ya da USB gerekmez).

## Malzeme

| Parça | Not |
|---|---|
| ESP32 DevKit (esp32dev) | ~150–250 TL |
| SCT-013-000 (100 A : 50 mA) | Akım çıkışlı model; içinde burden **yok** |
| Burden direnci **22 Ω** (R3) | Orijinalde 56 Ω; o değer 5 V Uno içindi, aşağıya bak |
| 2 × 100 kΩ (R1, R2) | 3,3 V'u ikiye bölen öngerilim (1,65 V) |
| 10 µF elektrolitik (C1) | Öngerilim noktası ile GND arasına |
| 3,5 mm dişi jak veya klemens | CT fişi için |

## Bağlantı

```
3V3 ──R1(100k)──┬──R2(100k)── GND
                │
                ├── C1 (10µF, + bu tarafa) ── GND
                │
                └── CT ucu 1
                    CT ucu 2 ──┬── GPIO34
                               │
         R3 (22Ω) CT'nin iki ucu arasına (paralel)
```

- ADC pini **GPIO34** (ADC1). ADC2 pinleri Wi-Fi açıkken çalışmaz.
- 5 V'a **bağlama**: ESP32 ADC'si en çok 3,3 V kaldırır.

### Neden 22 Ω?
SCT-013-000 oranı 2000:1. ESP32 ADC'si (12 dB zayıflatma) yaklaşık 0,15–2,45 V arasında
doğrusal; 1,65 V öngerilim etrafında güvenli salınım ≈ ±0,8 V.

- 22 Ω: 0,8 V / 22 Ω = 36 mA tepe sekonder → 72 A tepe primer → **~51 A RMS** üst sınır.
- 56 Ω (orijinal): aynı hesapla ~20 A RMS'de kırpılır → ev ana hattı için yetersiz.

Evin sigortası 63 A ise 18 Ω kullan ve YAML'da `ct_amps_per_volt` = 2000/18 = 111.1 yap.

**Alternatif:** SCT-013-030 (30 A / 1 V, burden içinde) alırsan R3'ü takma ve
`ct_amps_per_volt: "30"` yap. 30 A üstünde doymaya girer.

## Kurulum

1. `esphome/secrets.yaml.example` dosyasını `esphome/secrets.yaml` adıyla kopyala ve dört değeri kendi
   değerlerinle doldur: `wifi_ssid`, `wifi_password`, `fallback_ap_password` ve `api_encryption_key`.
   `secrets.yaml` `.gitignore`'dadır. Örnek dosyadaki `CHANGE_ME_…` değerleri yer tutucudur; doldurmadan
   `esphome config` hata verir.
   `api_encryption_key` 32 baytlık base64 bir anahtardır; şu komutla üret: `openssl rand -base64 32`
2. ESP32'yi USB ile bilgisayara tak ve ilk yüklemeyi yap:
   ```bash
   cd esphome && uv tool run --from esphome esphome run energy-meter.yaml
   ```
   İlk derleme ESP-IDF araç zincirini indirir (~1 GB, 5–10 dk). Sonraki yüklemeler Wi-Fi
   üzerinden (OTA) yapılabilir.
3. Home Assistant (örneğin `http://homeassistant.local:8123`) → Ayarlar → Cihazlar ve Hizmetler.
   "ESPHome: energy-meter bulundu" çıkarsa Yapılandır de. Çıkmazsa (Docker'da mDNS
   ağı görmüyorsa) **Entegrasyon ekle → ESPHome → host: `energy-meter.local`** veya
   cihazın IP'si, port 6053. Şifreleme anahtarı istenirse `secrets.yaml`'daki
   `api_encryption_key`'i yapıştır.
4. Ayarlar → Panolar → **Enerji** → "Şebeke tüketimi" olarak `sensor.energy_meter_enerji`'yi
   seç. Tarifeyi (TL/kWh) oraya gir; maliyet HA'da hesaplanır.

## Kalibrasyon

1. CT'yi bir uzatma kablosunun yalnız **faz** iletkenine tak (ikisine birden takarsan 0 okur).
2. Bilinen bir rezistif yük çalıştır (ısıtıcı, su ısıtıcı, akkor ampul) ve gerçek akımı
   pens ampermetre ile ya da `P / V` ile hesapla.
3. HA'daki "Akım" değerini oku:
   `yeni ct_amps_per_volt = eski × (gerçek akım / okunan akım)`
4. YAML'ı güncelle, `esphome run` ile tekrar yükle (OTA).
5. Yük yokken "Akım" 0 değilse, `lambda` filtresindeki 0.05 eşiğini gürültü seviyesinin hemen
   üstüne çek.

## Ev panosuna takarken

- Pano çalışmasını **ana şalter kapalıyken** yap; sayaç öncesi (mühürlü) bölüme dokunma.
  Emin değilsen elektrikçiye taktır.
- CT'yi ana şalter **sonrasındaki** tek faz iletkenine tak; nötr ve toprak hariç.
- Üç fazlı evde her faza ayrı CT + ayrı ADC pini (GPIO32, GPIO33, GPIO35) gerekir; YAML'daki
  `adc` + `ct_clamp` bloğunu çoğalt ve güçleri topla.

## Sınırlamalar (dürüst)

- **Gerilim ölçülmüyor.** Güç = akım × 230 V sabit kabul ediliyor. Bu *görünür* güçtür (VA).
  Motor, kompresör, SMPS gibi düşük güç faktörlü yüklerde gerçek watt'tan %10–40 fazla
  gösterebilir. Gerçek W için gerilimi de ölçen bir modül gerekir: en kolayı PZEM-004T v3
  (ESPHome `pzemac`) ya da ATM90E32 tabanlı bir kart (ESPHome `atm90e32`).
- ESP32 ADC çözünürlüğü ≈ 0,07 A/adım (22 Ω'da) → ~0,1–0,2 A (25–50 W) altı güvenilir değil.
- Enerji değeri ESP32'de entegre ediliyor; cihaz kapalıyken tüketim kaydedilmez.
- Bu YAML şema doğrulamasından geçti (`esphome config`, ESPHome 2026.10.2, çıkış kodu 0);
  donanımda **henüz test edilmedi** ve tam derleme yapılmadı.

## Lisans

[MIT](LICENSE). Şebeke geriliminde çalışmak tehlikelidir; sorumluluk kullanıcıya aittir.

# 🧠 Deneyap Kumanda Kolu ile Yön Hafızası Oyunu (Simon Says)

<p align="center">
  <a href="#donanim-ve-baglantilar"><img src="https://img.shields.io/badge/Donan%C4%B1m-Deneyap%20Kart%20(ESP32)-24292e?style=flat-square&logo=espressif" alt="Donanım" /></a>
  <a href="#donanim-ve-baglantilar"><img src="https://img.shields.io/badge/Ekran-Deneyap%20OLED%20(0x7A)-24292e?style=flat-square" alt="OLED" /></a>
  <a href="#donanim-ve-baglantilar"><img src="https://img.shields.io/badge/Giri%C5%9F-Deneyap%20Kumanda%20Kolu%20(0x1B)-24292e?style=flat-square" alt="Joystick" /></a>
  <a href="#donanim-ve-baglantilar"><img src="https://img.shields.io/badge/%C4%B0leti%C5%9Fim-I2C%20Protokol%C3%BC-24292e?style=flat-square" alt="I2C" /></a>
  <a href="#oyun-kurallari-ve-mekanigi"><img src="https://img.shields.io/badge/Oyun-Haf%C4%B1za%20%26%20Refleks-24292e?style=flat-square" alt="Oyun Türü" /></a>
  <a href="#lisans"><img src="https://img.shields.io/badge/Lisans-MIT-24292e?style=flat-square" alt="Lisans" /></a>
</p>
> **Retro arcade tarzı görsel hafıza ve refleks oyunu:** Deneyap Kart, Deneyap I2C OLED ekran ve analog kumanda kolu (joystick) kullanılarak geliştirilen; her turda uzayan rastgele yön dizisini (Yukarı, Aşağı, Sol, Sağ) ezberleyip joystick ile doğru sırayla girmeyi gerektiren interaktif Simon Says klonu hafıza oyunu.

---

<a id="icindekiler"></a>
## 📑 İçindekiler

1. [Proje Özeti ve Oyun Konsepti](#proje-ozeti)
2. [Donanım Mimarisi ve I2C Bağlantıları](#donanim-ve-baglantilar)
3. [Oyun Döngüsü ve Algoritma Mimarisi](#oyun-dongusu)
4. [Kumanda Kolu (Joystick) Eksen ve Yön Eşik Algoritması](#joystick-algoritmasi)
5. [Proje Dosya Yapısı](#dosya-yapisi)
6. [Kurulum ve Çalıştırma Rehberi](#kurulum-ve-calistirma)
7. [Geliştirici](#gelistirici)
8. [Lisans](#lisans)

---

<a id="proje-ozeti"></a>
## 🎮 1. Proje Özeti ve Oyun Konsepti

Bu proje, klasik **Simon Says** hafıza oyununun analog joystick ve OLED ekranla fiziksel donanıma uyarlanmış halidir:

- **Dinamik Dizi Üretimi:** Oyun başlangıçta 1. seviyeden başlar (`level = 1`) ve `randomSeed(analogRead(A0))` ile donanımsal gürültüden faydalanarak tahmin edilemez rastgele yönler (`Sol`, `Sag`, `Yukari`, `Asagi`) üretir.
- **Ezberleme Aşaması (4 Saniye):** Üretilen yön dizisi OLED ekranda seviye bilgisiyle birlikte gösterilir (`Örn: Seviye 3 -> Sol | Yukari | Sag`). Oyuncunun diziyi ezberlemesi için 4 saniye süre verilir ve ardından ekran temizlenir.
- **Giriş ve Doğrulama:** Ekranda `"Giris yapiniz..."` komutu belirdiğinde oyuncu analog kumanda kolunu sırasıyla ilgili yönlere yatırır.
- **Kusursuz Seviye Atlama:** Tüm dizi sırasıyla doğru girildiğinde ekranda `"Tebrikler!"` mesajı çıkar, seviye 1 artar (`level++`), dizi 1 eleman uzar ve yeni tur başlar (Maksimum 10 seviye).
- **Hata Cezası:** Oyuncu dizinin herhangi bir adımında yanlış bir yöne hareket ederse ekranda `"Yanlis! Tekrar dene"` uyarısı çıkar, seviye sıfırlanır (`level = 1`) ve oyun baştan başlar.

---

<a id="donanim-ve-baglantilar"></a>
## 🔌 2. Donanım Mimarisi ve I2C Bağlantıları

Tüm modüller Deneyap ekosisteminin hazır I2C portları üzerinden **SDA / SCL** hatlarıyla birbirine zincirleme (daisy-chain) bağlanmıştır. Ekstra direnç ya da karmaşık kablolama gerektirmez.

| Donanım Bileşeni | Model / Çip | I2C Adresi / Pin | Görev |
| :--- | :--- | :--- | :--- |
| **Deneyap Kart** | ESP32-WROOM-32 | Ana Denetleyici | Oyun mantığı, dizi üretimi ve kontrol |
| **Deneyap OLED Ekran** | 0.96" SSD1306 | `0x7A` (I2C) | Seviye, yön dizisi ve durum mesajları |
| **Deneyap Kumanda Kolu** | 2 Eksen Analog Joystick | `0x1B` (I2C) | Kullanıcı yön girişleri (Sol, Sağ, Yukarı, Aşağı) |
| **Analog Gürültü Pini** | Dahili ADC | `A0` (Floating) | `randomSeed()` için gerçek rastgele tohum |

---

<a id="oyun-dongusu"></a>
## 🔄 3. Oyun Döngüsü ve Algoritma Mimarisi

```
                 +-------------------------------+
                 |     setup(): I2C Başlatma     |
                 |  OLED (0x7A) + Joystick (0x1B)|
                 +-------------------------------+
                                 |
                                 v
                 +-------------------------------+
                 |   randomSeed(analogRead(A0))  |
                 |   generateSequence() [Lvl: 1] |
                 +-------------------------------+
                                 |
                                 v
                 +-------------------------------+
                 |      showSequenceOnOLED()     |
                 |   Yönler Ekranda (4 sn Bekle) |
                 +-------------------------------+
                                 |
                                 v
                 +-------------------------------+
                 |   OLED: "Giris yapiniz..."    |
                 +-------------------------------+
                                 |
                                 v
                 +-------------------------------+
          +----->|    getJoystickDirection()     |<-----+
          |      +-------------------------------+      |
          |                      |                      |
          |            (Bir yöne çekildi mi?)           |
          |             /              \                |
       Hayır          Evet             Hayır            |
          |            /                 \              |
          +-----------+         [Dizideki Sıra Doğru mu?]|
                                 /                     \
                             Evet                      Hayır
                              /                           \
               [Tüm Dizi Bitti mi?]            [HATA: "Yanlis! Tekrar dene"]
                 /              \                   Level = 1;
              Evet             Hayır                userIndex = 0;
               /                  \                 Yeniden Başlat;
      ["Tebrikler!"]        userIndex++;                  |
      level++;                   |                        |
      Yeni Seviye                +------------------------+
           |
           +-------------------------------------------->
```

---

<a id="joystick-algoritmasi"></a>
## 🕹️ 4. Kumanda Kolu Eksen ve Yön Eşik Algoritması

Analog joystick analog 0-1023 aralığında değer üretir. Merkez nötr konumu yaklaşık **512**'dir. İstem dışı küçük titreşimlerin (deadzone) yön olarak algılanmaması için **300 eşik değeri (`threshold`)** kullanılmıştır:

```cpp
const int threshold = 300;

int getJoystickDirection() {
  int x = joystick.xRead();   
  int y = joystick.yRead();  

  if (x < (512 - threshold)) return DOWN;   // X < 212  -> Aşağı
  if (x > (512 + threshold)) return UP;     // X > 812  -> Yukarı
  if (y < (512 - threshold)) return RIGHT;  // Y < 212  -> Sağ
  if (y > (512 + threshold)) return LEFT;   // Y > 812  -> Sol

  return -1; // Merkez pozisyon (Hareket yok)
}
```

> Her hamleden sonra `delay(300)` ve `delay(200)` bekleme süreleri verilerek oyuncunun tek bir yön hareketinin birden fazla kez sayılması (debounce sorunu) engellenmiştir.

---

<a id="dosya-yapisi"></a>
## 📁 5. Proje Dosya Yapısı

```
kumanda_koluyla_hafiza_yarisi/
├── README.md                          # Proje dokümantasyonu ve bağlantı rehberi
└── kumanda_koluyla_hafiza_yarisi.ino  # Deneyap Kart Arduino oyun kaynak kodu
```

---

<a id="kurulum-ve-calistirma"></a>
## 🚀 6. Kurulum ve Çalıştırma Rehberi

### 1. Gereksinimler
- [Arduino IDE](https://www.arduino.cc/en/software) (v1.8.19 veya v2.x)
- **Deneyap Kart Kart Paketi:** `Araçlar > Kart > Kart Yöneticisi` kısmından *Deneyap Geliştirme Kartları* paketini yükleyin.

### 2. Gerekli Kütüphaneler
Arduino IDE'de **Taslak > Kütüphane Ekle > Kütüphaneleri Yönet...** menüsünden aşağıdaki kütüphaneleri kurun:
- `Deneyap_OLED`
- `Deneyap_KumandaKolu`

### 3. Kodu Yükleme
1. Depoyu bilgisayarınıza indirin:
   ```bash
   git clone https://github.com/kvsrustun/kumanda_koluyla_hafiza_yarisi.git
   ```
2. `kumanda_koluyla_hafiza_yarisi.ino` dosyasını Arduino IDE ile açın.
3. **Araçlar > Kart > Deneyap Kart** seçimini yapın.
4. Doğru COM portunu belirleyin ve **Yükle** butonuna basın.
5. Kart açıldığında OLED ekranda seviye ve ezberlemeniz gereken yönler görüntülenecektir!

---

<a id="gelistirici"></a>
## 👩‍💻 7. Geliştirici

**Kevser ÜSTÜN**  
🎓 Giresun Üniversitesi — Bilgisayar Mühendisliği  
🔗 [GitHub Profilim](https://github.com/kvsrustun)

---

<a id="lisans"></a>
## 📄 8. Lisans

Bu proje [MIT Lisansı](LICENSE) kapsamında açık kaynak olarak paylaşılmıştır.

# Güneş Santrali Verisiyle Nümerik Analiz – Plan

**Amaç:** Nümerik Analiz'den 3 temel konuyu gerçek bir veri setinde, sade ve anlaşılır şekilde uygulamak. CV ve GitHub için.

**İlke:** Az ama anlaşılır. Her defterdeki her satır açıklanabilir olmalı. Gösteriş için ekstra yöntem, ileri teori veya gereksiz grafik yok.

---

## 1. Veri

Kaggle – *Solar Power Generation Data* (Ani Kannal). Hindistan'da iki güneş santrali, 15 Mayıs – 17 Haziran 2020, 15 dakikada bir ölçüm. Bu projede sadece **Plant 1** kullanılıyor.

| Dosya | Kullanılan sütunlar |
|---|---|
| `Plant_1_Generation_Data.csv` | `DATE_TIME`, `SOURCE_KEY` (inverter), `AC_POWER` (kW), `TOTAL_YIELD` (kWh, sürekli artan sayaç) |
| `Plant_1_Weather_Sensor_Data.csv` | `DATE_TIME`, `IRRADIATION` (güneş ışınımı, kW/m²), `MODULE_TEMPERATURE` (°C) |

**Bilinmesi gerekenler**
- Tarih formatları farklı: Üretim dosyası `%d-%m-%Y %H:%M`, sensör dosyası `%Y-%m-%d %H:%M:%S`.
- Bazı ölçümler eksik. Analizlerde sadece 04:00–20:00 arasındaki 65 noktanın hepsi mevcut olan günler kullanılır.
- Gece güç 0; üretim 06:00–18:30 arasında.
- Enerji referansı: `TOTAL_YIELD` farkı (inverterin kendi sayacı). `DAILY_YIELD` her gün düzgün sıfırlanmadığı için kullanılmaz.

## 2. Kurallar

1. **Yöntemler NumPy ile sıfırdan yazılır**, sonra SciPy/NumPy'nin hazır fonksiyonuyla aynı sonucu verdiği kontrol edilir.
2. **Kısa defterler:** Her defter yaklaşık 10–15 hücre. Her kod hücresinden önce ne yapıldığını anlatan 1–3 cümlelik Türkçe açıklama.
3. **Önce basit test:** Yöntem, sonucu bilinen bir örnekte denenir, sonra veriye uygulanır.
4. **Bir iki grafik yeterli.** Her defterin sonunda 3–5 maddelik "Sonuç" bölümü.
5. Defter çalıştırılmış ve çıktılarıyla kaydedilmiş olarak teslim edilir. CSV'ler defterle aynı klasörde.

---

## 3. Defterler

| # | Defter | Konu | Veri sorusu |
|---|---|---|---|
| 1 | `01_integrasyon.ipynb` | Trapez, Simpson, Romberg | Günde kaç kWh enerji üretildi? |
| 2 | `02_en_kucuk_kareler.ipynb` | Gauss eliminasyonu, en küçük kareler | Panel ısınınca verim ne kadar düşüyor? |
| 3 | `03_kok_bulma.ipynb` | Bisection, Newton, secant | Günün enerjisinin yarısı saat kaçta üretildi? |

---

### Defter 1 – Sayısal integrasyon

**Not:** Daha önce hazırlanan `gunes_integrasyon.ipynb` bu defterin geniş halidir. Onu **sadeleştirerek** bu defteri oluştur: sayısal türev bölümünü ve periyodik fonksiyon testini çıkar.

**Adımlar**
1. Veriyi yükle; tek bir inverter ve tek bir gün için güç eğrisini çiz.
2. Trapez, Simpson ve Romberg fonksiyonlarını yaz.
3. Test: $\int_0^1 e^x\,dx = e - 1$. Adım boyu küçüldükçe hatanın nasıl azaldığını log-log grafikte göster (trapez $h^2$, Simpson $h^4$).
4. Veri: Her inverter-gün için 04:00–20:00 integralini üç yöntemle al, `TOTAL_YIELD` farkıyla karşılaştır. Medyan göreli hata tablosu.
5. Seyreltme: Veriyi 15 dk → 30 dk → 60 dk → 120 dk aralıklara seyrelt (her 2., 4., 8. noktayı al). Trapez ve Simpson hatası nasıl büyüyor? Tek grafik.
6. Sonuç.

**Beklenen sonuçlar**
- 15 dakikalık veride medyan hata: trapez **≈ %0,28**, Simpson ≈ %0,43, Romberg ≈ %0,50. Trapez en iyisi.
- Seyreltince hata büyüyor: 30 dk'da ≈ %0,5–1,3, 60 dk'da ≈ %2–3.

**Basit açıklama (sonuç bölümüne):** Simpson ve Romberg fonksiyonun pürüzsüz olduğunu varsayar. Bu veride bulutlar yüzünden güç eğrisi sık sık zıplıyor, o yüzden yüksek mertebeli yöntemler avantajını kaybediyor. Ayrıca eğrinin iki ucu sıfır, bu da trapezin işine yarıyor.

**Mülakat soruları ve kısa cevaplar**
- *Trapez kuralı nedir?* Eğriyi noktalar arasında düz çizgilerle birleştirip altındaki yamukların alanlarını toplar. Hatası $O(h^2)$.
- *Simpson neden genelde daha iyi?* Üç noktadan parabol geçirir; hatası $O(h^4)$. Ama bu, fonksiyonun 4. türevinin küçük olmasına bağlı.
- *Romberg nedir?* Farklı adım boylarıyla hesaplanan trapez sonuçlarını Richardson ekstrapolasyonuyla birleştirip hatanın baş terimini yok eder.
- *O zaman bu veride neden trapez kazandı?* Veri pürüzsüz değil (bulutlar). Yüksek mertebeli yöntemlerin varsayımı sağlanmıyor.
- *Referansı nereden aldın?* İnverterin kendi enerji sayacından (`TOTAL_YIELD`).

---

### Defter 2 – En küçük kareler ve Gauss eliminasyonu

**Veri sorusu:** Güneş panelleri ısındıkça daha az verimli çalışır. Bu kayıp ne kadar? Model:
$$P = a\,G + c\,G\,(T - 25)$$
$P$: santralin toplam gücü (22 inverterin toplamı), $G$: ışınım, $T$: panel sıcaklığı. **Sıcaklık katsayısı** $\beta = -c/a$: Panel 25 °C'nin üstünde her 1 °C ısındığında gücün yüzde kaç düştüğünü söyler.

Elimizde binlerce ölçüm (denklem), ama sadece 2 bilinmeyen ($a$, $c$) var. Bu bir **aşırı belirli sistem**; tam çözümü yok, en küçük kareler çözümünü arıyoruz.

**Adımlar**
1. Üretim ve sensör verisini zamana göre birleştir. $G > 0{,}05$ olan satırları al (gece verisini çıkar).
2. Kısmi pivotlamalı **Gauss eliminasyonu** fonksiyonu yaz. Küçük bir 3×3 sistemde `np.linalg.solve` ile karşılaştır.
3. Pivotlamanın neden gerekli olduğunu göster: $\begin{bmatrix}10^{-17} & 1\\ 1 & 1\end{bmatrix}x = \begin{bmatrix}1\\2\end{bmatrix}$ sistemini pivotlamalı ve pivotlamasız çöz.
4. $A = [\,G,\ G(T-25)\,]$ matrisini kur. **Normal denklemleri** $A^TA\,x = A^Ty$ kendi Gauss eliminasyonunla çöz. `np.linalg.lstsq` ile karşılaştır.
5. $a$, $\beta$ ve $R^2$'yi hesapla. Gerçek ve modelin tahmin ettiği gücü aynı grafikte çiz (saçılım grafiği yeterli).
6. Sonuç: Panel 25 °C'den 60 °C'ye ısınınca güç yüzde kaç düşüyor?

**Beklenen sonuçlar**
- $\beta \approx$ **0,0046 /°C** (yani her derece için ≈ %0,46 kayıp). Silikon panellerin bilinen değeri %0,4–0,5 civarındadır, sonuç bununla uyumlu.
- $R^2 \approx$ **0,98**.
- 25 → 60 °C: ≈ %16 güç kaybı.

**Mülakat soruları ve kısa cevaplar**
- *Aşırı belirli sistem nedir?* Denklem sayısı bilinmeyen sayısından fazla. Genelde tam çözüm yoktur; hataların kareleri toplamını en küçük yapan çözüm aranır.
- *Normal denklemler nereden gelir?* $\|Ax - y\|^2$'nin $x$'e göre türevi sıfıra eşitlenince $A^TA\,x = A^Ty$ çıkar.
- *Kısmi pivotlama neden?* Çok küçük bir sayıya bölmek yuvarlama hatalarını büyütür. Her adımda sütundaki en büyük elemanı pivot yaparak bunu önleriz.
- *Normal denklemlerin dezavantajı?* $A^TA$'nın koşul sayısı $A$'nınkinin karesidir; kötü koşullu problemlerde hassasiyet kaybı olur. Bu yüzden pratikte QR tercih edilir. (Bu problemde $\kappa(A) \approx 87$, yani sorun yok.)
- *$R^2$ ne demek?* Güçteki değişimin ne kadarının modelle açıklandığı. 0,98 = %98.

---

### Defter 3 – Kök bulma

**Veri sorusu:** Günün toplam enerjisinin **yarısı saat kaçta** üretilmiş oldu? Açık bir günde öğlen civarı olmalı; sabah bulutluysa daha geç, öğleden sonra bulutluysa daha erken.

$E(t)$: sabah 04:00'ten $t$ anına kadar üretilen enerji. Çözülecek denklem:
$$F(t) = E(t) - \tfrac12 E_{\text{gün}} = 0$$
Güzel bir detay: $F'(t) = P(t)$, yani **Newton yönteminin ihtiyaç duyduğu türev, ölçülen güçtür**.

**$E(t)$ nasıl hesaplanır (basit yol):** Güç noktalarını düz çizgilerle birleştir (`np.interp`). $E(t)$ bu çizgilerin altındaki alandır. Bu, Defter 1'deki trapez kuralının birikimli halidir. Ölçüm noktalarında $E$ = birikimli trapez toplamı; iki ölçüm arasında $E(t)$ bir paraboldür ve tam olarak hesaplanabilir.

**Adımlar**
1. Bisection, Newton ve secant fonksiyonlarını yaz. Her biri kökü ve her iterasyondaki tahminleri döndürsün.
2. Test: $x^3 - 2 = 0$ (kök $\sqrt[3]{2}$). İterasyon sayılarını karşılaştır.
3. Veri: Plant 1 santral toplamı için bir gün seç, $E(t)$ ve $F(t)$'yi çiz, kökü üç yöntemle bul.
4. Her yöntemde hatanın iterasyonlara göre nasıl azaldığını tabloda göster (bisection her adımda yarıya iner, Newton'da doğru basamak sayısı her adımda kabaca ikiye katlanır).
5. Newton'u gece saatinden (ör. 05:00) başlat: Güç 0 olduğu için sıfıra bölme oluşuyor. Bu yüzden iyi bir başlangıç noktası gerektiğini göster.
6. Tam veri olan bütün günler için yarı enerji saatini hesapla, histogram çiz.

**Beklenen sonuçlar**
- Yarı enerji saati: medyan **≈ 12:10**, günlere göre ≈ 11:00 – 13:15 arası.
- Bisection [8, 16] aralığından başlayınca 10⁻¹⁰ hassasiyet için ≈ 37 iterasyon; Newton öğlenden başlayınca 4–5 iterasyon.

**Mülakat soruları ve kısa cevaplar**
- *Bisection nasıl çalışır?* İşaret değiştiren bir aralığı her adımda ikiye böler. Her zaman yakınsar, ama yavaştır (lineer, oran 1/2).
- *Newton nasıl çalışır?* Teğet doğrunun sıfırını yeni tahmin olarak alır: $x_{k+1} = x_k - F(x_k)/F'(x_k)$. Kökün yakınında çok hızlıdır (kuadratik), ama türev gerekir ve kötü başlangıçta ıraksayabilir.
- *Secant nedir?* Türev yerine son iki noktadan geçen doğrunun eğimini kullanan Newton. Mertebesi ≈ 1,618.
- *Newton'u gece başlatınca neden bozuldu?* $F'(t) = P(t) = 0$, yani teğet yatay; sıfırın kesiştiği nokta yok.
- *Hangisini seçerdin?* Güvenilirlik için bisection, hız için Newton; pratikte ikisini birleştiren yöntemler kullanılır.

---

## 4. GitHub

```
gunes-numerik-analiz/
├── README.md
├── notebooks/
│   ├── 01_integrasyon.ipynb
│   ├── 02_en_kucuk_kareler.ipynb
│   └── 03_kok_bulma.ipynb
└── data/        # .gitignore'a ekle; CSV'ler repoya konmaz
```

**README:** Kısa proje tanımı, Kaggle veri linki, her defter için bir grafik ve bir cümlelik bulgu:
- "15 dakikalık veride trapez kuralı günlük enerjiyi %0,3 hatayla buluyor ve bu veride Simpson ile Romberg'den daha isabetli."
- "En küçük kareler ile bulunan sıcaklık katsayısı β ≈ 0,0046/°C, silikon panellerin bilinen değeriyle uyumlu."
- "Newton yöntemi, türev olarak ölçülen gücü kullanarak yarı enerji saatini 4–5 iterasyonda buluyor."

**CV satırı:** *Nümerik Analiz Projesi (Python, NumPy): Güneş santrali verisinde sayısal integrasyon (trapez, Simpson, Romberg), en küçük kareler (Gauss eliminasyonu) ve kök bulma (bisection, Newton, secant) yöntemlerinin sıfırdan uygulanması ve karşılaştırılması.*

---

## 5. Yeni sohbette kullanım

Yeni sohbete şunları ekle: bu `plan.md`, CSV dosyaları (en az `Plant_1_Generation_Data.csv` ve `Plant_1_Weather_Sensor_Data.csv`), ve `gunes_integrasyon.ipynb`. Sonra şunu yaz:

> "plan.md'deki Defter 1'i uygula. Bölüm 2'deki kurallara uy, sonuçları beklenen sonuçlarla karşılaştır, çalıştırılmış .ipynb olarak teslim et."

Her sohbette bir defter iste. Her defteri teslim aldıktan sonra satır satır oku ve mülakat sorularını kendi cümlelerinle cevaplayabildiğinden emin ol.

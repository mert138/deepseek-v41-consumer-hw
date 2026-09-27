# DeepSeek V4.1 Flash + Colibrì — Tüketici Donanımında Optimizasyon Günlüğü

**Durum:** 🚧 Devam ediyor — 0.15 tok/s'den başlayıp 1-2 tok/s hedefliyoruz.

Colibrì'nin kendi ölçümleri kurumsal donanımda (çift enterprise NVMe, striped) yapılmış.
Bu repo, **tek bir tüketici NVMe'si olan sıradan bir masaüstünde** aynı modeli
çalıştırmaya çalışırken karşılaşılan gerçek darboğazları ve çözümleri belgeliyor.

## Donanım

| Bileşen | Detay |
|---|---|
| CPU | Intel i7, 12. nesil (tam model TBD) |
| RAM | 64 GB |
| GPU | RTX 4060 8GB — **kullanılmıyor**, bkz. GPU notu aşağıda |
| SSD (model) | NVMe, "MLD M300" ailesi, 1TB, tek M.2 yuvası |
| İkincil disk | 1TB HDD, 7200RPM (model için kullanılmıyor, sadece diğer dosyalar) |
| İşletim sistemi | Windows 11, native (WSL2 değil) |

## Model & Motor

- **Model:** DeepSeek-V4.1-Flash — resmi checkpoint (fp8 dense + fp4 experts), 552B toplam parametre, disk üzerinde 510 GB
- **Motor:** [Colibrì](https://github.com/JustVugg/colibri) v1.12.1

## Başlangıç noktası

**0.15 tok/s** — Görev Yöneticisi'nde inceleme: GPU %0, CPU %11, disk okuma 300-580 MB/s arası.

## Test edilen değişiklikler

| # | Değişiklik | Sonuç | Karar |
|---|---|---|---|
| 1 | McAfee gerçek zamanlı taramayı geçici kapatma | Ölçülebilir fark yok | ❌ Elendi |
| 2 | Disk boş alanı: 2-3 GB → 85 GB (1TB'ta) | Motorun kendi raporu: **0.95 GB/s** disk okuma (öncesinde Görev Yöneticisi tahminiyle 300-580 MB/s) | ✅ Büyük katkı — en etkili değişiklik şu ana kadar |
| 3 | Güç planı: "Nihai Performans" | Uygulandı, etkisi izole ölçülmedi henüz | ⏳ Beklemede |
| 4 | NVMe "aygıtı kapatmaya izin verme" kapatma | Bu sürücüde Aygıt Yöneticisi'nde bu sekme yok/bulunamadı | ⏳ Beklemede |
| 5 | PCI Express Link State Power Management → Kapalı | Uygulanma durumu teyit edilmedi | ⏳ Beklemede |
| 6 | Disk sıcaklığı kontrolü | Yük altında sabit 44°C | ❌ Isınma/throttling elendi |
| 7 | OMP_NUM_THREADS=8 (i7-12700F'nin 8 P-çekirdeği) | disk 0.86 GB/s → 15 tok · 113s · 0.13 tok/s | ❌ Ayarsız halden kötü |
| 8 | OMP_NUM_THREADS=20 (tüm P+E thread) | disk 0.69 GB/s → 15 tok · 153s · 0.10 tok/s | ❌ 8'den de kötü |
| 9 | GPU/CUDA hızlandırma | Araştırıldı | ❌ Mevcut değil — bkz. GPU notu |

**Not (#7-8) — güncelleme:** Soğutma sonrası temiz test (OMP_NUM_THREADS sıfırlanmış,
sıcaklık 41°C başlangıç / 46-48°C çalışırken) yine farklı bir sonuç verdi: 0.78 GB/s.
Isınma kesin olarak elendi (46-48°C, throttle eşiğinin çok altında), ama disk hızı
dört testte de (0.95 / 0.86 / 0.69 / 0.78 GB/s) düzenli bir örüntü izlemiyor.

**Metodoloji notu:** 15-16 token'lık yanıtlar gerçek hızı ölçmek için çok kısa —
prefill süresi (17-33s) toplam sürenin büyük kısmını yutuyor ve DSpark'ın "son 10
taslak" penceresi bu kadar kısa bir yanıtta hiç dolmuyor. Yani her ölçüm aslında
motorun ısınma/kalibrasyon evresini yakalıyor olabilir, kararlı-durumu değil.
Bundan sonraki testler en az 300-400 token'lık yanıtlarla yapılacak.

## `V41_STATS=1` ile ölçülen gerçek motor verisi

Tek bir kısa mesaj ("merhaba", 16 token yanıt) için:

```
expert I/O: 2246 misses, 42226.2 MB in 44.49s (0.95 GB/s)
prefill: 19270.7 MB, disk 17.16s, matmul 5.57s, wall 28.64s
toplam: 16 tok · 107s · 0.15 tok/s
DSpark: 2 of 10 drafts accepted this turn (20%, eşik %60 altı → spekülatif kod duraklatıldı)
```

**Gözlem:** Disk I/O toplamda sadece 44.49s tutuyor ama toplam süre 107s —
yaklaşık **62 saniye diskle açıklanamıyor**. Şüpheliler: CPU'nun tam kullanılmaması
(matmul tek/az thread'de çalışıyor olabilir) ve/veya DSpark'ın düşük kabul oranıyla
boşa harcanan hesaplama. 16 token çok küçük bir örneklem, daha uzun bir yanıtla
tekrar ölçülmesi gerekiyor.

## GPU / CUDA notu

Colibrì'nin `deepseek_v41` motorunda şu an (v1.12.1) CUDA/GPU desteği **yok**.
Resmi v1.11.0 sürüm notlarında ekip, expert okumalarını matmul hesaplamasının
arkasına gizleyen iki farklı yöntemi denediklerini, ikisinin de daha yavaş
çıktığını ve kaldırdıklarını belirtiyor (`docs/deepseek-v41.md`). RTX 4060 şu an
bu model için devre dışı.

## Referans sayılar (Colibrì ekibinin kendi ölçümü, v1.11.0 sürüm notları)

- 5 mesajlık bir sohbet oturumu: **1.14 – 1.58 tok/s**
- Bizim hedefimiz de bu aralık.

## Kritik bulgu: DSpark ile dil bozulması (400+ kelimelik test)

Uzun bir yanıt istendiğinde (1491 token, 6929s, **0.22 tok/s** — kısa testlerden daha iyi,
önbellek ısınması teyit edildi) iki önemli şey ortaya çıktı:

1. **Dil bozulması:** Yanıtın tamamı boyunca standart Türkçe'ye Kırım Tatarcası/Kiril-etkili
   yazımlı kelimeler karışmış ("suniy intellekt", "deñişiklikler", "añlatacaqmız", "meqalesi",
   "Con Makkarti" gibi) — tek seferlik değil, sistematik. Hız sorunundan **önce** çözülmesi
   gereken bir kalite sorunu. `DRAFT=0` ile DSpark'ı kapatma denemesi başarısız oldu (yanlış
   değişken adı, DSpark log'larda hâlâ aktif görünüyor) — doğru kapatma yöntemi hâlâ aranıyor.
2. **Düşük kabul oranı bir arıza değilmiş:** DeepSeek V4.1 Flash'ın resmi (SGLang) belgelerine
   göre DSpark'ın kabul oranı iş yüküne bağlı — matematik/kodda yüksek, **açık uçlu sohbette
   düşük**, dokümante edilmiş normal davranış. Bizim testlerimiz hep sohbet tarzı olduğu için
   %10-30 kabul oranı beklenen bir şey, araştırmaya gerek yok.
3. **Ama bu, dil bozulmasını daha şüpheli kılıyor:** Spekülatif kod matematiksel olarak
   "lossless" olmalı — kabul oranı ne olursa olsun reddedilen taslaklar çıktıya sızmamalı.
   Sızıyor olması, colibri'nin bu motordaki "reddet" mantığında gerçek bir hataya işaret
   ediyor olabilir.
4. **Açıklanamayan süre büyüyor:** disk I/O 1653s, toplam süre 6929s → ~5275s (1.5 saat) hâlâ
   diskle açıklanamıyor. Şüpheli: yüzlerce reddedilen taslağın her biri boşa doğrulama turu
   demek olabilir.

**Sıradaki test:** Upstream'e hata raporu açıldı (bkz. "Açık Sorun" bölümü) — DSpark'ı
güvenilir şekilde kapatmanın yolu netleşene kadar, odak dil bozulmasını çözmeye kaydı.

## Açık Sorun: Dil Bozulması (upstream'e bildirildi)

`colibri` reposuna hata raporu açıldı: **[issue linki buraya eklenecek]**

Özet: Uzun Türkçe yanıtlarda sistematik olarak Kırım Tatarcası/Kiril-etkili kelimeler
karışıyor (bkz. yukarıdaki örnekler). DSpark'ın kabul oranı düşük olsa da (bu normal,
iş yüküne bağlı), reddedilen taslakların çıktıya sızması "lossless" spekülatif kod
garantisine aykırı — gerçek bir hata gibi duruyor. `DRAFT=0` denemesi işe yaramadı,
`/help`'te DSpark'ı kapatan bir komut yok. Cevap beklerken bizim odağımız da bu soruna
kaydı.

## Güncel durum özeti

| Metrik | Değer |
|---|---|
| Başlangıç hızı | 0.15 tok/s |
| En iyi stabil hız (uzun yanıt, önbellek ısınmış) | **0.22-0.23 tok/s** |
| Disk hızı (motor ölçümü) | 0.78-1.02 GB/s arası (dalgalı, muhtemelen normal varyans) |
| Isınma | Elendi (46-48°C, güvenli aralık) |
| **Açık blocker** | **Dil bozulması — hız optimizasyonundan önce çözülmeli** |

## Kırılma noktası: kök neden bulundu (tokenizer, DSpark değil)

Colibri'nin geliştiricisi issue'ya yanıt verdi ve gerçek kök nedeni buldu:
**DSpark değil, tokenizer hatası.** Colibri'nin DeepSeek V4/V4.1 için kullandığı
pre-tokenizer, Türkçe apostrof+ek yapılarını yanlış bölüyor:

| İfade | Referans (doğru) | Eski Colibri (yanlış) |
|---|---|---|
| `1956'da` | `1956` + `'` + `da` | `1956` + `'d` + `a` |
| `Türkiye'de` | `Türkiye` + `'` + `de` | `Türkiye` + `'d` + `e` |
| `1950'deki` | `'` + `de` + `ki` | `'d` + `eki` |

Model eğitiminde bu yanlış bölünmüş diziler hiç görülmemiş — özellikle model kendi
önceki Türkçe cevabını (çok turlu sohbette) yeniden tokenize ederken eğitimde
görmediği dizilimlerle karşılaşıp bozuluyor. Bu, bizim "greedy/düşük sıcaklıkta daha
kötüleşiyor" bulgumuzu aslında daha iyi açıklıyor: model eğitimde hiç görmediği bir
girdiyle karşılaşınca en yüksek olasılıklı tahmini bile güvenilir olmuyor — bellek
hatası değil, eğitim-dışı girdiye verilen kafası karışık bir cevap. `DRAFT=0`'ın işe
yaramamasının sebebi de netleşti: o GLM engine'e ait bir ayarmış, DeepSeek V4.1 ile
ilgisi yok. Doğru DSpark kapatma değişkeni: **`V41_DSPARK=0`** (geliştirici tarafından
teyit edildi; kendi testlerinde reddedilen taslakların token'ları değiştirmediğini,
yani DSpark'ın doğrudan suçlu olduğuna dair kanıt olmadığını belirtti).

**Düzeltme: PR #1775** — pre-tokenizer'ı referans/HuggingFace tokenizer ile birebir
eşleştiriyor. ~18.000 rastgele string + çok dilli test setinde (V4 ve V4.1) referansla
fark göstermediği bildirildi.

### Kaynaktan derleme süreci

Mevcut çalışan kurulum (`C:\colibri`) dokunulmadan bırakıldı, ayrı bir kopya oluşturuldu:

```bash
git clone https://github.com/JustVugg/colibri.git colibri-source
cd colibri-source
git fetch origin pull/1775/head:tokenizer-fix
git switch tokenizer-fix   # commit 101c6a1
```

Araç zinciri (Windows, MSYS2 UCRT64): `make` (GNU Make 4.4.1), `mingw-w64-ucrt-x86_64-gcc`
(GCC 16.2.0), `mingw-w64-ucrt-x86_64-libgomp`. CUDA 12.6 kurulu ama Colibri'nin Windows/CUDA
dokümantasyonu 12.8+ önerdiği için ve Visual Studio 2022 kurulmak istenmediği için **CPU-only
build** tercih edildi (amaç zaten hız değil, tokenizer doğruluğunu test etmek).

```bash
make deepseek_v41.exe ARCH=native   # → c/deepseek_v41.exe (~759 KB)
```

Bu executable, `coli.cmd`'den farklı, kendi stdin/stdout serve protokolüne sahip:
```
SNAP=C:\deepseekv4 SERVE=1 ./deepseek_v41.exe
# sonra: SUBMIT id slot bytes max_tokens temperature top_p + ham prompt bytes
# engine: READY / STAT / EMAP / ACCEPT / DATA frame'leriyle yanıtlıyor
```
Bunu sürmek için `C:\colibri-source\test_v41.py` adlı bir Python istemcisi yazıldı.

### Şu anki durum

İlk test (150 kelime, `max_tokens=200`) `ACCEPT` aldı ama CPU-only build'de tahmini
üretim süresi ~1.5 saat — pratik değil. **Sıradaki adım:** çok daha kısa bir
`max_tokens` (örn. 25-30) ile apostrof+ek içeren kısa bir promptla hızlı test,
sonra `V41_DSPARK` açık/kapalı karşılaştırması (`V41_STATS=1` ile).

## Genel özet

| Metrik | Değer |
|---|---|
| Başlangıç hızı | 0.15 tok/s |
| En iyi stabil hız (uzun yanıt, önbellek ısınmış, tokenizer düzeltmesi öncesi) | **0.22-0.23 tok/s** |
| Disk hızı (motor ölçümü) | 0.78-1.02 GB/s arası (dalgalı, muhtemelen normal varyans) |
| Isınma | Elendi (46-48°C, güvenli aralık) |
| Dil bozulması kök nedeni | **Bulundu — tokenizer hatası (apostrof+ek), PR #1775 ile düzeltiliyor** |
| **Şu anki blocker** | **PR #1775 build'inin kısa testle doğrulanması (CPU-only, hızlı test bekleniyor)** |

## Sıradaki adımlar

- [x] Tam i7 modelini ve çekirdek/thread sayısını doğrula — **i7-12700F: 8P+4E, 20 thread**
- [x] `OMP_NUM_THREADS` değerini açıkça ayarlayıp test et — **8 ve 20, ikisi de belirgin bir kazanç göstermedi**
- [x] Soğutma sonrası temiz baseline + sıcaklık ölçümü — **46-48°C, ısınma kesin olarak elendi**
- [x] En az 300-400 token'lık bir yanıtla `V41_STATS=1` ölç — **0.22-0.23 tok/s stabil**
- [x] Dil bozulmasını izole et, upstream'e bildir — **kök neden bulundu: tokenizer hatası, PR #1775 ile düzeltiliyor**
- [ ] **PR #1775 build'iyle kısa (max_tokens~25-30) test — bozulma gitti mi?**
- [ ] `V41_DSPARK=0` vs açık karşılaştırması (tokenizer düzeltmesi sonrası DSpark'ın rolü kaldı mı?)
- [ ] Temiz sonuç alınırsa: `C:\colibri`'yi de PR #1775 ile güncelleyip hız+kalite testini birlikte tekrarla
- [ ] PCI Express Link State Power Management ayarını teyit et

---
*Bu belge, gerçek zamanlı bir optimizasyon sürecinin günlüğüdür — sayılar ilerledikçe güncellenecek.*

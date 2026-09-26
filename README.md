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

**Not (#7-8):** Üç ölçüm art arda, monoton şekilde kötüleşti (0.95 → 0.86 → 0.69 GB/s).
Bu thread sayısından çok, arka arkaya üç ağır yükleme/üretimden kaynaklanan bir
ısınma birikimi olabilir — sıcaklık daha önce sadece tek seferlik ölçülmüştü.
Soğutma sonrası temiz tekrar testi gerekiyor (bkz. Sıradaki adımlar).

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

## Sıradaki adımlar

- [x] Tam i7 modelini ve çekirdek/thread sayısını doğrula — **i7-12700F: 8P+4E, 20 thread**
- [x] `OMP_NUM_THREADS` değerini açıkça ayarlayıp test et — **8 ve 20, ikisi de ayarsız halden kötü**
- [ ] **Soğutma sonrası temiz baseline:** 5 dk bekle → sıcaklık ölç → `OMP_NUM_THREADS`'i sıfırla (boş bırak) → tekrar ölç → hemen sonra sıcaklık tekrar ölç (ısınma birikimi hipotezini test etmek için)
- [ ] Daha uzun bir yanıtla `V41_STATS=1` tekrar ölç (DSpark istatistiği için daha iyi örneklem)
- [ ] PCI Express Link State Power Management ayarını teyit et
- [ ] `/brio` modunu dene ve karşılaştır

---
*Bu belge, gerçek zamanlı bir optimizasyon sürecinin günlüğüdür — sayılar ilerledikçe güncellenecek.*

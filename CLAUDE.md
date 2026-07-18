# Sevkiyat Rota Planlayıcı

Tek dosyalık (`index.html`) bir sevkiyat/rota optimizasyon aracı. Sunucu yok, tamamen istemci tarafında çalışıyor.

## Ne yapıyor
- Adres listesi (sevkiyat noktaları) girilir, geocoding ile koordinatlara çevrilir.
- Coğrafi kodlama ve mesafe/süre matrisi için üç sağlayıcı destekleniyor: OpenRouteService (ORS), Mapbox, HERE. Sağlayıcı ve API anahtarları `localStorage`'da tutuluyor (`sevkiyat-provider`, provider key'leri).
- TSP tabanlı rota optimizasyonu: iki başlangıç turu üretilip (`nnConstruct` nearest-neighbor, `ciConstruct` cheapest-insertion) ikisi de `optimizeSeq` (2-opt + forward/reversed or-opt) ile iyileştiriliyor, `solveTSP` ikisinden en iyisini seçiyor; `bestOrder` bunu ORS'nin VROOM sonucuyla (`refine`) karşılaştırıp en iyisini döndürüyor.
- Rota, Leaflet haritası üzerinde çiziliyor (`drawRoute`), duraklar sürükle-bırak ile yeniden sıralanabiliyor.
- Sonuçlar CSV/Excel olarak dışa aktarılabiliyor (`exportToExcel`).
- Açık/koyu tema desteği var, tercih `localStorage`'da saklanıyor.

## Mimari notları
- Tüm mantık `index.html` içinde: inline `<style>` ve `<script>`. Ayrı build adımı, paket yöneticisi yok.
- Harici bağımlılıklar CDN üzerinden: Leaflet (harita), Google Fonts (Inter).
- Sağlayıcıya özel geocode/matrix/geometry fonksiyonları ayrı isimlerle yazılmış (`orsGeocode`/`mbGeocode`/`hereGeocode` vb.) — ortak bir arayüz/adaptör katmanı yok, provider seçimine göre if/else ile yönlendiriliyor.

## Repo / iş akışı kararları
- GitHub: `ademkucuk33-collab/sevkiyat-rota`, `main` branch.
- Kullanıcı birden fazla bilgisayarda çalışıyor; yerel geliştirme yerine GitHub'ı tek kaynak olarak kullanmayı tercih ediyor. Bu yüzden proje bağlamının (kararlar, güncel durum) bilgisayarlar arasında taşınması için bu `CLAUDE.md` dosyası oluşturuldu ve repoya push edildi.
- `gh` CLI bu makinede kurulup (`winget install GitHub.cli`) `ademkucuk33-collab` hesabıyla auth edildi.

## Güncel durum (2026-07-18 itibarıyla)
- Repoda sadece `index.html` + bu `CLAUDE.md` var, başka dosya/klasör yok.
- **Bilinen sorun (çözüldü, izleniyor):** Kullanıcı impargo.de ile karşılaştırdı — aynı 26 adreste impargo 2648 km, bizim uygulama ~2800+ km çıkarıyordu; ayrıca rota bazen bir teslimattan sonra ana yola geri dönüp başka bölgeye atlıyordu. Kök neden: Mapbox/HERE sağlayıcılarında ORS'deki gibi gerçek bir optimizasyon motoru (VROOM) yok, sadece nearest-neighbor kullanılıyordu. Düzeltme: `nnConstruct` + `ciConstruct` (cheapest-insertion) iki başlangıç turu üretilip ikisi de 2-opt/or-opt (artık segment ters çevirmeli) ile iyileştiriliyor, en iyisi seçiliyor (index.html, `solveTSP`/`optimizeSeq`, 2026-07-18 commit). Sentetik testle (Node, rastgele + kümeli senaryolar) doğrulandı, hiçbir denemede eski algoritmadan kötü çıkmadı. **Gerçek adreslerle henüz impargo'ya karşı tekrar ölçülmedi** — kullanıcı sonraki hesaplamada km farkının kapanıp kapanmadığını teyit etmeli.

## Bu dosyanın amacı
Bu dosya, farklı bilgisayarlarda Claude Code ile çalışırken proje bağlamının kaybolmaması için tutuluyor. Önemli bir karar alındığında veya proje durumu değiştiğinde bu dosya güncellenip commit edilmeli.

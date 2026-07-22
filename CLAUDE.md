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
- **Düzeltildi:** HERE sağlayıcısında 26 adresle "HERE matris hatası (400)" alınıyordu. Neden: `hereMatrix` (index.html:694) `regionDefinition: {type:'world'}` + özel `transportMode` ("Flexible mode") kullanıyordu, bu kombinasyon matrisi en fazla 15x100 ile sınırlıyor — 15'ten fazla durakta 400 dönüyor. Düzeltme: `transportMode` yerine önceden tanımlı `profile: 'carFast'` kullanılarak "Profile mode"a geçildi (senkron istekte 500x500'e kadar destekliyor). Hata mesajına HERE'in döndürdüğü gövde de eklendi, ileride benzer sorunlar daha kolay teşhis edilsin diye.

## Yeni özellikler (bu oturum — istemci tarafı, DEMO güvenlik)
- **Giriş/Auth:** Site giriş ekranıyla açılıyor (ayrı `<script>` IIFE, `</body>` öncesi). Kullanıcılar `localStorage['sevkiyat-users-v1']`. Rol: admin/user. Varsayılan `admin/admin1234`.
- **Paylaşımlı hesaplar:** `DEFAULT_USERS` sabiti koda gömülü — linki alan HER cihazda çalışır (localStorage üstüne merge edilir). Yeni kullanıcı: admin panelinde oluştur → "Kullanıcıları kod için kopyala" → çıkan JSON'u `DEFAULT_USERS`'a göm + push. (Backend olmadığı için tek yol bu.)
- **Admin paneli:** kullanıcı ekle/sil/şifre + her kullanıcının **güncel şifresi ve profil bilgileri** görünür (`renderUsers`). Sadece admin görür.
- **Profil:** sağ üstte avatar → modal: foto (canvas ile 96px'e küçültülüp dataURL), ad/telefon/eposta/not, şifre değiştirme. `users[n].profile`.
- **Sağlayıcı/anahtar alanı** sadece **admin**'e görünür (`sbProvider`), normal kullanıcıda gizli.
- **Yerleşim:** API/sağlayıcı + Excel + Ücret artık üstte tam genişlik `.settings-bar` (eski alttaki `formWrap` "Ayarlar" kaldırıldı).
- **Mod bazlı yol tercihi:** En kısa süre → `fastest`/otoban (ORS `preference`, HERE `routingMode=fast`), en kısa mesafe → `shortest`. `renderActive` içinde `pref`, dirCache key'ine dahil.
- **Rota paylaşımı:** "Rotayı Paylaş" → rota verisi (`{v,km,sure,ucret,st,en,p[]}`) base64 olarak `#r=` URL hash'ine gömülür. Link açan kişi **giriş yapmadan** salt-görüntüleme modunda görür (`renderSharedView`, `body.viewer`): harita (düz çizgi rota + markerlar), sıralı liste, KPI'lar, "Excel indir". Auth IIFE `#r=` varsa login'i atlıyor.
- **"Listeyi Temizle"** butonu (adres kutusunu boşaltır).

## Önemli notlar / TODO
- Tüm auth/kullanıcı/profil verisi **istemci tarafında, güvenli DEĞİL** (kod açık, aşılabilir). Satış öncesi **backend** (öneri: Supabase) şart — o zaman kullanıcılar cihazdan bağımsız paylaşılır, anahtarlar korunur.
- Provider API anahtarları da istemci tarafında; her sağlayıcının anahtarı `localStorage['sevkiyat-provider-keys-v1']` içinde ayrı.
- ORS aşırı test edilince günlük/dakikalık ücretsiz kotaya takılabilir; `orsFetch` 20sn timeout + 429 retry içeriyor.

## Bu dosyanın amacı
Bu dosya, farklı bilgisayarlarda Claude Code ile çalışırken proje bağlamının kaybolmaması için tutuluyor. Önemli bir karar/değişiklik olduğunda güncelle + commit et. (Yeni oturumda: önce bu dosyayı oku, tüm projeyi baştan tarama.)

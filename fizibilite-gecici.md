# Geçici Fizibilite Çalışması — US-1/2/3 Senaryoları

> **Geçici çalışma dokümanı.** `senaryolar.md`'deki üç senaryonun mevcut tasarıma
> (gereksinimler v0.3, mimari-claude v0.1, nihai-mimari) etkisini değerlendirir.
> Kararlar gereksinim + mimari dokümanlarına işlendikten sonra bu dosya silinebilir
> veya arşivlenir.

## 0. Yönetici özeti

- **Üç senaryo da yapılabilir.** Hiçbiri araştırma problemi değil; tamamı mevcut
  mimarinin (port-adapter, tek servis katmanı, config-driven) üzerine entegrasyon işi.
- Senaryolar 13 ayrık yeteneğe (C1–C13) ayrışıyor. Çoğu küçük/orta efor.
  En pahalı ikisi: **federasyon modu (C11)** ve **LLM'li otomatik tarif üretimi (C10)**.
- **V1 kapsam kilidi korunabilir** — tek şartla: iki *sözleşme* kararı (implementasyon
  değil) V1'e şimdi girmeli, yoksa sonradan kırılma yaşanır:
  1. Config şemasında agent tanımı **baştan çoğul** yazılmalı (`agents:` listesi,
     V1'de tek eleman).
  2. REST/MCP cevapları **baştan bloklu zarf** dönmeli (llm_text + verbatim blok +
     atıf + usage), pass-through bloklarına yer hazır.
- **Kütüphane fikri üç senaryonun hiçbirinde gerekmedi** → buzdolabı önerisi.
  "İmajı dışa açma" ise zaten FR-10'un kendisi; eksik olan yalnız yayınlama/sürümleme
  operasyonu.
- Üç senaryo tek ipe diziliyor: US-3 (federasyon) "isabet"i US-2'nin kataloğuna,
  "hız"ı US-1'in pass-through/LLM'siz sorgu fikrine dayanıyor. Sıralama bu yüzden
  US-1 → US-2 → US-3 mekanizmaları şeklinde olmalı.

## 1. Yetenek envanteri ve fizibilite

Efor ölçeği (tek kişi, göreli): **S** küçük (günler) · **M** orta (~hafta) · **L** büyük (haftalar).

| # | Yetenek | Senaryo | Fizibilite | Efor | Öneri sürüm |
|---|---|---|---|---|---|
| C1 | Agent profilleri (config'de çoklu named agent; istekte `agent:` seçimi) | US-1 | Kolay — persona map + seçim | S | Şema **V1**, impl V1.5 |
| C2 | Pass-through + bildirimsel routing (`pass_through: [...]`; sonuç LLM'e girmeden zarfa) | US-1, US-3 | Orta — agent döngüsünde dal + zarf sözleşmesi | M | Zarf **V1**, impl V1.5 |
| C3 | SSE stream endpoint (`/chat` stream modu; MCP progress zaten var) | US-1 | Kolay-orta — FastAPI SSE standart | S-M | V1.5 |
| C4 | Inbound webhook (`/hooks/{name}`: HMAC, idempotency, config'te hook→aksiyon; 202+job modeli) | US-1 | Orta — arka plan iş modeli yeni kavram | M | V1.5 |
| C5 | İstek günlüğü (opsiyonel içerikli log, retention; `query_history` tool'u) | US-1 | Kolay — ledger'ın yanına tablo | S | V1.5 |
| C6 | KB'ye geri-besleme (`save_to_kb` builtin; `origin=agent` etiketi; temizlenebilir) | US-1 | Kolay — mevcut ingest'i çağırır | S | V1.5–V2 |
| C7 | Repo özet wiki entegrasyonu (DeepWiki-muadili **harici MCP** + C4 tetiği + C6 yazımı) | US-1 | Reçete işi — çekirdeğe kod girmez | S (döküman/örnek) | V1.5 (examples/) |
| C8 | Konfigüre edilebilir MCP tool projeksiyonu (`expose.tools[]`: ad, açıklama, talimat, hedef: kb_query \| agent \| mcp_proxy \| http) | US-2 | Orta — SDK dinamik tool destekler; 5 generic tool "varsayılan projeksiyon"a dönüşür | M | Statik V1.5, mcp_proxy V2 |
| C9 | Yetenek kataloğu (`describeCapabilities` tool + `/capabilities`; MCP server instructions) | US-2, US-3 | Kolay — config+katalog+KB istatistiği derlemesi | S | V1.5 |
| C10 | Otomatik tarif üretimi (`agent describe` build / `POST /describe` runtime; utility LLM; **manual > auto > fallback** katmanları) | US-2 | Orta — LLM'li tarama; kalite riski var | M | Katalog şeması V1.5, LLM'li üretim V2 |
| C11 | Federasyon/hub modu (banka = `describeCapabilities` sunan MCP — otomatik tespit, ayrıca config işareti yok; katalog federasyonu; banka seçimi → paralel LLM'siz `queryKB` fan-out → füzyon; birleşik atıf `banka:doc`; timeout + kısmi cevap politikası; topoloji V2'de tek seviye hub→banka) | US-3 | Orta-büyük — spoke tarafı bedava (D akışı), iş hub'da | M-L | V2 |
| C12 | Derinlik/döngü guard'ları (`X-Agent-Depth`, max_depth, çevrim tespiti) | US-3 | Kolay ama bugün gereksiz — tek seviyeli hub→banka'da çevrim ancak yanlış config'le oluşur | S | **V3+ (A2A ile)**; timeout/kısmi cevap C11'de kalır |
| C13 | Toplu health (hub `/health` spoke durumlarını toplar) | US-3 | Kolay — degraded raporlama zaten var | S | V2 (C11 ile) |

## 2. Senaryo bazında hüküm

**US-1 (tamamlayıcı beyin plugin'i): YAPILABİLİR — C1+C2+C3+C4+C5+C6+C7.**
En kritik parça pass-through zarfı (C2): "milimetrik hesap LLM'e uğramasın" garantisi
ancak cevap sözleşmesi bloklu tasarlanırsa verilebilir. Bu yüzden zarf V1 sözleşmesine
şimdi girmeli. "1 ay sonra analiz" akışı = C5 (günlük) + C1 (analiz profili) + C6
(bulguları KB'ye yaz) — üçü de config-driven, senaryodaki "al-ver" ortadan kalkıyor.
DeepWiki kısmı (C7) bilinçli olarak çekirdek dışı: harici MCP + webhook reçetesi;
container generic kalır.

**US-2 (kendini tarif eden MCP): YAPILABİLİR — C8+C9+C10.**
Projeksiyon mekaniği (C8) ve katalog (C9) düşük riskli. Riskli tek parça otomatik
tarif kalitesi (C10) — hafifletme: katman zinciri (elle tarif > otomatik > jenerik
fallback) + `--review` çıktısıyla insan onayı. Mimari felsefeyle (env > yaml >
varsayılan) birebir aynı desen.

**US-3 (9 banka + 1 hub): YAPILABİLİR — C11+C13 (C12 → V3+); C9/C10 ve C2'ye bağımlı.**
Spoke tarafı bugünkü tasarımda hazır (MCP server + LLM'siz `queryKnowledgeBase` =
mimarideki D akışı). Bütün iş hub tarafında: katalog-tabanlı banka seçimi (utility
model), paralel fan-out, füzyon, birleşik atıf. En büyük risk isabet — katalog
kalitesine dayanır → C9/C10 önkoşuldur. İkinci risk gecikme → varsayılan yol
"LLM'siz queryKB fan-out" (timeout + kısmi cevap politikasıyla), `askAgent`
zinciri opsiyonel. Topoloji V2'de tek seviyedir (hub → banka; federated/centralized);
decentralized/mesh ve onunla gelen derinlik-döngü guard'ları A2A sonrası (V3+)
gündemidir — bugün tasarlamak zaman kaybı.
Hub = aynı generic imajın 10. konfigürasyonu; yeni ürün yok. Gereksinimlerdeki
S5 "kapsam dışı/V3" statüsü revize edilmeli (federasyon → V2; serbest A2A ağları V3'te kalır).

**Karar (açık soru kapandı):** "tek yerden kontrol" = **max esneklik**. Bankalar
MCP olarak bağlı olduğundan hub, onların yüzeyindeki her tool'u kullanabilir —
`queryKnowledgeBase`, `askAgent` ve upload/reindex gibi opsiyonel yönetim
endpoint'leri dahil. Federasyon tasarımı yüzeyi daraltmaz; LLM'siz `queryKB`
fan-out yalnız varsayılan hız yoludur, `askAgent`/yönetim çağrıları hub
agent'ının tercihine açıktır. Ö11'in kapsam cümlesi buna göre yazılır.

## 3. Çapraz bulgular

1. **Kütüphane olarak dışa açma:** üç senaryoda da ihtiyaç doğmadı; tüm entegrasyon
   MCP/REST yüzeyi + imaj konfigürasyonuyla çözülüyor. Python public-API taahhüdünün
   bedeli (semver, kırılma riski, dokümantasyon) karşılıksız kalır → **buzdolabı**;
   gereksinimlerin Bölüm 5'ine gerekçeli kayıt.
2. **Webhook-out (callback_url):** senaryolarda talep yok (çıkış yönü stream C3 ile
   karşılanıyor) → buzdolabı; C4 altyapısı ileride simetrik genişlemeye uygun.
3. **V1 kilidi:** korunur. V1'e giren yalnız iki sözleşme kararı (çoğul `agents:`
   şeması; bloklu cevap zarfı) — ikisi de kod maliyeti değil, tasarım kararı.
4. **Sıralama zorunluluğu:** C2 (zarf) → C8/C9 (projeksiyon+katalog) → C10 (otomatik
   tarif) → C11 (federasyon). Federasyonu öne çekmek katalogsuz "isabetsiz hub" üretir.

## 4. Gereksinim değişikliği önerileri (onay bekliyor)

Hedef: `gereksinimler.md` v0.3 → v0.4. Ö-numaraları onay/red için kullanılacak.

| Ö# | Değişiklik | İlgili | Sürüm etkisi |
|---|---|---|---|
| Ö1 | Senaryo tablosu: S3 genişler (self-describing yüzey), S5 "kapsam dışı" ibaresi kalkar → "federasyon/hub, V2 hedefi"; yeni S6 "tamamlayıcı beyin plugin'i"; `senaryolar.md` referansı | US-hepsi | — |
| Ö2 | FR-6 revizyonu → **Agent Profilleri**: config'de çoklu named agent (persona+tool seti+routing); istekte `agent:` seçimi; **config şeması V1'den çoğul** | C1 | Şema V1, impl V1.5 |
| Ö3 | Yeni FR: **Yanıt zarfı + pass-through** — bloklu cevap sözleşmesi (llm_text, verbatim bloklar, atıf, usage) V1'de; tool/MCP başına `passthrough: true` + istek düzeyi `pass_through: []`, sonuç bit-bit korunur | C2 | Zarf V1, impl V1.5 |
| Ö4 | FR-9 eki: **SSE streaming** — `/chat` stream modu; MCP tarafında progress | C3 | V1.5 |
| Ö5 | Yeni FR: **Inbound webhook / otomasyon** — `/hooks/{name}`, HMAC imza + idempotency, config'te hook→aksiyon (reindex \| agent profili \| MCP tool); 202 + arka plan iş modeli | C4 | V1.5 |
| Ö6 | Yeni FR (+NFR-3 eki): **İstek günlüğü** — opsiyonel içerik loglama, retention, `query_history` tool'u; gizlilik notu | C5 | V1.5 |
| Ö7 | Yeni FR: **KB geri-besleme** — `save_to_kb` builtin tool; `origin=agent` etiketi atıflarda görünür; listele/temizle | C6 | V1.5–V2 |
| Ö8 | FR-4 eki: **repo özet wiki reçetesi** — DeepWiki-muadili harici MCP + webhook tetikli güncelleme, `examples/` altında uçtan uca örnek (çekirdeğe kod girmez) | C7 | V1.5 (döküman) |
| Ö9 | FR-3.2 revizyonu: **konfigüre edilebilir MCP yüzeyi** — `expose.tools[]` projeksiyonu (kb_query\|agent\|mcp_proxy\|http), server instructions, ad/açıklama/talimat config'den; bugünkü 5 tool varsayılan set | C8 | Statik V1.5, proxy V2 |
| Ö10 | Yeni FR: **Yetenek kataloğu + otomatik tarif** — `describeCapabilities` + `/capabilities`; tarif katmanları manual > auto > fallback; `agent describe` (build) / `POST /describe` (runtime, utility model) | C9, C10 | Katalog V1.5, LLM'li üretim V2 |
| Ö11 | Yeni FR + S5 revizyonu: **Federasyon modu** — banka otomatik tespit (`describeCapabilities` sunan MCP; `role` işareti yok), katalog federasyonu, fan-out+füzyon+birleşik atıf, timeout+kısmi cevap, toplu health; topoloji V2'de **tek seviye** (hub→banka); kontrol kapsamı **max esneklik**: hub, bankaların tüm MCP tool'larını kullanabilir (LLM'siz `queryKB` = varsayılan hız yolu, `askAgent`/yönetim serbest). Derinlik/döngü guard'ları V3+ (A2A) | C11, C13 | V2 |
| Ö12 | NFR eki: **base image dağıtımı** — registry yayını, semver, "minor sürümde config kırılmaz" taahhüdü, CHANGELOG | imaj fikri | operasyonel |
| Ö13 | Bölüm 5 kaydı: **kütüphane fikri buzdolabına** (senaryo desteği yok, gerekçeli); webhook-out da talep doğana kadar buzdolabı | çapraz | — |
| Ö14 | Yol haritası güncellemesi: V1 kilidi korunur (+Ö2/Ö3 sözleşme istisnaları); V1.5 = Ö3–Ö6, Ö8, Ö9-statik, Ö10-katalog; V2 = Ö7, Ö9-proxy, Ö10-otomatik, Ö11; V3+ = serbest A2A ağları (decentralized) + derinlik/döngü guard'ları (C12) | — | harita |

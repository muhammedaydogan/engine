# Generic AI Agent Container — Gereksinim Dokümanı (v0.6 — Taslak)

> Amaç: Herhangi bir kişi veya projenin, kendi bilgi tabanı (knowledge base / knowledge graph), araçları (MCP, repo, web, harici API) ve talimatlarıyla özelleştirebileceği; hem sohbet asistanı hem kod/doküman üretim aracı olarak çalışabilen; ve dışarıya kendisini bir **MCP server** olarak sunabilen generic bir Docker container'ı geliştirmek.
>
> **v0.4 değişikliği:** Web search özellik setinden çıkarıldı — bir ürün özelliği değil, ihtiyaç hâlinde FR-3.1 üzerinden config'e eklenen sıradan bir arama MCP'sidir; offline davranışı da diğer MCP'lerle aynı genel kurala tabidir (FR-5.2 kaldırıldı; NFR-7, Bölüm 6 ve yol haritası sadeleştirildi).
>
> **v0.5 değişikliği:** `OFFLINE_MODE` kaldırıldı — kapalı ağ, bildirilecek bir mod değil **varsayılan duruştur**: container'ın internet varsayımı yoktur, dış bağlantı yalnız config'te tanımlı adreslere yapılır (default-deny); internet gerektiren özellikler config'le bilinçli açılır (opt-in). NFR-7 yeniden yazıldı.
>
> **v0.6 değişikliği:** `senaryolar.md`'deki üç kullanım senaryosu (US-1/2/3) fizibiliteden geçirilip işlendi. Senaryo tablosunda S3 genişledi (kendini tarif eden yüzey), S5 federasyon hedefi olarak V2'ye alındı, S6 (tamamlayıcı beyin plugin'i) eklendi. FR-3.2 konfigüre edilebilir MCP yüzeyine, FR-6 agent profilleri + skill'lere genelleştirildi; FR-4.4 (repo özet wiki reçetesi), FR-9.5 (SSE streaming), FR-11–FR-16 (yanıt zarfı/pass-through, inbound webhook, istek günlüğü, KB geri-besleme, yetenek kataloğu/otomatik tarif, federasyon modu) ve NFR-8 (base image dağıtımı) eklendi; Bölüm 5'e gerekçeli buzdolabı kayıtları girdi (kütüphane API'si, webhook-out, derinlik/döngü guard'ları). V1 kapsam kilidi korunur; V1'e giren yalnız iki **sözleşme** kararıdır: çoğul `agents:`/`skills:` şeması (FR-6.1) ve bloklu yanıt zarfı (FR-11.1). Onay turu ekleri: FR-2.8 (dosya volume'unun dışa mount edilebilirliği), hint'li tarif yöntemi (FR-3.2.3, FR-15.3), buzdolabına dört ham fikir.

---

## 1. Kapsam ve Kullanım Senaryoları

| # | Senaryo | Açıklama |
|---|---------|----------|
| S1 | Doküman üretimi | Şablonlar önden KB'ye yüklenir; container ayağa kalkınca "mevcut şablona göre YGÖ yaz" denir, proje bilgileri verilir, çıktı üretilir. |
| S2 | Kod asistanı | Repo(lar) taranır; örnek kod, refactor, analiz yaptırılır. |
| S3 | MCP olarak dış kullanım (kendini tarif eden yüzey) | Kişi/sistem kendi agent'ına (ör. Claude Code) bu container'ı MCP server olarak tanıtır; tek bağlantıyla büyük bir knowledge base'e ve tool'lara erişir. Yüzey kendini tarif eder: konfigüre edilebilir tool projeksiyonu (FR-3.2) + yetenek kataloğu ve otomatik tarif (FR-15) sayesinde istemci agent hangi işi hangi tool'a götüreceğini açıklamalardan anlar. (`senaryolar.md` US-2) |
| S4 | Canlı veri sorgulama | Spring AI destekli harici Java servisinin endpoint'leri üzerinden canlı DB verisi vb. çekilir. |
| S5 | Container'lar arası iletişim (federasyon/hub) | Bir agent container başka agent container'lara bağlanır: hub, veri bankası container'larını MCP client olarak ekler; katalog federasyonu + LLM'siz fan-out ile tek kapıdan isabetli ve hızlı cevap (FR-16). **V2 hedefi — tek seviye hub→banka topolojisi**; serbest/decentralized A2A ağları V3+ gündemidir. (`senaryolar.md` US-3) |
| S6 | Tamamlayıcı beyin plugin'i | Otonom bir dış sisteme (ör. kod yazma sistemi) MCP/REST üzerinden bağlanan bilgi/karar destek eklentisi: çoklu agent profili (FR-6), pass-through (FR-11), inbound webhook (FR-12), stream (FR-9.5), istek günlüğü + KB geri-besleme (FR-13/FR-14) ile tam otomasyon. (`senaryolar.md` US-1) |

> Not: ALM entegrasyonu (Azure DevOps / Jira) ayrı bir senaryo değildir; ilgili MCP server'ın config'e eklenmesinden ibarettir. FR-3.1'in bir kullanım örneği olarak geçer.
>
> Not: S3/S5/S6'nın ayrıntılı hikâyeleri `senaryolar.md`'dedir (US-2→S3, US-3→S5, US-1→S6). **Kabul çizgisi:** üç senaryo anahatlarıyla **en geç V3'te** uçtan uca çalışır olmalı; US-3 bu çizgide centralized/federated topolojiyle sınırlıdır (decentralized sonrası).

---

## 2. Fonksiyonel Gereksinimler

### FR-1: LLM Erişimi
- **FR-1.1** Container içindeki uygulama en az bir LLM sağlayıcısına istek atabilmeli.
- **FR-1.2** Sağlayıcı **konfigürasyonla seçilebilir** olmalı (provider-agnostic): Anthropic, OpenAI, Azure OpenAI, ve on-prem/lokal (Ollama, vLLM gibi OpenAI-uyumlu endpoint). Kurumsal ortamlarda dış API'ye çıkış yasak olabileceği için OpenAI-uyumlu generic endpoint desteği şart. **İlk hedef sağlayıcı OpenAI-uyumlu bir endpoint'tir**; geliştirme ve testler bu profil üzerinden yürütülür, embedding de öncelikle aynı endpoint'ten alınır (NFR-4).
- **FR-1.3** Farklı görevler için farklı model atanabilmeli (ör. embedding için küçük model, üretim için büyük model).
- **FR-1.4** API anahtarları imaja gömülmemeli; environment variable / secret / mount ile verilmeli.

### FR-2: Knowledge Base / Knowledge Graph
- **FR-2.1** Statik dosyalardan (md, txt, pdf, docx, xlsx, kod dosyaları) bilgi tabanı oluşturulabilmeli (ingest → chunk → embed → index).
- **FR-2.2** V1'de **vektör tabanlı RAG yeterli**; knowledge graph (GraphRAG/Graphiti/Cognee tarzı entity-ilişki çıkarımı) **V2 hedefi**. V2'de hedeflenen yapı Cognee tarzı üçlü depolamadır: vektör + graph + ilişkisel/metadata DB. (Gerekçe için Bölüm 5.)
- **FR-2.3** KB iki şekilde hazırlanabilmeli:
  - **Build-time (bake):** Dosyalar image build sırasında işlenir, index imajın içinde hazır gelir. → V1'in ana yolu.
  - **Runtime:** Container ayaktayken `addSource`, `addRepo`, `uploadFile` gibi API/tool çağrılarıyla eklenir. → V1'de en az `uploadFile` + `reindex`, gerisi V1.5/V2.
- **FR-2.4** KB kalıcı olabilmeli (volume mount) — container yeniden başlayınca runtime'da eklenenler kaybolmamalı.
- **FR-2.5** Kaynaklar listelenebilmeli ve silinebilmeli (`listSources`, `removeSource`).
- **FR-2.6** Cevaplarda kaynak atfı yapılabilmeli (hangi doküman/bölümden geldiği).
- **FR-2.7** Retrieval modu config'den seçilebilmeli: `vector` (varsayılan) veya `hybrid` (FTS/BM25 + vektör, skor füzyonu — RRF veya ağırlıklı toplam). Gerekçe: hata kodu, fonksiyon adı, resmi terim gibi birebir eşleşme gereken sorgularda ve Türkçe içerikte saf vektör zayıf kalabilir; gömülü FTS desteği olan bir vektör DB (ör. LanceDB) veya SQLite FTS5 ile ek maliyet düşüktür. Reranking **opsiyonel bir config anahtarıdır ve varsayılan kapalıdır**; implementasyonu V1.5'e bırakılabilir ama config şemasında yeri baştan ayrılmalı.
- **FR-2.8 (Dosya volume'unun dışa görünürlüğü):** KB'yi besleyen dosyaların durduğu volume (build'de gelenler + runtime eklenenler) **opsiyonel olarak dışarıya mount edilebilmeli** — host'a veya aynı volume'u paylaşan başka container'lara. Dosyalar dışarıdan görünür ve yönetilebilir sade bir dizin düzeninde tutulmalı (opak bir depo formatına hapsedilmemeli); dışarıdan eklenen/değişen dosyalar `reindex` ile index'e alınır (FR-2.3). Gerekçe: mevcut ve sonradan eklenen dosyalar dışarıdan izlenebilmeli; ileride ayrı bir **fileUploader** container'ı aynı volume'u paylaşıp dosyaları hızlı ve kolay yönetebilir (fileUploader'ın kendisi kapsam dışıdır — burada yalnız kapı açık bırakılır).

### FR-3: MCP — İki Yönlü
- **FR-3.1 (MCP Client):** Uygulama, konfigürasyonda tanımlı harici MCP server'lara bağlanıp onların tool'larını kullanabilmeli (stdio ve HTTP/SSE transport). Örnek: Azure DevOps / Jira MCP server'ı config'e eklenerek iş kayıtları okunabilir/yazılabilir — bu tür ALM entegrasyonları ayrı bir gereksinim değil, bu maddenin doğal sonucudur.
- **FR-3.2 (MCP Server):** Uygulama kendisini dışarıya MCP server olarak sunabilmeli; böylece Claude Code / Claude Desktop / başka agent'lar bu container'ı tek bir MCP olarak ekleyip KB'ye ve tool'larına erişebilmeli. *(v0.6'da genişletildi: yüzey sabit değil, config'den tanımlanır.)*
  - **FR-3.2.1 (Konfigüre edilebilir tool projeksiyonu):** MCP yüzeyi config'den tanımlanabilmeli (`expose.tools[]`): her tool'un adı, açıklaması ve kullanım talimatı config'den gelir; hedef tipleri `kb_query` (ön-filtreli KB sorgusu) \| `agent` (FR-6 profili) \| `mcp_proxy` (bağlı harici MCP tool'una köprü) \| `http` (manuel HTTP tool). Server `instructions` alanı katalog özetiyle doldurulur (FR-15.2). Gerekçe: sabit generic yüzeyde istemci agent içeride ne olduğunu anlayamıyor, her iş `askAgent` hunisine düşüyor; anlamlı ad + açıklama, istemci tarafındaki route isabetinin ön şartı (`senaryolar.md` US-2). Sürüm: statik projeksiyon V1.5; `mcp_proxy` hedefi V2.
  - **FR-3.2.2 (Varsayılan projeksiyon):** `expose` config'i yoksa bugünkü beş generic tool (`queryKnowledgeBase`, `askAgent` — talimatlı üretim, `uploadFile`/`addSource`, `listSources`, `removeSource`) varsayılan projeksiyon olarak aynen sunulur — geriye dönük kırılma yok; V1 yüzeyi budur.
  - **FR-3.2.3 (Tarif katmanlaması):** Tool tarifi üç yolla verilebilir: **(a) elle tam tarif** — config'de yazılmış ad/açıklama/talimat her zaman en üst katmandır; **(b) hint** — tam tarif yerine config'de kısa bir `hint` verilir, tarifi bu hint'i yönlendirme alarak otomatik üretim (FR-15.3) oluşturur (sürümü FR-15.3'ü izler); **(c) hiçbiri** — otomatik üretim (varsa) ya da generic fallback devreye girer. Katmanlama: **manual > hint'li auto > auto > generic fallback** — mekanizmalar çakışmaz, otomatik üretim elle yazılanı asla ezmez.
- **FR-3.3** MCP tanımları önden (config dosyası) veya sonradan (`addMCP` endpoint'i) yapılabilmeli. `addMCP` → V2.
- **FR-3.4 (Kapalı ağ / air-gap):** İlgili ortamdaki sonatype-nexus imajın içerisinde imaj build edilmeden önce tanımlanır. Böylelikle npm install pip install gibi paketlerin çalışması beklenir.

### FR-4: Repo Erişimi
- **FR-4.1** Git repo'ları (HTTPS/SSH, token'lı private repo dahil) clone edilip taranabilmeli.
- **FR-4.2** Repo içeriği KB'ye kaynak olarak eklenebilmeli (`addRepo`).
- **FR-4.3** V1'de repo, build-time'da statik dosya gibi kopyalanıp index'lenebilir; canlı `addRepo` endpoint'i V1.5.
- **FR-4.4 (Repo özet wiki reçetesi):** DeepWiki-muadili repo özetleme **çekirdeğe alınmaz**; harici bir MCP server olarak bağlanır (FR-3.1) ve salt config'le kurulur: repo değişince inbound webhook tetiklenir (FR-12) → wiki MCP'sinin güncelleme tool'u çağrılır → çıktı KB'ye yazılır (FR-14). `examples/` altında uçtan uca çalışan örnek kompozisyon verilir (iki servisli docker-compose + `agent.yaml` + hook config'i). Ürünün bildiği tek şey "hook gelince config'teki aksiyon zincirini çalıştır"; DeepWiki'ye özgü her şey kullanıcının config'inde kalır — container generic kalır. Zincirin agent/LLM içeren adımları best-effort'tur (FR-12.4). Sürüm: V1.5 (doküman/örnek; yeni özellik değil, FR-12 + FR-14 + FR-3.1'in kullanım tarifi). (`senaryolar.md` US-1)

### FR-5: Web Erişimi
- **FR-5.1** Belirli URL'lerin içeriği çekilebilmeli (fetch) ve isteğe bağlı olarak KB'ye eklenebilmeli.
- **FR-5.2** *(v0.4'te kaldırıldı — numara, çapraz referanslar bozulmasın diye korunuyor.)* Web araması ürün özelliği değildir; arama ihtiyacı doğarsa bir arama MCP'si (Brave/Tavily/SearXNG MCP vb.) FR-3.1 üzerinden config'e eklenir. Kapalı ağ davranışı da özel kural gerektirmez — erişemeyen her MCP gibi degraded işaretlenir (NFR-7).
- **FR-5.3** Erişilebilecek domain'ler allowlist/denylist ile sınırlandırılabilmeli (güvenlik).

### FR-6: Agent Profilleri ve Skill'ler *(v0.6'da "Talimat / Persona Yönetimi"nden genelleştirildi)*
- **FR-6.1** Config'de **birden çok named agent** tanımlanabilmeli (`agents:` listesi): her agent = persona/system prompt + skill listesi + tool seti (+ pass-through/routing ayarları, FR-11). **Skill** = yeniden kullanılabilir talimat modülü + opsiyonel tool alt-kümesi (`skills:` listesi); agent'larla serbestçe kombine edilir (x/y/z agent'ları a/b/c skill'lerini paylaşabilir). İstek düzeyinde `agent: <ad>` ile profil seçilir; seçilmezse varsayılan agent çalışır. **Config şeması V1'den itibaren çoğuldur** (V1'de tek elemanlı liste) — sonradan tekil→çoğul şema geçişi tüm config'leri kırar, bu yüzden sözleşme şimdi çoğul yazılır. İmplementasyon (çoklu profil + skill kompozisyonu) V1.5. Gerekçe: `senaryolar.md` US-1 (asistan + analiz profili) ve US-2 (hedefi `agent` olan tool projeksiyonları, FR-3.2.1).
- **FR-6.2** Agent ve skill dosyaları **build-time** (imajdan) veya **runtime** (volume'dan) yüklenebilmeli; runtime'da eklenip güncellenebilmeli (eski `setInstructions`'ın genellemesi). Kalıcı değişiklik volume'a yazılmalı.
- **FR-6.3** Aynı imajdan farklı agent/skill setleriyle farklı container'lar ayağa kaldırılabilmeli (config-driven).

### FR-7: Harici Servis Entegrasyonu (Spring AI Java Servisi vb.)
- **FR-7.1** Harici bir HTTP servisinin endpoint'leri, agent'ın kullanabileceği **tool** olarak tanımlanabilmeli. Tanım yöntemi: OpenAPI spec verilir veya config'de manuel tool tanımı yapılır (URL, method, parametre şeması, açıklama).
- **FR-7.2** Bu sayede canlı DB sorguları doğrudan DB bağlantısıyla değil, **Java servisinin kontrollü endpoint'leri üzerinden** yapılır. (Bu, kullanıcı tarafında zaten doğru düşünülmüş bir tasarım: agent'a doğrudan DB erişimi vermek yerine servis katmanından geçirmek hem güvenli hem generic.)
- **FR-7.3** Servis kimlik doğrulaması (API key, bearer token) config'den verilebilmeli.
- **FR-7.4** Alternatif/kolay yol: Spring AI servisi kendini MCP server olarak sunuyorsa (Spring AI'ın MCP server desteği mevcut), FR-3.1 üzerinden bağlanmak yeterli — ayrı bir mekanizma gerekmez. **Öneri: önce bu yol denenmeli.**

### FR-8: Doküman Üretimi (YGÖ senaryosu)
- **FR-8.1** KB'deki şablon dokümanlar referans alınarak, verilen proje bilgileriyle yeni doküman üretilebilmeli.
- **FR-8.2** Çıktı formatı belirlenebilmeli: en azından Markdown; YGÖ'nün resmi teslim formatı **docx** V1.5'e alınmalı (Pandoc veya python-docx ile). V1 kabul senaryosunda md çıktı yeterli.
- **FR-8.3** Uzun dokümanlar için bölüm bölüm üretim (tek prompt'a sığmayan şablonlar) desteklenmeli.

### FR-9: Arayüzler
- **FR-9.1** **REST API** (asgari): `/chat`, `/query`, `/sources` (list/add/delete), `/upload`, `/health`. `/chat` tek atımlık RAG'den ibaret değildir: FR-3.1'deki MCP tool'larını **sınırlı bir agent döngüsü** içinde çok adımlı çağırabilmeli (max iterasyon sayısı config'den).
- **FR-9.2** **MCP server endpoint'i** (FR-3.2).
- **FR-9.3** Basit bir web UI (chat ekranı) — **opsiyonel, V1'de gereksiz**; API + MCP yeterli. İstenirse V2.
- **FR-9.4** CLI ile hızlı test imkânı (curl örnekleri dokümante edilir, ayrı CLI aracı gerekmez).
- **FR-9.5 (SSE streaming):** `/chat` bir de **stream** modunda sunulmalı: **SSE** (`text/event-stream`), OpenAI-uyumlu delta deseninde (`stream: true` → `data: {delta}` satırları) — istemciler mevcut OpenAI stream istemcileriyle dinleyebilmeli ("typing/shimmering" efektinin standardı budur; WebSocket değil). Yanıt zarfının (FR-11) blokları stream'de "blok başladı / delta / blok bitti" event'leri olarak akar. MCP tarafında karşılık, progress bildirimidir (MCP streamable HTTP transport'u zaten SSE kullanır — REST ve MCP aynı desende birleşir). Gerekçe: harici sistemlerin cevabı anlık dinleyebileceği çıkış yolu (`senaryolar.md` US-1). Sürüm: V1.5.

### FR-10: Genericlik / Özelleştirme
- **FR-10.1** Tüm özelleştirme **tek bir config yapısından** yapılabilmeli (ör. `agent.yaml` + `knowledge/` klasörü + `.env`):
  - persona/talimatlar, LLM sağlayıcı ve modeller, MCP listesi, tool tanımları, KB kaynak listesi, güvenlik ayarları.
- **FR-10.2** Yeni bir proje için özelleştirme akışı: base image al → config + dosyalar koy → build → deploy. Kod değişikliği gerektirmemeli.
- **FR-10.3** Base image ile proje imajı ayrılmalı: `FROM agent-base` + `COPY knowledge/ config/` şeklinde ince bir katman.

### FR-11: Yanıt Zarfı ve Pass-through *(v0.6'da eklendi)*
- **FR-11.1 (Bloklu yanıt zarfı — V1 sözleşmesi):** `/chat` ve `askAgent` cevabı düz metin değil, **bloklu zarf** (response envelope) olarak döner: `blocks[]` listesi (`llm_text` — LLM metni + kaynak atıfları; `verbatim` — tool çıktısı bit-bit aynen, `source` ve `content_hash` alanlarıyla) + `usage` (token muhasebesi). V1'de zarf yalnız `llm_text` blokları içerir ama **format baştan zarftır** — düz metinden zarfa sonradan geçmek tüm istemcileri kırar; bu yüzden zarf, kod maliyeti değil V1 sözleşme kararıdır.
- **FR-11.2 (Pass-through):** Tool/MCP başına config'de `passthrough: true` işaretlenebilmeli ve istek düzeyinde `pass_through: [...]` listesi verilebilmeli. İşaretli tool'un çıktısı **LLM bağlamına hiç girmez**; zarfa `verbatim` bloğu olarak bit-bit aynen konur (`content_hash` bütünlüğün kanıtı). Böylece "milimetrik hesabı LLM bozar" riski yapısal olarak imkânsızlaşır — LLM içeriği hiç görmez. Federasyondaki LLM'siz hız yolunun da temelidir (FR-16.3). Gerekçe: `senaryolar.md` US-1. Sürüm: zarf V1, pass-through implementasyonu V1.5.

### FR-12: Inbound Webhook ve Arka Plan İşleri *(v0.6'da eklendi)*
- **FR-12.1 (Hook → aksiyon):** Config'den `POST /hooks/{name}` endpoint'leri tanımlanabilmeli; her hook bir **aksiyon zincirine** bağlanır: `reindex` \| agent profili çalıştır (FR-6) \| MCP tool çağır. Dış sistem olay anında haber verir (polling'in tersi) — ör. "repo güncellendi → wiki MCP'sini tetikle → çıktıyı KB'ye yaz" (FR-4.4). Gerekçe: `senaryolar.md` US-1 tam otomasyon varsayar, manuel adım kaldırmaz.
- **FR-12.2 (Güvenlik):** Hook gövdesi paylaşılan secret ile **HMAC-SHA256** imzalanır (GitHub webhook deseni, `X-Hub-Signature-256` muadili); imzasız/yanlış imzalı istek reddedilir. Teslimat ID'siyle **idempotency**: aynı teslimat tekrar geldiğinde (retry) aksiyon ikinci kez çalıştırılmaz.
- **FR-12.3 (İş modeli):** Hook aksiyonları uzun sürebilir (reindex gibi) → çağrıya hemen **`202 Accepted` + `job_id`** dönülür; iş arka planda koşar, durumu `GET /jobs/{id}` ile izlenir — gönderen sistemin timeout'una takılınmaz.
- **FR-12.4 (Güvenilirlik notu):** Deterministik aksiyonlar (`reindex`, tekil MCP tool çağrısı) güvenilirdir; **agent profili çalıştıran (LLM'li) aksiyonlar best-effort'tur** — modelin işi başaramama ihtimali vardır. Bu risk bilinçli kabul edilmiştir: başarısızlık job durumunda görünür olmalı; retry/kalite iyileştirmeleri ileriki sürüm konusudur. Sürüm: V1.5.

### FR-13: İstek Günlüğü *(v0.6'da eklendi)*
- **FR-13.1** İstek/cevap günlüğü **opsiyonel** tutulabilmeli: varsayılan kapalı (opt-in); açılırsa prompt + cevap + metadata (zaman, agent profili, tool çağrıları, token) kaydedilir. **Retention** süresi config'den verilir; süresi dolan kayıt silinir.
- **FR-13.2** Geçmiş, `query_history` tool'u/endpoint'i ile sorgulanabilmeli — ör. bir analiz agent'ı "geçmiş sorulardaki tekrar eden pattern'leri bul" işini salt config'le yapabilmeli; bulgular FR-14 ile KB'ye yazılınca döngü konfigürasyonla kapanır (`senaryolar.md` US-1'in "1 ay sonra analiz" akışı).
- **FR-13.3 (Gizlilik ve yerleşim):** İçerik loglama açıkça opt-in'dir (NFR-3 uzantısı); kayıtlar KB'ye karışmaz, volume'daki gömülü DB'nin (SQLite) yanına tablo olarak eklenir. Depolama erişimi bir kontrat arkasındadır; adapter ayrımı mimari dokümanın konusudur. Sürüm: V1.5.

### FR-14: KB'ye Geri-besleme *(v0.6'da eklendi)*
- **FR-14.1** Builtin **`save_to_kb`** tool'u: agent, ürettiği bulguyu/dokümanı KB'ye kaynak olarak yazabilmeli. Mevcut ingest hattını **çekirdek kontrat üzerinden** çağırır (adapter'a doğrudan dokunmaz — bağımlılık yönü korunur).
- **FR-14.2** Agent'ın yazdığı içerik **`origin=agent`** etiketi taşır ve bu etiket kaynak atıflarında görünür (FR-2.6) — okuyan, bilginin insan kaynaklı mı agent üretimi mi olduğunu ayırt edebilir.
- **FR-14.3** `origin=agent` kayıtları filtreyle listelenebilir ve topluca temizlenebilir (FR-2.5'in filtreli hâli) — KB'nin agent çıktısıyla kirlenmesi geri alınabilir olmalı. Sürüm: V2 (mekanik olarak basit, V1.5'e öne alınabilir).

### FR-15: Yetenek Kataloğu ve Otomatik Tarif *(v0.6'da eklendi)*
- **FR-15.1 (Katalog):** Container kendini tarif edebilmeli: **`describeCapabilities(toolNames?: string[], detail?: "summary" | "full")`** MCP tool'u + REST **`/capabilities`**. Parametresiz çağrı özet katalog döner; `toolNames` verilirse yalnız seçili tool'ların tam tarifi döner — büyük katalogda istemcinin context'i şişirilmez. İçerik: tool projeksiyonu (FR-3.2), bağlı MCP/servisler, KB alanları ve istatistikleri.
- **FR-15.2 (MCP-native yüzey):** Katalog için format icat edilmez; MCP spec'inin üç native yüzeyine döşenir: server **`instructions`** (initialize cevabında katalog özeti), tool **`description` + JSON Schema** (+ `annotations`), ve istenirse tam kataloğun **MCP resource** olarak yayını.
- **FR-15.3 (Otomatik tarif üretimi):** Tarifler katmanlıdır: **manual (config) > hint'li auto > auto (LLM'li üretim) > generic fallback** — elle tarif zorunlu değildir (hiç yazılmadan bağlanılabilir), elle tarif otomatik üretimle asla ezilmez (FR-3.2.3 ile aynı katmanlama). Otomatik üretim: build-time **`agent describe`** komutu veya runtime **`POST /describe`** (utility model bağlı kaynakları + KB'yi tarayıp tarif taslağı çıkarır); `--review` çıktısıyla insan onayına sunulabilir. Üretim, config'de tool başına verilebilen kısa **`hint`** alanıyla yönlendirilebilir — tam tarif yazmak istemeyen entegratör niyetini hint'le söyler, tarifi agent yazar (FR-3.2.3'ün (b) yolu). Gerekçe: tarif kalitesi bütün entegrasyonun verimini belirler (`senaryolar.md` US-2); LLM'li üretimde kalite riski katman zinciri + insan onayıyla hafifletilir. Sürüm: katalog şeması V1.5; LLM'li üretim V2.

### FR-16: Federasyon / Hub Modu *(v0.6'da eklendi)*
- **FR-16.1 (Otomatik tespit):** Hub ayrı bir ürün değildir — aynı generic imajın konfigürasyonudur; config'de `role` benzeri bir işaret **yoktur**. Hub'ın MCP client kayıtlarından `describeCapabilities` sunanlar **veri bankası (spoke)** olarak otomatik tanınır. Spoke tarafında sıfır ek iş: banka, sıradan bir MCP server'dır (bugünkü tasarım yeter).
- **FR-16.2 (Katalog federasyonu):** Hub, bankaların kataloglarını (FR-15) toplar; soru geldiğinde banka seçimini onlarca tool'luk düz listeyle değil **katalogla** yapar (utility model — ucuz ve isabetli). Gerekçe: naif hâlde 9 banka × ~6 tool ≈ 54 tool hub LLM'ine gider — yavaş, pahalı, isabetsiz.
- **FR-16.3 (Fan-out + füzyon):** Seçilen bankalara **paralel, LLM'siz** `queryKnowledgeBase` fan-out yapılır (bankada agent döngüsü çalışmaz; FR-11 pass-through'un akrabası); sonuçlar füzyonlanıp tek cevap derlenir. **Birleşik atıf:** `banka:doküman § bölüm` (FR-2.6'nın federasyona genellemesi).
- **FR-16.4 (Kontrol kapsamı — max esneklik):** LLM'siz fan-out yalnız **varsayılan hız yoludur**; hub, bankaların MCP yüzeyindeki her tool'u kullanabilir (`askAgent`, upload/reindex dahil) — federasyon yüzeyi daraltmaz.
- **FR-16.5 (Dayanıklılık ve toplu health):** Timeout + kısmi cevap politikası: geç/ölü banka cevabı bekletmez; cevap "X bankasına ulaşılamadı" notuyla döner. Hub `/health`'i, bağlı bankalardan çekilen durum özetini (`federation:` bölümü) içerir — filo durumu tek bakışta.
- **FR-16.6 (Topoloji sınırı):** V2'de topoloji **tek seviyedir** (hub → banka; centralized/federated). Decentralized/mesh ağlar ve derinlik/döngü guard'ları kapsam dışıdır (buzdolabı — Bölüm 5). Sürüm: V2. Gerekçe: `senaryolar.md` US-3.

---

## 3. Fonksiyonel Olmayan Gereksinimler

- **NFR-1 Güvenlik:**
  - Secrets asla imaja gömülmez (env/secret mount).
  - API'ye erişim en azından basit bir API key ile korunmalı (container internete/ortak ağa açılırsa şart).
  - Prompt injection farkındalığı: web'den/repodan çekilen içerik "talimat" değil "veri" olarak ele alınmalı (tam çözümü yok, ama tasarımda ayrım gözetilmeli).
  - Outbound ağ erişimi kısıtlanabilmeli (allowlist).
- **NFR-2 Kalıcılık:** KB index'i, yüklenen dosyalar, talimatlar volume'da; container stateless kalabilmeli.
- **NFR-3 Gözlemlenebilirlik:** Yapılandırılabilir loglama; her LLM çağrısının token/maliyet kaydı (kurumsalda maliyet takibi kaçınılmaz soru olur). İstek günlüğü (FR-13) bu başlığın uzantısıdır: içerik loglama **opt-in**'dir ve retention'a tabidir.
- **NFR-4 Kaynak:** Tek container, makul RAM ile çalışmalı. Embedding lokalde yapılacaksa CPU'da çalışan küçük bir model tercih edilmeli; aksi halde embedding de LLM sağlayıcısından alınmalı (config). İlk hedef sağlayıcı OpenAI-uyumlu endpoint olduğundan (FR-1.2) varsayılan yol embedding'in de bu endpoint'ten alınmasıdır; lokal CPU modeli yedek seçenektir.
- **NFR-5 Taşınabilirlik:** Hedef deploy ortamı ikilidir: tek sunucuda docker-compose **ve** kurumsal orkestratör (Kubernetes/OpenShift vb.) — ikisi de desteklenmeli. Temel gereksinim sadece Docker (ve docker-compose) ile çalışabilmek, Kubernetes zorunluluğu olmamalı; tek container + volume + env-config yapısı sayesinde imaj orkestratör üzerinde de değişiklik gerektirmeden koşabilmeli.
- **NFR-6 Çoklu kullanıcı:** V1'de **tek kiracı / tek KB** varsayımı. Kullanıcı bazlı izolasyon istenirse "proje başına container" modeli kullanılır (zaten tasarımın doğası bu). Container içi multi-tenancy → gereksiz efor, yapılmamalı.
- **NFR-7 Offline-first varsayılan duruş:** Container'ın internet varsayımı yoktur; kapalı/internetsiz ağda tam işlevlilik **varsayılan durumdur** ve ayrıca bir mod/bayrak gerektirmez. Dış bağlantı yalnız config'te tanımlı adreslerledir: LLM/embedding endpoint'i (FR-1.2 — tipik kurulumda aynı ağda/lokal; internet erişimi olan kurulumda cloud endpoint de config'le mümkündür), config'teki MCP'ler ve açıksa web fetch allowlist'i (FR-5.3). Bunların dışına çıkış yoktur (default-deny). MCP bağımlılıkları imaja gömülüdür, runtime'da paket indirme yoktur (FR-3.4). İnternet gerektiren bir özellik ancak config ile bilinçli açılır (opt-in). Erişilemeyen/kapalı bir bileşen uygulamayı düşürmez: degraded işaretlenir, çağrıldığında hata yerine "bu kurulumda kapalı/erişilemiyor" türünde anlamlı bir yanıt döner.
- **NFR-8 Base image dağıtımı:** Base image bir container registry'de yayınlanır ve **semver** ile sürümlenir; **"minor sürümde config kırılmaz"** taahhüdü verilir (config şemasını kıran değişiklik ancak major sürümde), her sürümle CHANGELOG yayınlanır. Gerekçe: "imajı dışa açma" fikri zaten FR-10'un kendisidir; eksik olan yalnız bu yayınlama/sürümleme operasyonudur — proje imajları `FROM agent-base:X.Y` ile güvenle sabitlenebilmeli.

---

## 4. Kullanıcının Tarifinde Eksik Kalan Noktalar (tamamlandı)

1. **Kimlik doğrulama:** Container MCP olarak dışarı açılınca kim erişebilecek? → asgari API key (NFR-1).
2. **Kalıcılık stratejisi:** Runtime'da eklenen kaynaklar restart'ta ne olacak? → volume (FR-2.4).
3. **Embedding modeli kararı:** KB kurmak için LLM'den ayrı bir embedding ihtiyacı var; lokal mi API mi? → config'e bağlandı (NFR-4).
4. **Çıktı formatı:** YGÖ gibi resmi dokümanlar md ile teslim edilemez → docx üretimi (FR-8.2).
5. **Maliyet/limit:** Token kullanımı loglanmalı, istek başına max token limiti konabilmeli (NFR-3).
6. **KB güncelleme:** Kaynak silme/yeniden index'leme olmadan KB zamanla çöplüğe döner (FR-2.5).
7. **Kurumsal ağ kısıtı:** Dış LLM API'sine erişimin yasak olduğu ortam senaryosu → OpenAI-uyumlu lokal endpoint desteği (FR-1.2).
8. **Türkçe içerik kalitesi:** Şablonlar ve çıktılar Türkçe olacaksa embedding ve chunk stratejisi Türkçe metinde test edilmeli (kabul kriterlerine eklenecek).

## 5. Mümkün / Mümkün Değil / Gereksiz Efor Değerlendirmesi

**Mümkün ve makul (hepsi ispatlanmış teknolojiler):**
- Provider-agnostic LLM erişimi, RAG, MCP client+server, repo tarama, web fetch, config-driven genericlik, Azure DevOps/Jira'yı MCP ile bağlama, harici HTTP servislerini tool yapma. Bunların hiçbiri araştırma problemi değil; entegrasyon işi.

**Mümkün ama V1 için gereksiz efor (ertelenecek):**
- **Sıfırdan knowledge graph motoru yazmak.** Entity/ilişki çıkarımı, graph kurma ve graph üzerinde sorgulama (GraphRAG tarzı) hem ingest maliyetini (her doküman için çok sayıda LLM çağrısı) hem karmaşıklığı katlar. Çoğu senaryoda (şablondan doküman yazma, kod örneği, dokümantasyon sorgusu) iyi kurgulanmış vektör RAG + metadata yeterlidir. Graph gerekirse V2'de hazır bir kütüphane/altyapı (ör. Cognee, LightRAG, Graphiti+Neo4j) entegre edilir — kesinlikle sıfırdan yazılmaz.
- **RAPTOR tarzı hiyerarşik özet ağacı:** Ayrı bir iş olarak kurulmayacak; korpus-geneli soruların gerektirdiği özetleri V2'de Cognee zaten ingest sırasında üretir (SUMMARIES arama tipi). Vektör + graph + özet üçlüsü bu ihtiyacı kapatır.
- **Web UI:** API + MCP varken V1'de vitrin.
- **`addMCP` runtime endpoint'i:** MCP listesi config'den geliyorken, restart'la MCP eklemek V1 için kabul edilebilir.
- **Container içi multi-tenancy:** "Proje başına container" modeli varken gereksiz.

**Kapsam dışı / sonraya:**
- **Serbest A2A ağları (decentralized/mesh):** S5'in somut hâli (9 banka + 1 hub, `senaryolar.md` US-3) fizibiliteden geçti ve v0.6'da **federasyon/hub modu olarak V2'ye alındı (FR-16)** — teknik temel zaten bedavaydı (her container MCP server olduğundan hub onları client olarak ekler); eksik olan isabet/hız/atıf davranış paketiydi. Kapsam dışı kalan yalnız **tek seviyeli hub→banka'nın ötesi**: decentralized/mesh topolojiler, çok seviyeli zincirler. V3+ gündemidir; onunla gelen derinlik/döngü guard'ları buzdolabındadır (aşağıda).

**Buzdolabı (gerekçeli beklemede — gündem doğarsa açılır):**
- **Python kütüphanesi olarak dışa açma:** Üç kullanım senaryosunun (`senaryolar.md`) hiçbirinde ihtiyaç doğmadı; tüm entegrasyon MCP/REST yüzeyi + imaj konfigürasyonuyla çözülüyor. Public API taahhüdünün bedeli (semver disiplini, kırılma riski, API dokümantasyonu) karşılıksız kalır. İmaj dağıtımı ayrı konudur ve gereksinimdir (NFR-8).
- **Webhook-out (`callback_url`):** Senaryolarda talep doğmadı; çıkış yönündeki "anlık bildirim" ihtiyacını stream (FR-9.5) karşılıyor. FR-12'nin arka plan iş modeli ileride simetrik genişlemeye (iş bitince dışarı POST) uygundur.
- **Derinlik/döngü guard'ları (`X-Agent-Depth`, `max_depth`, çevrim tespiti):** Tek seviyeli hub→banka topolojisinde (FR-16.6) çevrim ancak yanlış config'le oluşur; timeout + kısmi cevap (FR-16.5) yeterlidir. Bugün tasarlamak zaman kaybı; decentralized/mesh gündemi açılırsa geri alınır.

**Buzdolabı — ham fikirler (v0.6'da kabaca not edildi; fizibilitesi yapılmadı, gündem doğarsa değerlendirilir):**
- **Çoklu iç agent tartışması:** Birden fazla iç agent spawn edilip çeşitli formatlarda tartıştırılması (debate/kritik/oylama vb.).
- **Knowledge Core'un MCP'ye dönüştürülmesi:** Knowledge Core'un kendisinin bağımsız bir MCP server olarak sunulabilmesi/paketlenebilmesi.
- **Süre telemetrisi:** Hangi MCP/tool ne kadar süre harcadı, KB sorgusu ne kadar sürdü — loglanması ve ek olarak yanıta da yazılması (yanıt zarfı FR-11, `usage` yanına `timings` bloğuyla genişlemeye uygun).
- **Sorgu türevleri (5 tip):** Sorgu yüzeyinin retrieval ve/veya generation kombinasyonlarına ayrışan ~5 türev tip olarak sunulması (retrieval-only, generation-only, karışımları); tipler netleştiğinde değerlendirilecek.

**Mümkün olmayan / yanlış beklenti olabilecekler:**
- "Canlı database bilgisi çekme"nin güvenli hali ancak servis katmanı (FR-7.2) üzerinden olur; agent'a doğrudan DB credential'ı vermek teknik olarak mümkün ama önerilmez.
- Halüsinasyonsuz doküman üretimi garanti edilemez; kaynak atfı (FR-2.6) ve insan onayı süreçle çözülür.
- Tek container'a sınırsız büyüklükte KB sığmaz; pratikte yüzbinlerce sayfa mertebesine kadar sorunsuz. KB büyüklüğü V1 kararlarını değiştirmiyor (gömülü vektör DB yeterli); büyüme ve graph ihtiyacı doğduğunda V2'de Cognee tarzı üçlü depolamaya (vektör + graph + ilişkisel) geçilir.

---

## 6. V1 Kapsamı (kilitlenecek)

**Hedef:** Statik dosyalarla knowledge base oluşturulmuş bir Docker imajı build edilir, deploy edilir, hem REST hem MCP üzerinden sorgulanabilir.

**Teknoloji kararları:** Uygulama dili **Python**'dur (FastAPI + resmi MCP SDK + LlamaIndex/LangChain ekosistemi). LLM ve embedding ilk hedefte OpenAI-uyumlu bir endpoint'ten alınır (FR-1.2, NFR-4); deploy hem tek sunucuda docker-compose hem kurumsal orkestratör üzerinde hedeflenir (NFR-5).

Dahil:
1. `knowledge/` klasörüne konan md/txt/pdf/docx dosyalarının **build sırasında** index'lenmesi (chunk + embed + vektör index + hybrid mod için FTS index'i, FR-2.7; ör. gömülü Qdrant/Chroma/LanceDB — mimaride seçilecek, LanceDB'nin gömülü FTS desteği bu seçimi etkiler).
2. `agent.yaml` ile: agent tanımı — **V1'den itibaren çoğul şema** (`agents:`/`skills:` listeleri, V1'de tek elemanlı; FR-6.1), persona/system prompt, LLM sağlayıcı + model, embedding kaynağı.
3. REST API: `/chat` (talimatlı üretim, KB'yi RAG olarak kullanır, MCP tool'larını sınırlı agent döngüsüyle çağırabilir — FR-9.1), `/query` (saf KB sorgusu), `/upload` + `/reindex` (runtime dosya ekleme, volume'a kalıcı), `/sources` (listele/sil), `/health`. `/chat` cevabı **V1'den itibaren bloklu zarf** formatındadır (FR-11.1; V1'de yalnız `llm_text` blokları).
4. **MCP server modu:** `queryKnowledgeBase`, `askAgent`, `uploadFile`, `listSources` tool'ları (varsayılan projeksiyon — FR-3.2.2) — Claude Code'a eklenebilir olduğu uçtan uca test edilir.
5. MCP client: config'de tanımlı en az bir harici MCP'ye bağlanabilme (herhangi bir MCP ile kanıtlanır; Azure DevOps/Jira gibi kayıtları kullanıcı kendi ortamına göre config'e ekler).
6. Basit API key koruması, docker-compose örneği, "kendi projen için özelleştirme" README'si.
7. Kabul senaryosu: YGÖ şablonları `knowledge/`'a konur → image build → deploy → Claude Code'dan MCP ile bağlanılır → "şablona göre şu proje için YGÖ yaz" → md çıktı üretilir.

V1'e **dahil değil:** knowledge graph, reranking (config anahtarı ayrılır, implementasyon V1.5), `addRepo`/`addMCP` runtime endpoint'leri, docx çıktı, web UI, federasyon (FR-16), OpenAPI'den otomatik tool üretimi (V1'de harici servis ihtiyacı MCP client ile karşılanır). v0.6 eklerinin hiçbirinin **implementasyonu** V1'de değildir — V1'e giren yalnız iki **sözleşme** kararıdır: çoğul `agents:`/`skills:` şeması (FR-6.1) ve bloklu yanıt zarfı (FR-11.1); ikisi de kod maliyeti değil, sonradan istemci/config kırmamak için baştan doğru yazılan format kararıdır.

**Yol haritası özeti:** V1 (yukarısı; sözleşme istisnaları: çoğul `agents:`/`skills:` şeması FR-6.1, bloklu yanıt zarfı FR-11.1) → V1.5 (docx çıktı, reranking, `addRepo`, web fetch, manuel tool tanımı; çoklu agent/skill implementasyonu FR-6, pass-through FR-11.2, SSE FR-9.5, inbound webhook FR-12, istek günlüğü FR-13, statik tool projeksiyonu FR-3.2.1, katalog şeması FR-15.1/15.2, repo özet wiki örneği FR-4.4) → V2 (knowledge graph — Cognee tarzı üçlü depolama, `addMCP`, UI, OpenAPI tool'ları; KB geri-besleme FR-14, `mcp_proxy` hedefi FR-3.2.1, LLM'li otomatik tarif FR-15.3, federasyon/hub FR-16) → V3+ (serbest A2A ağları — decentralized; tampon sürüm). **Kabul çizgisi:** `senaryolar.md`'deki üç use case anahatlarıyla **en geç V3'te** uçtan uca çalışır (US-3: centralized/federated) — bu plan çizgiyi V2'de yakalar, V3 tampondur. **Sıralama zorunluluğu:** zarf (FR-11) → projeksiyon+katalog (FR-3.2.1, FR-15) → otomatik tarif (FR-15.3) → federasyon (FR-16); federasyonu öne çekmek katalogsuz "isabetsiz hub" üretir.
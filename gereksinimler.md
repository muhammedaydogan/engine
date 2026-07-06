# Generic AI Agent Container — Gereksinim Dokümanı (v0.3 — Taslak)

> Amaç: Herhangi bir kişi veya projenin, kendi bilgi tabanı (knowledge base / knowledge graph), araçları (MCP, repo, web, harici API) ve talimatlarıyla özelleştirebileceği; hem sohbet asistanı hem kod/doküman üretim aracı olarak çalışabilen; ve dışarıya kendisini bir **MCP server** olarak sunabilen generic bir Docker container'ı geliştirmek.

---

## 1. Kapsam ve Kullanım Senaryoları

| # | Senaryo | Açıklama |
|---|---------|----------|
| S1 | Doküman üretimi | Şablonlar önden KB'ye yüklenir; container ayağa kalkınca "mevcut şablona göre YGÖ yaz" denir, proje bilgileri verilir, çıktı üretilir. |
| S2 | Kod asistanı | Repo(lar) taranır; örnek kod, refactor, analiz yaptırılır. |
| S3 | MCP olarak dış kullanım | Kişi kendi Claude Code'una bu container'ı MCP server olarak tanıtır; tek bağlantıyla büyük bir knowledge base'e erişir. |
| S4 | Canlı veri sorgulama | Spring AI destekli harici Java servisinin endpoint'leri üzerinden canlı DB verisi vb. çekilir. |
| S5 | Container'lar arası iletişim | Bir agent container başka bir agent container'a istek atar. **(Kapsam dışı — sonraki iş, sadece mimaride kapı açık bırakılacak.)** |

> Not: ALM entegrasyonu (Azure DevOps / Jira) ayrı bir senaryo değildir; ilgili MCP server'ın config'e eklenmesinden ibarettir. FR-3.1'in bir kullanım örneği olarak geçer.

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

### FR-3: MCP — İki Yönlü
- **FR-3.1 (MCP Client):** Uygulama, konfigürasyonda tanımlı harici MCP server'lara bağlanıp onların tool'larını kullanabilmeli (stdio ve HTTP/SSE transport). Örnek: Azure DevOps / Jira MCP server'ı config'e eklenerek iş kayıtları okunabilir/yazılabilir — bu tür ALM entegrasyonları ayrı bir gereksinim değil, bu maddenin doğal sonucudur.
- **FR-3.2 (MCP Server):** Uygulama kendisini dışarıya MCP server olarak sunabilmeli; böylece Claude Code / Claude Desktop / başka agent'lar bu container'ı tek bir MCP olarak ekleyip KB'ye ve tool'larına erişebilmeli. Sunulacak asgari tool'lar: `queryKnowledgeBase`, `askAgent` (talimatlı üretim), `uploadFile`/`addSource`, `listSources`.
- **FR-3.3** MCP tanımları önden (config dosyası) veya sonradan (`addMCP` endpoint'i) yapılabilmeli. `addMCP` → V2.
- **FR-3.4 (Kapalı ağ / air-gap):** stdio tipi MCP'ler genelde `npx`/`uvx` ile ilk çalıştırmada paket indirir; internetsiz ortamda bu çalışmaz. Bu yüzden MCP bağımlılıkları **build sırasında imaja gömülebilmeli** (proje imajının Dockerfile'ında `npm install`/`pip install` katmanı); runtime'da hiçbir paket indirme varsayımı olmamalı. Container, imaja gömülü MCP binary/script'ini doğrudan çalıştırabilmeli.

### FR-4: Repo Erişimi
- **FR-4.1** Git repo'ları (HTTPS/SSH, token'lı private repo dahil) clone edilip taranabilmeli.
- **FR-4.2** Repo içeriği KB'ye kaynak olarak eklenebilmeli (`addRepo`).
- **FR-4.3** V1'de repo, build-time'da statik dosya gibi kopyalanıp index'lenebilir; canlı `addRepo` endpoint'i V1.5.

### FR-5: Web Erişimi
- **FR-5.1** Belirli URL'lerin içeriği çekilebilmeli (fetch) ve isteğe bağlı olarak KB'ye eklenebilmeli.
- **FR-5.2** Web araması (search) — bir arama API'si anahtarı gerektirir (Brave, Tavily, SearXNG vb.), **opsiyonel özellik** olarak tasarlanmalı. Kurumsal kapalı ağlarda kapatılabilmeli.
- **FR-5.3** Erişilebilecek domain'ler allowlist/denylist ile sınırlandırılabilmeli (güvenlik).

### FR-6: Talimat / Persona Yönetimi
- **FR-6.1** Agent'ın davranışı bir **system prompt / persona dosyası** ile tanımlanabilmeli (build-time config).
- **FR-6.2** Runtime'da talimat eklenebilmeli/güncellenebilmeli (`setInstructions` endpoint'i veya sohbet içinden). Kalıcı talimat değişikliği volume'a yazılmalı.
- **FR-6.3** Aynı imajdan farklı persona'larla farklı container'lar ayağa kaldırılabilmeli (config-driven).

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

### FR-10: Genericlik / Özelleştirme
- **FR-10.1** Tüm özelleştirme **tek bir config yapısından** yapılabilmeli (ör. `agent.yaml` + `knowledge/` klasörü + `.env`):
  - persona/talimatlar, LLM sağlayıcı ve modeller, MCP listesi, tool tanımları, KB kaynak listesi, güvenlik ayarları.
- **FR-10.2** Yeni bir proje için özelleştirme akışı: base image al → config + dosyalar koy → build → deploy. Kod değişikliği gerektirmemeli.
- **FR-10.3** Base image ile proje imajı ayrılmalı: `FROM agent-base` + `COPY knowledge/ config/` şeklinde ince bir katman.

---

## 3. Fonksiyonel Olmayan Gereksinimler

- **NFR-1 Güvenlik:**
  - Secrets asla imaja gömülmez (env/secret mount).
  - API'ye erişim en azından basit bir API key ile korunmalı (container internete/ortak ağa açılırsa şart).
  - Prompt injection farkındalığı: web'den/repodan çekilen içerik "talimat" değil "veri" olarak ele alınmalı (tam çözümü yok, ama tasarımda ayrım gözetilmeli).
  - Outbound ağ erişimi kısıtlanabilmeli (allowlist).
- **NFR-2 Kalıcılık:** KB index'i, yüklenen dosyalar, talimatlar volume'da; container stateless kalabilmeli.
- **NFR-3 Gözlemlenebilirlik:** Yapılandırılabilir loglama; her LLM çağrısının token/maliyet kaydı (kurumsalda maliyet takibi kaçınılmaz soru olur).
- **NFR-4 Kaynak:** Tek container, makul RAM ile çalışmalı. Embedding lokalde yapılacaksa CPU'da çalışan küçük bir model tercih edilmeli; aksi halde embedding de LLM sağlayıcısından alınmalı (config). İlk hedef sağlayıcı OpenAI-uyumlu endpoint olduğundan (FR-1.2) varsayılan yol embedding'in de bu endpoint'ten alınmasıdır; lokal CPU modeli yedek seçenektir.
- **NFR-5 Taşınabilirlik:** Hedef deploy ortamı ikilidir: tek sunucuda docker-compose **ve** kurumsal orkestratör (Kubernetes/OpenShift vb.) — ikisi de desteklenmeli. Temel gereksinim sadece Docker (ve docker-compose) ile çalışabilmek, Kubernetes zorunluluğu olmamalı; tek container + volume + env-config yapısı sayesinde imaj orkestratör üzerinde de değişiklik gerektirmeden koşabilmeli.
- **NFR-6 Çoklu kullanıcı:** V1'de **tek kiracı / tek KB** varsayımı. Kullanıcı bazlı izolasyon istenirse "proje başına container" modeli kullanılır (zaten tasarımın doğası bu). Container içi multi-tenancy → gereksiz efor, yapılmamalı.
- **NFR-7 Offline çalışabilirlik:** Container, internetsiz/kapalı ağda tam işlevli çalışabilmeli: LLM ve embedding lokal endpoint'ten (FR-1.2), MCP bağımlılıkları imajdan (FR-3.4), KB imaj/volume'dan. İnternetsiz ortamda çalışıldığı tek bir environment variable ile bildirilebilmeli (ör. `OFFLINE_MODE=true`); bu bayrak set edildiğinde internet gerektiren tüm özellikler (web fetch/search, cloud MCP'ler, dış API çağrıları) otomatik olarak devre dışı kalmalı — tek tek config ile kapatmak gerekmemeli. Devre dışı özellik çağrıldığında uygulama hata üretmemeli; "offline modda kapalı" şeklinde anlamlı bir yanıt dönmeli.

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

**Kapsam dışı / sonraya (kullanıcının da dediği gibi):**
- **Container'lar arası agent iletişimi (S5):** Teknik olarak kolay (her container zaten MCP server olduğu için A container'ı B'yi MCP olarak ekleyebilir — mimari bu kapıyı bedavaya açıyor), ama orkestrasyon/döngü/maliyet soruları var. V3+.

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
2. `agent.yaml` ile: persona/system prompt, LLM sağlayıcı + model, embedding kaynağı.
3. REST API: `/chat` (talimatlı üretim, KB'yi RAG olarak kullanır, MCP tool'larını sınırlı agent döngüsüyle çağırabilir — FR-9.1), `/query` (saf KB sorgusu), `/upload` + `/reindex` (runtime dosya ekleme, volume'a kalıcı), `/sources` (listele/sil), `/health`.
4. **MCP server modu:** `queryKnowledgeBase`, `askAgent`, `uploadFile`, `listSources` tool'ları — Claude Code'a eklenebilir olduğu uçtan uca test edilir.
5. MCP client: config'de tanımlı en az bir harici MCP'ye bağlanabilme (herhangi bir MCP ile kanıtlanır; Azure DevOps/Jira gibi kayıtları kullanıcı kendi ortamına göre config'e ekler).
6. Basit API key koruması, docker-compose örneği, "kendi projen için özelleştirme" README'si.
7. Kabul senaryosu: YGÖ şablonları `knowledge/`'a konur → image build → deploy → Claude Code'dan MCP ile bağlanılır → "şablona göre şu proje için YGÖ yaz" → md çıktı üretilir.

V1'e **dahil değil:** knowledge graph, reranking (config anahtarı ayrılır, implementasyon V1.5), web search, `addRepo`/`addMCP` runtime endpoint'leri, docx çıktı, web UI, agent'lar arası iletişim, OpenAPI'den otomatik tool üretimi (V1'de harici servis ihtiyacı MCP client ile karşılanır).

**Yol haritası özeti:** V1 (yukarısı) → V1.5 (docx çıktı, reranking, `addRepo`, web fetch, manuel tool tanımı) → V2 (knowledge graph — Cognee tarzı üçlü depolama, `addMCP`, web search, UI, OpenAPI tool'ları) → V3 (agent-to-agent).
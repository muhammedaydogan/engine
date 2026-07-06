# Generic AI Agent Container — Mimari Doküman (v0.1 — Taslak)

> Bu doküman, [gereksinimler.md](gereksinimler.md)'de tanımlanan sistemin **UI'sız container** halinin mimarisini tanımlar. Arayüz olarak yalnızca REST API ve MCP server vardır. Temel tasarım hedefi **modülerlik**tir: LLM sağlayıcısı, hafıza (memory/KB) sistemi, web search, vektör deposu gibi her dış bağımlılık tak-çıkar (pluggable) olmalı; örneğin V1'deki vektör RAG hafızası ileride cognee, Graphiti gibi alternatiflerle kod değişikliği minimumda tutularak değiştirilebilmelidir.

---

## 1. Mimari İlkeler

1. **Port & Adapter (Hexagonal):** Çekirdek uygulama (core) hiçbir somut sağlayıcıyı tanımaz; yalnızca kendi tanımladığı arayüzlerle (port) konuşur. Anthropic, LanceDB, cognee, Brave Search — hepsi birer adapter'dır ve core'a dokunmadan eklenir/çıkarılır.
2. **Config-driven composition:** Hangi adapter'ın kullanılacağı `agent.yaml` + environment variable'lardan okunur; uygulama açılışında bir composition root tüm bağımlılıkları kurar. Adapter değiştirmek = config'de bir isim değiştirmek.
3. **Framework kilidi yok:** LangChain/LlamaIndex gibi framework'ler **core'a giremez**. Gerekirse yalnızca bir adapter'ın iç implementasyonunda kullanılabilir. Bu, "hafıza sistemini değiştirince yarım framework'ü söküyorum" durumunu engeller.
4. **İnce arayüz katmanları:** REST ve MCP server, aynı uygulama servislerini çağıran ince adaptörlerdir. İş mantığı hiçbir zaman route/tool handler içinde yaşamaz — böylece bir yetenek her iki arayüzden de otomatik ve tutarlı sunulur.
5. **Fail-fast config:** Hatalı/eksik config ile container yarı çalışır durumda ayağa kalkmaz; açılışta şema doğrulaması yapılır, net hata mesajıyla düşülür.
6. **Offline-first (NFR-7):** `OFFLINE_MODE=true` iken internet gerektiren tüm adapter'lar otomatik olarak "disabled" davranışlı null implementasyonlarına düşer; uygulama hata üretmez.
7. **Stub ile kapı açma:** Henüz yapılmayacak özellikler (web search) port + null adapter olarak mimaride yer alır; implementasyon sonradan yalnızca yeni bir adapter yazarak gelir.

---

## 2. Teknoloji Seçimi

| Bileşen             | Seçim                                 | Gerekçe                                                                                                                               |
| ------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Dil                 | **Python 3.12+**                      | Tak-çıkar denenecek hafıza sistemlerinin (cognee, Graphiti, LightRAG) tamamı Python. MCP resmi SDK'sı olgun. Ekosistem en geniş.      |
| Web framework       | **FastAPI**                           | Async, Pydantic entegrasyonu, OpenAPI otomatik; MCP SDK ile aynı ASGI süreçte koşabilir.                                              |
| MCP                 | **Resmi `mcp` Python SDK**            | Hem client hem server; stdio + streamable HTTP transport.                                                                             |
| Config/şema         | **Pydantic v2 + pydantic-settings**   | `agent.yaml` doğrulama, env override, fail-fast.                                                                                      |
| Vektör DB (default) | **LanceDB** (embedded, dosya tabanlı) | Sunucusuz, tek container'a gömülür (NFR-4/5), volume'da kalıcı (NFR-2). Port arkasında olduğu için Chroma/Qdrant'a geçiş adapter işi. |
| HTTP istemci        | httpx                                 | Async, OpenAI-uyumlu generic endpoint çağrıları için.                                                                                 |

> Not: Bu tablo bir öneri-karardır; gereksinimler.md Bölüm 7 Soru 2'nin cevabıdır. Java/Spring AI tercih edilirse aynı port/adapter deseni orada da uygulanabilir, ancak cognee tarzı deneyler için Python güçlü tavsiyedir.

---

## 3. Yüksek Seviye Görünüm

```
                    ┌─────────────────────── Container ──────────────────────────────┐
                    │                                                                │
  Claude Code ──MCP──▶ ┌────────────┐      ┌──────────────────────────┐              │
  (stdio/HTTP)      │  │ MCP Server │──┐   │      Application Core     │             │
                    │  └────────────┘  │   │                          │              │
  curl / servis ─REST─▶┌────────────┐  ├──▶│  AgentService  (chat/üretim döngüsü)
                    │  │  REST API  │──┘   │  KnowledgeService (ingest/query/sources)
                    │  └────────────┘      │  ToolRegistry  (tool toplama/çağırma)
                    │   (driving           │  SessionStore  (konuşma geçmişi)
                    │    adapters)         └──────────┬───────────────┘              │
                    │                                 │ portlar (interface)          │
                    │        ┌────────────────────────┼────────────────────┐         │
                    │        ▼            ▼           ▼          ▼         ▼         │
                    │  ┌──────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌───────────┐
                    │  │   LLM    │ │  Memory  │ │WebSearch│ │  MCP   │ │ HTTP Tool │
                    │  │ Provider │ │ Backend  │ │ (stub)  │ │ Client │ │ (FR-7)    │
                    │  └────┬─────┘ └────┬─────┘ └────┬────┘ └───┬────┘ └─────┬─────┘
                    │       │            │            │          │            │
                    └───────┼────────────┼────────────┼──────────┼────────────┼──────┘
                            ▼            ▼            ▼          ▼            ▼
                     Anthropic /   vector_rag    NoopSearch   harici     Spring AI
                     OpenAI-uyumlu (LanceDB) |   (V2'de       MCP        Java servisi
                     lokal endpoint  cognee* |    Brave vb.)  server'lar  vb.
                                     (iskelet)
```

- **Driving adapters** (sol): dış dünyadan çağrı alan REST ve MCP server. İkisi de aynı servisleri çağırır.
- **Application core** (orta): iş mantığı. Yalnızca port arayüzlerini bilir.
- **Driven adapters** (alt): dış sistemlere giden somut implementasyonlar. Config ile seçilir.

---

## 4. Modül / Dizin Yapısı

```
ngine/
├── app/
│   ├── main.py                    # Composition root: config oku → adapter'ları kur → API'yi başlat
│   ├── registry.py                # Adapter kayıt defteri (isim → factory)
│   │
│   ├── config/
│   │   ├── schema.py              # Pydantic modelleri: AgentConfig, LLMConfig, MemoryConfig...
│   │   └── loader.py              # agent.yaml + env birleşimi, doğrulama, OFFLINE_MODE uygulaması
│   │
│   ├── core/                      # ← dış bağımlılık YOK (FastAPI, LanceDB vb. import edilemez)
│   │   ├── models.py              # Domain tipleri: Message, ToolCall, MemoryHit, SourceInfo, Citation...
│   │   ├── agent.py               # AgentService: LLM tool-calling döngüsü
│   │   ├── knowledge.py           # KnowledgeService: ingest/query/list/remove orkestrasyonu
│   │   ├── tools.py               # ToolRegistry: tüm ToolSource'ları toplar, isim çakışmasını çözer
│   │   ├── session.py             # SessionStore portunu kullanan oturum mantığı
│   │   └── ports/                 # ← tak-çıkar noktaları (Protocol tanımları)
│   │       ├── llm.py             # LLMProvider
│   │       ├── embedding.py       # EmbeddingProvider
│   │       ├── memory.py          # MemoryBackend  ★ cognee buraya takılır
│   │       ├── websearch.py       # WebSearchProvider (stub'ın portu)
│   │       ├── toolsource.py      # ToolSource (MCP client, HTTP tool, websearch hepsi bunu uygular)
│   │       └── storage.py         # SessionStore, FileStore (volume erişimi)
│   │
│   ├── adapters/
│   │   ├── llm/
│   │   │   ├── anthropic.py
│   │   │   └── openai_compat.py   # OpenAI, Azure OpenAI, Ollama, vLLM — tek adapter (FR-1.2)
│   │   ├── embedding/
│   │   │   ├── openai_compat.py   # API'den embedding
│   │   │   └── local.py           # CPU'da küçük model (ör. ONNX) — NFR-4 opsiyonu
│   │   ├── memory/
│   │   │   ├── vector_rag/        # V1 default MemoryBackend implementasyonu
│   │   │   │   ├── backend.py     # MemoryBackend'i uygular; alttaki üçünü orkestre eder
│   │   │   │   ├── chunker.py     # iç port: Chunker (md/pdf/docx/kod farkındalıklı)
│   │   │   │   ├── extractor.py   # iç port: dosya → metin (pdf, docx, xlsx...)
│   │   │   │   └── stores/
│   │   │   │       ├── lancedb.py # iç port: VectorStore
│   │   │   │       └── chroma.py  # (opsiyonel alternatif)
│   │   │   └── cognee/            # ★ BOŞ İSKELET — MemoryBackend'i uygulayan sınıf,
│   │   │       └── backend.py     #   tüm metodları NotImplementedError; kayıt defterinde adı var
│   │   ├── websearch/
│   │   │   └── noop.py            # ★ STUB — arayüzü uygular, "disabled" cevabı döner
│   │   ├── mcp/
│   │   │   ├── client.py          # Harici MCP'lere bağlanır, tool'larını ToolSource olarak sunar
│   │   │   └── server.py          # Kendini MCP server olarak sunar (driving adapter)
│   │   └── tools/
│   │       └── http_tool.py       # Config'den manuel HTTP tool tanımı (FR-7, V1.5)
│   │
│   ├── api/                       # REST driving adapter (FastAPI)
│   │   ├── server.py              # ASGI app, API-key middleware, lifespan
│   │   └── routes/
│   │       ├── chat.py            # POST /chat
│   │       ├── query.py           # POST /query
│   │       ├── sources.py         # GET/POST/DELETE /sources, POST /upload, POST /reindex
│   │       └── health.py          # GET /health
│   │
│   └── ingest_cli.py              # `python -m app.ingest_cli` — build-time bake (FR-2.3)
│
├── config/
│   ├── agent.yaml                 # Ana konfigürasyon (örnek: Bölüm 7)
│   └── persona.md                 # System prompt / persona (FR-6.1)
├── knowledge/                     # Build-time index'lenecek dosyalar
├── docker/
│   ├── Dockerfile.base            # agent-base imajı
│   ├── Dockerfile.project         # FROM agent-base + COPY knowledge/ config/ + RUN ingest
│   └── docker-compose.yml
└── tests/
```

**Bağımlılık kuralı:** `core/` → hiçbir şeyi import etmez (stdlib + Pydantic hariç). `adapters/` ve `api/` → `core/`'u import eder. Ters yön yasak; CI'da import-linter ile zorlanır.

---

## 5. Portlar (Tak-Çıkar Noktaları)

Portlar Python `Protocol` olarak tanımlanır. Kritik olanların eskizleri:

### 5.1 MemoryBackend — hafıza sistemi değişim noktası ★

Port bilinçli olarak **domain dilinde** konuşur: içeri "kaynak", dışarı "atıflı sonuç". Chunk, embedding, vektör, graph — bunlar port'un DIŞINDA, adapter'ın iç detayıdır. Bu sayede graph tabanlı cognee ile vektör tabanlı RAG aynı arayüze sığar.

```python
class MemoryBackend(Protocol):
    async def ingest(self, source: SourceSpec) -> IngestReport:
        """Bir kaynağı (dosya/dizin/repo içeriği) hafızaya işler."""
    async def query(self, question: str, top_k: int = 8,
                    filters: QueryFilters | None = None) -> list[MemoryHit]:
        """Soruya en alakalı parçaları, kaynak atfıyla döner (FR-2.6)."""
    async def list_sources(self) -> list[SourceInfo]: ...
    async def remove_source(self, source_id: str) -> None: ...
    async def stats(self) -> MemoryStats:
        """Doküman/parça sayısı, index boyutu — /health ve gözlemlenebilirlik için."""

@dataclass
class MemoryHit:
    text: str
    score: float
    citation: Citation      # source_id, dosya adı, bölüm/sayfa
    metadata: dict[str, Any]
```

- **`vector_rag`** (V1 default): extract → chunk → embed → LanceDB. Chunker/Extractor/VectorStore kendi iç portlarıdır; yani hafıza backend'ini değiştirmeden sadece vektör DB'yi de değiştirebilirsin.
- **`cognee`** (iskelet): `CogneeMemoryBackend(MemoryBackend)` sınıfı ve registry kaydı vardır; metodlar `NotImplementedError("cognee backend V2")`. Config'de `memory.backend: cognee` seçilirse açılışta "bu backend henüz implement edilmedi" hatasıyla fail-fast.

### 5.2 LLMProvider ve EmbeddingProvider

```python
class LLMProvider(Protocol):
    async def complete(self, messages: list[Message],
                       tools: list[ToolSpec] | None = None,
                       params: LLMParams | None = None) -> LLMResponse: ...
    # LLMResponse: text | tool_calls + usage (token sayıları → NFR-3 loglama)

class EmbeddingProvider(Protocol):
    async def embed(self, texts: list[str]) -> list[list[float]]: ...
    @property
    def dimension(self) -> int: ...
```

Görev-bazlı model ataması (FR-1.3) composition root'ta çözülür: config'de `models.chat`, `models.embedding` ayrı tanımlanır; aynı adapter sınıfı farklı model adlarıyla iki kez örneklenebilir.

### 5.3 ToolSource — tüm tool'ların ortak kapısı

Agent'ın kullanabildiği her şey (KB sorgusu, web search, harici MCP tool'ları, HTTP tool'lar) tek tip bir kaynaktan gelir:

```python
class ToolSource(Protocol):
    async def list_tools(self) -> list[ToolSpec]: ...
    async def call_tool(self, name: str, arguments: dict) -> ToolResult: ...
```

`ToolRegistry` açılışta tüm ToolSource'ları toplar, isim çakışmalarını kaynak öneki ile çözer (`ado.createWorkItem`), LLM'e verilecek tool listesini üretir. Yeni bir tool türü eklemek = yeni bir ToolSource adapter'ı yazmak; agent döngüsüne dokunulmaz.

### 5.4 WebSearchProvider — stub

```python
class WebSearchProvider(Protocol):
    async def search(self, query: str, max_results: int = 5) -> WebSearchResponse: ...

class NoopWebSearch:
    """V1 stub'ı. Arayüz ve tool tanımı gerçek; implementasyon 'kapalı' cevabı döner."""
    async def search(self, query, max_results=5) -> WebSearchResponse:
        return WebSearchResponse(results=[], status="disabled",
            message="Web search bu kurulumda etkin değil.")
```

Web search, `WebSearchToolSource` üzerinden agent'a normal bir tool olarak görünür (`webSearch`). Böylece V2'de Brave/Tavily/SearXNG adapter'ı yazıldığında **agent, API ve MCP tarafında sıfır değişiklik** gerekir. `websearch.provider: none` (default) → Noop; `OFFLINE_MODE=true` → config ne derse desin Noop'a zorlanır.

### 5.5 SessionStore

```python
class SessionStore(Protocol):
    async def get(self, session_id: str) -> list[Message]: ...
    async def append(self, session_id: str, messages: list[Message]) -> None: ...
    async def delete(self, session_id: str) -> None: ...
```

Default: volume'a JSONL yazan `FileSessionStore` (NFR-2). `/chat` opsiyonel `session_id` alır; verilmezse tek atımlık (stateless) çalışır. İleride Redis vb. yine adapter işi.

---

## 6. Registry ve Adapter Seçimi

`registry.py` basit bir isim → factory sözlüğüdür; her adapter modülü kendini kaydeder:

```python
MEMORY_BACKENDS:    dict[str, MemoryFactory]    = {"vector_rag": ..., "cognee": ...}
LLM_PROVIDERS:      dict[str, LLMFactory]        = {"anthropic": ..., "openai_compat": ...}
WEBSEARCH_PROVIDERS: dict[str, SearchFactory]    = {"none": NoopWebSearch}
VECTOR_STORES:      dict[str, StoreFactory]      = {"lancedb": ..., "chroma": ...}
```

- Composition root (`main.py`) config'deki ismi registry'den çözer; isim yoksa **açılışta** hata: `Bilinmeyen memory backend: 'cognee2'. Mevcutlar: vector_rag, cognee`.
- Container dışından paketle adapter eklemek istenirse Python **entry-points** mekanizmasıyla registry genişletilebilir (`ngine.plugins` grubu) — V2 kapısı, bedava açık.

**cognee'yi gerçekten takma adımları (ileride):** ① `adapters/memory/cognee/backend.py` içinde beş metodu doldur ② `pyproject.toml`'a `cognee` bağımlılığını ekle ③ `agent.yaml`'da `memory.backend: cognee` yap. Core, API, MCP, agent döngüsü — hiçbirine dokunulmaz.

---

## 7. Konfigürasyon

```yaml
# config/agent.yaml
persona_file: persona.md

llm:
  provider: openai_compat # anthropic | openai_compat
  base_url: ${LLM_BASE_URL} # lokal vLLM/Ollama olabilir (FR-1.2)
  models:
    chat: qwen2.5-72b
    embedding: bge-m3 # FR-1.3: görev başına model

memory:
  backend: vector_rag # ★ tak-çıkar noktası: vector_rag | cognee
  vector_rag:
    store: lancedb
    data_dir: /data/index
    chunk: { size: 1000, overlap: 150 }

websearch:
  provider: none # V1: sadece 'none' (stub). V2: brave | tavily | searxng

tools:
  mcp_servers:
    - name: ado
      transport: stdio
      command: ["node", "/opt/mcp/azure-devops/index.js"] # imaja gömülü (FR-3.4)
      requires_internet: true # OFFLINE_MODE'da otomatik atlanır
  http_tools: [] # FR-7 manuel tool tanımları (V1.5)

api:
  require_api_key: true # anahtar env'den: API_KEY

limits:
  max_tokens_per_request: 8000 # NFR-3
```

**Environment variable'lar (secrets + ortam bayrakları):** `LLM_API_KEY`, `LLM_BASE_URL`, `API_KEY`, `OFFLINE_MODE`, `LOG_LEVEL`. Kural: config dosyası _davranışı_, env _sırları ve ortamı_ tanımlar (FR-1.4). Env, yaml'daki `${VAR}` yer tutucularına açılır.

### OFFLINE_MODE=true davranış matrisi (NFR-7)

| Bileşen       | Davranış                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------------------- |
| WebSearch     | Config'e bakılmaksızın Noop'a zorlanır; tool "kapalı" cevabı döner                                   |
| MCP client    | `requires_internet: true` işaretli server'lar bağlanmaz; tool listesinden düşer, log'a bilgi yazılır |
| LLM/Embedding | `base_url` zorunlu (lokal endpoint); public cloud URL'i tespit edilirse açılışta uyarı               |
| Ingest        | Sadece lokal dosya/volume kaynakları; URL kaynakları reddedilir (anlamlı mesajla)                    |
| /health       | `"offline_mode": true` ve devre dışı bileşen listesi raporlanır                                      |

---

## 8. Temel Akışlar

### 8.1 /chat (ve MCP `askAgent`) — agent döngüsü

```
istek → SessionStore.get(geçmiş) → persona + geçmiş + soru → LLM.complete(tools=ToolRegistry.specs())
  ↺ LLM tool_call döndürdükçe: ToolRegistry.call() → sonucu mesajlara ekle → tekrar LLM
    (queryKnowledgeBase → MemoryBackend.query; webSearch → Noop "kapalı"; ado.* → MCP client)
→ nihai metin + Citation listesi + usage → SessionStore.append → cevap
```

Döngü sınırlıdır (`max_tool_rounds`, default 8) — sonsuz tool döngüsü koruması. Her LLM çağrısının token kullanımı yapılandırılmış log'a yazılır (NFR-3).

### 8.2 Ingest — build-time ve runtime aynı kod

- **Build-time (V1 ana yolu):** `Dockerfile.project` içinde `RUN python -m app.ingest_cli --dir knowledge/` → `KnowledgeService.ingest()` → aktif MemoryBackend. Index imaja gömülür.
- **Runtime:** `POST /upload` + `POST /reindex` aynı `KnowledgeService`'i çağırır; çıktı volume'a (`/data`) yazılır. Açılışta volume'daki index imajdakinin üstüne monte edilirse o kullanılır (NFR-2, FR-2.4).

### 8.3 MCP server modu

`adapters/mcp/server.py`, MCP SDK ile şu tool'ları tanımlar ve **doğrudan core servislerine** bağlar (FR-3.2):

| MCP tool             | Bağlandığı servis                    |
| -------------------- | ------------------------------------ |
| `queryKnowledgeBase` | `KnowledgeService.query`             |
| `askAgent`           | `AgentService.run`                   |
| `uploadFile`         | `KnowledgeService.ingest` (volume'a) |
| `listSources`        | `KnowledgeService.list_sources`      |

Transport: streamable HTTP (REST ile aynı porttan `/mcp` path'i) + istenirse stdio. REST route'ları ile MCP tool'ları aynı servis metodlarını çağırdığı için davranış farkı oluşamaz.

---

## 9. Docker İmaj Mimarisi (FR-10)

```
Dockerfile.base  → agent-base:X.Y
  Python + uygulama kodu + MCP runtime bağımlılıkları (node dahil, FR-3.4 için)
  (opsiyonel) lokal embedding modeli

Dockerfile.project → FROM agent-base
  COPY config/ knowledge/  →  RUN python -m app.ingest_cli  →  index imajda hazır
```

- Proje imajı ince bir katmandır; **kod değişikliği gerektirmez** (FR-10.2/10.3).
- stdio MCP bağımlılıkları base veya proje imajına build sırasında kurulur (`npm ci`/`pip install`); runtime'da paket indirme yoktur (FR-3.4).
- Volume düzeni: `/data/index` (KB), `/data/uploads`, `/data/sessions`, `/data/instructions` — container stateless (NFR-2).

---

## 10. Kapsam Eşlemesi ve Stub Envanteri

| Bileşen                                                                                                     | V1 durumu                                                                                  |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| REST API, MCP server/client, vector_rag memory, build-time ingest, upload/reindex, session (dosya), API key | **Implement edilir**                                                                       |
| Web search                                                                                                  | **Port + Noop stub** — tool görünür, "kapalı" cevabı döner; implementasyon yok             |
| cognee memory backend                                                                                       | **Boş iskelet** — sınıf + registry kaydı var, metodlar NotImplementedError                 |
| HTTP tool (FR-7 manuel tanım)                                                                               | Port + config şeması hazır, adapter V1.5                                                   |
| Web UI                                                                                                      | **Yok** — bilinçli olarak mimaride yer almaz; ihtiyaç olursa REST'in üstüne ayrı container |
| addMCP, addRepo runtime endpoint'leri                                                                       | Yok (V1.5/V2); ToolSource/SourceSpec tasarımı kapıyı açık tutar                            |

---

## 11. Açık Mimari Kararlar

1. **LanceDB vs Chroma:** İkisi de embedded; LanceDB default önerisi (dosya tabanlı, sıfır süreç). İlk prototipte Türkçe içerikle ikisi de VectorStore portu arkasında denenebilir.
2. **Lokal embedding modeli imaj boyutu:** ONNX modeli base'e gömmek imajı ~0.5-1 GB büyütür; alternatif proje imajına opsiyonel katman. Karar: proje imajına.
3. **MCP server transport önceliği:** Claude Code hedefi için streamable HTTP birincil; stdio ikincil (aynı imaj `--stdio` bayrağıyla).

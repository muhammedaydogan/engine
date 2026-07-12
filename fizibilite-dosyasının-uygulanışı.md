# Fizibilite Maddelerinin Uygulanışı — Yorumlar ve Cevaplar

> **Cevap turu (Claude):** Tikli maddeler = yorumun kabul edildi, gereksinim
> güncellemesine o şekilde girecek. Tiksiz maddelerde sorunun cevabı altta —
> okuyup karar verince tiki sen atarsın. Hepsi netleşince Ö1–Ö14'ü buradaki
> eklerle birlikte numaralı öneri listesi olarak çıkarıp onayına sunacağım.
> Sürüm notu: fizibilite "gereksinimler v0.3 → v0.4" diyordu; doküman bu arada
> v0.5'e ilerlediği için değişiklikler **v0.5 → v0.6** olarak işlenecek.

- [x] C1. uygun. ek olarak agent'a skiller de yüklenebilecek. x y z agentlar a b c de skill olsun. bunlar konfigürasyon üzerinde birbirleriyle kombin yapılabilecek. Bu da şunu gerektirir. dışarıdan (MCP veya REST katmanından) gelen istek toollara veya knowledge-graph core sorgu atmadan önce ara bir katmana uğrar. Burada LangChain benzeri birşey tercih edilebilir. Bu katmanda toollar mcp'ler agent ve skiller ve knowledge-core bağlanır ve burada oluşturulan agent prompt a göre bunları ihtiyaca göre kullanır. agent ve skill dosyaları build-time veya runtime'da yüklenebilir. runtime'da güncellenebilir.
  - **Cevap:** Kabul — Ö2 genişletilerek işlenecek: config'de **çoklu agent + çoklu skill** (`agents:` ve `skills:` listeleri). Skill = yeniden kullanılabilir talimat modülü + opsiyonel tool alt-kümesi; agent = persona + skill listesi + tool seti (+ pass-through/routing ayarları). Kombinasyon serbest, istekte `agent:` ile seçilir. Agent/skill dosyaları build-time imajdan veya runtime volume'dan yüklenir, runtime'da güncellenebilir (FR-6.2'nin genellemesi). Şema V1'den çoğul, implementasyon V1.5 — fizibilitedeki yerleşim korunur.
  - Tarif ettiğin ara katman mimaride zaten var: **Agent Core** (API katmanı → Agent Core → tool'lar / KB / LLM; her istek buradan geçer). C1 bu katmanı "config'den kompoze edilen agent+skill"i çalıştırır hâle getiriyor; yeni bir katman gerekmiyor.
  - **LangChain notu:** teknoloji seçimi olduğu için gereksinim dokümanına yazılmaz, mimari turunun konusu. Mevcut mimari karar (ADR-3) agent döngüsünü bilinçli olarak kendi kodumuz tutuyor — pass-through, kaynak atfı ve token muhasebesi üzerinde tam kontrol için; C2'nin "bit-bit aynen dön" garantisi de tam bu kontrolü gerektiriyor. Skill kompozisyonu framework'süz kurulabilir; mimari revizyonuna "karmaşıklık artarsa LangGraph vb. değerlendirilir" notu düşeceğim. Farklı düşünüyorsan mimari turunda tartışalım.

- [ ] C2. dal+zarf sözleşmesi nedir? konsept ney? ingilizcesini söylersen anlardım muhtemelen.
  - **Cevap:** İngilizcesi: **response envelope** (zarf) + agent döngüsünde bir **branch** (dal) + **pass-through**.
  - **Zarf (response envelope):** `/chat` ve `askAgent` cevabının düz metin değil, **bloklu yapı** olarak dönmesi sözleşmesi:

    ```json
    {
      "blocks": [
        { "type": "llm_text", "text": "Analize göre ...", "citations": ["ygo-sablon.md § Giriş"] },
        { "type": "verbatim", "source": "spring-ai__hesapla",
          "content": { "not": "tool çıktısı bit-bit aynen" }, "content_hash": "sha256:..." }
      ],
      "usage": { "prompt_tokens": 1234, "completion_tokens": 567 }
    }
    ```

  - **Dal (branch):** agent döngüsüne eklenen tek koşul — tool config'te `passthrough: true` işaretliyse sonucu LLM bağlamına **sokma**, zarfa `verbatim` bloğu olarak aynen koy (`content_hash` = bit-bit korunduğunun kanıtı). LLM içeriği hiç görmediği için "milimetrik hesabı bozması" yapısal olarak imkânsızlaşır.
  - **Neden V1'e girmeli:** zarf, cevap formatının kendisi — sonradan düz metinden zarfa geçmek tüm istemcileri kırar. V1'de tek blok tipiyle (`llm_text`) döner ama format baştan zarftır; V1.5'teki pass-through implementasyonu sadece yeni blok tipi ekler.

- [x] C3. Burada kullanacağımız şey open-ai destekli LLM modellerinin shimmering effect gibi yerlerde de kullandıkları standart ile aynı olursa iyi olur. sse mi websocket mi bilmem. standart neyse oradan devam edelim.
  - **Cevap:** Standart tam olarak önerilendir: **SSE** (Server-Sent Events). OpenAI ve tüm uyumlu sağlayıcılar `stream: true`'da `text/event-stream` + `data: {delta chunk}` satırlarıyla akıtır; "shimmering/typing" efekti bu delta'lardan gelir — WebSocket değil. MCP'nin streamable HTTP transport'u da zaten SSE kullanır; yani REST ve MCP tarafı aynı desende birleşiyor. `/chat` stream modu OpenAI delta deseninde tanımlanacak; zarf (C2) blokları stream'de "blok başladı / delta / blok bitti" event'leri olarak akacak.

- [ ] C4. ne anlattığını anlamadım. Inbound webhook nedir nasıl çalışır HMAC nedir?
  - **Cevap — inbound webhook:** dış sistemin, bir olay olduğunda bizim container'daki bir URL'e (`POST /hooks/{name}`) kendiliğinden istek atması; polling'in tersi — biz sormuyoruz, olay olunca haber geliyor. US-1'deki kullanım: harici kod sistemi bir işi bitirince `POST /hooks/repo-updated` çağırır; config'te o hook'a bağlı aksiyon zinciri çalışır (`reindex` | agent profili çalıştır | MCP tool çağır — ör. "wiki MCP'sini tetikle, çıktıyı KB'ye yaz").
  - **HMAC (hash-based message authentication code):** endpoint dışarı açık, herkes POST atabilir. Paylaşılan bir secret ile gövde imzalanır: gönderen `HMAC-SHA256(secret, body)` değerini header'a koyar, biz aynı hesabı yapıp karşılaştırırız. Tutuyorsa istek gerçekten anlaşmalı sistemden gelmiş ve gövde yolda değişmemiş demektir. GitHub webhook'ları birebir bu deseni kullanır (`X-Hub-Signature-256`).
  - **Idempotency:** dış sistemler webhook'u garanti için tekrar gönderebilir (retry). Her teslimatın ID'si kaydedilir; görülen ID ikinci kez geldiğinde aksiyon tekrar çalıştırılmaz.
  - **202 + arka plan işi:** hook'un tetiklediği iş (reindex gibi) uzun sürebilir. Çağrıya hemen `202 Accepted + job_id` dönülür, iş arkada koşar, durumu `GET /jobs/{id}` ile izlenir — gönderen sistemin timeout'una takılmayız. (C4'ün eforunun "orta" olmasının sebebi bu küçük arka plan iş modelinin yeni kavram olması.)

- [x] C5. uygun zaten sqlite veya postgres db var. db-bağımsız olduğumuz için, bu kolay yapılır. Yani contract olur. sonra sqlite-adapter postgres adapter bunu implement eder.
  - **Cevap:** Kabul, bir düzeltmeyle: bugünkü tasarımda Postgres yok ve "db-bağımsızlık" henüz yazılı değil — katalog + usage ledger doğrudan **SQLite** (gömülü, `/data`). Dediğin gibi yapılacak: depolama bir **kontrat** arkasına alınır (`CatalogStore`, `UsageLedger`, yanına `RequestLog` eklenir); V1'de tek adapter SQLite'tır, Postgres adapter'ı ihtiyaç doğduğunda yazılır. "Tek container, ayrı sunucu süreci yok" sadeliği (NFR-4/5) korunur. Gereksinime giren kısım: istek günlüğü opsiyonel + retention + `query_history` tool'u; port/adapter kısmı mimari dokümanına işlenir.

- [x] C6. uygun. ama unutma içten dışa bağımlılık yok. dıştan içe kontraktlar aracılığıyla var. her şey adapter lü.
  - **Cevap:** Kabul. Bağımlılık yönü zaten mimarinin sözleşmesi: çekirdek hiçbir adapter'ı import edemez ve bu kural CI'da **import-linter** ile mekanik doğrulanıyor (mimari §9 — dokümanla değil araçla korunuyor). `save_to_kb`, çekirdek kontrat olan `KnowledgeBackend.ingest`'i çağıran bir builtin tool olur; adapter'a doğrudan dokunmaz. `origin=agent` etiketi atıflarda görünür; listelenip temizlenebilir.

- [ ] C7. anlamadım. genericlik mi korunmalı ama bunu içeri almadan nasıl generic olacağını anlamadım. hook falan mı koyacağız? kod yazıp push'layıp konfig'de onStart'ta run'la falan mı diyeceğiz.
  - **Cevap:** Kod push / onStart hook yok. "Reçete"nin anlamı: **çekirdeğe tek satır DeepWiki kodu girmiyor**; senaryo, üçü de zaten generic olan mevcut mekanizmaların salt config'le birbirine bağlanmasından ibaret:
    1. DeepWiki-muadili araç **ayrı bir MCP server** olarak koşar (kendi container'ı/süreci; bizim config'e FR-3.1 ile eklenir — Azure DevOps MCP'si eklemekten farksız).
    2. Repo değişince dış sistem `POST /hooks/repo-updated` çağırır (C4).
    3. Config'teki hook→aksiyon eşlemesi çalışır: "wiki MCP'sinin `updateWiki` tool'unu çağır → çıktıyı KB'ye yaz (C6)".
  - Ürünün bildiği tek şey "hook gelince config'teki aksiyon zincirini çalıştır". DeepWiki'ye özgü her şey kullanıcının config'inde ve kullanıcının bağladığı harici MCP'de. `examples/` altına konacak olan da kod değil, çalışan bir örnek kompozisyon: iki servisli docker-compose + agent.yaml + hook config'i. C7'nin eforunun "S (döküman/örnek)" olması bundan — yeni özellik değil, C4 + C6 + FR-3.1'in kullanım tarifi.

- [ ] C8. anlamadım.
  - **Cevap:** Bugün MCP yüzeyimiz sabit 5 generic tool: `queryKnowledgeBase`, `askAgent`, `uploadFile`, `listSources`, `removeSource`. US-2'nin şikâyeti: istemci agent bu 5 huniye bakınca içeride ne olduğunu anlamıyor; her iş "askAgent'a sor"a düşüyor. C8 = MCP yüzeyinin **config'den tanımlanması** (tool "projeksiyonu"):

    ```yaml
    expose:
      tools:
        - name: "mevzuat_ara"
          description: "Muhasebe mevzuatı KB alanında arama; madde no veya serbest metinle sor."
          target: { type: kb_query, filters: { source: "mevzuat/" } }
        - name: "malzeme_hesabi"
          description: "Milimetrik malzeme hesabı; sonuç LLM'e girmeden aynen döner."
          target: { type: mcp_proxy, server: "spring-ai-servis", tool: "hesapla", passthrough: true }
        - name: "ygo_yaz"
          description: "Şablona göre YGÖ üretir."
          target: { type: agent, profile: "ygo-yazari" }
    ```

  - Dört hedef tipi: `kb_query` (ön-filtreli KB sorgusu) | `agent` (C1 profili) | `mcp_proxy` (bağlı harici MCP tool'una köprü) | `http` (manuel HTTP tool). İstemci `tools/list` çektiğinde 5 generic huni yerine anlamlı ad + açıklamalar görür ve doğru tool'a route eder — US-2'nin "kendini iyi tarif etme"sinin mekanik yarısı (diğer yarısı C9/C10 kataloğu). `expose` config'i yoksa bugünkü 5 tool **varsayılan projeksiyon** olarak aynen kalır; kırılma yok.

- [x] C9. uygun mcp formatına uygun olmalı. MCP'ler bir standart falan var mı acaba?
  - **Cevap:** Kabul. MCP'nin resmî spec'i var (modelcontextprotocol.io) ve katalog, spec'in üç native yüzeyine döşenir — format icat etmiyoruz:
    1. **Server `instructions`** — initialize cevabındaki serbest metin alanı: sunucu kendini burada anlatır; katalog özeti buraya yazılır (istemci agent'lar bunu bağlamına alır).
    2. **Tool metadata** — her tool'un `description` + JSON Schema'sı (+ `annotations`: readOnlyHint vb.); C8 projeksiyonu bunları config'den doldurur.
    3. **Resources** — salt-okunur içerik yüzeyi; kataloğun tamamı bir MCP resource olarak da yayınlanabilir.
  - "Yetenek kataloğu" diye ayrıca bir MCP standardı yok; `describeCapabilities` sıradan bir tool olarak eklenir, içeriği yukarıdaki üç yüzeyle tutarlı üretilir.

- [x] C10. describe derse hepsi describe(toolNames: string[]) gibi bir sey yaparsa sadece secili olanlar.
  - **Cevap:** Kabul — imza `describeCapabilities(toolNames?: string[], detail?: "summary" | "full")` olacak: parametresiz → özet katalog; `toolNames` verilirse yalnız seçili tool'ların tam tarifi. Büyük kataloglarda istemcinin context'ini şişirmemenin doğru yolu. (Tarif üretimindeki **manual > auto > fallback** katmanlaması aynen kalıyor.)

- [ ] C11. böyle bir şeye niye ihtiyaç var anlamadım. diğer bağlı container'lar bir mcp'den ibaret değil mi?
  - **Cevap:** Haklısın — **bağlantı** için hiçbir şey gerekmiyor: hub bugün de 9 bankayı config'ine 9 MCP client kaydı olarak ekler ve çalışır ("kapı bedava" dediğimiz bu). C11 bağlantı değil, hub'ın **isabetli + hızlı + ucuz** cevap vermesi için gereken davranış paketi. Naif hâlde olan şu:
    - 9 banka × ~6 tool ≈ 54 tool tek listede hub LLM'ine gider; model her soruda "hangi bankaya sorsam" diye sırayla tool deneyerek gezer → yavaş, pahalı, isabetsiz (US-3'ün "hızlı ve isabetli" hedefinin tam tersi).
    - Kimse birleşik atıf üretmez — cevap "hangi bankanın hangi dokümanından geldi" diyemez.
    - Çökük bir banka döngüyü timeout'larla süründürür.
  - C11'in eklediği (tamamı hub tarafında config/davranış; spoke'lara sıfır iş — onlar gerçekten "bir MCP'den ibaret" kalıyor):
    1. **Katalog federasyonu:** hub, bankaların `describeCapabilities` çıktılarını toplar; banka seçimini 54 tool'luk listeyle değil katalogla yapar (utility model, ucuz).
    2. **Paralel LLM'siz fan-out:** seçilen bankalara eşzamanlı `queryKnowledgeBase` (bankada LLM döngüsü çalışmaz — mimarideki D akışı); sonuçlar füzyonlanır.
    3. **Birleşik atıf:** `banka:doküman § bölüm`.
    4. **Timeout + kısmi cevap:** geç/ölü banka cevabı bekletmez; "X bankasına ulaşılamadı" notuyla dönülür.
  - Onayladığın "max esneklik" kararı korunuyor: fan-out yalnız varsayılan hız yolu; hub istediğinde `askAgent`/upload/reindex dahil her tool'u çağırabilir. Ayrı ürün de yok: hub = aynı generic imajın config'le açılan davranışı. Kabul çizgin (üç senaryo en geç V3'te uçtan uca; US-3 federated) bu paketi zaten gerektiriyor — soru "yapılsın mı" değil "V2'de mi kalsın"; önerim: evet, V2.

- [x] C12. Belirsizlik çok. Buzdolabına.
  - **Cevap:** Kabul — buzdolabına. Ö11/Ö14 metinlerindeki "derinlik/döngü guard'ları V3+ (A2A)" kaydı yol haritasından çıkarılıp gerekçeli buzdolabı kaydına dönüşecek (decentralized/mesh gündemi açılırsa geri alınır). Tek seviyeli hub→banka topolojisinde timeout + kısmi cevap (C11 içinde) yeterli.

- [ ] C13. böyle bir şeye niye ihtiyaç var anlamadım. diğer bağlı container'lar bir mcp'den ibaret değil mi?
  - **Cevap:** Fark şu: mevcut `/health`, hub'ın kendi MCP **bağlantılarının** durumunu bilir (bağlı / degraded — bu kadarı bugün var). Bankanın **içinin** sağlığını (KB'de kaç doküman var, LLM'i erişilebilir mi, hangi yetenekleri kapalı) bilmez — o bilgi bankanın kendi `/health`/`stats`'ında. C13 = hub'ın, zaten açık olan MCP bağlantıları üzerinden bu özeti de çekip kendi `/health`'inde `federation:` bölümü olarak göstermesi. Onsuz da yaşanır (9 bankanın health'ine tek tek bakarsın); katkısı operasyonel: tek bakışta filo durumu.
  - **Önerim:** ayrı gereksinim maddesi yapmayalım — Ö11 federasyon maddesinin bir alt bendi olsun ("hub `/health`'i spoke durum özetini içerir"). Onaylarsan öyle işlerim.

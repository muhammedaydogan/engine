# Kullanım Senaryoları (beyin fırtınası girdisi — v0.1)

> Amaç: Webhook ve "kütüphane/imaj olarak dışa açma" fikirlerini gerçek kullanım
> senaryolarına çarpıp test etmek. Bu bir gereksinim dokümanı DEĞİLDİR; buradan
> elenen/onaylanan fikirler gerekçeleriyle `gereksinimler.md`'ye işlenecek.

## Nasıl doldurulur

- Her senaryo bir kart. Önce **Hikâye**'yi serbest yaz (5–10 cümle, konkret isimlerle:
  "bir kullanıcı" değil "kapalı ağdaki Java ekibi"). Sonra alt alanları doldur.
- Bilmediğin alanı boş bırak, "?" koy — boşluklar da bilgi.
- Kesin olmayan, "bir gün belki" fikirlerini en alttaki Buzdolabı bölümüne at;
  ana listeyi bugün gerçekten yaşadığın/öngördüğün ihtiyaçlarla sınırlı tut.

---

## US-1: Otonom kod sistemine "tamamlayıcı beyin" plugin'i

**Hikâye:**
Dışarıdan satın alınmış, bizden bağımsız çalışan otonom bir kod yazma sistemi var.
Bizim container bu sisteme **tamamlayıcı beyin plugin'i** oluyor: harici sistem her
ihtiyacında bize soruyor, biz elimizdeki bilgiyi derleyip onun anlayacağı biçimde
döndürüyoruz. İmaj şöyle konfigüre ediliyor:

1. **Agent Core'a bir agent (asistan modu) yüklenir.** Talimatı: "şu MCP'lerden ve
   Knowledge Core'dan gelen bilgiyi, gelen prompt ile birlikte LLM'e gönder; şu şu
   kısımları ise LLM'e sokmadan **direkt getir**." Yani her tool ayrı ayrı
   konfigüre edilebilmeli: bazı bilgiler LLM'den geçmemeli — ör. Spring AI
   servisinden gelen milimetrik hesapları LLM bozabilir, onlar aynen dönmeli
   (pass-through). Bu davranışın bir kısmı YAML/JSON benzeri bir yapıyla
   **scriptize edilebilmeli**: identity bir kez yüklenir; sonrasında hangi agent'ın
   çalışacağı ve nelerin pass-through olacağı bildirimsel olarak seçilir —
   ör. `agent: code_assistant`, `pass-through: [mcp-x, spring-ai-service-3]`.
   Cevap iki biçimde dönebilmeli: ya **direkt bilgi**, ya da harici sistemlerin
   **anlık dinleyebileceği bir stream endpoint'i** verilir.
2. **Knowledge Core'a** hem build-time'da hem runtime'da bilgi gönderilir.
3. Birkaç **MCP** bağlanır.
4. **Spring AI servisi** bağlanır; agent isterse ilgili servisi run'layıp Spring AI
   desteğiyle test eder.
5. Kod yazacak harici sistemin repoları, **DeepWiki'nin open-source muadili** ile
   özetlettirilip Knowledge Core'a gönderilir.
6. Agent'a şu **identity** verilir: "Sen asistansın; kod yazacak harici sisteme
   destek vereceksin. Elindeki bilgilere göre isteği değerlendirip olumlu/olumsuz
   bir cevap vereceksin. Gelen bilgileri, yazılım geliştiren sistemin anlayacağı
   şekilde derleyeceksin."
7. Harici sistem bir işi bitirince container **webhook benzeri bir yapıyla
   tetiklenir**: değişen dosyalar DeepWiki-muadili pakete bildirilir, o da özet
   repoyu (KB'deki wiki'yi) günceller.
8. **1 ay sonra analiz:** Agent Core'a ikinci bir agent eklenir: "geçmişte gelen
   sorulardaki tekrar eden pattern'leri ve dikkat etmem gereken ince noktaları bul."
   Çıkan bulgular knowledge base'e geri gönderilir. (Bu al-ver bilerek "gereksiz"
   yapılıyor — container generic kalabilsin diye; ama bu döngü salt konfigürasyonla
   kurulabilse süper olurdu.)

| Alan | |
|---|---|
| Kim | Dışarıdan alınmış otonom kod yazma sistemi (makine istemci) + onu işleten ekip; bizim container = bilgi/karar destek plugin'i |
| Ortam | Kurum ağı; harici sistem bağımsız çalışıyor, bize MCP/API ile geliyor |
| Tetik | Harici sistem her ihtiyaçta soru sorar (sürekli); her iş bitiminde webhook ile repo-değişti bildirimi gönderir |
| Bugün neden olmuyor | (a) inbound webhook yok; (b) tool başına "LLM'e girmesin, direkt dön" pass-through config'i yok — her şey agent döngüsünden geçiyor; ve bu yönlendirme (hangi agent, neler pass-through) YAML/JSON ile bildirimsel tanımlanamıyor; (c) Agent Core tek persona — config'le birden fazla agent tanımlanamıyor; (d) V1 stateless: geçmiş soruların günlüğü tutulmuyor, 1 aylık pattern analizi için ham veri yok; (e) DeepWiki tarzı repo özetleme entegrasyonu yok (FR-4 repoyu ham index'ler, özet wiki üretmez); (f) agent çıktısını KB'ye geri yazan döngü config'le kurulamıyor; (g) yanıt stream'i yok — V1 `/chat` senkron istek/cevap; harici sistemlerin anlık dinleyeceği bir stream endpoint'i sunulmuyor |
| Başarı işareti | Harici sistem tek MCP/API bağlantısıyla derlenmiş, anlayacağı formatta cevap alıyor; agent seçimi + pass-through listesi YAML/JSON'la bildirilebiliyor; milimetrik hesaplar LLM'e uğramadan bit-bit aynı dönüyor; cevap istendiğinde stream endpoint'inden anlık dinlenebiliyor; her iş bitiminde özet wiki kendiliğinden güncel kalıyor; 1 ay sonunda soru geçmişi üzerinden pattern analizi salt config'le yapılabiliyor |
| İlgili fikir | webhook-in (repo değişikliği → KB güncelleme); imaj (hazır plugin olarak dağıtım); + yeni fikirler: tool pass-through, bildirimsel agent/routing script'i (YAML/JSON), stream endpoint, çoklu agent tanımı, soru günlüğü, KB'ye geri-besleme |
| Sıklık / önem | Sorgu: sürekli/günde çok sayıda (otonom sistem). Webhook: her iş bitiminde. Analiz: ayda bir. Önem: yüksek — senaryo tam otomasyon varsayıyor, manuel adım kaldıramaz |

---

## US-2: Kod yazan sisteme "kendini iyi tarif eden" MCP olarak bağlanma

**Hikâye:**
Yazılım yazan harici bir tool/sistem, bizim container'ı kendisine **MCP olarak
bağlıyor**. Ama asıl iş bağlanmadan önce, entegrasyonu kuran tarafın yaptığı
tariflerde:

1. **İmaj konfigüre edilir:** entegratör kendi veri kaynaklarını bağlar — birkaç
   MCP, Spring AI servisi, Knowledge Core'a build-time + runtime içerik.
2. **Tarifler iki katmanlıdır — taban katman otomatik:** sistem, MCP tariflerini
   bir endpoint üzerinden **build-time veya runtime kendisi üretebilir** (bağlı
   MCP'leri, servisleri ve Knowledge Core içeriğini tarayıp tool açıklamalarını /
   kataloğu çıkarır). Elle tarif zorunlu değildir; hiç yazılmasa da sistem
   bağlanabilir durumdadır.
3. **Opsiyonel verim katmanı — kullanıcı tarifi:** hangi serviste, hangi tool'da,
   Knowledge Core'un hangi alanında ne var; MCP ile bağlanıldığında oradan ne
   çekilir — entegratör bunları kendisi güzelce tarif ederek otomatik tabanın
   üzerine yazar. Kullanma talimatları ve query çeşitleri de bu katmanda tanımlanır.
   Amaç verimi **uçurmak**: istemcinin agent'ı, bizim sistemde neler
   yapılabileceğini bilsin ve yeteneklerimiz arasında en doğru şekilde **route**
   edebilsin. Bağladığı bir sürü veriyi gerçekten efektif yapacak olan kendisidir.
4. **Bağlanma:** kod yazan sistem bizi tek MCP olarak ekler; artık her ihtiyacında
   hangi tool'u çağıracağını bu açıklamalardan bilerek gelir.

Bu senaryoda routing zekâsı **istemci tarafındadır**; bizim imajın işi kendini
(ve bağlı kaynaklarını) iyi tarif etmektir.

| Alan | |
|---|---|
| Kim | Yazılım yazan harici tool/sistem (makine istemci, kendi agent'ı var) + bizim imajı konfigüre eden entegratör |
| Ortam | ? (belirtilmedi; muhtemelen kurum ağı) |
| Tetik | Kurulumda: tarifler otomatik üretilir (endpoint, build-time/runtime); istenirse elle zenginleştirilir. Sonra her istekte: istemci agent bu tariflere bakarak route eder |
| Bugün neden olmuyor | (a) MCP tool yüzeyi sabit — 5 generic tool (`queryKnowledgeBase`, `askAgent`...), isim/açıklama config'le özelleştirilemiyor; kullanım talimatı ve query çeşitleri MCP yüzeyine işlenemiyor; (b) bağlanan kaynaklar (harici MCP'ler, Spring AI, KB alanları) dışarıya ayrı ve iyi tarif edilmiş tool'lar olarak yansıtılamıyor — her şey tek `askAgent`/`queryKnowledgeBase` hunisine düşüyor, istemci agent'ın route edeceği granülerlik yok; (c) "nerede ne var, ne zaman ne çekilir" bilgisini istemcinin keşfedeceği bir yetenek kataloğu yok — `listSources` dosya listesi verir, semantik tarif vermez; (d) tariflerin otomatik üretimi yok — sistemin bağlı kaynakları tarayıp kendi tool tariflerini/kataloğunu build-time ya da runtime bir endpoint'le üretmesi diye bir mekanizma yok |
| Başarı işareti | Elle hiç tarif yazmadan bağlanılabiliyor ve otomatik üretilen tariflerle makul route çalışıyor; entegratör elle tarif ekleyince verim gözle görülür sıçrıyor; istemci agent tool listesine bakıp hangi işi hangi tool'a götüreceğini açıklamalardan anlıyor; yanlış tool'a gitme / gereksiz geniş sorgu nadir; yeni kaynak eklendiğinde route haritası tarif config'i ya da otomatik üretimle güncelleniyor |
| İlgili fikir | imaj (S3 senaryosunun ileri hali); + yeni fikirler: konfigüre edilebilir MCP tool yüzeyi (custom tool projeksiyonu + açıklama/talimat metinleri config'den), yetenek kataloğu/manifest, **otomatik tarif üretimi** (self-describing yüzey; build-time/runtime endpoint) — kullanıcı tarifi bunun üstünde opsiyonel verim katmanı |
| Sıklık / önem | Tarif yazımı kurulumda bir kez (+ her yeni kaynakta); etkisi her çağrıda — tarif kalitesi bütün entegrasyonun verimini belirler. Önem: yüksek |

---

## US-3: 9 veri bankası + 1 hub — tek kapıdan federasyon

**Hikâye:**
1. Müşteri **9 adet imaj** hazırlar. Her biri **apayrı bir konuda** hizmet verir;
   farklı farklı sistemlere bağlanmış birer **veri bankası** konumundadırlar
   (her biri kendi MCP'leri, servisleri ve Knowledge Core içeriğiyle ayrı
   konfigüre edilmiş aynı generic imaj).
2. Sonra müşteri **yeni bir imaj daha** hazırlar (10.) ve diğer 9'unu bu imaja
   bağlayıp güzelce konfigüre eder — her banka zaten MCP server olduğu için hub
   onları MCP client olarak ekler.
3. Artık **bütün sistemler tek yerden kontrol edilir**: hub gelen soruyu doğru
   bankaya/bankalara yönlendirir, cevapları derler. Sonuç: **çok büyük bir veriye
   hızlı ve isabetli** ulaşım, tek bağlantı noktasından.

Not: Hub da aynı generic imajdır — yeni bir ürün değil, 10. konfigürasyon.
Bu, gereksinimlerdeki S5'in ("container'lar arası iletişim", V3'e ertelenmişti)
somut hâlidir.

| Alan | |
|---|---|
| Kim | Çok sistemli bir müşteri kurum; 9 domain veri bankası imajı + 1 hub imajı; son kullanıcılar ve harici sistemler hub'a bağlanır |
| Ortam | Değişken (her deployment farklı olabilir) |
| Tetik | Veri bankaları çoğalıp silolaştıkça tek kapı ihtiyacı doğar; kurulum bir kez, sonrasında her sorguda hub route eder |
| Bugün neden olmuyor | (a) S5 bilinçli olarak kapsam dışı/V3 — mekanik olarak mümkün (hub, 9 bankayı MCP client olarak ekler; kapı bedava) ama bunun için tasarlanmış hiçbir şey yok; (b) hub'ın "isabetli" route etmesi her bankanın yetenek kataloğuna muhtaç — US-2'deki tarif/katalog mekanizması olmadan hub 9 kapıdan hangisini çalacağını bilemez; (c) "hızlı" olması ucuz sorgu moduna muhtaç — naif zincirde her bankada ayrı LLM döngüsü döner (maliyet ve gecikme katlanır); hub'ın bankalara LLM'siz, pass-through/query-only inebilmesi gerekir (US-1 akrabası); (d) derinlik/döngü guard'ı yok (A→B→A çevrimi, max hop, timeout, kısmi cevap politikası); (e) bankalar arası birleşik kaynak atfı yok — cevap "hangi bankanın hangi dokümanından" diyebilmeli; (f) 9 bağlantının sağlığı tek yerden izlenemiyor — hub `/health`'inin spoke durumlarını toplaması gerekir |
| Başarı işareti | Tek bağlantı noktası; sorular doğru bankaya düşüyor (isabet ölçülebilir); gecikme kabul edilebilir; cevapta banka+doküman atfı var; 10. banka eklemek = hub config'ine bir kayıt + katalog tazeleme; hub `/health` 9 bankanın durumunu gösteriyor |
| İlgili fikir | imaj (genericliğin nihai testi: hub = aynı imajın 10. konfigürasyonu); + yeni fikirler: federasyon/hub modu, katalog federasyonu (US-2'nin üstüne), bankalara ucuz/LLM'siz sorgu modu (US-1 pass-through akrabası), derinlik-döngü guard'ları, toplu health. Açık soru: "kontrol" salt sorgu mu, yönetim de mi (hub üzerinden bankaya upload/reindex/talimat)? → ? |
| Sıklık / önem | Kurulum nadir; sorgu sürekli. Önem: stratejik — ürünün çok-imajlı ölçek hikâyesini tanımlıyor |

---

## US-4: <kısa isim>

**Hikâye:**


| Alan | |
|---|---|
| Kim | |
| Ortam | |
| Tetik | |
| Bugün neden olmuyor | |
| Başarı işareti | |
| İlgili fikir | |
| Sıklık / önem | |

---

## Buzdolabı ("bir gün belki")

- <tek satırlık fikir — detaylandırma>

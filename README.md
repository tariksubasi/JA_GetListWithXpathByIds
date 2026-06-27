# Mendix Logging Kit — Trace + Object/Workflow Logging (Graylog / DB)

Workflow → microflow → sub-microflow zincirinde **nokta-atışı bug bulma** için kurulan logging sistemi.
Bir `correlationId` (ör. talep no) ile bir akışın tüm log satırlarını birbirine bağlar; objelerin
null/empty/association durumunu ve onay olaylarını tek satırda yazar. Loglar config'e göre **Graylog'a**
ve/veya **DB'deki bir entity'ye** gider. Logging kodu hata verse bile çağıran akışı asla bozmaz.

---

## 0) Gereksinim

- Projede **CommunityCommons** modülü olmalı (log seviyesi için `CommunityCommons.LogLevel` enum'u ve
  `commitInSeparateDatabaseTransaction` buradan gelir). Marketplace'ten ekli değilse ekle.

---

## 1) Dosya → Mendix'teki yeri

| GitHub dosyası | Mendix'teki yeri | Türü |
|---|---|---|
| `TraceContext` | `javasource/logging/TraceContext.java` | Düz Java sınıfı (action değil) |
| `LogSink` | `javasource/logging/LogSink.java` | Düz Java sınıfı (action değil) |
| `StartTrace` | `logging` modülü → Java action | Java action |
| `EndTrace` | `logging` modülü → Java action | Java action |
| `LogObjectState` | `logging` modülü → Java action | Java action |
| `LogWorkflowContext` | `logging` modülü → Java action | Java action |

> Her GitHub dosyasının içeriği **tam Java dosyasıdır**. Action'larda Mendix yalnızca import listesi,
> `BEGIN/END USER CODE` ve `BEGIN/END EXTRA CODE` bloklarını korur — yapıştırırken bu blokları birebir kopyala.

---

## 2) Studio Pro kurulumu (sırayla)

### 2.1 `logging` modülü
Yoksa oluştur (App Explorer → Add module → `logging`). İsim **birebir** `logging` olmalı (paket adları buna bağlı).

### 2.2 `Logger` entity (DB sink için)
`logging` modülünün domain model'ine **persistable** bir `Logger` entity ekle. Attribute'lar:

| Attribute | Tip |
|---|---|
| `LogNodeName` | String |
| `Message` | String (sınırsız / unlimited uzunluk) |
| `LogLevel` | Enumeration → `CommunityCommons.LogLevel` |
| `CorrelationId` | String |
| `Timestamp` | DateTime |

Access rules: sadece admin okuyabilsin (loglar hassas veri taşıyabilir).

### 2.3 İki Constant
`logging` modülüne iki **Boolean Constant** ekle:

| Constant | Tip | Default |
|---|---|---|
| `LogToGraylog` | Boolean | `true` |
| `LogToDatabase` | Boolean | `false` |

Graylog'un olmadığı ortamda (deployment ayarlarından) `LogToDatabase = true` yap (istersen `LogToGraylog = false`).

### 2.4 Java action'ları oluştur (parametreleriyle)
`logging` modülünde 4 Java action oluştur. **İsim, parametre adı/tipi ve sıra birebir aynı olmalı.**
Hepsinin dönüş tipi **Nothing**.

**StartTrace**
| Sıra | Parametre | Tip |
|---|---|---|
| 1 | `CorrelationId` | String |

**EndTrace** — parametre yok.

**LogObjectState**
| Sıra | Parametre | Tip |
|---|---|---|
| 1 | `InputObject` | Object |
| 2 | `LogNodeName` | String |
| 3 | `Level` | Enumeration → `CommunityCommons.LogLevel` |
| 4 | `MaxLength` | Integer/Long |

**LogWorkflowContext**
| Sıra | Parametre | Tip |
|---|---|---|
| 1 | `LogNodeName` | String |
| 2 | `Action` | String |
| 3 | `ContextObject` | Object |
| 4 | `TargetUser` | Object → `System.User` |
| 5 | `Level` | Enumeration → `CommunityCommons.LogLevel` |
| 6 | `MaxLength` | Integer/Long |

### 2.5 Kodları yapıştır
1. **Düz sınıflar:** `javasource/logging/` klasöründe `TraceContext.java` ve `LogSink.java` dosyalarını oluştur,
   GitHub'daki `TraceContext` ve `LogSink` içeriğini **tam** yapıştır.
2. **Action'lar:** Studio Pro'nun oluşturduğu 4 action iskeletini aç; GitHub'daki ilgili dosyadan
   **import listesini**, **USER CODE** ve **EXTRA CODE** bloklarını yapıştır (constructor/alanları Studio Pro üretir, dokunma).

### 2.6 Deploy
`Run` / `Deploy for Eclipse` → proxy'ler ve constant'lar üretilir, derlenir. Hata yoksa kurulum tamam.

---

## 3) Kullanım

**Ana (parent) microflow:**
1. İlk activity: **StartTrace** → `CorrelationId` = iş anahtarın (ör. `$Request/RequestNumber`; String değilse `toString(...)`).
2. Şüpheli noktalarda **LogObjectState** → `InputObject` = ilgili obje, `Level` = `Debug`/`Info`, `MaxLength` = boş.
3. Onay/karar noktasında **LogWorkflowContext** → `Action` = "Request approved" vb., `ContextObject`, `TargetUser`.
4. Son activity: **EndTrace**.
5. **Error handler'a da EndTrace koy** (pooled thread'de bayat cid kalmasın).

**Sub-microflow'larda:** StartTrace çağırma — aynı thread'de cid otomatik miras alınır. Sadece gerekirse `LogObjectState` koy.

**Workflow / async / background:** ThreadLocal pod/thread sınırını geçmez. Workflow'un çağırdığı her microflow
girişinde `StartTrace($key)` ile cid'i **aynı iş anahtarıyla yeniden tohumla** → cid tutarlı kalır.

---

## 4) Çıktı ve bug bulma

Loglar tek satır, `cid` ile etiketli:
```
ObjectState cid=REQ-123 Request[id=..] {Status=<empty>, Amount=1000, Customer=ref(55), Lines=<empty list>}
WorkflowEvent cid=REQ-123 { actor=ahmet, action=Request approved, targetUser=ayse(42), context=Request[id=..] {...} }
```
- **Graylog:** `cid:REQ-123` ara → tüm zincir sırayla gelir.
- **DB:** `Logger` tablosunda `CorrelationId = 'REQ-123'` filtrele.
- Satırları bir LLM'e verip "kök neden nerede?" diye sorabilirsin (null/empty/association net görünür).

---

## 5) Davranış / güvenlik notları

- **Performans:** `Trace`/`Debug` log node kapalıyken hiç string kurulmaz (≈0 maliyet). Rutin teşhis için `Debug`,
  iş olayları için `Info`, anormallik için `Warning/Error` kullan. Yüksek frekanslı yerde `Info` basma.
- **Seviye guard her iki sink için geçerli:** DB'ye `Debug` yazmak istersen ilgili log node'unu `Debug`'a al.
- **Truncate:** Alan başına `MaxLength` (boş = 500), sert tavan 2000; tüm satır tavanı 8000 karakter — geniş entity'de bile patlamaz.
- **PII:** `TargetUser` için tüm obje değil sadece `Name(id)` loglanır.
- **DB dayanıklılığı:** DB sink ayrı system context ile yazar → iş transaction'ı rollback olsa bile log kalır.
- **Fail-safe:** Logging herhangi bir hata verse bile çağıran akış bozulmaz; hata sessizce yutulup konsola yazılır.
- **Retention:** DB tablosu büyür → "N günden eski `Logger` kayıtlarını sil" scheduled event kur.

---

## 6) Dosya özeti

| Dosya | Görev |
|---|---|
| `TraceContext` | Thread'e bağlı `correlationId` (start/get/clear) |
| `LogSink` | Config'e göre Graylog ve/veya DB'ye yazar (fail-safe, ayrı tx) |
| `StartTrace` / `EndTrace` | cid'i koyar / temizler |
| `LogObjectState` | Bir objenin tüm alan/association durumunu tek satırda yazar |
| `LogWorkflowContext` | Onay olayını (actor/action/assignee/context) tek satırda yazar |

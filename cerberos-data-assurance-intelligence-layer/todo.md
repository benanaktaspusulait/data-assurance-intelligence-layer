# TODO / Değişiklik Kaydı - Cerberos Data Assurance Intelligence Layer

Durum: **Tüm review turları TAMAMLANDI** ✅

Bu dosya, doküman paketi üzerinde yapılan incelemeleri ve düzeltmeleri özetler.
Kaynak doğruluk her zaman `confluence/` altındaki sayfalardır; `*-combined.md` dosyası
`build-combined.sh` ile üretilir.

## 1. Tur - Hata / Tutarsızlık Düzeltmeleri
- Kanonik **flat** Rule YAML formatı seçildi; `06` örneği buna çevrildi.
- `13` JSON Schema tüm gerekli alanları kapsayacak şekilde düzeltildi; örnekler jsonschema ile doğrulandı.
- Veri modeli tek kanonik sete hizalandı (`dq_audit_event` tanımı eklendi).
- Backtest alan adı her yerde `missed_confirmed_issues` oldu.

## 2. Tur - Eksikler / Çapraz Referans
- `09` "Later Enhancements" `13` ile senkronlandı.
- README'ye combined dosya + `build-combined.sh` açıklaması eklendi.
- `03`'ün safe-operating-model atfı parent sayfaya yönlendirildi.
- `14` analitik kuralların `05` ile eşlemesi açıklandı.
- Header metadata, H1 yapısı, örnek kaynak adı, maliyet alanı, README sayfa sayısı standartlaştırıldı.
- Tüm dış URL'ler HTTP 200 (kırık link yok).

## 3. Tur - Paylaşım Stratejisi / Polish
- Executive sayfası **Executive Discovery Proposal** olarak adlandırıldı, 700-900 kelimeye indirildi.
- Parent page'e **Suggested Reading Paths** (senior / architecture / technical katmanları) eklendi.
- "Current state unknown; discovery first" mesajı parent + executive'e eklendi.
- RACI'ye "discovery sırasında validate edilecek" notu; rule YAML'ine `classification` bloğu;
  `missing_rate_percent` tutarsızlığı düzeltildi; build-vs-buy ifadesi dengelendi.

## 4. Tur - Fikrin Kalbini Parlatma
- `03`'e **AI-Assisted Suggestion Lifecycle** alt bölümü eklendi.
- `06`'ya somut **Agent Suggestion Review Example** (`dq-suggestion-2026-06-001`) eklendi.

## 5. Tur - Uzun Sayfaları Bölme (Confluence okunabilirliği)
Çok uzun sayfalar mantıklı alt sayfalara bölündü; en uzun sayfa 6739 → ~2964 kelimeye indi.

- **`10` Technology Selection** (6739 kelime) üçe ayrıldı:
  - `10` Technology Selection (özet/karar) → ~2275 kelime
  - `10a` Component Technology Choices (bileşen-bazlı teknoloji seçimleri) → ~2964 kelime
  - `10b` Architecture Decision Records and Reference Architecture (matrix + ADR-001..014 + ref mimari) → ~1755 kelime
- **`14` S3/Athena** (3139) ikiye ayrıldı:
  - `14` concept/governance/cost/scope → ~2239 kelime
  - `14a` Athena Analytical DQ Examples and Sub-Architectures → ~1030 kelime
- **`13`** Glossary ayrı `13a` Glossary sayfasına alındı.

Bölme sırasında: işaretçiler + çapraz referanslar eklendi, iç numaralar yeniden sıralandı,
parent page / README / `build-combined.sh` güncellendi, dosya adları mantıksal sırayla hizalandı
(`10a` = Component, `10b` = ADR), combined yeniden üretildi (19 sayfa), YAML örnekleri doğrulandı.

## Mevcut Yapı
- 1 parent page + 18 child page = 19 section.
- Bölünmeyenler: `06` (kod-ağırlıklı, 1603 kelime) ve diğer tüm sayfalar (≤~2275 kelime) Confluence için uygun.

## Bakım
- Herhangi bir `confluence/*.md` düzenlemesinden sonra: `./build-combined.sh` çalıştır.

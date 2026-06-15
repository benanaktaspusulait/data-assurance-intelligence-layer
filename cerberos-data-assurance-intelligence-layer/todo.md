# TODO / Değişiklik Kaydı

Kaynak doğruluk `confluence/`'tır; combined dosya `build-combined.sh` ile üretilir.
(Bu dosya ortam tarafından turlar arası silinebiliyor; her tur güncel haliyle yeniden yazılıyor.)

## 10. Tur - Uzun Sayfaları Bölme (devam) - TAMAMLANDI ✅
İki kalan uzun sayfa bölündü; en uzun sayfa artık 2280 kelime (önce 3216).

- `10a` Component Technology Choices (3216 kelime) ikiye:
  - `10a` (bölüm 1-6: backend, registry, query exec, sources, store, UI) → ~1856 kelime
  - yeni `10b` Component Technology Choices - Platform Services (learning, agent, AWS, scheduling, observability, security, deployment) 
  - mevcut ADR sayfası `10b` → `10c` olarak yeniden adlandırıldı
- `13` Supporting Technical Detail (2806 kelime) ikiye:
  - `13` (evidence model, data model, ER, YAML schema, sequences, confidence, adaptive threshold, backtesting) → ~1912 kelime
  - yeni `13b` Supporting Technical Detail - Operational Artefacts (notification templates, agent prompt, integration tests, tooling comparison, complexity/value/quota)
  - Glossary `13a` → `13c` olarak yeniden adlandırıldı; yanlış yerdeki Glossary stub'ı temizlendi

Güncellenenler: işaretçiler + iç numaralandırma, parent page (22 child / 23 section),
README, `build-combined.sh` (PAGES + TOC + sayım). Stale dosya adı referansı yok.
Combined yeniden üretildi (23 sayfa). YAML örnekleri JSON Schema'yı geçiyor (PASS/PASS).

## Mevcut Yapı
- 1 parent + 22 child = 23 section. Tüm sayfalar ≤ ~2280 kelime (dengeli).

## Bakım
- `confluence/*.md` düzenlemesi sonrası: `./build-combined.sh`.

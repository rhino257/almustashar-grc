# سجلّ قرارات المعمارية (ADR)

قرارات المعمارية المهمّة، **قرار واحد لكل ملف**، بصيغة مستوحاة من MADR.

## القواعد
- **Immutable**: بعد اعتماد ADR لا تُعدَّل. إذا تغيّر القرار، تُكتب ADR جديدة
  وتُحدَّث حالة القديمة إلى `Superseded by ADR-XXXX`.
- ترقيم متسلسل: `ADR-0001`, `ADR-0002`, …
- **فضاء ترقيم مستقل لكل مستودع**: أرقام ADR في هذا المستودع (`almustashar-grc`) خاصة به ولا تتقاطع مع ترقيم مستودعات المشروع الأخرى.
- القالب: `templates/adr-template.md`.
- الحالات الممكنة: `Proposed` / `Accepted` / `Superseded` / `Deprecated`.

## الفهرس
| # | العنوان | الحالة |
| --- | --- | --- |
| [0001](ADR-0001-contract-first-development.md) | التطوير بمنهج العقد أولاً | Accepted |
| [0002](ADR-0002-authentication-model.md) | نموذج المصادقة | Accepted |
| [0003](ADR-0003-2fa-sms-drop-totp.md) | التحقق الثنائي عبر SMS وإسقاط TOTP | Accepted |
| [0004](ADR-0004-biometric-pin-local-only.md) | البصمة/PIN محلي على الجهاز فقط | Accepted |
| [0005](ADR-0005-consent-and-policy-versioning.md) | الموافقات وإصدارات السياسات | Accepted |
| [0006](ADR-0006-personalization-advisor-role-settings-jsonb.md) | التخصيص: advisor_role + settings JSONB | Accepted |
| [0007](ADR-0007-blind-index-and-password-hashing.md) | الفهرس الأعمى وتجزئة كلمات المرور | Accepted |
| [0008](ADR-0008-field-encryption-deferred.md) | تأجيل تشفير الحقول عبر CryptoService | Accepted |
| [0009](ADR-0009-walking-skeleton-mock-repositories.md) | الهيكل السائر عبر mock repositories | Accepted |
| [0010](ADR-0010-evolutionary-platform-architecture.md) | المعمارية التطوّرية للمنصّة | Accepted |
| [0011](ADR-0011-three-repos-auth-slice-first.md) | ثلاثة مستودعات منفصلة والبدء بشريحة المصادقة | Accepted |
| [0012](ADR-0012-observability-foundation-dependencies.md) | أساس الرصد والمراقبة: التبعيات المعتمدة (structlog وOpenTelemetry) | Accepted |
| [0013](ADR-0013-cicd-quality-gate-pipeline.md) | خط تحقّق الجودة في CI/CD وضوابط الأمن | Accepted |

# HANDOFF — ابدأ من هنا

> **الغرض:** هذه الصفحة هي أول ما يفتحه أي شخص جديد (فريق استلام، مطوّر جديد،
> أو مساعد ذكي في محادثة جديدة). تجيب على: أين نحن؟ ما المُنجَز؟ ما المتبقّي؟
> كيف أُشغّل المشروع؟ أين أجد التفاصيل؟
>
> **حالة الوثيقة:** حيّة (living) — تُحدَّث مع كل مرحلة. آخر تحديث: 2026-07-04.

---

## 1) ما هو المشروع في سطر واحد
«المستشار القانوني الذكي» — مساعد قانوني يمني يعمل محلياً (On-Prem) عبر RAG،
يجيب بالعربية مسنَداً بالنصوص القانونية اليمنية والشريعة، مع حياد سياسي كطبقة
فحص مستقلة.

## 2) أين نحن الآن (Status)
| المسار | الحالة |
| --- | --- |
| تصميم الواجهة (UI/UX) | ✅ مكتمل |
| مواصفات مُحكومة (specs) | ✅ متسقة عند data-model v0.4 / api-contract v0.3.1 |
| عقد الـ API (OpenAPI) | ✅ v0.3.0 |
| تطبيق Flutter | ✅ مبنيّ على mocks ومُواءَم مع العقد (`flutter analyze` نظيف) |
| الخادم (backend) | ⛔️ لم يُبنَ بعد (المرحلة القادمة) |
| ربط SMS (الأوائل) / SMTP | ⛔️ لم يُنفَّذ |
| هجرات قاعدة البيانات (Alembic) | ⛔️ لم تُكتب |
| CI/CD | ⛔️ لم يُعدّ |

## 3) المُنجَز باختصار (روابط للتفاصيل)
- **البحث والسياق اليمني** → `docs/journal/2026-07-04-inception-to-app-alignment.md`
- **نموذج البيانات** → `docs/data-model.md` (v0.4)
- **عقد الـ API** → `docs/api-contract.md` (v0.3.1، ARC-API-001)
- **خط أنابيب RAG** → `docs/rag-pipeline-spec.md` (v0.2)
- **الـ prompts** → `docs/system-prompt.md` (v0.2)
- **معيار التشفير** → `docs/cryptography-standard.md` (v0.2)
- **قرارات المعمارية** → `docs/adr/` (ADR-0001 … ADR-0009)

## 4) القرارات الكبرى (اقرأ الـ ADRs)
| ADR | القرار |
| --- | --- |
| 0001 | التطوير بمنهج العقد أولاً (contract-first) |
| 0002 | نموذج المصادقة: هاتف + كلمة مرور + OTP إجباري عند أول تسجيل |
| 0003 | التحقق الثنائي عبر SMS (الأوائل)، وإسقاط TOTP |
| 0004 | البصمة/رمز PIN محلي على الجهاز فقط (لا عمود في قاعدة البيانات) |
| 0005 | الموافقات وإصدارات السياسات (consent + policy versioning) |
| 0006 | التخصيص: advisor_role نص طبيعي + الحقول الخام في settings JSONB |
| 0007 | الفهرس الأعمى HMAC-SHA-256 + كلمات المرور Argon2id |
| 0008 | تأجيل تشفير الحقول عبر CryptoService (NullCipher→AesGcmCipher) |
| 0009 | الهيكل السائر (walking skeleton) عبر mock repositories |

## 5) كيف أُشغّل المشروع
راجع `README.md` (الجذر) و`docs/README.md` لخطوات قاعدة البيانات والنموذج
والخادم والواجهة ومتغيّرات البيئة.

## 6) المرحلة القادمة (Next)
1. بناء backend رفيع (thin FastAPI + LangGraph) كـ walking skeleton.
2. استبدال mock repositories بالتنفيذ الحقيقي (تبديل provider بسطرين — ADR-0009).
3. ربط SMS عبر الأوائل + SMTP للبريد.
4. كتابة هجرات Alembic لمخطط data-model v0.4.
5. إعداد CI/CD.

## 7) قواعد التوثيق (كيف نحافظ على هذا حيّاً)
- **Journal**: `docs/journal/` — مدخلات مؤرّخة، **append-only** (لا تُعدَّل السابقة).
- **ADR**: `docs/adr/` — قرار واحد لكل ملف، **immutable**؛ عند التغيير تُكتب ADR
  جديدة بحالة `Superseded by ADR-XXXX`.
- **CHANGELOG**: لكل إصدار (Keep a Changelog + SemVer).
- **Commits**: بصيغة Conventional Commits (`feat:`/`fix:`/`docs:`…).
- القوالب في `templates/` (`adr-template.md`، `journal-entry-template.md`).

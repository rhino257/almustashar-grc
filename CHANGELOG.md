# سجل التغييرات (Changelog)

جميع التغييرات الملحوظة على هذا المستودع تُوثَّق في هذا الملف.

يلتزم هذا الملف بمعيار [Keep a Changelog](https://keepachangelog.com/ar/1.1.0/)،
ويتبع المشروع [الإصدار الدلالي (Semantic Versioning)](https://semver.org/lang/ar/).

## [غير مُصدَر بعد / Unreleased]

### مُضاف (Added)

- **حوكمة الذكاء الاصطناعي:** بطاقة نموذج Gemma 4 31B
  (`08-ai-governance/gemma-model-card.md`, `AIG-MODELCARD-001`) — الإصدار 0.1،
  تصنيف سرّي للغاية، تتضمّن بناء التكميم 4-بت (QAT) وخطة التقييم §7.1.
- **حوكمة الذكاء الاصطناعي:** مواصفات ضوابط الأمان والحياد السياسي
  (`08-ai-governance/guardrails-spec.md`, `AIG-GUARDRAILS-001`) — الإصدار 0.1،
  مع الربط الكامل بـ OWASP Top 10 for LLM (2025).
- **حوكمة الذكاء الاصطناعي:** مجموعة التقييم الذهبية
  (`08-ai-governance/evaluation-golden-set.md`, `AIG-EVALSET-001`) — الإصدار 0.1،
  أربع طبقات تقييم (الجودة، الاستناد، الحياد، OCR) وعتبات الإطلاق.
- **السياسات:** سياسة أمن المعلومات الأمّ
  (`01-policies/information-security-policy.md`, `POL-POLICY-001`) — الإصدار 0.1،
  مبنية على ISO/IEC 27001:2022.
- **الأسلوب والقوالب:** دليل الأسلوب الموحّد
  (`STYLE_GUIDE.md`, `GOV-STYLEGUIDE-001`) — الإصدار 1.1، والقالب الرسمي
  (`templates/document-template.md`) الذي يعتمد الترويسة الموحّدة (frontmatter + rh-id/rh-class + بطاقة docmeta).
- **قرارات المعمارية:** `ADR-0012` أساس الرصد والمراقبة (structlog ومنظومة OpenTelemetry)
  و`ADR-0013` خط تحقّق الجودة في CI/CD وضوابط الأمن — كلاهما `Accepted` بتاريخ 2026-07-26،
  معرّبان وموحّدان على صيغة ADR-0010/0011 (رُقّما لتفادي تعارض الترقيم).

### مُعدَّل (Changed)

- إعادة تنسيق مخطّط بطاقة المصدر (§7) في `guardrails-spec.md` ليُعرض بشكل سليم
  (قيم إنجليزية داخل كتلة الكود + جدول وصف عربي) لتفادي تشوّه اتجاه النص.
- **مواءمة `README.md` الجذري مع الواقع:** جدول البنية يميّز المجلدات الموجودة (✅) عن المخطّطة (⏳)،
  إضافة صفّ `docs/`، إعادة بناء «حالة الإنجاز»، وفصل سير العمل الحالي (كتابة مباشرة على `main`)
  عن المستهدف (PR/CODEOWNERS مخطّط)، وتصحيح إشارة القالب إلى `templates/document-template.md` و`STYLE_GUIDE.md`.
- **توحيد سجل قرارات المعمارية:** نقل `ADR-0011` (ثلاثة مستودعات منفصلة) إلى `docs/adr/`،
  تعريبه وتوحيد صيغته، وتحديث فهرس `docs/adr/README.md` (إضافة صفوف 0010–0013
  وملاحظة فضاء الترقيم المستقل لكل مستودع).
- **نقل معيار التشفير** إلى `02-standards/cryptography-standard.md` (`STD-STD-001` v0.3)
  وتحديث الإحالات في `docs/HANDOFF.md`.
- **تحديث `docs/HANDOFF.md`** ليطابق الواقع والمرجع الأساسي (SSOT).

### مُزال (Removed)

- `templates/_document-header.md` — قالب الترويسة المهجور، استُبدِل بـ `templates/document-template.md`.
- `docs/cryptography-standard.md` — نُقل محتواه إلى `02-standards/cryptography-standard.md`.
- `docs/decisions/ADR-0011-three-repos-auth-slice-first.md` — نُقل إلى مجلد `docs/adr/` الموحّد.

### مُصلَح (Fixed)

- تصحيح روابط المقارنة والإصدار في نهاية هذا الملف: `<org>` → `rhino257`.

---

## [0.1.0] - 2026-06-21

### مُضاف (Added)

- **الحوكمة:** ميثاق الامتثال (`00-governance/compliance-charter.md`) — الأقسام 1–5،
  16 مبدأً، ونموذج إشراف بشري متدرّج.
- **الحوكمة:** الأدوار ومصفوفة RACI (`00-governance/roles-and-raci.md`, `GOV-RACI-001`).
- **القوالب:** قالب ترويسة الوثيقة (`templates/_document-header.md`).
- **الجذر:** ملف التعريف (`README.md`) وسجل التغييرات (`CHANGELOG.md`).

### ملاحظات (Notes)

- هذا هو الإصدار التأسيسي (نواة المرحلة الأولى) لمستودع حوكمة المشروع.
- جميع الوثائق في حالة مسودّة (Draft) ما لم يُنصّ على اعتمادها.

[غير مُصدَر بعد / Unreleased]: https://github.com/rhino257/almustashar-grc/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/rhino257/almustashar-grc/releases/tag/v0.1.0

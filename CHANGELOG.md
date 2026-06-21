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

### مُعدَّل (Changed)

- إعادة تنسيق مخطّط بطاقة المصدر (§7) في `guardrails-spec.md` ليُعرض بشكل سليم
  (قيم إنجليزية داخل كتلة الكود + جدول وصف عربي) لتفادي تشوّه اتجاه النص.

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

[غير مُصدَر بعد / Unreleased]: https://github.com/<org>/almustashar-grc/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/<org>/almustashar-grc/releases/tag/v0.1.0

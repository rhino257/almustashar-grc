# almustashar-grc — وثائق الحوكمة والمخاطر والامتثال

> مستودع وثائق **الحوكمة والمخاطر والامتثال (GRC)** لمشروع **المستشار القانوني الذكي** —
> نظام ذكاءٍ اصطناعيٍّ قانونيٍّ بالشراكة مع بنك اليمن والكويت.

> ⚠️ **سرّي وملكية خاصة (Private & Proprietary):** هذا المحتوى ملكٌ لأطراف الشراكة وفق
> عقد 11 سبتمبر 2025، ولا يجوز نسخه أو مشاركته خارج نطاق المشروع دون موافقةٍ كتابية.

---

## 1. الغرض من المستودع

توثيقٌ منظّمٌ وقابلٌ للتدقيق لإطار الامتثال الذي تبنّاه المشروع طوعاً (`ISO/IEC 27001`،
`ISO/IEC 42001`، `PDPL`، `NCA`...)، بهدفٍ مزدوج: **تأهيل المشروع للإطلاق الآمن**،
و**بناء القدرات** عبر تطبيق أرقى الممارسات.

> المرجع الأساسي (SSOT) لحقائق المشروع هو مساحة Notion؛ عند أي تعارض تُقدّم Notion.

## 2. بنية المستودع

الحالة: ✅ موجود فعلاً · ⏳ مخطّط (لمّا يُنشَأ بعد).

| المجلد | الرمز | المحتوى | الحالة |
|---|---|---|---|
| `00-governance/` | GOV | الميثاق، ميثاق المبادئ، الأدوار وRACI | ✅ موجود |
| `01-policies/` | POL | السياسات (سياسة أمن المعلومات) | ✅ موجود |
| `02-standards/` | STD | المعايير التقنية (معيار التشفير) | ✅ موجود |
| `03-procedures/` | PRO | الإجراءات التشغيلية | ⏳ مخطّط |
| `04-registers/` | REG | السجلّات (الأصول، المخاطر...) `.csv` | ⏳ مخطّط |
| `05-assessments/` | ASM | التقييمات، SoA، مصفوفة المحاذاة | ⏳ مخطّط |
| `06-privacy/` | PRV | وثائق الخصوصية (PDPL) | ⏳ مخطّط |
| `07-plans/` | PLN | خطط الاستجابة والاستمرارية | ⏳ مخطّط |
| `08-ai-governance/` | AIG | حوكمة الذكاء الاصطناعي، guardrails، بطاقة النموذج | ✅ موجود |
| `09-evidence/` | EVD | الأدلّة والمحاضر | ⏳ مخطّط |
| `docs/` | — | الوثائق التقنية، سجل قرارات المعمارية (`adr/`)، الدفتر (`journal/`) | ✅ موجود |
| `templates/` | — | القوالب (`document-template.md`، `adr-template.md`) | ✅ موجود |
| `.github/` | — | سير العمل الآلي (`build-pdf.yml`) | ✅ موجود |

> ملاحظة: المجلدات المخطّطة ستُنشَأ عند إيداع أول وثيقة فيها (Git لا يتتبّع المجلدات الفارغة).

## 3. أعراف التوثيق

- **المُعرّف (doc_id):** `<DOMAIN>-<TYPE>-<NNN>` — مثل `GOV-CHARTER-001`.
- **الإصدارات:** `MAJOR.MINOR` — المسودات `0.x`، وأول معتمَدٍ `1.0`.
- **مستويات الالتزام (آلية الصدق):** `مُطبّق (بدليل)` · `محاذاة (قيد التنفيذ)` · `مرجعي (فجوة)` · `خارج النطاق`.
- **القالب الرسمي والترويسة الموحّدة:** `templates/document-template.md`، وقواعد الأسلوب في `STYLE_GUIDE.md`.

## 4. سير العمل (Workflow)

### الحالي (مرحلة التأسيس)
تُكتب الوثائق حالياً مباشرةً على الفرع `main` مع رسائل commit واضحة وتدوين كل دفعة في `CHANGELOG.md`.

### المستهدف (مخطّط)
1. فرع لكل وثيقة: `docs/<اسم-الوثيقة>-vX.Y`.
2. إضافة/تعديل الوثيقة عبر `Pull Request`.
3. المراجعة والاعتماد من `CODEOWNERS` (⏳ لمّا يُنشَأ بعد).
4. عند الدمج: `Git tag` + `Release` + تدوينٌ في `CHANGELOG.md`.

> ملاحظة صدق: ملفّا `CODEOWNERS` و`CONTRIBUTING.md` و`LICENSE` لمّا تُنشَأ بعد؛ فالمراجعة الرسمية عبر CODEOWNERS مستهدفة وليست مُفعّلة بعد.

## 5. حالة الإنجاز

**مُنجَز (موجود فعلاً):**
- [x] `00-governance/compliance-charter.md` — الميثاق (GOV-CHARTER-001 · v0.1)
- [x] `00-governance/principles-charter.md` — ميثاق المبادئ (GOV-PRINCIPLES-001 · v0.3)
- [x] `00-governance/roles-and-raci.md` — الأدوار وRACI (GOV-RACI-001 · v0.1)
- [x] `01-policies/information-security-policy.md` — سياسة أمن المعلومات (POL-POLICY-001 · v0.1)
- [x] `02-standards/cryptography-standard.md` — معيار التشفير (STD-STD-001 · v0.3)
- [x] `08-ai-governance/gemma-model-card.md` — بطاقة النموذج (AIG-MODELCARD-001 · v0.1)
- [x] `08-ai-governance/guardrails-spec.md` — مواصفة الضوابط (AIG-GUARDRAILS-001 · v0.1)
- [x] `08-ai-governance/evaluation-golden-set.md` — مجموعة التقييم الذهبية (AIG-EVALSET-001 · v0.1)
- [x] `docs/adr/` — سجل قرارات المعمارية (ADR-0001 … ADR-0013)
- [x] `templates/document-template.md` · `STYLE_GUIDE.md` — القالب الرسمي ودليل الأسلوب

**مخطّط (لمّا يُنجَز):**
- [ ] `00-governance/isms-scope.md` — نطاق ISMS
- [ ] `05-assessments/risk-methodology.md` · `statement-of-applicability.md` · `framework-crosswalk.csv`
- [ ] `04-registers/asset-inventory.csv` · `risk-register.csv`
- [ ] `03-procedures/` · `06-privacy/` · `07-plans/` · `09-evidence/`
- [ ] `CODEOWNERS` · `CONTRIBUTING.md` · `LICENSE`

## 6. المعايير المرجعية

`ISO/IEC 27001:2022` · `ISO/IEC 27002:2022` · `ISO/IEC 42001:2023` ·
`PDPL` (السعودي) · `NCA ECC` · `NCS` · `NIST CSF 2.0` · `OWASP Top 10 for LLM`.

## 7. القانون الحاكم

القانون اليمني والشريعة الإسلامية.

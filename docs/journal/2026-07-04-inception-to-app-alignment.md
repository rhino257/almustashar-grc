# 2026-07-04 — من الفكرة إلى مواءمة التطبيق (سجلّ رجعي شامل)

> **نوع المدخلة:** سجلّ رجعي (retrospective) يغطّي كل ما أُنجز حتى تاريخه.
> **المشاركون:** Marya Davis (مالكة المشروع)، mohammed (مساعد Notion AI للتخطيط
> والتوثيق)، Claude (منفّذ الكود داخل IDE محلياً).
> **ملاحظة على أدوار الأدوات:** التخطيط/التوثيق/مراجعة القرارات تمّت عبر
> المساعد؛ تعديلات كود Flutter نفّذها Claude محلياً (لا يملك المساعد وصولاً
> مباشراً للمشروع المحلي، فالتواصل عبر prompts).

---

## المرحلة 0 — البحث والسياق اليمني
قبل أي قرار تقني، بحثنا في واقع اليمن لأنه يفرض قيوداً على المصادقة والخصوصية:
- **لا يوجد قانون حماية بيانات نافذ في اليمن** (مصادر: SAMRL، DataGuidance).
- **نفاذية الهاتف ~60% والإنترنت ~17.7%**، ومرونة الشبكة ~28% (DataReportal،
  Internet Society Pulse) → تصميم يتحمّل ضعف الاتصال وانقطاعه.
- **قيود الرسائل SMS** في اليمن (Messente، Twilio) → اختيار مزوّد محلي: **الأوائل**.
- **NIST SP 800-63B** يصنّف الرسائل قناةً «مقيّدة» → أثّر على قرار التحقق الثنائي
  (انظر ADR-0003).

## المرحلة 1 — تصميم الواجهة (UI/UX)
- شعار متحرّك (logo animation) بمحور ارتكاز عند ≈(106,85) على viewBox `0 0 212 326`.
- شاشات المنتج الأساسية + إعادة تصميم شاشات: الإعدادات، التخصيص، الأمان.
- لغة تصميم موحّدة: بطاقات مجمّعة بحواف دائرية، عناوين خافتة، دعم RTL، bottom
  sheets بمقبض سحب مع blur، بلا chevrons، والإجراءات المدمّرة بلون
  `colorScheme.error` مع حوارات تأكيد.
- ثلاث قواعد إلزامية لكل شاشة: `AppLocalizations`، `colorScheme`، وحالات
  التحميل/الفراغ/الخطأ. (المرجع: `docs/design-language.md`، `docs/frontend-spec.md`)
- **النتيجة:** مرحلة الواجهة مكتملة.

## المرحلة 2 — نموذج البيانات (data-model v0.4)
وُثّق المخطط الكامل: `users`, `otp_codes`, `auth_sessions`, `conversations`,
`messages`, `message_sources`, `feedback`, `attachments`, `memories`,
`user_preferences`, `policy_versions`, `user_consents`، وقاعدة المعرفة
(`legal_documents`, `legal_articles`, `legal_chunks` مع pgvector + pg_trgm + RLS).
(التفاصيل الكاملة في `docs/data-model.md`.)

## المرحلة 3 — عقد الـ API (contract-first)
- كتابة `docs/api-contract.md` (v0.3) و`openapi.yaml` (v0.3.0).
- **المبدأ:** العقد يُكتب ويُعتمد **قبل** مواءمة التطبيق (انظر ADR-0001).
- استخراج واقع التطبيق عبر prompt مُوجّه لـ Claude، ثم تقييمه ومطابقته بالعقد.

## المرحلة 4 — مواءمة تطبيق Flutter مع العقد
نفّذ Claude المواءمة (بعد مراجعة خطته واعتمادها مع 5 تصحيحات):
- **27 ملفاً** تغيّرت، و`flutter analyze` = «No issues found!».
- أبرزها: `token_storage.dart`, `api_client.dart` (مع single-flight 401 refresh),
  إثراء `UserModel`, `ChatMessageModel` + `MessageStatus` (5 حالات),
  إعادة محاذاة SSE, نماذج جديدة منها `preferences_dto` بحقل `settings` من نوع Map,
  `AuthStatus.mfaRequired` + `MfaRequiredException`, وهياكل repositories.
- ترقية `flutter_secure_storage` إلى `^10.3.1` (بسبب تعارض win32 مع `^9.0.0`).
- **تحذير معلّق:** `flutter_secure_storage` على الويب غير آمن فعلياً (المفاتيح في
  IndexedDB/localStorage) → الإنتاج على الويب يحتاج access token في الذاكرة
  و refresh في httpOnly cookie.

## المرحلة 5 — توحيد وثائق الحوكمة
مواءمة كل الوثائق لتتّسق عند data-model v0.4 / api-contract v0.3.1:
- `rag-pipeline-spec.md` → v0.2 (ربط `messages.status` بعقدة التوليد، تسجيل
  `retrieval_path`/`model_version`، استثناء المرفقات من استرجاع MVP).
- `system-prompt.md` → v0.2 (§9 حقن التخصيص/الذاكرة مع حارس حوكمة: التخصيص
  يؤثّر على الأسلوب فقط لا على الحياد/الإسناد، ومعاملة الذكريات كبيانات لا تعليمات).
- `cryptography-standard.md` → v0.2 (الفهرس الأعمى HMAC، Argon2id مفعّل، جدول
  الحقول المشفّرة، MinIO at-rest).
- `api-contract.md` §8 → v0.3.1 (إضافة كائن `settings` للتفضيلات).

## سجلّ الأخطاء والتصحيحات (Errors & Corrections)
- ترميز advisor_role بأنابيب (pipe-encoding) **مرفوض** → الحقول الخام تذهب إلى
  `user_preferences.settings JSONB`، و advisor_role نص طبيعي نظيف (ADR-0006).
- ملاحظة SHA قديمة كادت تسبّب تعديلاً على نسخة قديمة → **قاعدة:** أعِد جلب الملف
  دائماً قبل التعديل.
- تعديل Claude على `auth_state.dart` أصاب `}` خاطئة داخل `copyWith` → صحّحه
  بإعادة كتابة الملف كاملاً.
- فشل حلّ `flutter_secure_storage ^9.0.0` (تعارض win32) → الترقية إلى `^10.3.1`.

## الحالة عند إغلاق هذه المدخلة
الواجهة مكتملة ومُواءَمة مع العقد على mocks؛ كل وثائق الحوكمة متّسقة؛ الخادم لم
يُبنَ بعد. الخطوة التالية: walking skeleton للـ backend (ADR-0009) ثم استبدال
الـ mocks بالتنفيذ الحقيقي.

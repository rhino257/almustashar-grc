---
title: "عقد الواجهة البرمجية (REST + SSE) — المستشار القانوني الذكي"
doc_id: ARC-API-001
version: 0.4
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-07-26
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "OWASP API Security Top 10", status: محاذاة }
related:
  - ARC-DESIGN-001  # المعمارية
  - ARC-DATA-001    # نموذج البيانات (v0.5)
  - ARC-RAG-001     # خط أنابيب الاسترجاع (مصدر أحداث SSE)
  - ARC-PROMPT-001  # التخصيص والذاكرة في المولّد
  - SEC-CRYPTO-001  # معيار التشفير (كلمة المرور، الهاتف، البريد، الرموز)
---

# عقد الواجهة البرمجية (REST + SSE)

> **حالة التنفيذ (وسوم هذه الوثيقة):**
> - 🟢 **مطبّق:** موجودٌ في `contracts/openapi.json` المُولّد من الباك اند.
> - 🟡 **مستهدف:** مصمّمٌ للبناء اللاحق (مرجعٌ للفريق)، ليس منقوصاً.
>
> المطبّق حالياً = **شريحة المصادقة (M0)**: `/v1/auth/*` و`/v1/policies/current`
> و`/v1/users/me` (GET/PATCH/DELETE) ومفتاح MFA مفرد و`/health` و`/`. بقية المسارات مستهدفة.

## 1. الغرض والنطاق
يعرّف هذا العقد نقاط النهاية بين الـ backend (FastAPI) وواجهة Flutter:
المسارات، الترويسات، أجسام الطلب/الاستجابة، أحداث البثّ (SSE)، ونموذج
الأخطاء. هو المرجع الملزِم للطرفين، ومُواءَمٌ مع نموذج البيانات ARC-DATA-001 v0.5.

## 2. المبادئ
- بادئة الإصدار `/v1` لكل مسارات الأعمال (عدا `/` و`/health` الجذريين).
- الجسم JSON (`Content-Type: application/json`) عدا بثّ SSE ورفع الملفات (presigned).
- الأوقات بصيغة ISO-8601 (UTC). المعرّفات UUID.
- أكواد HTTP قياسية + نموذج خطأ موحّد (القسم 4).
- **الحدّ الأدنى من البيانات:** الهاتف إلزاميّ (الهوية)، والبريد اختياريّ (استرداد + قناة بديلة).

## 2.1 علاقة هذا العقد بـ contracts/openapi.json
يُصدّر الباك اند مواصفة OpenAPI فعلية إلى `contracts/openapi.json` آلياً عبر
`scripts/export_openapi.py` (من `app.openapi()` مع APP_ENV=production)، ويحرسها CI
بفحص انحراف (`git diff --exit-code`). لذا:
- `contracts/openapi.json` = **المصدر الموثوق للسطح المُطبّق فعلياً**.
- هذه الوثيقة = **عقد التصميم** (تجمع المطبّق 🟢 والمستهدف 🟡 مع الشرح والحيثيات).
- ملف `openapi.yaml` المكتوب يدوياً في مستودع الواجهة **قديمٌ ومكسور** (أخطاء صياغة)
  ويجب إهماله لصالح `contracts/openapi.json` المُولّد.

## 3. المصادقة (هاتف + كلمة مرور + OTP + 2FA اختياري)
النموذج المعتمد:

- التسجيل والدخول عبر **رقم الهاتف + كلمة المرور**.
- **OTP إلزاميّ مرّةً عند أول تسجيل** (تفعيل الهاتف)، ولإعادة تعيين كلمة المرور،
  وللتحقّق من البريد الاختياري، وللتحقّق بخطوتين.
- **التحقّق بخطوتين (2FA) اختياريّ لكل مستخدم** عبر **SMS** (مزوّد محلّي مثل الأوائل).
- **البصمة/الـ PIN** وسيلة قفلٍ **محلية على الجهاز فقط** — لا تمرّ بالخادم ولا تُخزَّن.
- بعد الدخول يُصدر الخادم **رمز وصول (JWT قصير العمر)** و**رمز تجديد (مبهم قابل للإبطال)**.
- تُرسل الواجهة رمز الوصول: `Authorization: Bearer <access_token>`.
- يستخرج الـ backend `user_id` من الرمز ويضبط `app.current_user_id` لتفعيل RLS (ARC-DATA-001 §6).

### حماية بيانات الاعتماد (يحيل إلى SEC-CRYPTO-001)
- **كلمة المرور:** مجزّأة بـ **Argon2id** (time_cost=3, memory=64MiB, parallelism=2) — لا تُخزَّن نصّاً صريحاً.
- **الهاتف:** بصمة حتمية **HMAC-SHA-256** فقط (`phone_hash`) — **لا يوجد `phone_enc`**،
  ولا يُعاد الهاتف في `GET /v1/users/me`.
- **البريد (وبريد الاسترداد 🟡):** بصمة `HMAC-SHA-256` (`*_hash`) للبحث + قيمة مشفّرة `AES-256-GCM` (`*_enc`) للعرض.
- **رمز التجديد:** مجزّأ (`SHA-256`) وفريد في `auth_sessions` وقابلٌ للإبطال.
- **رمز OTP:** مجزّأ بـ **Argon2id** في `otp_codes`، قصير العمر، بحدّ محاولاتٍ ومهلة، مع `purpose` و`channel`.

### أعمار الرموز (مطابقة لـ config)
| الرمز | العمر | ملاحظة |
| --- | --- | --- |
| access (JWT) | 30 دقيقة (`ACCESS_TOKEN_TTL_SECONDS=1800`) | يُجدّد عبر refresh |
| refresh | 30 يوماً (`REFRESH_TOKEN_TTL_DAYS=30`) | مخزّنٌ مجزّأً، يُبطَل عند الخروج/الإبطال |
| OTP | 5 دقائق (`OTP_TTL_SECONDS=300`) | 5 محاولات (`OTP_MAX_ATTEMPTS=5`) ثم قفل مؤقّت (423) |

> ملاحظة تشغيلية: مزوّد الرسائل (OTP SMS) خلف علم (`SMS_PROVIDER=fake|alawael`)؛ في التطوير
> يُفعّل مسارٌ وهمٌّ (`fake`) ويُتاح مسار تطويري `POST /v1/dev/otp` (يُستبعد عند APP_ENV=production).

## 4. نموذج الأخطاء الموحّد
```json
{
  "error": {
    "code": "validation_error",
    "message": "human-readable message",
    "details": {}
  }
}
```

خريطة الأخطاء المطبّقة (تطابق `errors.py`؛ الرسائل بالعربية):

| كود HTTP | code | الدلالة |
| --- | --- | --- |
| 400 | validation_error | جسم/معامل غير صالح |
| 401 | unauthorized | رمز الوصول مفقود/منتهٍ/غير صالح |
| 403 | forbidden | محاولة وصولٍ لمورد مستخدمٍ آخر |
| 404 | not_found | المورد غير موجود |
| 409 | conflict | رقم الهاتف/البريد/اسم المستخدم مسجّلٌ مسبقاً |
| 422 | otp_invalid | رمز OTP خاطئ أو منتهٍ أو استُهلك |
| 423 | account_locked | تجاوز حدّ محاولات الدخول/OTP |
| 429 | service_busy | تجاوز سقف التزامن؛ أعد المحاولة بعد Retry-After |
| 500 | internal_error | خطأ غير متوقّع |

> 🟡 `policy_reconsent_required` (حجب الدخول حتى إعادة قبول إصدار سياسةٍ جديد) **مستهدف** — غير موجود في `errors.py` بعد.

## 5. خريطة نقاط النهاية
| الطريقة | المسار | الغرض | محمي؟ | الحالة |
| --- | --- | --- | --- | --- |
| GET | / | جذر الخدمة (معلومات أساسية) | لا | 🟢 |
| GET | /health | فحص الصحّة (قاعدة البيانات فقط) | لا | 🟢 |
| GET | /v1/policies/current | جلب الإصدارات السارية | لا | 🟢 |
| POST | /v1/auth/register | تسجيل حساب ويطلق OTP | لا | 🟢 |
| POST | /v1/auth/otp/request | إرسال/إعادة إرسال OTP | لا | 🟢 |
| POST | /v1/auth/otp/verify | التحقق بالـ OTP | لا | 🟢 |
| POST | /v1/auth/login | الدخول (قد يردّ mfa_required) | لا | 🟢 |
| POST | /v1/auth/refresh | تجديد رمز الوصول | لا (refresh) | 🟢 |
| POST | /v1/auth/logout | إبطال رمز التجديد | نعم | 🟢 |
| POST | /v1/auth/password/reset-request | طلب رمز إعادة التعيين | لا | 🟢 |
| POST | /v1/auth/password/reset | تعيين كلمة مرور جديدة | لا | 🟢 |
| GET | /v1/auth/sessions | قائمة الجلسات النشطة | نعم | 🟢 |
| DELETE | /v1/auth/sessions/{id} | إبطال جلسةٍ محدّدة | نعم | 🟢 |
| GET | /v1/users/me | قراءة الملف | نعم | 🟢 |
| PATCH | /v1/users/me | تحديث الملف | نعم | 🟢 (جزئيّ) |
| DELETE | /v1/users/me | حذف الحساب (PDPL) | نعم | 🟢 |
| POST | /v1/users/me/mfa | مفتاح تفعيل/إلغاء 2FA (مؤقّت) | نعم | 🟢 (مؤقّت) |
| POST | /v1/dev/otp | إظهار آخر OTP (تطوير فقط) | لا | 🟢 (dev) |
| POST | /v1/users/me/mfa/enable | بدء تفعيل 2FA (OTP للتأكيد) | نعم | 🟡 |
| POST | /v1/users/me/mfa/verify | تأكيد تفعيل 2FA | نعم | 🟡 |
| DELETE | /v1/users/me/mfa | إلغاء 2FA (بتأكيد) | نعم | 🟡 |
| POST | /v1/users/me/avatar/upload-url | رابط رفع صورة الملف | نعم | 🟡 |
| PATCH | /v1/users/me/avatar | تثبيت مفتاح الصورة | نعم | 🟡 |
| DELETE | /v1/users/me/avatar | إزالة الصورة | نعم | 🟡 |
| GET | /v1/users/me/preferences | قراءة التفضيلات | نعم | 🟡 |
| PATCH | /v1/users/me/preferences | تحديث التفضيلات | نعم | 🟡 |
| GET | /v1/memories | قائمة الذاكرة | نعم | 🟡 |
| POST | /v1/memories | إضافة ذكرى يدوية | نعم | 🟡 |
| PATCH | /v1/memories/{id} | تعديل ذكرى | نعم | 🟡 |
| DELETE | /v1/memories/{id} | حذف ذكرى | نعم | 🟡 |
| DELETE | /v1/memories | مسح الذاكرة | نعم | 🟡 |
| GET | /v1/conversations | قائمة المحادثات | نعم | 🟡 |
| POST | /v1/conversations | إنشاء محادثة | نعم | 🟡 |
| GET | /v1/conversations/{id} | محادثة + رسائلها | نعم | 🟡 |
| PATCH | /v1/conversations/{id} | تسمية/تثبيت/أرشفة | نعم | 🟡 |
| DELETE | /v1/conversations/{id} | حذف محادثة | نعم | 🟡 |
| POST | /v1/conversations/{id}/messages | طرح سؤال (بثّ SSE) | نعم | 🟡 |
| POST | /v1/messages/{id}/stop | إيقاف بثّ | نعم | 🟡 |
| POST | /v1/messages/{id}/feedback | تغذية راجعة | نعم | 🟡 |
| POST | /v1/attachments/upload-url | رابط رفع مرفق | نعم | 🟡 |
| PATCH | /v1/attachments/{id} | تثبيت جاهزية المرفق | نعم | 🟡 |
| GET | /v1/attachments/{id}/download-url | رابط تنزيل مؤقّت | نعم | 🟡 |

## 6. السياسات والموافقة
### GET /v1/policies/current 🟢
```json
{ "policies": [
  { "id": "uuid", "policy_type": "terms",      "version": "v1.0-2026-07" },
  { "id": "uuid", "policy_type": "privacy",    "version": "v1.0-2026-07" },
  { "id": "uuid", "policy_type": "disclaimer", "version": "v1.0-2026-07" }
] }
```
> الموافقة تُلتقط **عند إنشاء الحساب**. 🟡 حجب الدخول عند نشر إصدارٍ جديد
> (`policy_reconsent_required`) مستهدفٌ لم يُفرَض بعد.

## 7. المصادقة والحساب
### POST /v1/auth/register 🟢
```json
// request — email اختياري
{ "full_name": "عبدالله محمد", "phone": "777000000", "password": "********",
  "email": "user@example.com",
  "consent": { "accepted": true,
               "policy_version_ids": ["uuid-terms", "uuid-privacy", "uuid-disclaimer"] } }
// response 201
{ "user_id": "uuid", "phone_verified": false, "verification_required": true }
```
الرقم/البريد المسجّل مسبقاً يردّ `409 conflict`. يسجّل الخادم صفّ `user_consents`
بـ `method='signup'` مع `ip`/`user_agent`.

### POST /v1/auth/otp/request 🟢
```json
{ "phone": "777000000", "purpose": "verify_phone", "channel": "sms" }
// purpose ∈ verify_phone | reset_password | verify_email | mfa_login
// response 200
{ "sent": true, "expires_in": 300, "resend_after": 60 }
```

### POST /v1/auth/otp/verify 🟢
```json
{ "phone": "777000000", "code": "123456", "purpose": "verify_phone" }
// response 200 — verify_phone و mfa_login تُصدران الرموز
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "phone_verified": true } }
```
غرض `verify_email` يردّ `{ "email_verified": true }` دون رموز. رمزٌ خاطئ/منتهٍ
يردّ `422 otp_invalid`؛ تجاوز الحدّ يردّ `423 account_locked`.

### POST /v1/auth/login 🟢
```json
{ "phone": "777000000", "password": "********" }
// response 200 — عند تعطيل 2FA
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "full_name": "عبدالله محمد", "phone_verified": true } }
// response 200 — عند تفعيل 2FA (لا رموز بعد؛ أُرسل OTP)
{ "mfa_required": true, "mfa_channel": "sms", "otp_sent": true, "expires_in": 300 }
```
بعد `mfa_required` تُكمِل الواجهة عبر `POST /v1/auth/otp/verify` بغرض `mfa_login`.

### POST /v1/auth/refresh 🟢
```json
{ "refresh_token": "opaque" }  // -> 200 { "access_token": "jwt", "token_type": "Bearer", "expires_in": 1800 }
```

### POST /v1/auth/logout 🟢
```json
{ "refresh_token": "opaque" }  // -> 204 (يُبطل revoked_at)
```

### POST /v1/auth/password/reset-request 🟢
```json
// ردّ موحّد دائماً لتفادي كشف الأرقام
{ "phone": "777000000" }  // -> 200 { "sent": true, "expires_in": 300 }
```

### POST /v1/auth/password/reset 🟢
```json
{ "phone": "777000000", "code": "123456", "new_password": "********" }  // -> 200 { "reset": true }
```
بعد إعادة التعيين تُبطَل كل جلسات refresh القائمة.

### GET /v1/auth/sessions 🟢
```json
{ "items": [
  { "id": "uuid", "device_label": "Chrome · Windows", "ip": "x.x.x.x",
    "last_seen_at": "ISO", "created_at": "ISO", "current": true }
] }
```

### DELETE /v1/auth/sessions/{id} 🟢
الاستجابة 204. يُبطل الجلسة المحدّدة (`revoked_at`). لا يمكن حذف جلسات مستخدمٍ آخر (403).

### GET /v1/users/me 🟢
```json
// الحقول المطبّقة فعلياً (لا يُعاد الهاتف):
{ "user_id": "uuid", "full_name": "عبدالله محمد", "username": "abdullah",
  "phone_verified": true, "email": "u***@example.com", "email_verified": false,
  "mfa_enabled": false }
// 🟡 حقول مستهدفة (عند إضافة الأعمدة): user_type, profession, governorate, recovery_email, avatar_url
```
> البريد يُعرض مُقنّعاً. **الهاتف لا يُعاد** (مخزّنٌ كبصمة فقط).

### PATCH /v1/users/me 🟢 (جزئيّ)
```json
// المطبّق حالياً: full_name, username, email
{ "full_name": "...", "username": "...", "email": "new@example.com" }
// response 200 — تغيير البريد يعيد email_verified=false ويستلزم verify_email
{ "user_id": "uuid", "username": "...", "email_verified": false }
// 🟡 حقول مستهدفة: user_type, profession, governorate, recovery_email
```
تعارض اسم المستخدم/البريد يردّ `409 conflict`.

### DELETE /v1/users/me 🟢
الاستجابة 204. يحذف المستخدم وجلساته ورموزه وموافقاته متتالياً (ومحادثاته/مرفقاته عند بنائها).

### التحقّق بخطوتين (2FA)
**🟢 المطبّق حالياً (مؤقّت):** مفتاحٌ مفرد `POST /v1/users/me/mfa` يبدّل `mfa_enabled`.
```json
// POST /v1/users/me/mfa  -> 200 { "mfa_enabled": true }  // أو false عند الإلغاء
```
**🟡 المستهدف (تدفّق مؤكّد بـ OTP):** يُستبدل المفتاح المؤقّت بتدفّقٍ يمنع التفعيل دون تأكيد:
```json
// POST /v1/users/me/mfa/enable -> 200 { "otp_sent": true, "channel": "sms", "expires_in": 300 }
// POST /v1/users/me/mfa/verify { "code": "123456" } -> 200 { "mfa_enabled": true }
// DELETE /v1/users/me/mfa (بتأكيد) -> 200 { "mfa_enabled": false }
```

### 🟡 صورة الملف (MinIO) — مستهدف
```json
// POST /v1/users/me/avatar/upload-url  { "mime_type": "image/jpeg", "size_bytes": 240000 }
// -> 200 { "avatar_key": "avatars/uuid/....jpg", "upload_url": "https://minio/...signed-PUT", "expires_in": 300 }
// PATCH /v1/users/me/avatar { "avatar_key": "avatars/uuid/....jpg" } -> 200
```

## 8. 🟡 التفضيلات (مستهدف)
### GET /v1/users/me/preferences
```json
{ "theme": "system", "language": "ar", "memory_enabled": true,
  "advisor_role": null, "improve_service": false, "settings": {} }
```
### PATCH /v1/users/me/preferences
```json
{ "theme": "dark", "language": "ar", "memory_enabled": false,
  "advisor_role": "صياغة موجزة ومباشرة", "improve_service": true,
  "settings": { "response_style": "concise", "user_role": "lawyer", "nickname": "أبو محمد" } }
// response 200 — يعيد الحالة الكاملة
```
> `advisor_role` نصّ لغةٍ طبيعية موجز يُحقن في برومبت المولّد (ARC-PROMPT-001 §9)
> لضبط **الأسلوب فقط**؛ و`settings` خريطة JSON حرّة. الشخصنة أسلوبٌ فقط ولا تتجاوز الإسناد أو الحياد.

## 9. 🟡 الذاكرة (مستهدف)
### GET /v1/memories
```json
{ "items": [ { "id": "uuid", "content": "يفضّل الأمثلة العملية", "source": "auto", "is_active": true, "created_at": "ISO" } ] }
```
### POST /v1/memories
```json
{ "content": "يعمل في القطاع المصرفي" }  // -> 201 (source=manual)
```
### PATCH /v1/memories/{id}
```json
{ "is_active": false }  // أو { "content": "..." } -> 200
```
### DELETE /v1/memories/{id} -> 204   ·   DELETE /v1/memories -> 204 (مسح الكل)
> إن كانت `preferences.memory_enabled=false` لا تُحقن الذاكرة في التوليد (ARC-PROMPT-001 §9).

## 10. 🟡 المحادثات (مستهدف)
### GET /v1/conversations?limit=20&before=<ISO>&archived=false
```json
{ "items": [ { "id": "uuid", "title": "string|null", "pinned": false,
               "archived_at": null, "created_at": "ISO", "updated_at": "ISO" } ],
  "next_before": "ISO|null" }
```
### POST /v1/conversations
```json
{ "title": "string|null" }  // -> 201
```
### GET /v1/conversations/{id}
```json
{ "id": "uuid", "title": "string|null", "pinned": false, "archived_at": null,
  "messages": [
    { "id": "uuid", "role": "user", "content": "string", "status": "complete",
      "attachments": [ { "id": "uuid", "file_name": "عقد.pdf",
                         "mime_type": "application/pdf", "size_bytes": 12345, "status": "ready" } ],
      "created_at": "ISO" },
    { "id": "uuid", "role": "assistant", "content": "string", "status": "complete",
      "is_grounded": true,
      "sources": [ { "article_id": "uuid|null", "law_name": "string", "article": "string",
                     "variant": "original|sanaa_amended", "source_ref": "string",
                     "snippet": "string", "score": 0.0, "rank": 1 } ],
      "created_at": "ISO" }
  ] }
```
### PATCH /v1/conversations/{id}
```json
{ "title": "عنوان جديد", "pinned": true, "archived": true }
// response 200 — archived=true يضبط archived_at=now()، وfalse يصفّره
{ "id": "uuid", "title": "عنوان جديد", "pinned": true, "archived_at": "ISO" }
```
### DELETE /v1/conversations/{id}
الاستجابة 204. حذف متتالٍ للرسائل والمصادر والتغذية الراجعة والمرفقات.

## 11. 🟡 طرح سؤال + بثّ SSE (مستهدف — نواة النظام)
### POST /v1/conversations/{id}/messages
```http
Headers:
  Authorization: Bearer <access_token>
  Accept: text/event-stream
```
```json
{ "content": "نص سؤال المستخدم بالعربية", "attachment_ids": ["uuid"] }
```
الاستجابة `200 text/event-stream`. تُنشأ رسالة المستخدم ثم رسالة المساعد بحالة
`streaming`، وعند الاكتمال `complete` (أو `stopped`/`error`). عند الازدحام يُصدر حدث `queued`
المخصّص (انظر §12)، أو يردّ `429` مع `Retry-After` عند تجاوز السقف.

### POST /v1/messages/{id}/stop
يوقف بثّاً جارٍ؛ يضبط `messages.status='stopped'`. الاستجابة `200 { "stopped": true }`.

## 12. 🟡 مخطّط أحداث SSE (مستهدف)
كل حدثٍ سطر `event:` يتبعه `data:` (JSON):
```text
event: status
data: {"node":"analyze","label":"جارٍ تحليل السؤال..."}
event: queued
data: {"position":2,"label":"طلبك في قائمة الانتظار..."}
event: token
data: {"delta":"جزء من الإجابة"}
event: sources
data: {"sources":[
  {"article_id":"uuid|null","law_name":"...","article":"...","variant":"original",
   "source_ref":"...","snippet":"...","score":0.0,"rank":1}]}
event: done
data: {"message_id":"uuid","is_grounded":true,"status":"complete"}
event: error
data: {"code":"internal_error","message":"..."}
```

| الحدث | الدلالة | ربط الحالة |
| --- | --- | --- |
| status | حالة تنفيذٍ مرتبطة باسم عقدة LangGraph | messages.status='streaming' |
| queued | الطلب في الطابور (موقعه التقريبي) | pending |
| token | جزء من الإجابة (بثّ تدريجي) | streaming |
| sources | محتوى خانة المصادر (يُحفظ في message_sources) | — |
| done | اكتمال + علم الإسناد + الحالة النهائية | complete |
| error | خطأ أثناء البثّ | error |

> قرار محسوم: `queued` **حدثٌ مخصّص** (لا حالة فرعية ضمن status). `is_grounded=false` تعرض الواجهة وسماً «غير مسنَد».

## 13. 🟡 التغذية الراجعة (مستهدف)
### POST /v1/messages/{id}/feedback
```json
{ "rating": "down", "issue_category": "دقة", "safety_subcategory": null,
  "conversation_included": true, "comment": "string|null" }
// response 201 { "id": "uuid", "message_id": "uuid", "rating": "down", "created_at": "ISO" }
```
> `model_version` و`retrieval_path` يضبطهما الخادم تلقائياً (لا يرسلهما العميل).

## 14. 🟡 المرفقات (MinIO) — مستهدف
```json
// POST /v1/attachments/upload-url
{ "file_name": "عقد.pdf", "mime_type": "application/pdf", "size_bytes": 240000 }
// -> 200 { "attachment_id": "uuid", "storage_key": "attachments/uuid/....pdf",
//          "upload_url": "https://minio/...signed-PUT", "expires_in": 300 }
// PATCH /v1/attachments/{id}  { "status": "ready" } -> 200
// GET  /v1/attachments/{id}/download-url -> 200 { "url": "...signed-GET", "expires_in": 300 }
```
- الحدّ الأقصى **5 مرفقات** للرسالة؛ يتحقّق الخادم من MIME والحجم.
- المرفق يُنشأ بحالة `preparing` ثم `ready` وقد يصبح `error`.

## 15. الصحّة
### GET /health 🟢
```json
// المطبّق حالياً: فحص قاعدة البيانات فقط (SELECT 1)
{ "status": "ok", "db": "up" }
// 🟡 فحوصات مستهدفة لاحقاً: model, vector_index, object_store, sms_gateway
```

## 16. اعتبارات أمنية
- 🟢 كل نقلٍ حصراً عبر TLS (SEC-CRYPTO-001 §2).
- 🟢 رمز الوصول JWT قصير العمر؛ رمز التجديد مبهمٌ ومخزّنٌ مجزّأً وقابلٌ للإبطال.
- 🟢 كلمات المرور بـ Argon2id؛ الهاتف بصمة HMAC-SHA-256 فقط؛ البريد enc + hash.
- 🟢 عزل الصفوف (RLS FORCE + fail-closed) على جداول المصادقة؛ التحقّق من الملكية (403/404).
- 🟢 تنقيح السجلّات (PII/أسرار): مطبّق فعلياً (`redact_processor`) — لا تُسجّل كلمات المرور/الرموز/الهاتف.
- 🟢 سقف محاولات OTP (5 → 423) مطبّق في طبقة الخدمة.
- 🟡 **تحديد المعدّل (rate limiting) كوسيط (middleware) لكل هاتف/IP مستهدف** — غير مطبّق بعد (الموجود فقط سقف محاولات OTP).
- 🟢 ردودٌ موحّدة على مسارات OTP/إعادة التعيين لتفادي كشف الأرقام.
- 🟡 روابط MinIO الموقّعة وإدماج المرفقات (مع المراحل اللاحقة).

## 17. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | نموذج المصادقة (هاتف+كلمة مرور+OTP+2FA SMS) | **مُعتمد** |
| 2 | ترقيم بالصفحات: cursor مقابل offset | مبدئياً cursor زمني |
| 3 | دلالة queued مقابل 429 | **محسوم: حدث `queued` مخصّص** (مستهدف مع طبقة الطابور) |
| 4 | حقل sources: إضافة article_id/rank | **محسوم: مُضافان** |
| 5 | مزوّد رسائل OTP (SMS) الفعلي | خلف feature flag (`SMS_PROVIDER`)؛ الأوائل مرشّحاً |
| 6 | مصادقة TOTP كطبقةٍ إضافية | مؤجّلة؛ 2FA الحالي عبر SMS |
| 7 | تدفّق MFA: مفتاح مفرد مقابل enable/verify/disable | **محسوم: المفتاح الحالي مؤقّت؛ التدفّق المؤكّد بـ OTP مستهدف** |
| 8 | الدخول الاجتماعي (social login) | **محسوم: خارج نطاق MVP** (يُزال من الواجهة) |
| 9 | استخراج نصّ المرفقات للسياق | خارج نطاق MVP |

## 18. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لعقد REST + SSE (نموذج الضيف) |
| 0.2 | 2026-07-02 | اعتماد المصادقة الكاملة (هاتف+كلمة مرور+OTP)؛ رموز Bearer؛ أخطاء 401/409/422/423؛ ربط SEC-CRYPTO-001 |
| 0.3 | 2026-07-04 | مواءمةٌ مع ARC-DATA-001 v0.4: بريدٌ اختياري، 2FA عبر SMS، الجلسات، الصورة والمرفقات، التفضيلات، الذاكرة، المحادثات، البثّ والإيقاف، التغذية الراجعة، السياسات |
| 0.3.1 | 2026-07-04 | إضافة حقل `settings` (JSONB) للتفضيلات؛ توضيح فصل الشخصنة عن الإسناد (ARC-PROMPT-001 §9) |
| 0.4 | 2026-07-26 | **مطابقة الواقع مع `contracts/openapi.json` المُولّد:** وسم 🟢/🟡 لكل نقطة نهاية؛ الهاتف بصمةً فقط ولا يُعاد في GET /users/me؛ MFA الحالي مفتاحٌ مفرد (مؤقّت) مع إبقاء enable/verify/disable كهدف؛ تصحيح /health (فحص قاعدة البيانات فقط) والمسار /؛ مطابقة خريطة الأخطاء لـ errors.py (policy_reconsent_required مستهدف)؛ حدث `queued` مخصّص؛ إضافة article_id وrank إلى sources؛ توضيح أن تحديد المعدّل غير مطبّق كوسيط وأن تنقيح السجلّات مطبّق؛ إضافة قسم علاقة العقد بـ openapi.json وإهمال openapi.yaml القديم |

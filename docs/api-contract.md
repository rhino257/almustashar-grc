---
title: "عقد الواجهة البرمجية (REST + SSE) — المستشار القانوني الذكي"
doc_id: ARC-API-001
version: 0.3
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-07-04
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "OWASP API Security Top 10", status: محاذاة }
related:
  - ARC-DESIGN-001  # المعمارية
  - ARC-DATA-001    # نموذج البيانات (v0.4)
  - ARC-RAG-001     # خط أنابيب الاسترجاع (مصدر أحداث SSE)
  - SEC-CRYPTO-001  # معيار التشفير (كلمة المرور، الهاتف، البريد، الرموز)
---

# عقد الواجهة البرمجية (REST + SSE)

## 1. الغرض والنطاق
يعرّف هذا العقد نقاط النهاية بين الـ backend (FastAPI) وواجهة Flutter:
المسارات، الترويسات، أجسام الطلب/الاستجابة، أحداث البثّ (SSE)، ونموذج
الأخطاء. هو المرجع الملزِم للطرفين، ومُواءَمٌ مع نموذج البيانات ARC-DATA-001 v0.4.

## 2. المبادئ
- بادئة الإصدار `/v1` لكل المسارات.
- الجسم JSON (`Content-Type: application/json`) عدا بثّ SSE ورفع الملفات (presigned).
- الأوقات بصيغة ISO-8601 (UTC). المعرّفات UUID.
- أكواد HTTP قياسية + نموذج خطأ موحّد (القسم 4).
- **الحدّ الأدنى من البيانات:** الهاتف إلزاميّ (الهوية)، والبريد اختياريّ (استرداد + قناة بديلة).

## 3. المصادقة (هاتف + كلمة مرور + OTP + 2FA اختياري)
النموذج المعتمد:

- التسجيل والدخول عبر **رقم الهاتف + كلمة المرور**.
- **OTP إلزاميّ مرّةً عند أول تسجيل** (تفعيل الهاتف)، ولإعادة تعيين كلمة المرور،
  وللتحقّق من البريد الاختياري، وللتحقّق بخطوتين.
- **التحقّق بخطوتين (2FA) اختياريّ لكل مستخدم** عبر **SMS** (مزوّد محلّي مثل الأوائل).
  عند تفعيله (`users.mfa_enabled=true`) يتطلّب الدخول خطوة OTP إضافية بغرض `mfa_login`.
- **البصمة/الـ PIN** وسيلة قفلٍ ودخولٍ **محلية على الجهاز فقط** — لا تمرّ بالخادم ولا تُخزَّن في القاعدة.
- بعد الدخول يُصدر الخادم **رمز وصول (JWT قصير العمر)** و**رمز تجديد (قيمة مبهمة قابلة للإبطال)**.
- تُرسل الواجهة رمز الوصول في كل طلبٍ محمي: `Authorization: Bearer <access_token>`.
- يستخرج الـ backend `user_id` من الرمز ويضبط `app.current_user_id` لتفعيل RLS (ARC-DATA-001 §6).

### حماية بيانات الاعتماد (يحيل إلى SEC-CRYPTO-001)
- **كلمة المرور:** مجزّأة بـ **Argon2id** — لا تُخزَّن ولا تُسجَّل نصّاً صريحاً أبداً.
- **الهاتف/البريد/بريد الاسترداد:** مشفّرة `AES-256-GCM` (`*_enc`) للعرض، مع بصمة
  حتمية `HMAC-SHA-256` (`*_hash`) للبحث/الفرادة دون نصٍّ صريح.
- **رمز التجديد:** مجزّأ (`SHA-256`) في `auth_sessions` وقابلٌ للإبطال.
- **رمز OTP:** مجزّأ في `otp_codes`، قصير العمر، بحدّ محاولاتٍ ومهلة، مع `purpose` و`channel`.

### أعمار الرموز (مبدئي)
| الرمز | العمر | ملاحظة |
| --- | --- | --- |
| access (JWT) | 30 دقيقة | يُجدَّد عبر refresh |
| refresh | 30 يوماً | مخزّنٌ مجزّأً، يُبطَل عند الخروج/الإبطال |
| OTP | 5 دقائق | 5 محاولات كحدٍّ أقصى ثم قفل مؤقّت |

> ملاحظة تشغيلية: مزوّد الرسائل (OTP SMS) خلف علمٍ (feature flag)؛ في التطوير
> يُفعَّل مسار وهمي يعيد الرمز في السجل الآمن فقط. البريد عبر SMTP خلف العلم ذاته.

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

| كود HTTP | code | الدلالة |
| --- | --- | --- |
| 400 | validation_error | جسم/معامل غير صالح |
| 401 | unauthorized | رمز الوصول مفقود/منتهٍ/غير صالح |
| 403 | forbidden | محاولة وصولٍ لمورد مستخدمٍ آخر |
| 403 | policy_reconsent_required | يلزم قبول إصدار سياسةٍ جديد قبل المتابعة |
| 404 | not_found | المورد غير موجود |
| 409 | conflict | رقم الهاتف/البريد/اسم المستخدم مسجّلٌ مسبقاً |
| 422 | otp_invalid | رمز OTP خاطئ أو منتهٍ أو استُهلك |
| 423 | account_locked | تجاوز حدّ محاولات الدخول/OTP |
| 429 | service_busy | تجاوز سقف التزامن؛ أعد المحاولة بعد Retry-After |
| 500 | internal_error | خطأ غير متوقّع |

## 5. خريطة نقاط النهاية
| الطريقة | المسار | الغرض | محمي؟ |
| --- | --- | --- | --- |
| GET | /v1/policies/current | جلب الإصدارات السارية (شروط/خصوصية/إخلاء) | لا |
| POST | /v1/auth/register | تسجيل حساب (اسم+هاتف+كلمة مرور+بريد اختياري+موافقة) ويطلق OTP | لا |
| POST | /v1/auth/otp/request | إرسال/إعادة إرسال OTP (purpose+channel) | لا |
| POST | /v1/auth/otp/verify | التحقق بالـ OTP (تفعيل هاتف/2FA يصدر الرموز) | لا |
| POST | /v1/auth/login | الدخول بهاتف+كلمة مرور (قد يردّ mfa_required) | لا |
| POST | /v1/auth/refresh | تجديد رمز الوصول | لا (refresh) |
| POST | /v1/auth/logout | إبطال رمز التجديد للجلسة الحالية | نعم |
| POST | /v1/auth/password/reset-request | طلب رمز لإعادة تعيين كلمة المرور | لا |
| POST | /v1/auth/password/reset | تعيين كلمة مرور جديدة برمز OTP | لا |
| GET | /v1/auth/sessions | قائمة الجلسات النشطة | نعم |
| DELETE | /v1/auth/sessions/{id} | إبطال جلسةٍ محدّدة | نعم |
| GET | /v1/users/me | قراءة الملف | نعم |
| PATCH | /v1/users/me | تحديث الاسم/المستخدم/النوع/المهنة/المحافظة/البريد/بريد الاسترداد | نعم |
| DELETE | /v1/users/me | حذف الحساب (PDPL، حذف متتالٍ) | نعم |
| POST | /v1/users/me/avatar/upload-url | طلب رابط رفع صورة الملف (MinIO presigned) | نعم |
| PATCH | /v1/users/me/avatar | تثبيت مفتاح الصورة بعد الرفع | نعم |
| DELETE | /v1/users/me/avatar | إزالة صورة الملف | نعم |
| POST | /v1/users/me/mfa/enable | بدء تفعيل 2FA (يرسل OTP للتأكيد) | نعم |
| POST | /v1/users/me/mfa/verify | تأكيد تفعيل 2FA بالـ OTP | نعم |
| DELETE | /v1/users/me/mfa | إلغاء تفعيل 2FA | نعم |
| GET | /v1/users/me/preferences | قراءة التفضيلات | نعم |
| PATCH | /v1/users/me/preferences | تحديث السمة/اللغة/الذاكرة/تخصيص المستشار/تحسين الخدمة | نعم |
| GET | /v1/memories | قائمة الذاكرة | نعم |
| POST | /v1/memories | إضافة ذكرى يدوية | نعم |
| PATCH | /v1/memories/{id} | تفعيل/تعطيل/تعديل ذكرى | نعم |
| DELETE | /v1/memories/{id} | حذف ذكرى | نعم |
| DELETE | /v1/memories | مسح كل الذاكرة | نعم |
| GET | /v1/conversations | قائمة المحادثات (تدعم فلترة الأرشيف) | نعم |
| POST | /v1/conversations | إنشاء محادثة | نعم |
| GET | /v1/conversations/{id} | محادثة + رسائلها | نعم |
| PATCH | /v1/conversations/{id} | تسمية/تثبيت/أرشفة | نعم |
| DELETE | /v1/conversations/{id} | حذف محادثة | نعم |
| POST | /v1/conversations/{id}/messages | طرح سؤال (يردّ ببثّ SSE) | نعم |
| POST | /v1/messages/{id}/stop | إيقاف بثّ رسالةٍ جارٍ | نعم |
| POST | /v1/messages/{id}/feedback | تغذية راجعة على رسالة | نعم |
| POST | /v1/attachments/upload-url | طلب رابط رفع مرفق (MinIO presigned) | نعم |
| PATCH | /v1/attachments/{id} | تثبيت جاهزية المرفق بعد الرفع | نعم |
| GET | /v1/attachments/{id}/download-url | رابط تنزيلٍ مؤقّت للمرفق | نعم |
| GET | /v1/health | فحص الصحّة | لا |

## 6. السياسات والموافقة
### GET /v1/policies/current
```json
{ "policies": [
  { "id": "uuid", "policy_type": "terms",      "version": "v1.0-2026-07" },
  { "id": "uuid", "policy_type": "privacy",    "version": "v1.0-2026-07" },
  { "id": "uuid", "policy_type": "disclaimer", "version": "v1.0-2026-07" }
] }
```
> الموافقة تُلتقط **عند إنشاء الحساب** (لا شاشة مستقلّة). عند نشر إصدارٍ جديد
> (`policy_versions.is_current`) يُحجب الدخول حتى إعادة القبول (`policy_reconsent_required`).

## 7. المصادقة والحساب
### POST /v1/auth/register
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

### POST /v1/auth/otp/request
```json
// request — channel افتراضياً sms
{ "phone": "777000000", "purpose": "verify_phone", "channel": "sms" }
// purpose ∈ verify_phone | reset_password | verify_email | mfa_login
// للبريد: { "email": "user@example.com", "purpose": "verify_email", "channel": "email" }
// response 200
{ "sent": true, "expires_in": 300, "resend_after": 60 }
```

### POST /v1/auth/otp/verify
```json
// request
{ "phone": "777000000", "code": "123456", "purpose": "verify_phone" }
// response 200 — verify_phone و mfa_login تُصدران الرموز
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "user_type": "citizen", "phone_verified": true } }
```
غرض `verify_email` يردّ `{ "email_verified": true }` دون رموز. رمزٌ خاطئ/منتهٍ
يردّ `422 otp_invalid`؛ تجاوز الحدّ يردّ `423 account_locked`.

### POST /v1/auth/login
```json
// request
{ "phone": "777000000", "password": "********" }
// response 200 — عند تعطيل 2FA
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "user_type": "citizen", "full_name": "عبدالله محمد",
            "phone_verified": true } }
// response 200 — عند تفعيل 2FA (لا رموز بعد؛ أُرسل OTP)
{ "mfa_required": true, "mfa_channel": "sms", "otp_sent": true, "expires_in": 300 }
```
بعد `mfa_required` تُكمِل الواجهة عبر `POST /v1/auth/otp/verify` بغرض `mfa_login`
لاستلام الرموز.

### POST /v1/auth/refresh
```json
// request
{ "refresh_token": "opaque" }
// response 200
{ "access_token": "jwt", "token_type": "Bearer", "expires_in": 1800 }
```

### POST /v1/auth/logout
```json
// request
{ "refresh_token": "opaque" }
// response 204 — يُبطل رمز التجديد (revoked_at) في auth_sessions
```

### POST /v1/auth/password/reset-request
```json
// request — ردٌّ موحّد دائماً لتفادي كشف الأرقام المسجّلة
{ "phone": "777000000" }
// response 200
{ "sent": true, "expires_in": 300 }
```

### POST /v1/auth/password/reset
```json
// request
{ "phone": "777000000", "code": "123456", "new_password": "********" }
// response 200
{ "reset": true }
```
بعد إعادة التعيين تُبطَل كل جلسات refresh القائمة للمستخدم.

### GET /v1/auth/sessions
```json
{ "items": [
  { "id": "uuid", "device_label": "Chrome · Windows", "ip": "x.x.x.x",
    "last_seen_at": "ISO", "created_at": "ISO", "current": true }
] }
```

### DELETE /v1/auth/sessions/{id}
الاستجابة 204. يُبطل الجلسة المحدّدة (`revoked_at`). لا يمكن حذف جلسات مستخدمٍ آخر (403).

### GET /v1/users/me
```json
{ "user_id": "uuid", "full_name": "عبدالله محمد", "username": "abdullah",
  "user_type": "citizen", "profession": null, "governorate": null,
  "phone_verified": true,
  "email": "u***@example.com", "email_verified": false,
  "recovery_email": null, "avatar_url": "https://minio/.../signed",
  "mfa_enabled": false }
```
> البريد وبريد الاسترداد يُعرضان مُقنَّعين؛ `avatar_url` رابطٌ مؤقّتٌ من MinIO.

### PATCH /v1/users/me
```json
// request — كل الحقول اختيارية
{ "full_name": "...", "username": "...", "user_type": "lawyer",
  "profession": "attorney", "governorate": "Sanaa",
  "email": "new@example.com", "recovery_email": "rec@example.com" }
// response 200 — تغيير البريد يعيد email_verified=false ويستلزم verify_email
{ "user_id": "uuid", "user_type": "lawyer", "username": "...",
  "email_verified": false }
```
تعارض اسم المستخدم/البريد يردّ `409 conflict`.

### DELETE /v1/users/me
الاستجابة 204. يحذف المستخدم وكل محادثاته ورسائله وجلساته ورموزه ومرفقاته
وذاكرته وتفضيلاته وسجلّات موافقته متتالياً، وتُحذف ملفات MinIO المرتبطة.

### صورة الملف (MinIO)
```json
// POST /v1/users/me/avatar/upload-url
// request
{ "mime_type": "image/jpeg", "size_bytes": 240000 }
// response 200
{ "avatar_key": "avatars/uuid/....jpg", "upload_url": "https://minio/...signed-PUT",
  "expires_in": 300 }
// PATCH /v1/users/me/avatar   { "avatar_key": "avatars/uuid/....jpg" }  -> 200
```
يتحقّق الخادم من MIME والحجم قبل إصدار الرابط.

### التحقّق بخطوتين (2FA عبر SMS)
```json
// POST /v1/users/me/mfa/enable  -> يرسل OTP إلى هاتف المستخدم
// response 200
{ "otp_sent": true, "channel": "sms", "expires_in": 300 }
// POST /v1/users/me/mfa/verify  { "code": "123456" } -> 200 { "mfa_enabled": true }
// DELETE /v1/users/me/mfa       -> 200 { "mfa_enabled": false }
```

## 8. التفضيلات
### GET /v1/users/me/preferences
```json
{ "theme": "system", "language": "ar", "memory_enabled": true,
  "advisor_role": null, "improve_service": false }
```
### PATCH /v1/users/me/preferences
```json
// request — كل الحقول اختيارية
{ "theme": "dark", "language": "ar", "memory_enabled": false,
  "advisor_role": "صياغة موجزة ومباشرة", "improve_service": true }
// response 200 — يعيد الحالة الكاملة بعد التحديث
```

## 9. الذاكرة
### GET /v1/memories
```json
{ "items": [
  { "id": "uuid", "content": "يفضّل الأمثلة العملية", "source": "auto",
    "is_active": true, "created_at": "ISO" }
] }
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
> إن كانت `preferences.memory_enabled=false` لا تُحقن الذاكرة في التوليد (ARC-PROMPT-001).

## 10. المحادثات
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
                         "mime_type": "application/pdf", "size_bytes": 12345,
                         "status": "ready" } ],
      "created_at": "ISO" },
    { "id": "uuid", "role": "assistant", "content": "string", "status": "complete",
      "is_grounded": true,
      "sources": [ { "law_name": "string", "article": "string",
                     "variant": "original|sanaa_amended",
                     "source_ref": "string", "snippet": "string", "score": 0.0 } ],
      "created_at": "ISO" }
  ] }
```
### PATCH /v1/conversations/{id}
```json
// تسمية/تثبيت/أرشفة — كل الحقول اختيارية
{ "title": "عنوان جديد", "pinned": true, "archived": true }
// response 200 — archived=true يضبط archived_at=now()، وfalse يصفّره
{ "id": "uuid", "title": "عنوان جديد", "pinned": true, "archived_at": "ISO" }
```
### DELETE /v1/conversations/{id}
الاستجابة 204. حذف متتالٍ للرسائل والمصادر والتغذية الراجعة والمرفقات.

## 11. طرح سؤال + بثّ SSE (نواة النظام)
### POST /v1/conversations/{id}/messages
```http
Headers:
  Authorization: Bearer <access_token>
  Accept: text/event-stream
```
```json
// request body — attachment_ids اختيارية (مرفقات جاهزة سبق رفعها)
{ "content": "نص سؤال المستخدم بالعربية", "attachment_ids": ["uuid"] }
```
الاستجابة `200 text/event-stream`. تُنشأ رسالة المستخدم ثم رسالة المساعد بحالة
`streaming`، وعند الاكتمال `complete` (أو `stopped`/`error`). عند تجاوز سقف
التزامن يردّ `429` مع `Retry-After`، أو يُصدر حدث `status` بحالة `queued`.

### POST /v1/messages/{id}/stop
يوقف بثّاً جارياً؛ يضبط `messages.status='stopped'`. الاستجابة `200 { "stopped": true }`.

## 12. مخطّط أحداث SSE
كل حدثٍ سطر `event:` يتبعه `data:` (JSON):
```text
event: status
data: {"node":"analyze","label":"جارٍ تحليل السؤال..."}
event: status
data: {"node":"retrieve","label":"جارٍ البحث في النصوص القانونية..."}
event: status
data: {"node":"rerank","label":"جارٍ ترتيب أهم المصادر..."}
event: token
data: {"delta":"جزء من الإجابة"}
event: sources
data: {"sources":[
  {"law_name":"...","article":"...","variant":"original",
   "source_ref":"...","snippet":"...","score":0.0}]}
event: done
data: {"message_id":"uuid","is_grounded":true,"status":"complete"}
event: error
data: {"code":"internal_error","message":"..."}
```

| الحدث | الدلالة | ربط الحالة |
| --- | --- | --- |
| status | حالة تنفيذٍ مرتبطة باسم عقدة LangGraph | messages.status='streaming' |
| token | جزء من الإجابة (بثّ تدريجي) | streaming |
| sources | محتوى خانة المصادر (يُحفظ في message_sources) | — |
| done | اكتمال + علم الإسناد + الحالة النهائية | complete |
| error | خطأ أثناء البثّ | error |

> عند إيقاف المستخدم للبثّ تُغلق الواجهة الاتصال وتستدعي `/stop` (status='stopped').
> إن كان `is_grounded=false` تعرض الواجهة وسماً أحمر «غير مسنَد».

## 13. التغذية الراجعة
### POST /v1/messages/{id}/feedback
```json
// request — issue_category/safety_subcategory عند rating=down
{ "rating": "down", "issue_category": "دقة",
  "safety_subcategory": null, "conversation_included": true,
  "comment": "string|null" }
// response 201
{ "id": "uuid", "message_id": "uuid", "rating": "down", "created_at": "ISO" }
```
> `model_version` و`retrieval_path` يضبطهما الخادم تلقائياً لحلقة التقييم (لا يرسلهما العميل).

## 14. المرفقات (MinIO)
```json
// POST /v1/attachments/upload-url
// request
{ "file_name": "عقد.pdf", "mime_type": "application/pdf", "size_bytes": 240000 }
// response 200
{ "attachment_id": "uuid", "storage_key": "attachments/uuid/....pdf",
  "upload_url": "https://minio/...signed-PUT", "expires_in": 300 }
// PATCH /v1/attachments/{id}  { "status": "ready" } -> 200 (بعد اكتمال الرفع)
// GET  /v1/attachments/{id}/download-url -> 200 { "url": "...signed-GET", "expires_in": 300 }
```
- الحدّ الأقصى **5 مرفقات** للرسالة؛ يتحقّق الخادم من MIME والحجم.
- المرفق يُنشأ بحالة `preparing`، ثم `ready`، وقد يصبح `error`.
- يُربط المرفق بالرسالة عبر `attachment_ids` في مسار إنشاء الرسالة؛ قبلها `message_id=NULL`.

## 15. الصحّة
### GET /v1/health
```json
{ "status": "ok", "model": "up", "db": "up", "vector_index": "up",
  "object_store": "up", "sms_gateway": "up" }
```

## 16. اعتبارات أمنية
- كل نقلٍ حصراً عبر TLS (SEC-CRYPTO-001 §2).
- رمز الوصول JWT قصير العمر؛ رمز التجديد مبهمٌ ومخزّنٌ مجزّأً وقابلٌ للإبطال (شاشة الجلسات).
- كلمات المرور بـ Argon2id؛ الهاتف/البريد مشفّرة AES-256-GCM + بصمة HMAC للبحث.
- **2FA عبر SMS اختياريّ**؛ البصمة/الـ PIN محلية على الجهاز ولا تمرّ بالخادم.
- تحديد المعدّل على مسارات المصادقة وOTP لكل هاتف/بريد ولكل IP، مع قفلٍ مؤقّت (`423`).
- ردودٌ موحّدة على مسارات OTP/إعادة التعيين لتفادي كشف الأرقام/العناوين المسجّلة.
- روابط MinIO موقّعةٌ وقصيرة العمر؛ لا وصول مباشر للكائنات.
- لا تُسجَّل أجسام الطلب/الرسائل/كلمات المرور/الرموز (PII وأسرار) في اللوقات.
- التحقّق من ملكية المورد قبل أي قراءة/حذف (403/404)، مدعوماً بـ RLS.
- حجب الدخول عند وجود إصدار سياسةٍ جديد حتى إعادة القبول (`policy_reconsent_required`).

## 17. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | نموذج المصادقة (هاتف+كلمة مرور+OTP+2FA SMS) | **مُعتمد** |
| 2 | ترقيم بالصفحات: cursor مقابل offset | مبدئياً cursor زمني |
| 3 | دلالة queued مقابل 429 الصريح | تُحسم عند بناء طبقة الطابور |
| 4 | مزوّد رسائل OTP (SMS) الفعلي | خلف feature flag؛ الأوائل مرشّحاً |
| 5 | مصادقة TOTP كطبقةٍ إضافية | مؤجّلة؛ 2FA الحالي عبر SMS |
| 6 | استخراج نصّ المرفقات لإدماجه في السياق | خارج نطاق MVP (الاسترجاع على قاعدة المعرفة فقط) |

## 18. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لعقد REST + SSE (نموذج الضيف) |
| 0.2 | 2026-07-02 | اعتماد المصادقة الكاملة (هاتف+كلمة مرور+OTP): مسارات `/v1/auth/*`، رموز Bearer، أخطاء 401/409/422/423، ربط SEC-CRYPTO-001 |
| 0.3 | 2026-07-04 | مواءمةٌ مع ARC-DATA-001 v0.4: بريدٌ اختياري وتحقّقه، **2FA عبر SMS** (mfa_required + mfa_login)، شاشة الجلسات النشطة، رفع صورة الملف والمرفقات عبر MinIO presigned، التفضيلات، الذاكرة (CRUD)، تثبيت/أرشفة المحادثات، حالة الرسالة وإيقاف البثّ، إثراء التغذية الراجعة، السياسات والموافقة وإعادة القبول |

---
title: "عقد الواجهة البرمجية (REST + SSE) — المستشار القانوني الذكي"
doc_id: ARC-API-001
version: 0.2
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-07-02
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "OWASP API Security Top 10", status: محاذاة }
related:
  - ARC-DESIGN-001  # المعمارية
  - ARC-DATA-001    # نموذج البيانات
  - ARC-RAG-001     # خط أنابيب الاسترجاع (مصدر أحداث SSE)
  - SEC-CRYPTO-001  # معيار التشفير (كلمة المرور، الهاتف، الرموز)
---

# عقد الواجهة البرمجية (REST + SSE)

## 1. الغرض والنطاق
يعرّف هذا العقد نقاط النهاية بين الـ backend (FastAPI) وواجهة Flutter:
المسارات، الترويسات، أجسام الطلب/الاستجابة، أحداث البثّ (SSE)، ونموذج
الأخطاء. هو المرجع الملزِم للطرفين في النسخة الأولى.

## 2. المبادئ
- بادئة الإصدار `/v1` لكل المسارات.
- الجسم JSON (`Content-Type: application/json`) عدا بثّ SSE.
- الأوقات بصيغة ISO-8601 (UTC).
- المعرّفات UUID.
- أكواد HTTP قياسية + نموذج خطأ موحّد (القسم 4).

## 3. المصادقة (هاتف + كلمة مرور + OTP)
النموذج المعتمد للنسخة الأولى:

- التسجيل والدخول عبر **رقم الهاتف + كلمة المرور**.
- **OTP** للتحقق من الهاتف عند التسجيل ولإعادة تعيين كلمة المرور.
- بعد الدخول يُصدر الخادم **رمز وصول (access token — JWT قصير العمر)**
  و**رمز تجديد (refresh token — قيمة مبهمة قابلة للإبطال)**.
- تُرسل الواجهة رمز الوصول في كل طلبٍ محمي:
  `Authorization: Bearer <access_token>`
- يستخرج الـ backend `user_id` من الرمز ويضبط `app.current_user_id`
  لتفعيل RLS (انظر ARC-DATA-001 §6).

### حماية بيانات الاعتماد (يحيل إلى SEC-CRYPTO-001)
- **كلمة المرور:** تُخزَّن مجزّأة بـ **Argon2id** — لا تُخزَّن ولا تُسجَّل نصّاً صريحاً أبداً.
- **الهاتف:** يُخزَّن مشفّراً `AES-256-GCM` (`phone_enc`) للعرض، مع بصمة
  حتمية `HMAC-SHA-256` (`phone_hash`) للبحث/الفرادة دون نصٍّ صريح.
- **رمز التجديد:** يُخزَّن مجزّأً (`SHA-256`) في `auth_sessions` وقابلٌ للإبطال.
- **رمز OTP:** يُخزَّن مجزّأً في `otp_codes`، قصير العمر، بحدّ محاولاتٍ ومهلة.

### أعمار الرموز (مبدئي)
| الرمز | العمر | ملاحظة |
| --- | --- | --- |
| access (JWT) | 30 دقيقة | يُجدَّد عبر refresh |
| refresh | 30 يوماً | مخزّنٌ مجزّأً، يُبطَل عند الخروج |
| OTP | 5 دقائق | 5 محاولات كحدٍّ أقصى ثم قفل مؤقّت |

> ملاحظة تشغيلية: مزوّد الرسائل (OTP SMS) يُضبط خلف علمٍ (feature flag)؛
> في بيئة التطوير يُفعَّل مسار وهمي (mock) يعيد الرمز في السجل الآمن فقط.

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
| 404 | not_found | المورد غير موجود |
| 409 | conflict | رقم الهاتف مسجّلٌ مسبقاً |
| 422 | otp_invalid | رمز OTP خاطئ أو منتهٍ أو استُهلك |
| 423 | account_locked | تجاوز حدّ محاولات الدخول/OTP |
| 429 | service_busy | تجاوز سقف التزامن؛ أعد المحاولة بعد Retry-After |
| 500 | internal_error | خطأ غير متوقّع |

## 5. خريطة نقاط النهاية
| الطريقة | المسار | الغرض | محمي؟ |
| --- | --- | --- | --- |
| POST | /v1/auth/register | تسجيل حساب جديد (اسم+هاتف+كلمة مرور) ويطلق OTP | لا |
| POST | /v1/auth/otp/request | إرسال/إعادة إرسال رمز OTP | لا |
| POST | /v1/auth/otp/verify | التحقق من الهاتف بالـ OTP وإصدار الرموز | لا |
| POST | /v1/auth/login | الدخول بهاتف+كلمة مرور وإصدار الرموز | لا |
| POST | /v1/auth/refresh | تجديد رمز الوصول | لا (refresh) |
| POST | /v1/auth/logout | إبطال رمز التجديد | نعم |
| POST | /v1/auth/password/reset-request | طلب رمز لإعادة تعيين كلمة المرور | لا |
| POST | /v1/auth/password/reset | تعيين كلمة مرور جديدة برمز OTP | لا |
| GET | /v1/users/me | قراءة الملف | نعم |
| PATCH | /v1/users/me | تحديث النوع/المهنة/المحافظة | نعم |
| DELETE | /v1/users/me | حذف الحساب (PDPL، حذف متتالٍ) | نعم |
| GET | /v1/conversations | قائمة المحادثات | نعم |
| POST | /v1/conversations | إنشاء محادثة | نعم |
| GET | /v1/conversations/{id} | محادثة + رسائلها | نعم |
| DELETE | /v1/conversations/{id} | حذف محادثة | نعم |
| POST | /v1/conversations/{id}/messages | طرح سؤال (يردّ ببثّ SSE) | نعم |
| POST | /v1/messages/{id}/feedback | تغذية راجعة على رسالة | نعم |
| GET | /v1/health | فحص الصحّة | لا |

## 6. المصادقة والحساب
### POST /v1/auth/register
```json
// request
{ "full_name": "عبدالله محمد", "phone": "777000000", "password": "********" }
// response 201
{ "user_id": "uuid", "phone_verified": false, "verification_required": true }
```
إن كان الرقم مسجّلاً مسبقاً يردّ `409 conflict`.

### POST /v1/auth/otp/request
```json
// request
{ "phone": "777000000", "purpose": "verify_phone" }  // أو "reset_password"
// response 200
{ "sent": true, "expires_in": 300, "resend_after": 60 }
```

### POST /v1/auth/otp/verify
```json
// request
{ "phone": "777000000", "code": "123456", "purpose": "verify_phone" }
// response 200 — عند verify_phone تُصدر الرموز مباشرة
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "user_type": "citizen", "phone_verified": true } }
```
رمزٌ خاطئ/منتهٍ يردّ `422 otp_invalid`؛ تجاوز الحدّ يردّ `423 account_locked`.

### POST /v1/auth/login
```json
// request
{ "phone": "777000000", "password": "********" }
// response 200
{ "access_token": "jwt", "refresh_token": "opaque", "token_type": "Bearer",
  "expires_in": 1800,
  "user": { "user_id": "uuid", "user_type": "citizen", "full_name": "عبدالله محمد",
            "phone_verified": true } }
```

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
// response 204 (بلا جسم) — يُبطل رمز التجديد في auth_sessions
```

### POST /v1/auth/password/reset-request
```json
// request
{ "phone": "777000000" }
// response 200 — ردٌّ موحّد دائماً لتفادي كشف الأرقام المسجّلة
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

### GET /v1/users/me
```json
{ "user_id": "uuid", "full_name": "عبدالله محمد", "user_type": "citizen",
  "profession": null, "governorate": null, "phone_verified": true }
```

### PATCH /v1/users/me
```json
// request
{ "user_type": "lawyer", "profession": "attorney", "governorate": "Sanaa" }
// response 200
{ "user_id": "uuid", "user_type": "lawyer", "profession": "attorney",
  "governorate": "Sanaa" }
```

### DELETE /v1/users/me
الاستجابة 204 (بلا جسم). يحذف المستخدم وكل محادثاته ورسائله وجلساته ورموزه متتالياً.

## 7. المحادثات
### GET /v1/conversations?limit=20&before=<ISO-8601>
```json
{ "items": [ { "id": "uuid", "title": "string|null",
               "created_at": "ISO", "updated_at": "ISO" } ],
  "next_before": "ISO|null" }
```

### POST /v1/conversations
```json
// request
{ "title": "string|null" }
// response 201
{ "id": "uuid", "title": "string|null", "created_at": "ISO", "updated_at": "ISO" }
```

### GET /v1/conversations/{id}
```json
{ "id": "uuid", "title": "string|null",
  "messages": [
    { "id": "uuid", "role": "user", "content": "string", "created_at": "ISO" },
    { "id": "uuid", "role": "assistant", "content": "string",
      "is_grounded": true,
      "sources": [
        { "law_name": "string", "article": "string",
          "variant": "original|sanaa_amended",
          "source_ref": "string", "snippet": "string", "score": 0.0 }
      ],
      "created_at": "ISO" }
  ] }
```

### DELETE /v1/conversations/{id}
الاستجابة 204. حذف متتالٍ للرسائل والمصادر والتغذية الراجعة.

## 8. طرح سؤال + بثّ SSE (نواة النظام)
### POST /v1/conversations/{id}/messages
```http
Headers:
  Authorization: Bearer <access_token>
  Accept: text/event-stream
```
```json
// request body
{ "content": "نص سؤال المستخدم بالعربية" }
```
الاستجابة: `200 text/event-stream`. تيار الأحداث (انظر القسم 9). عند تجاوز
سقف التزامن يردّ `429` مع `Retry-After`، أو يُصدر حدث `status` بحالة
`queued` ثم يُكمل عند توفّر السعة.

## 9. مخطّط أحداث SSE
كل حدثٍ سطر `event:` يتبعه `data:` (JSON). الأنواع:
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
data: {"message_id":"uuid","is_grounded":true}
event: error
data: {"code":"internal_error","message":"..."}
```

| الحدث | الدلالة |
| --- | --- |
| status | حالة تنفيذٍ مرتبطة باسم عقدة LangGraph الفعلي |
| token | جزء من الإجابة (بثّ تدريجي) |
| sources | محتوى خانة المصادر (يُحفظ في message_sources) |
| done | اكتمال + علم الإسناد is_grounded |
| error | خطأ أثناء البثّ |

> ملاحظة عرض: إن كان `is_grounded=false` تعرض الواجهة وسماً أحمر «غير مسنَد».

## 10. التغذية الراجعة
### POST /v1/messages/{id}/feedback
```json
// request
{ "rating": "up|down", "comment": "string|null" }
// response 201
{ "id": "uuid", "message_id": "uuid", "rating": "down", "created_at": "ISO" }
```

## 11. الصحّة
### GET /v1/health
```json
{ "status": "ok", "model": "up", "db": "up", "vector_index": "up" }
```

## 12. اعتبارات أمنية
- كل نقلٍ حصراً عبر TLS.
- رمز الوصول JWT قصير العمر؛ رمز التجديد مبهمٌ ومخزّنٌ مجزّأً وقابلٌ للإبطال.
- كلمات المرور بـ Argon2id؛ الهاتف مشفّرٌ AES-256-GCM + بصمة HMAC للبحث (SEC-CRYPTO-001).
- تحديد المعدّل (rate limiting) على مسارات المصادقة و OTP لكل هاتف ولكل IP،
  مع حدّ محاولاتٍ وقفلٍ مؤقّت (`423`).
- ردودٌ موحّدة على مسارات OTP/إعادة التعيين لتفادي كشف الأرقام المسجّلة.
- لا تُسجَّل أجسام الطلب/الرسائل/كلمات المرور/الرموز (PII وأسرار) في اللوقات.
- التحقّق من ملكية المورد قبل أي قراءة/حذف (يردّ 403/404).

## 13. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | نموذج المصادقة (هاتف+كلمة مرور+OTP) | **مُعتمد** (v0.2) بدل نموذج الضيف |
| 2 | ترقيم بالصفحات: cursor مقابل offset | مبدئياً cursor زمني |
| 3 | دلالة queued مقابل 429 الصريح | تُحسم عند بناء طبقة الطابور |
| 4 | مزوّد رسائل OTP (SMS) الفعلي | خلف feature flag؛ mock في التطوير |

## 14. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لعقد REST + SSE (نموذج الضيف) |
| 0.2 | 2026-07-02 | اعتماد المصادقة الكاملة (هاتف+كلمة مرور+OTP): مسارات `/v1/auth/*`، رموز Bearer، أخطاء 401/409/422/423، ربط SEC-CRYPTO-001 |

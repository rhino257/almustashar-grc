---
title: "عقد الواجهة البرمجية (REST + SSE) — المستشار القانوني الذكي"
doc_id: ARC-API-001
version: 0.1
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-06-24
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "OWASP API Security Top 10", status: محاذاة }
related:
  - ARC-DESIGN-001  # المعمارية
  - ARC-DATA-001    # نموذج البيانات
  - ARC-RAG-001     # خط أنابيب الاسترجاع (مصدر أحداث SSE)
---

# عقد الواجهة البرمجية (REST + SSE)

## 1. الغرض والنطاق
يعرّف هذا العقد نقاط النهاية بين الـ backend (FastAPI) وواجهة Flutter:
المسارات، الترويسات، أجسام الطلب/الاستجابة، أحداث البثّ (SSE)، ونموذج
الأخطاء. هو المرجع الملزِم للطرفين في النسخة الأولى.

## 2. المبادئ
- بادئة الإصدار `'/v1'` لكل المسارات.
- الجسم JSON (`Content-Type: application/json`) عدا بثّ SSE.
- الأوقات بصيغة ISO-8601 (UTC).
- المعرّفات UUID.
- أكواد HTTP قياسية + نموذج خطأ موحّد (القسم 4).

## 3. المصادقة (نموذج الضيف — MVP)
- يُصدر الخادم `user_id` (UUID) عند أول اتصال، وتُرسله الواجهة في كل طلب:
  `X-User-Id: <uuid>`.
- يضبط الـ backend منه `app.current_user_id` لتفعيل RLS.

> تنبيه أمني (دَين موثّق): الـ UUID هنا **سرٌّ حامل** — حائزه يصل للحساب.
> مقبولٌ لبيانات الضيف الدنيا. مسار الترقية: استبداله برمزٍ موقّع
> (signed token) مرتبطٍ بجلسة OTP عند تفعيل المصادقة، دون كسر هذا العقد.

## 4. نموذج الأخطاء الموحّد
​
{
"error": {
"code": "validation_error",
"message": "human-readable message",
"details": {}
}
}

| كود HTTP | code | الدلالة |
| --- | --- | --- |
| 400 | validation_error | جسم/معامل غير صالح |
| 401 | missing_identity | ترويسة X-User-Id مفقودة/غير صالحة |
| 403 | forbidden | محاولة وصولٍ لمورد مستخدمٍ آخر |
| 404 | not_found | المورد غير موجود |
| 429 | service_busy | تجاوز سقف التزامن؛ أعد المحاولة بعد Retry-After |
| 500 | internal_error | خطأ غير متوقّع |

## 5. خريطة نقاط النهاية
| الطريقة | المسار | الغرض |
| --- | --- | --- |
| POST | /v1/users | إنشاء مستخدم ضيف وإصدار UUID |
| GET | /v1/users/me | قراءة الملف |
| PATCH | /v1/users/me | تحديث النوع/المهنة/المحافظة |
| DELETE | /v1/users/me | حذف الحساب (PDPL، حذف متتالٍ) |
| GET | /v1/conversations | قائمة المحادثات |
| POST | /v1/conversations | إنشاء محادثة |
| GET | /v1/conversations/{id} | محادثة + رسائلها |
| DELETE | /v1/conversations/{id} | حذف محادثة |
| POST | /v1/conversations/{id}/messages | طرح سؤال (يردّ ببثّ SSE) |
| POST | /v1/messages/{id}/feedback | تغذية راجعة على رسالة |
| GET | /v1/health | فحص الصحّة |

## 6. المستخدمون
### POST /v1/users
الطلب: لا جسم. الاستجابة 201:
​
{ "user_id": "uuid", "user_type": "citizen", "created_at": "ISO-8601" }

### PATCH /v1/users/me
​
// request
{ "user_type": "lawyer", "profession": "attorney", "governorate": "Sanaa" }
// response 200
{ "user_id": "uuid", "user_type": "lawyer", "profession": "attorney",
"governorate": "Sanaa" }

### DELETE /v1/users/me
الاستجابة 204 (بلا جسم). يحذف المستخدم وكل محادثاته ورسائله متتالياً.

## 7. المحادثات
### GET /v1/conversations?limit=20&before=<ISO-8601>
​
{ "items": [ { "id": "uuid", "title": "string|null",
"created_at": "ISO", "updated_at": "ISO" } ],
"next_before": "ISO|null" }

### POST /v1/conversations
​
// request
{ "title": "string|null" }
// response 201
{ "id": "uuid", "title": "string|null", "created_at": "ISO", "updated_at": "ISO" }

### GET /v1/conversations/{id}
​
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

### DELETE /v1/conversations/{id}
الاستجابة 204. حذف متتالٍ للرسائل والمصادر والتغذية الراجعة.

## 8. طرح سؤال + بثّ SSE (نواة النظام)
### POST /v1/conversations/{id}/messages
​
Headers:
X-User-Id: <uuid>
Accept: text/event-stream
​
// request body
{ "content": "نص سؤال المستخدم بالعربية" }
الاستجابة: `200 text/event-stream`. تيار الأحداث (انظر القسم 9). عند تجاوز
سقف التزامن يردّ `429` مع `Retry-After`، أو يُصدر حدث `status` بحالة
`queued` ثم يُكمل عند توفّر السعة.

## 9. مخطّط أحداث SSE
كل حدثٍ سطر `event:` يتبعه `data:` (JSON). الأنواع:
​
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
​
// request
{ "rating": "up|down", "comment": "string|null" }
// response 201
{ "id": "uuid", "message_id": "uuid", "rating": "down", "created_at": "ISO" }

## 11. الصحّة
### GET /v1/health
​
{ "status": "ok", "model": "up", "db": "up", "vector_index": "up" }

## 12. اعتبارات أمنية
- X-User-Id سرٌّ حامل (انظر §3) — يُنقل حصراً عبر TLS.
- تحديد المعدّل (rate limiting) لكل user_id ولكل IP.
- لا تُسجَّل أجسام الطلب/الرسائل (PII) في اللوقات.
- التحقّق من ملكية المورد قبل أي قراءة/حذف (يردّ 403/404).

## 13. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | استبدال X-User-Id برمزٍ موقّع (OTP) | مؤجّل لتفعيل المصادقة |
| 2 | ترقيم بالصفحات: cursor مقابل offset | مبدئياً cursor زمني |
| 3 | دلالة queued مقابل 429 الصريح | تُحسم عند بناء طبقة الطابور |

## 14. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لعقد REST + SSE |

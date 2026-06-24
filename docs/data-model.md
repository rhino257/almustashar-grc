---
title: "نموذج البيانات — المستشار القانوني الذكي"
doc_id: ARC-DATA-001
version: 0.2
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-06-24
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "ISO/IEC 27701", status: محاذاة }
  - { name: NCS-1, status: مرجعي }
related:
  - ARC-DESIGN-001  # المعمارية التقنية
  - ARC-RAG-001     # خط أنابيب الاسترجاع (يفرض فهارس البحث اللفظي والتصنيف)
  - GOV-ADR-001     # سجل القرارات التقنية
---

# نموذج البيانات — المستشار القانوني الذكي

## 1. الغرض والنطاق
تصف هذه الوثيقة مخطّط قاعدة البيانات (PostgreSQL + pgvector) للنسخة الأولى:
الجداول، الأعمدة، العلاقات، الفهارس، عزل الصفوف (RLS)، الحذف، وتصنيف
البيانات. الهدف: مخطّطٌ يكفي لإطلاق سؤال/جواب مسنَد بذاكرةٍ داخل المحادثة،
ملتزمٌ بالحد الأدنى من البيانات (PDPL)، ومهيّأٌ لخط أنابيب الاسترجاع الهجين.

## 2. المبادئ الحاكمة للمخطّط
- **نطاقان منفصلان:** (أ) بيانات التطبيق ينشئها المستخدم، (ب) قاعدة المعرفة
  القانونية. الأولى نصمّمها كاملةً، والثانية نُحاذي ما هو موجودٌ لديك.
- **الحد الأدنى من البيانات:** لا هوية وطنية؛ فقط ما يلزم للخدمة.
- **العزل الافتراضي:** RLS صارمٌ يمنع تسرّب بياناتٍ بين المستخدمين.
- **الحذف حقٌّ أصيل:** `ON DELETE CASCADE` يجعل حذف الحساب يمحو كل أثره.
- **استراتيجية لغة التضمين:** يُخزَّن النص العربي للعرض، والإنجليزي للتضمين
  والبحث اللفظي الإنجليزي، مع بحثٍ لفظيٍّ عربيٍّ مساعد.

## 3. مخطّط الكيانات والعلاقات (ERD)
​
users 1--- conversations 1--- messages 1---* message_sources
*---1 feedback (per message)
legal_documents 1--- legal_articles 1--- legal_chunks
^
+--+ variant_of (self-reference: amended <-> original)
message_sources *---1 legal_articles   (citation link, when available)

## 4. نطاق (أ): جداول التطبيق
​
-- enable extensions before running this script:
--   CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()
--   CREATE EXTENSION IF NOT EXISTS vector;     -- pgvector
--   CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- trigram (Arabic lexical)
CREATE TABLE users (
id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
phone_enc    TEXT,        -- reversible-encrypted via CryptoService; NULL until OTP active
user_type    TEXT NOT NULL DEFAULT 'citizen',  -- citizen|lawyer|judge|researcher|bank
profession   TEXT,
governorate  TEXT,
created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE conversations (
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
title       TEXT,
created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_user ON conversations(user_id);
CREATE TABLE messages (
id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
conversation_id  UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
role             TEXT NOT NULL CHECK (role IN ('user','assistant')),
content          TEXT NOT NULL,
is_grounded      BOOLEAN,   -- NULL for user msgs; TRUE=sourced, FALSE=model-belief
created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE TABLE message_sources (
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
message_id  UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
article_id  UUID REFERENCES legal_articles(id),  -- link to KB when available
law_name    TEXT,
article     TEXT,
variant     TEXT,          -- original|sanaa_amended
source_ref  TEXT,
snippet     TEXT,
score       REAL,
rank        INT
);
CREATE INDEX idx_message_sources_message ON message_sources(message_id);
CREATE TABLE feedback (
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
message_id  UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
rating      TEXT NOT NULL CHECK (rating IN ('up','down')),
comment     TEXT,
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

| الجدول | الغرض | ملاحظة |
| --- | --- | --- |
| `users` | الحد الأدنى من بيانات المستخدم | `phone_enc` مشفّرٌ قابلٌ للفكّ؛ لا هوية وطنية |
| `conversations` | الجلسات المحفوظة على الـ UUID | حذفٌ متتالٍ عند حذف المستخدم |
| `messages` | رسائل المحادثة + علم الإسناد | `is_grounded` سجلّ الإجابات غير المسنَدة |
| `message_sources` | خانة «المصادر» تحت كل رسالة | `variant` و`article_id` يربطان بقاعدة المعرفة |
| `feedback` | تغذية المجموعة الذهبية | مرتبطٌ بالرسالة لا بالمستخدم (تقليل الربط) |

## 5. نطاق (ب): قاعدة المعرفة القانونية
​
CREATE TABLE legal_documents (
id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
doc_type     TEXT,        -- law|regulation|sharia_ref|ruling_principle
title        TEXT NOT NULL,
source_ref   TEXT,
issued_date  DATE
);
CREATE TABLE legal_articles (
id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
document_id  UUID NOT NULL REFERENCES legal_documents(id),
article_no   TEXT,
content_ar   TEXT NOT NULL,   -- original Arabic text (returned to the user)
content_en   TEXT,            -- English translation (embedding + EN lexical)
variant      TEXT,            -- original|sanaa_amended
variant_of   UUID REFERENCES legal_articles(id),  -- links amended <-> original
categories   TEXT[] DEFAULT '{}',  -- legal taxonomy; CAPTURED, not filtered in MVP
-- generated full-text search vectors (lexical retrieval L3/L4):
content_en_tsv tsvector
GENERATED ALWAYS AS (to_tsvector('english', coalesce(content_en,''))) STORED,
content_ar_tsv tsvector
GENERATED ALWAYS AS (to_tsvector('simple',  coalesce(content_ar,''))) STORED
);
CREATE INDEX idx_legal_articles_document  ON legal_articles(document_id);
CREATE INDEX idx_legal_articles_variantof ON legal_articles(variant_of);
-- lexical search indexes:
CREATE INDEX idx_articles_en_tsv  ON legal_articles USING GIN (content_en_tsv);
CREATE INDEX idx_articles_ar_tsv  ON legal_articles USING GIN (content_ar_tsv);
-- Arabic has no native PG stemmer; trigram index supports fuzzy/substring match:
CREATE INDEX idx_articles_ar_trgm ON legal_articles USING GIN (content_ar gin_trgm_ops);
-- category lookup (for future soft-boost, not MVP filtering):
CREATE INDEX idx_articles_categories ON legal_articles USING GIN (categories);
CREATE TABLE legal_chunks (
id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
article_id   UUID NOT NULL REFERENCES legal_articles(id) ON DELETE CASCADE,
chunk_index  INT NOT NULL,
content_en   TEXT NOT NULL,   -- English chunk that gets embedded
embedding    vector(1024),    -- DIMENSION PENDING final embedding-model decision
token_count  INT
);
-- ANN index, create AFTER the embedding model/dimension is fixed:
-- CREATE INDEX idx_legal_chunks_embedding
--   ON legal_chunks USING hnsw (embedding vector_cosine_ops);

| الجدول | الغرض | ملاحظة |
| --- | --- | --- |
| `legal_documents` | القانون/اللائحة/المرجع | الكيان الأعلى |
| `legal_articles` | المواد بنسختيها | `content_ar` للعرض، `content_en` للتضمين؛ `variant_of` يقرن المعدَّل بالأصلي؛ عمودا `tsv` للبحث اللفظي |
| `legal_chunks` | المقاطع المُضمَّنة | التضمين على الإنجليزية؛ الاسترجاع يُرجع العربية عبر `article_id` |

**ملاحظة على البحث اللفظي:** الدمج (RRF) يجري على **مستوى المادة** —
القائمتان المتجهيتان (L1/L2) تُرجعان مقاطع تُربط بمادّتها عبر `article_id`،
والقائمتان اللفظيتان (L3/L4) تُرجعان مواداً مباشرة.

**ملاحظة على العربية:** لا يملك Postgres مُجزّئاً (stemmer) عربياً أصيلاً،
لذا نستخدم إعداد `simple` لـ tsvector + فهرس `pg_trgm` للتطابق التقريبي.
قد ننتقل لاحقاً إلى إضافةٍ متخصّصة إن لزم تحسين الاستدعاء العربي.

## 6. عزل الصفوف (Row-Level Security)
​
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages       ENABLE ROW LEVEL SECURITY;
CREATE POLICY conv_isolation ON conversations
USING (user_id = current_setting('app.current_user_id')::uuid);
CREATE POLICY msg_isolation ON messages
USING (conversation_id IN (
SELECT id FROM conversations
WHERE user_id = current_setting('app.current_user_id')::uuid));
يضبط الـ backend المتغيّر `app.current_user_id` لكل طلبٍ من الـ UUID الموثَّق،
فلا يصل أيّ مستخدمٍ إلا لصفوفه.

## 7. التشفير وحماية الحقول
- `users.phone_enc`: يمرّ عبر `CryptoService` (الهدف: AES-256 قابل للفكّ؛
  مبدئياً `NullCipher`). العمود `TEXT` بحجمٍ يكفي النصّ المشفّر، فلا يلزم
  تعديل المخطّط عند تفعيل التشفير لاحقاً.
- رمز OTP (عند تفعيله): يُخزَّن **مجزّأً (hash)** في جدولٍ منفصلٍ قصير العمر.
- لا تُسجَّل بيانات شخصية في اللوقات.

## 8. الحذف والاحتفاظ (PDPL)
- تُحفظ المحادثات حتى يحذفها المستخدم أو يُغلق حسابه.
- حذف الحساب = حذف صفّ `users`؛ والـ CASCADE يمحو تلقائياً المحادثات
  والرسائل والمصادر والتغذية الراجعة.
- لا نستخدم الحذف الناعم للبيانات الشخصية كي لا نخالف مبدأ الحذف الفعلي.

## 9. تصنيف البيانات
| الجدول/الحقل | التصنيف |
| --- | --- |
| `users.phone_enc` | سرّي |
| `users` (نوع/مهنة/محافظة) | داخلي |
| `conversations.title` | سرّي |
| `messages.content` | سرّي |
| `message_sources` (نصوص قانونية) | عام (الربط داخلي) |
| `feedback` | داخلي |
| `legal_*` (النصوص القانونية) | عام |

## 10. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | بُعد `vector(n)` | مثبّت مبدئياً 1024؛ يُحسم مع نموذج التضمين |
| 2 | جدول `otp_codes` | يُضاف عند تفعيل ميزة OTP |
| 3 | فهرس HNSW مقابل IVFFlat | يُحسم بعد قياس حجم القاعدة والأداء |
| 4 | تطبيع `categories` إلى جدول taxonomy | مؤجّل؛ TEXT[] يكفي الآن |
| 5 | إعداد بحثٍ عربيٍّ متخصّص (إضافة) | يُعاد النظر إن ضعُف الاستدعاء العربي |

## 11. سجل التغييرات
| الإصدار | التاريخ | الوصف | المعتمِد |
| --- | --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لنموذج البيانات (تطبيق + معرفة) | — (Draft) |
| 0.2 | 2026-06-24 | إضافة فهارس البحث اللفظي (tsvector + pg_trgm) وعمود `categories`؛ ربط ARC-RAG-001 | — (Draft) |
​
أصبح نموذج البيانات الآن متّسقاً تماماً مع خطّ الأنابيب: القائمتان اللفظيتان (L3/L4) لهما فهارسهما، والتصنيف مهيّأٌ للتعزيز الناعم مستقبلاً. اكتملت من الطبقة الأولى: architecture.md وdata-model.md (v0.2) وrag-pipeline-spec.md.

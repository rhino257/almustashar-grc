---
title: "نموذج البيانات — المستشار القانوني الذكي"
doc_id: ARC-DATA-001
version: 0.3
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-07-02
next_review: 2026-12-24
frameworks:
  - { name: PDPL, status: محاذاة }
  - { name: "ISO/IEC 27701", status: محاذاة }
  - { name: NCS-1, status: مرجعي }
related:
  - ARC-DESIGN-001  # المعمارية التقنية
  - ARC-API-001     # عقد الواجهة (مصادقة هاتف+كلمة مرور+OTP)
  - ARC-RAG-001     # خط أنابيب الاسترجاع (يفرض فهارس البحث اللفظي والتصنيف)
  - SEC-CRYPTO-001  # معيار التشفير (كلمة المرور، الهاتف، الرموز)
  - GOV-ADR-001     # سجل القرارات التقنية
---

# نموذج البيانات — المستشار القانوني الذكي

## 1. الغرض والنطاق
تصف هذه الوثيقة مخطّط قاعدة البيانات (PostgreSQL + pgvector) للنسخة الأولى:
الجداول، الأعمدة، العلاقات، الفهارس، عزل الصفوف (RLS)، الحذف، وتصنيف
البيانات. الهدف: مخطّطٌ يكفي لإطلاق سؤال/جواب مسنَد بذاكرةٍ داخل المحادثة،
مع مصادقةٍ بهاتف+كلمة مرور+OTP، ملتزمٌ بالحد الأدنى من البيانات (PDPL)،
ومهيّأٌ لخط أنابيب الاسترجاع الهجين.

## 2. المبادئ الحاكمة للمخطّط
- **نطاقان منفصلان:** (أ) بيانات التطبيق ينشئها المستخدم، (ب) قاعدة المعرفة
  القانونية. الأولى نصمّمها كاملةً، والثانية نُحاذي ما هو موجودٌ لديك.
- **الحد الأدنى من البيانات:** لا هوية وطنية؛ فقط ما يلزم للخدمة والمصادقة.
- **العزل الافتراضي:** RLS صارمٌ يمنع تسرّب بياناتٍ بين المستخدمين.
- **الحذف حقٌّ أصيل:** `ON DELETE CASCADE` يجعل حذف الحساب يمحو كل أثره.
- **بيانات الاعتماد مصونة:** كلمة المرور مجزّأة (Argon2id)، الهاتف مشفّرٌ
  (AES-256-GCM) مع بصمة HMAC للبحث، ورموز OTP/التجديد مجزّأة (انظر SEC-CRYPTO-001).
- **استراتيجية لغة التضمين:** يُخزَّن النص العربي للعرض، والإنجليزي للتضمين
  والبحث اللفظي الإنجليزي، مع بحثٍ لفظيٍّ عربيٍّ مساعد.

## 3. مخطّط الكيانات والعلاقات (ERD)
```text
users 1---* conversations 1---* messages 1---* message_sources
                                    *---1 feedback (per message)
users 1---* otp_codes        (phone verification / password reset)
users 1---* auth_sessions    (refresh tokens, revocable)
legal_documents 1---* legal_articles 1---* legal_chunks
                          ^
                          +--+ variant_of (self-reference: amended <-> original)
message_sources *---1 legal_articles   (citation link, when available)
```

## 4. نطاق (أ): جداول التطبيق
```sql
-- enable extensions before running this script:
--   CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()
--   CREATE EXTENSION IF NOT EXISTS vector;     -- pgvector
--   CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- trigram (Arabic lexical)
CREATE TABLE users (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name      TEXT,
  phone_enc      TEXT,          -- AES-256-GCM reversible (display) via CryptoService
  phone_hash     TEXT UNIQUE,   -- HMAC-SHA-256 deterministic lookup (login/uniqueness), no plaintext
  phone_verified BOOLEAN NOT NULL DEFAULT FALSE,
  password_hash  TEXT,          -- Argon2id (SEC-CRYPTO-001); never plaintext
  user_type      TEXT NOT NULL DEFAULT 'citizen',  -- citizen|lawyer|judge|researcher|bank
  profession     TEXT,
  governorate    TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE otp_codes (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_hash  TEXT NOT NULL,   -- deterministic lookup (no plaintext phone)
  user_id     UUID REFERENCES users(id) ON DELETE CASCADE,  -- NULL before the account exists
  code_hash   TEXT NOT NULL,   -- OTP stored hashed, never plaintext
  purpose     TEXT NOT NULL CHECK (purpose IN ('verify_phone','reset_password')),
  attempts    INT NOT NULL DEFAULT 0,
  expires_at  TIMESTAMPTZ NOT NULL,
  consumed_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_otp_phone_purpose ON otp_codes(phone_hash, purpose);
CREATE TABLE auth_sessions (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id            UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  refresh_token_hash TEXT NOT NULL,   -- SHA-256 of the opaque refresh token
  user_agent         TEXT,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at         TIMESTAMPTZ NOT NULL,
  revoked_at         TIMESTAMPTZ
);
CREATE INDEX idx_auth_sessions_user ON auth_sessions(user_id);
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
```

| الجدول | الغرض | ملاحظة |
| --- | --- | --- |
| `users` | بيانات المستخدم + بيانات الاعتماد | `password_hash` بـ Argon2id؛ `phone_enc` مشفّر و`phone_hash` بصمة بحث؛ لا هوية وطنية |
| `otp_codes` | رموز التحقق/إعادة التعيين | مجزّأة، قصيرة العمر، بحدّ محاولات؛ حذفٌ متتالٍ |
| `auth_sessions` | جلسات رمز التجديد | `refresh_token_hash` قابلٌ للإبطال؛ حذفٌ متتالٍ |
| `conversations` | الجلسات المحفوظة | حذفٌ متتالٍ عند حذف المستخدم |
| `messages` | رسائل المحادثة + علم الإسناد | `is_grounded` سجلّ الإجابات غير المسنَدة |
| `message_sources` | خانة «المصادر» تحت كل رسالة | `variant` و`article_id` يربطان بقاعدة المعرفة |
| `feedback` | تغذية المجموعة الذهبية | مرتبطٌ بالرسالة لا بالمستخدم (تقليل الربط) |

## 5. نطاق (ب): قاعدة المعرفة القانونية
```sql
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
```

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
```sql
ALTER TABLE conversations  ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages       ENABLE ROW LEVEL SECURITY;
CREATE POLICY conv_isolation ON conversations
  USING (user_id = current_setting('app.current_user_id')::uuid);
CREATE POLICY msg_isolation ON messages
  USING (conversation_id IN (
    SELECT id FROM conversations
    WHERE user_id = current_setting('app.current_user_id')::uuid));
```
يستخرج الـ backend `user_id` من رمز الوصول (Bearer JWT) ويضبط المتغيّر
`app.current_user_id` لكل طلب، فلا يصل أيّ مستخدمٍ إلا لصفوفه. جداول المصادقة
(`otp_codes`, `auth_sessions`) يُتعامل معها في طبقة الخدمة بمفتاح الخدمة
(service role) خارج مسار RLS للمستخدم.

## 7. التشفير وحماية الحقول
- `users.password_hash`: مجزّأ بـ **Argon2id** (SEC-CRYPTO-001) — لا يُخزَّن ولا يُسجَّل نصّاً صريحاً.
- `users.phone_enc`: يمرّ عبر `CryptoService` (AES-256-GCM قابل للفكّ؛ مبدئياً `NullCipher`).
  العمود `TEXT` بحجمٍ يكفي النصّ المشفّر، فلا يلزم تعديل المخطّط عند تفعيل التشفير لاحقاً.
- `users.phone_hash`: بصمة **HMAC-SHA-256** حتمية بمفتاحٍ سرّي — للبحث والفرادة دون كشف الرقم.
- `otp_codes.code_hash`: رمز OTP مجزّأ، قصير العمر، بحدّ محاولاتٍ ومهلة.
- `auth_sessions.refresh_token_hash`: بصمة **SHA-256** لرمز التجديد المبهم؛ يُبطَل عند الخروج/إعادة التعيين.
- لا تُسجَّل بيانات شخصية أو أسرار في اللوقات.

## 8. الحذف والاحتفاظ (PDPL)
- تُحفظ المحادثات حتى يحذفها المستخدم أو يُغلق حسابه.
- حذف الحساب = حذف صفّ `users`؛ والـ CASCADE يمحو تلقائياً المحادثات
  والرسائل والمصادر والتغذية الراجعة و`otp_codes` و`auth_sessions`.
- رموز OTP المنتهية/المستهلكة تُنظَّف دورياً (مهمة مجدولة).
- لا نستخدم الحذف الناعم للبيانات الشخصية كي لا نخالف مبدأ الحذف الفعلي.

## 9. تصنيف البيانات
| الجدول/الحقل | التصنيف |
| --- | --- |
| `users.password_hash` | سرّي |
| `users.phone_enc` / `users.phone_hash` | سرّي |
| `users.full_name` | سرّي |
| `users` (نوع/مهنة/محافظة) | داخلي |
| `otp_codes` | سرّي |
| `auth_sessions` | سرّي |
| `conversations.title` | سرّي |
| `messages.content` | سرّي |
| `message_sources` (نصوص قانونية) | عام (الربط داخلي) |
| `feedback` | داخلي |
| `legal_*` (النصوص القانونية) | عام |

## 10. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | بُعد `vector(n)` | مثبّت مبدئياً 1024؛ يُحسم مع نموذج التضمين |
| 2 | جدول `otp_codes` و`auth_sessions` | **مُضافان** (v0.3) مع اعتماد المصادقة |
| 3 | فهرس HNSW مقابل IVFFlat | يُحسم بعد قياس حجم القاعدة والأداء |
| 4 | تطبيع `categories` إلى جدول taxonomy | مؤجّل؛ TEXT[] يكفي الآن |
| 5 | إعداد بحثٍ عربيٍّ متخصّص (إضافة) | يُعاد النظر إن ضعُف الاستدعاء العربي |

## 11. سجل التغييرات
| الإصدار | التاريخ | الوصف | المعتمِد |
| --- | --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لنموذج البيانات (تطبيق + معرفة) | — (Draft) |
| 0.2 | 2026-06-24 | إضافة فهارس البحث اللفظي (tsvector + pg_trgm) وعمود `categories`؛ ربط ARC-RAG-001 | — (Draft) |
| 0.3 | 2026-07-02 | اعتماد المصادقة: أعمدة `full_name`/`phone_hash`/`phone_verified`/`password_hash` في `users`، وجدولا `otp_codes` و`auth_sessions`؛ تحديث التشفير والحذف والتصنيف؛ ربط ARC-API-001 وSEC-CRYPTO-001 | — (Draft) |

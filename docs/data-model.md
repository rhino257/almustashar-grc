---
title: "نموذج البيانات — المستشار القانوني الذكي"
doc_id: ARC-DATA-001
version: 0.5
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-07-26
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

> **حالة التنفيذ (وسوم هذه الوثيقة):**
> - 🟢 **مطبّق:** موجودٌ فعلياً في مخطّط الباك اند (هجرة Alembic `e6c3057967fb_init_auth_schema`) على PostgreSQL 16 + pgvector.
> - 🟡 **تصميم مستهدف:** مخطّطٌ مرجعيٌّ للبناء اللاحق وفق الضوابط؛ ليس منقوصاً بل مقصودٌ كخارطة تسليمٍ للفريق.
>
> المطبّق حالياً = **شريحة المصادقة (M0)**: خمسة جداول فقط
> (`users`, `otp_codes`, `auth_sessions`, `policy_versions`, `user_consents`).
> الإضافات (المحادثة، الذاكرة، التفضيلات، المرفقات، قاعدة المعرفة) مستهدفة.

## 1. الغرض والنطاق
تصف هذه الوثيقة مخطّط قاعدة البيانات (PostgreSQL 16 + pgvector): الجداول،
الأعمدة، العلاقات، الفهارس، عزل الصفوف (RLS)، الحذف، وتصنيف البيانات. الهدف:
مخطّطٌ يكفي لإطلاق سؤال/جواب مسنَد بذاكرةٍ داخل المحادثة، مع مصادقةٍ بهاتف+كلمة
مرور+OTP (تفعيلٌ إلزاميٌّ عند أول تسجيل)، وبريدٍ اختياريٍّ للاسترداد، وتحقّقٍ
بخطوتين اختياريٍّ عبر SMS، ملتزمٌ بالحد الأدنى من البيانات (PDPL)، ومهيّأٌ لخط
أنابيب الاسترجاع الهجين.

## 2. المبادئ الحاكمة للمخطّط
- **نطاقان منفصلان:** (أ) بيانات التطبيق ينشئها المستخدم، (ب) قاعدة المعرفة
  القانونية. الأولى نصمّمها كاملةً، والثانية نُحاذي ما هو موجودٌ لديك.
- **الحد الأدنى من البيانات:** لا هوية وطنية؛ فقط ما يلزم للخدمة والمصادقة.
  الهاتف إلزاميٌّ (الهوية الأساسية)، والبريد اختياريٌّ (استرداد + قناة بديلة).
- **العزل الافتراضي:** RLS صارمٌ (FORCE) بشرطٍ fail-closed يمنع تسرّب بياناتٍ بين المستخدمين.
- **الحذف حقٌّ أصيل:** `ON DELETE CASCADE` يجعل حذف الحساب يمحو كل أثره —
  بما فيه سجلّات الموافقة (البرهان يلزم ما دامت المعالجة قائمة فقط).
- **بيانات الاعتماد مصونة:** كلمة المرور مجزّأة **Argon2id**؛ **الهاتف يُخزَّن كبصمة
  `HMAC-SHA-256` فقط دون نصٍّ صريح** (فهرسة عمياء)؛ البريد الاختياري مشفّر
  (AES-256-GCM عبر `CryptoService`) مع بصمة HMAC للبحث؛ رموز OTP مجزّأة **Argon2id**
  ورمز التجديد بصمة **SHA-256** (انظر SEC-CRYPTO-001).
- **المصادقة بلا اعتمادٍ على SMS كأساس:** الأساس هاتف+كلمة مرور؛ الـ OTP عبر
  SMS (مزوّد محلي مثل الأوائل) إلزاميٌّ **مرّةً** عند أول تسجيل (تفعيل الحساب)،
  ويُستخدم أيضاً في إعادة التعيين والتحقّق بخطوتين الاختياري. البصمة/الـ PIN
  وسيلة دخولٍ إضافية **محلية على الجهاز** لا تُخزَّن في القاعدة.
- **استراتيجية لغة التضمين:** يُخزَّن النص العربي للعرض، والإنجليزي للتضمين
  والبحث اللفظي الإنجليزي، مع بحثٍ لفظيٍّ عربيٍّ مساعد.

## 3. مخطّط الكيانات والعلاقات (ERD — البنية المستهدفة الكاملة)
> الجزء المطبّق حالياً من هذا المخطّط هو كتلة المصادقة فقط
> (`users`, `otp_codes`, `auth_sessions`, `policy_versions`, `user_consents`).
```text
users 1---* conversations 1---* messages 1---* message_sources
                                    *---1 feedback (per message)
                                    1---* attachments (per message; user-owned)
users 1---* memories
users 1---1 user_preferences
users 1---* user_consents *---1 policy_versions
users 1---* otp_codes        (phone/email verify | reset | 2FA login)
users 1---* auth_sessions    (refresh tokens, revocable)
legal_documents 1---* legal_articles 1---* legal_chunks
                          ^
                          +--+ variant_of (self-reference: amended <-> original)
message_sources *---1 legal_articles   (citation link, when available)
```

## 4. نطاق (أ): جداول التطبيق

### 4.1 🟢 المطبّق فعلياً — كتلة المصادقة (M0)
> يطابق هذا القسم مخطّط الهجرة الفعلي. الامتدادات المُفعّلة: `pgcrypto` و`vector`.
```sql
-- extensions enabled by the auth migration:
--   CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()
--   CREATE EXTENSION IF NOT EXISTS vector;     -- pgvector (متاح؛ يُستخدم مع قاعدة المعرفة لاحقاً)
CREATE TABLE users (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name      TEXT,
  username       TEXT UNIQUE,                        -- معرّف عرضٍ اختياري
  phone_hash     TEXT UNIQUE,                        -- HMAC-SHA-256 (الهوية الأساسية؛ لا يُخزَّن الهاتف صريحاً)
  phone_verified BOOLEAN NOT NULL DEFAULT FALSE,
  email_hash     TEXT UNIQUE,                        -- اختياري؛ HMAC-SHA-256 (UNIQUE يسمح بتعدّد NULL)
  email_enc      TEXT,                               -- اختياري؛ AES-256-GCM (عرض) عبر CryptoService
  email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  password_hash  TEXT,                               -- Argon2id (SEC-CRYPTO-001)؛ لا يُخزَّن نصّاً صريحاً
  mfa_enabled    BOOLEAN NOT NULL DEFAULT FALSE,     -- تحقّق بخطوتين عبر SMS (اختياري لكل مستخدم)
  deleted_at     TIMESTAMPTZ,                        -- موجودٌ في المخطّط الحالي — انظر §8 (قيدٌ مقابل سياسة الحذف الصلب)
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE otp_codes (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id) ON DELETE CASCADE,  -- NULL قبل إنشاء الحساب
  phone_hash   TEXT,            -- للقناة SMS
  email_hash   TEXT,            -- للقناة email
  code_hash    TEXT NOT NULL,   -- OTP مجزّأ بـ Argon2id (وليس SHA المجرّد)
  purpose      TEXT NOT NULL,   -- verify_phone | verify_email | reset_password | mfa_login
  channel      TEXT NOT NULL DEFAULT 'sms',   -- sms | email
  attempts     INT NOT NULL DEFAULT 0,
  max_attempts INT NOT NULL DEFAULT 5,         -- سقف المحاولات (تجاوزه => 423 Locked)
  expires_at   TIMESTAMPTZ NOT NULL,
  consumed_at  TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE auth_sessions (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id            UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  refresh_token_hash TEXT NOT NULL UNIQUE,   -- SHA-256 of the opaque refresh token
  device_label       TEXT,
  ip                 TEXT,
  user_agent         TEXT,
  last_seen_at       TIMESTAMPTZ,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at         TIMESTAMPTZ NOT NULL,
  revoked_at         TIMESTAMPTZ
);
CREATE TABLE policy_versions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  policy_type  TEXT NOT NULL,        -- terms | privacy | disclaimer
  version      TEXT NOT NULL,        -- e.g. 'v1.0-2026-07'
  content      TEXT,
  published_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_current   BOOLEAN NOT NULL DEFAULT FALSE,
  UNIQUE (policy_type, version)
);
CREATE TABLE user_consents (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  policy_version_id UUID NOT NULL REFERENCES policy_versions(id) ON DELETE RESTRICT,
  accepted_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  method            TEXT,           -- signup | reconsent
  ip                TEXT,
  user_agent        TEXT
);
```

| الجدول | الغرض | ملاحظة |
| --- | --- | --- |
| `users` | بيانات المستخدم + بيانات الاعتماد | `password_hash` بـ Argon2id؛ **الهاتف بصمةٌ فقط** (`phone_hash`)؛ البريد اختياريٌّ (enc/hash)؛ لا هوية وطنية |
| `otp_codes` | التحقّق/إعادة التعيين/2FA | مجزّأة (Argon2id)، قصيرة العمر، بحدّ محاولات (`max_attempts`)؛ حذفٌ متتالٍ |
| `auth_sessions` | جلسات رمز التجديد | `refresh_token_hash` (SHA-256) فريدٌ وقابلٌ للإبطال؛ `device_label`/`ip`/`last_seen_at` لشاشة الجلسات |
| `policy_versions` | إصدارات السياسات | `is_current` تحدّد النسخة السارية؛ افتراضياً FALSE |
| `user_consents` | سجلّ موافقات المستخدم | برهانٌ مؤرّخٌ بالإصدار؛ `RESTRICT` يمنع حذف إصدارٍ مرجَعٍ؛ حذفٌ متتالٍ عند حذف الحساب |

### 4.2 🟡 مستهدف — إضافات المستخدم وجداول المحادثة
> غير مطبّقة بعد؛ تُبنى في مراحل M2–M3 وفق «معايير الهجرات» (§12).
```sql
-- إضافات مستهدفة على users:
ALTER TABLE users ADD COLUMN user_type           TEXT NOT NULL DEFAULT 'citizen'; -- citizen|lawyer|judge|researcher|bank
ALTER TABLE users ADD COLUMN profession          TEXT;
ALTER TABLE users ADD COLUMN governorate         TEXT;
ALTER TABLE users ADD COLUMN avatar_key          TEXT;   -- كائن صورة الملف في MinIO
ALTER TABLE users ADD COLUMN recovery_email_enc  TEXT;   -- بريد استرداد اختياري (مشفّر)
ALTER TABLE users ADD COLUMN recovery_email_hash TEXT;   -- بصمة بريد الاسترداد (بحث)

CREATE TABLE conversations (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title       TEXT,
  pinned      BOOLEAN NOT NULL DEFAULT FALSE,
  archived_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_user        ON conversations(user_id);
CREATE INDEX idx_conversations_user_pinned ON conversations(user_id, pinned);
CREATE TABLE messages (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id  UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  role             TEXT NOT NULL CHECK (role IN ('user','assistant')),
  content          TEXT NOT NULL,
  status           TEXT NOT NULL DEFAULT 'complete'
                   CHECK (status IN ('pending','streaming','complete','error','stopped')),
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
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id            UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  rating                TEXT NOT NULL CHECK (rating IN ('up','down')),
  issue_category        TEXT,
  safety_subcategory    TEXT,
  conversation_included BOOLEAN NOT NULL DEFAULT FALSE,
  comment               TEXT,
  model_version         TEXT,
  retrieval_path        TEXT,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE attachments (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id  UUID REFERENCES messages(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  file_name   TEXT NOT NULL,
  mime_type   TEXT NOT NULL,
  size_bytes  BIGINT NOT NULL,
  storage_key TEXT NOT NULL,   -- مفتاح الكائن في MinIO
  status      TEXT NOT NULL DEFAULT 'preparing'
              CHECK (status IN ('preparing','ready','error')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_attachments_message ON attachments(message_id);
CREATE INDEX idx_attachments_user    ON attachments(user_id);
CREATE TABLE memories (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content           TEXT NOT NULL,
  source            TEXT NOT NULL DEFAULT 'auto' CHECK (source IN ('auto','manual')),
  source_message_id UUID REFERENCES messages(id) ON DELETE SET NULL,
  is_active         BOOLEAN NOT NULL DEFAULT TRUE,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_memories_user ON memories(user_id);
CREATE TABLE user_preferences (
  user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  theme           TEXT NOT NULL DEFAULT 'system',
  language        TEXT NOT NULL DEFAULT 'ar',
  memory_enabled  BOOLEAN NOT NULL DEFAULT TRUE,
  advisor_role    TEXT,
  improve_service BOOLEAN NOT NULL DEFAULT FALSE,
  settings        JSONB NOT NULL DEFAULT '{}',
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 5. 🟡 نطاق (ب): قاعدة المعرفة القانونية (مستهدف)
> **قيد مطابقة SSOT:** يعتمد المصدر الواحد (نوشن) تسمية `documents` + `content_units`
> لجداول قاعدة المعرفة. المخطّط أدناه (`legal_documents`/`legal_articles`/`legal_chunks`)
> تصميمٌ سابق؛ يلزم توحيده مع SSOT في تمريرةٍ مخصّصة لمخطّط قاعدة المعرفة (§10 #8).
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
  variant_of   UUID REFERENCES legal_articles(id),
  categories   TEXT[] DEFAULT '{}',
  content_en_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(content_en,''))) STORED,
  content_ar_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('simple',  coalesce(content_ar,''))) STORED
);
CREATE INDEX idx_legal_articles_document  ON legal_articles(document_id);
CREATE INDEX idx_legal_articles_variantof ON legal_articles(variant_of);
CREATE INDEX idx_articles_en_tsv  ON legal_articles USING GIN (content_en_tsv);
CREATE INDEX idx_articles_ar_tsv  ON legal_articles USING GIN (content_ar_tsv);
-- Arabic has no native PG stemmer; trigram index supports fuzzy/substring match
-- (يتطلّب امتداد pg_trgm — يُفعّل مع بناء قاعدة المعرفة):
CREATE INDEX idx_articles_ar_trgm ON legal_articles USING GIN (content_ar gin_trgm_ops);
CREATE INDEX idx_articles_categories ON legal_articles USING GIN (categories);
CREATE TABLE legal_chunks (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  article_id   UUID NOT NULL REFERENCES legal_articles(id) ON DELETE CASCADE,
  chunk_index  INT NOT NULL,
  content_en   TEXT NOT NULL,   -- English chunk that gets embedded
  embedding    vector(n),       -- البُعد (n) يُحسم من مواصفات نموذج التضمين jina-v5 (ذاتي الاستضافة)
  token_count  INT
);
-- ANN index, create AFTER the embedding dimension is fixed (نقطة انطلاق: m=16, ef_construction=64):
-- CREATE INDEX idx_legal_chunks_embedding
--   ON legal_chunks USING hnsw (embedding vector_cosine_ops)
--   WITH (m = 16, ef_construction = 64);
-- ويُضبط hnsw.ef_search لكل استعلامٍ حسب موازنة الدقّة/الزمن.
```

| الجدول | الغرض | ملاحظة |
| --- | --- | --- |
| `legal_documents` | القانون/اللائحة/المرجع | الكيان الأعلى |
| `legal_articles` | المواد بنسختيها | `content_ar` للعرض، `content_en` للتضمين؛ `variant_of` يقرن المعدَّل بالأصلي؛ عمودا `tsv` للبحث اللفظي |
| `legal_chunks` | المقاطع المُضمَّنة | التضمين على الإنجليزية بـ jina-v5؛ الاسترجاع يُرجع العربية عبر `article_id` |

**ملاحظة على البحث اللفظي:** الدمج (RRF) يجري على **مستوى المادة** —
القائمتان المتجهيتان تُرجعان مقاطع تُربط بمادّتها عبر `article_id`،
والقائمتان اللفظيتان تُرجعان مواداً مباشرة.

**ملاحظة على العربية:** لا يملك Postgres مُجزّئاً (stemmer) عربياً أصيلاً،
لذا نستخدم إعداد `simple` لـ tsvector + فهرس `pg_trgm` للتطابق التقريبي، مع تطبيع
التشكيل وتوحيد الألف/الياء/التاء المربوطة قبل الفهرسة (انظر §12). قد ننتقل لاحقاً
إلى إضافةٍ متخصّصة إن لزم تحسين الاستدعاء العربي.

## 6. عزل الصفوف (Row-Level Security)

### 6.1 🟢 المطبّق فعلياً
دورٌ تطبيقيٌّ منفصلٌ عن مالك المخطّط: `almustashar_app` بـ `NOSUPERUSER NOBYPASSRLS`،
مع `FORCE ROW LEVEL SECURITY` على كل جداول المصادقة الخمسة، وشرط عزلٍ **fail-closed**
(إن لم يُضبط المستخدم الحالي، لا يُرى أيّ صفّ):
```sql
-- ENABLE + FORCE RLS على: users, otp_codes, auth_sessions, policy_versions, user_consents
-- شرط العزل الموحّد (fail-closed):
--   (col = nullif(current_setting('app.current_user_id', true), '')::uuid)
--   OR (current_setting('app.system_op', true) = 'on')
CREATE POLICY users_isolation ON users
  USING ( (id = nullif(current_setting('app.current_user_id', true), '')::uuid)
          OR (current_setting('app.system_op', true) = 'on') );
CREATE POLICY sessions_isolation ON auth_sessions
  USING ( (user_id = nullif(current_setting('app.current_user_id', true), '')::uuid)
          OR (current_setting('app.system_op', true) = 'on') );
CREATE POLICY consents_isolation ON user_consents
  USING ( (user_id = nullif(current_setting('app.current_user_id', true), '')::uuid)
          OR (current_setting('app.system_op', true) = 'on') );
-- otp_codes: تُدار عبر عمليات النظام (الخدمة) أثناء التسجيل/الدخول
CREATE POLICY otp_system ON otp_codes
  USING ( current_setting('app.system_op', true) = 'on' );
-- policy_versions: قراءةٌ عامة، وكتابةٌ لعمليات النظام فقط
CREATE POLICY policy_read  ON policy_versions FOR SELECT USING ( true );
CREATE POLICY policy_write ON policy_versions FOR ALL
  USING     ( current_setting('app.system_op', true) = 'on' )
  WITH CHECK ( current_setting('app.system_op', true) = 'on' );
```
يستخرج الـ backend `user_id` من رمز الوصول (Bearer JWT) ويضبط `app.current_user_id`
لكل طلب؛ وتستخدم الخدمة `app.system_op = on` للعمليات النظامية (تسجيل، إدارة سياسات).

### 6.2 🟡 مستهدف — نفس النمط للجداول القادمة
عند إضافة جداول المحادثة تُطبَّق نفس صيغة fail-closed:
```sql
-- conversations/messages/attachments/memories/user_preferences:
-- USING ( (user_id = nullif(current_setting('app.current_user_id', true), '')::uuid)
--         OR (current_setting('app.system_op', true) = 'on') )
-- messages عبر الانتماء لمحادثة المستخدم (EXISTS على conversations).
```

## 7. التشفير وحماية الحقول
- `users.password_hash`: **Argon2id** (SEC-CRYPTO-001) بمعاملات: `time_cost=3`,
  `memory=65536 KiB (64 MiB)`, `parallelism=2`, `hash_len=32`, `salt_len=16` — لا نصّ صريح.
- `users.phone_hash`: بصمة **HMAC-SHA-256** حتمية بمفتاحٍ سرّي (فهرسة عمياء) — للبحث والفرادة.
  **لا يوجد `phone_enc`:** الهاتف لا يُخزَّن قابلاً للفكّ ولا يُعاد في `GET /v1/users/me`.
- `users.email_hash` / `users.email_enc`: بصمة HMAC للبحث + قيمةٌ مشفّرة (AES-256-GCM) للعرض عبر `CryptoService`
  (`NullCipher` مبدئياً؛ العمود `TEXT` يكفي النصّ المشفّر لاحقاً دون ترحيل).
- `otp_codes.code_hash`: رمز OTP مجزّأ بـ **Argon2id**، قصير العمر، بحدّ محاولات (`max_attempts`).
- `auth_sessions.refresh_token_hash`: بصمة **SHA-256** لرمز التجديد المبهم؛ يُبطَل عند الخروج/إعادة التعيين.
- 🟡 (مستهدف) `messages.content` و`memories.content`: مصنّفان سرّيّين؛ يُفعَّل تشفيرهما على مستوى التطبيق لاحقاً
  (لا يؤثّر على RAG لأنه يعمل على القوانين لا على رسائل المستخدم). و`recovery_email_enc/hash` عند إضافته.
- **البصمة/الـ PIN لقفل التطبيق:** محلياً على الجهاز (Keystore/Keychain) — لا عمود في القاعدة ولا إرسال للخادم.
- 🟡 (مستهدف) **المرفقات:** ملفاتها في MinIO مع تشفيرٍ في الراحة/النقل وتحكّم وصول؛ القاعدة تحفظ المرجع فقط.
- لا تُسجَّل بيانات شخصية أو أسرار في اللوقات (منقّح فعلياً في طبقة الرصد).

## 8. الحذف والاحتفاظ (PDPL)
- تُحفظ المحادثات حتى يحذفها المستخدم أو يُغلق حسابه.
- حذف الحساب = حذف صفّ `users`؛ والـ CASCADE يمحو تلقائياً `otp_codes` و`auth_sessions`
  و`user_consents` (والمحادثات والرسائل والمصادر والتغذية والمرفقات والذاكرة والتفضيلات عند بنائها).
- **الموافقات:** نحتفظ بالبرهان المُعرَّف ما دام الحساب قائماً؛ وعند الحذف تُمحى مع الحساب.
- 🟡 ملفات المرفقات في MinIO تُحذف بمهمةٍ متزامنةٍ مع حذف صفوف `attachments`.
- رموز OTP المنتهية/المستهلكة تُنظَّف دورياً (مهمة مجدولة).
- **قيدٌ يلزم حسمه:** يحتوي مخطّط `users` الحالي عمود `deleted_at` (حذفٌ ناعم)، بينما تعتمد
  سياستنا **الحذف الفعلي (hard-delete) عبر CASCADE**. يجب المواءمة: إمّا إزالة `deleted_at`
  أو حصر استخدامه في التعطيل المؤقّت مع ضمان المحو الفعلي عند طلب الحذف (§10 #9).

## 9. تصنيف البيانات
| الجدول/الحقل | التصنيف |
| --- | --- |
| `users.password_hash` | سرّي |
| `users.phone_hash` | سرّي |
| `users.email_enc` / `users.email_hash` | سرّي |
| `users.full_name` / `users.username` | سرّي |
| `users` (mfa_enabled، الطوابع الزمنية) | داخلي |
| `otp_codes` | سرّي |
| `auth_sessions` | سرّي |
| 🟡 `conversations.title` | سرّي |
| 🟡 `messages.content` | سرّي |
| 🟡 `attachments` (الملفات والبيانات الوصفية) | سرّي |
| 🟡 `memories.content` | سرّي |
| 🟡 `user_preferences` | داخلي |
| `user_consents` / `policy_versions` | داخلي |
| 🟡 `message_sources` (نصوص قانونية) | عام (الربط داخلي) |
| 🟡 `feedback` | داخلي |
| 🟡 `legal_*` (النصوص القانونية) | عام |

## 10. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | بُعد `vector(n)` | يُحسم من مواصفات نموذج التضمين jina-v5 |
| 2 | فهرس HNSW مقابل IVFFlat | HNSW نقطة الانطلاق (m=16, ef_construction=64)؛ يُراجَع بعد قياس الحجم/الأداء |
| 3 | تطبيع `categories` إلى جدول taxonomy | مؤجّل؛ TEXT[] يكفي الآن |
| 4 | إعداد بحثٍ عربيٍّ متخصّص (إضافة) | يُعاد النظر إن ضعُف الاستدعاء العربي |
| 5 | تضمين الذاكرة (`memories.embedding`) للاسترجاع الذكي | مؤجّل لمرحلةٍ لاحقة |
| 6 | مصادقة TOTP (تطبيق مصادقة) كطبقةٍ إضافية | مؤجّلة؛ 2FA الحالي عبر SMS (ADR-0003) |
| 7 | تفعيل التشفير الفعلي (استبدال NullCipher) للحقول السرّية | يُفعَّل قبل الإنتاج مع خزنة المفاتيح |
| 8 | **توحيد تسمية قاعدة المعرفة مع SSOT** (`documents`/`content_units`) | مفتوح — يتطلّب تمريرةً مخصّصة بمخطّط SSOT |
| 9 | **مواءمة `users.deleted_at` مع سياسة الحذف الصلب** | مفتوح — إزالة العمود أو تقييد دوره |
| — | تخزين الهاتف: enc+bidx مقابل hash-only | **محسوم: hash-only** (يطابق الكود؛ الهاتف لا يُعاد في الواجهة) |

## 11. سجل التغييرات
| الإصدار | التاريخ | الوصف | المعتمِد |
| --- | --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لنموذج البيانات (تطبيق + معرفة) | — (Draft) |
| 0.2 | 2026-06-24 | إضافة فهارس البحث اللفظي (tsvector + pg_trgm) وعمود `categories`؛ ربط ARC-RAG-001 | — (Draft) |
| 0.3 | 2026-07-02 | اعتماد المصادقة: أعمدة `full_name`/`phone_hash`/`phone_verified`/`password_hash` في `users`، وجدولا `otp_codes` و`auth_sessions`؛ تحديث التشفير والحذف والتصنيف؛ ربط ARC-API-001 وSEC-CRYPTO-001 | — (Draft) |
| 0.4 | 2026-07-04 | توسعةٌ شاملة موائمةٌ للواجهات: بريدٌ اختياري و`username`/`avatar_key`/بريد استرداد/`mfa_enabled`؛ توسعة `otp_codes`/`auth_sessions`/`conversations`/`messages`/`feedback`؛ جداول جديدة (`attachments`/`memories`/`user_preferences`/`policy_versions`/`user_consents`)؛ توسعة RLS والتشفير والحذف والتصنيف؛ 2FA عبر SMS وقفلٌ محليٌّ؛ تأجيل TOTP وتضمين الذاكرة | — (Draft) |
| 0.5 | 2026-07-26 | **مطابقة الواقع:** فصل المطبّق (5 جداول) عن المستهدف بوسوم 🟢/🟡؛ اعتماد الهاتف **بصمةً فقط** (حذف `phone_enc`)؛ تصحيح أعمدة `users` الفعلية (إضافة `updated_at`/`deleted_at`/`email_enc`، ونقل `user_type`/`profession`/`governorate`/`avatar_key`/بريد الاسترداد إلى المستهدف)؛ `otp_codes.max_attempts` وOTP بـ Argon2id؛ `refresh_token_hash` UNIQUE؛ `policy_versions.is_current` افتراضياً FALSE؛ `user_consents.policy_version_id` بـ RESTRICT؛ تحديث RLS إلى الصيغة المطبّقة (FORCE + fail-closed عبر `app.current_user_id`/`app.system_op`)؛ ربط التضمين بـ jina-v5؛ توثيق قيدَي `deleted_at` وتسمية قاعدة المعرفة؛ إضافة قسم «معايير الهجرات» | — (Draft) |

## 12. معايير الهجرات (Database Migration Standards)
معيارٌ مُعتمَدٌ للفريق يضمن هجراتٍ آمنةً دون توقّف، مع أدوات Alembic:
- **Expand-and-Contract:** كل تغييرٍ كاسرٍ يُقسَّم على مراحل (إضافة ← ترحيل ← تبديل القراءة/الكتابة ← إزالة القديم).
- **هجرات عكوسة ومختبَرة:** لكل هجرة `upgrade` و`downgrade` فعليّان ومختبَران.
- **الفهارس دون قفل:** على الجداول الكبيرة يُنشأ الفهرس بـ `CREATE INDEX CONCURRENTLY` **خارج** المعاملة.
- **التعبئة على دفعات:** backfill للبيانات على دفعاتٍ صغيرة لتفادي الأقفال الطويلة.
- **`NOT NULL` على مراحل:** إضافة العمود nullable ← تعبئة ← فرض `NOT NULL` بعد التحقّق.
- **اختبارٌ في بيئةٍ مماثلة للإنتاج:** تُجرَّب الهجرات على نسخةٍ بحجمٍ وبنيةٍ مقاربة للإنتاج قبل التطبيق.
- **ضبط `hnsw.ef_search` لكل استعلام:** موازنةً بين الدقّة والزمن (مع `m=16`, `ef_construction=64` كنقطة انطلاق).
- **توثيق المخطّط داخل القاعدة:** `COMMENT ON TABLE`/`COMMENT ON COLUMN` لبيان الغرض.
- **تطبيع البحث العربي:** معالجة التشكيل وتوحيد الألف/الياء/التاء المربوطة قبل الفهرسة اللفظية؛ ودراسة قاموسٍ عربيٍّ متخصّص عند الحاجة.

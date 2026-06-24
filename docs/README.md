# المستشار القانوني الذكي (Al-Mustashar)

مساعدٌ قانونيٌّ ذكيّ يعمل محلياً (On-Prem) عبر RAG، يجيب بالعربية مسنَداً
بالنصوص القانونية اليمنية والشريعة الإسلامية، موجّهٌ للعموم
(مواطن/محامٍ/قاضٍ/باحث).

## تنويه الحياد
يعمل التطبيق في بيئةٍ ذات مرجعيتين تشريعيتين. الحياد السياسي التام **طبقة
فحصٍ مستقلة** لا مجرّد تعليمة، وتُعرض النسختان (الأصلية والمعدّلة) بالتساوي.

## الحزمة التقنية
| الطبقة | التقنية |
| --- | --- |
| الواجهة | Flutter (mobile-first) |
| الخادم | FastAPI + LangGraph |
| النموذج | Gemma 4 31B (QAT 4-bit) |
| التشغيل | vLLM (إنتاج) / Ollama (تطوير) |
| التضمين | EmbeddingGemma (ترجمة النص إلى الإنجليزية ثم التضمين) |
| إعادة الترتيب | jina-reranker-v3 |
| قاعدة البيانات | PostgreSQL + pgvector (self-hosted) |
| الاسترجاع | Hybrid (Dense + Lexical) + RRF + HyDE |

## بنية المستودع
​
.
-- backend/            # FastAPI + LangGraph (RAG pipeline)
`-- .env.example
-- frontend/           # Flutter app
`-- pubspec.yaml
-- docs/               # governed specs
`-- api-contract.md         # ARC-API-001
-- docker-compose.yml  # postgres + pgvector
`-- README.md

## المتطلّبات المسبقة
- Python 3.11+ و Flutter 3.x
- Docker (لتشغيل Postgres + pgvector)
- GPU: NVIDIA H100 80GB (للإنتاج عبر vLLM)

## الإعداد والتشغيل
### 1) قاعدة البيانات
​
docker compose up -d postgres
extensions: pgcrypto, vector, pg_trgm (created by migrations)

### 2) تشغيل النموذج
​
production (vLLM, OpenAI-compatible):
vllm serve google/gemma-4-31B-it-qat --quantization awq --port 8001
development (Ollama):
ollama run gemma4:31b

### 3) الخادم (backend)
​
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # then fill values
alembic upgrade head        # create schema
uvicorn app.main:app --reload --port 8000

### 4) الواجهة (frontend)
​
cd frontend
flutter pub get
flutter run

## متغيّرات البيئة
انسخ `backend/.env.example` إلى `backend/.env` واملأ القيم. لا تُودِع
`.env` في Git إطلاقاً.

## التوثيق
المواصفات المحكومة في `docs/`. الحوكمة (السياسات والمعايير والسجلّات)
في مستودع الحوكمة المنفصل `almustashar-grc`.

## الترخيص والحوكمة
استخدامٌ داخليّ ضمن الشراكة. يخضع لإطار الحوكمة في `almustashar-grc`
(PDPL، NCA، ISO 42001).
​
 بطاقة إنشاء الملف (2)
الحقل
القيمة
المستودع
مستودع التطبيق
المسار
backend/.env.example
الفرع
draft/repo-bootstrap
رسالة الـ commit
chore(config): add backend .env.example template
# ============================================================
#  Al-Mustashar backend - environment template
#  Copy to .env and fill values. NEVER commit the real .env.
# ============================================================

# --- Application ---
APP_ENV=development            # development | production
APP_HOST=0.0.0.0
APP_PORT=8000
API_VERSION=v1
CORS_ALLOWED_ORIGINS=http://localhost:3000

# --- Database (self-hosted Postgres + pgvector) ---
DATABASE_URL=postgresql+asyncpg://almustashar:password@localhost:5432/almustashar
DB_POOL_SIZE=10

# --- LLM serving (OpenAI-compatible endpoint) ---
LLM_BASE_URL=http://localhost:8001/v1
LLM_MODEL=google/gemma-4-31B-it-qat
LLM_API_KEY=not-needed-for-local
GENERATOR_MAX_TOKENS=1024

# --- Analyzer (Model-1). MVP: same model as generator ---
ANALYZER_MODEL=google/gemma-4-31B-it-qat
ANALYZER_MAX_TOKENS=256

# --- Embeddings (translate-then-embed strategy) ---
EMBEDDING_BASE_URL=http://localhost:8002
EMBEDDING_MODEL=google/embeddinggemma-300m
EMBEDDING_DIM=1024             # MUST match legal_chunks.embedding dimension

# --- Reranker ---
RERANKER_MODEL=jinaai/jina-reranker-v3
RETRIEVE_TOP_K=50              # per list
FUSE_TOP_K=100                # candidates after RRF
RRF_K=60                      # RRF constant
RERANKER_TOP_N=15             # final context size

# --- Crypto (CryptoService). MVP: NullCipher ---
CIPHER_PROVIDER=null          # null | aes_gcm
CIPHER_KEY=                   # base64 32-byte key, required when aes_gcm

# --- Concurrency / queue ---
MAX_CONCURRENT_REQUESTS=50
QUEUE_TIMEOUT_SECONDS=30

# --- Security / logging ---
LOG_PII=false                 # never log message content / phone

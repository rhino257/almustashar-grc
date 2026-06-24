---
title: "مواصفة خط أنابيب الاسترجاع والتوليد (RAG) — المستشار القانوني الذكي"
doc_id: ARC-RAG-001
version: 0.1
owner: مالك النظام / المدير التقني
status: Draft
approved_by: راعي المشروع
last_review: 2026-06-24
next_review: 2026-12-24
frameworks:
  - { name: "ISO/IEC 42001", status: محاذاة }
  - { name: "OWASP Top 10 for LLMs", status: محاذاة }
related:
  - ARC-DESIGN-001  # المعمارية
  - ARC-DATA-001    # نموذج البيانات
  - GOV-ADR-001     # القرارات التقنية
---

# مواصفة خط أنابيب الاسترجاع والتوليد (RAG)

## 1. الغرض والنطاق
تصف هذه الوثيقة المسار الكامل من سؤال المستخدم (عربي) إلى إجابةٍ عربيةٍ
مسنَدة بالمصادر: التحليل، الاسترجاع الهجين، الدمج، إعادة الترتيب، التوليد،
وطبقات الفحص. المرجع التنفيذي لعقد LangGraph وعقود الواجهة (SSE).

## 2. المبادئ
- **لا إجابة بلا مصدر:** كل إجابةٍ إمّا مسنَدة، أو مُعلَّمة صراحةً كغير مسنَدة.
- **العربية للعرض، الإنجليزية للتضمين:** نسترجع ونعيد الترتيب على النص
  الإنجليزي، ونعرض النص العربي الأصلي للمستخدم.
- **تعدد المسارات يفشل بطرق متعامدة:** ندمج متجهي + لفظي لتغطيةٍ أوسع.
- **الحياد طبقة فحص:** لا تعليمة فقط، بل عقدة تحقّق مستقلة.

## 3. نظرة عامة (عقد LangGraph)
​
[intake] -> [analyze (Model-1)] -> [retrieve (hybrid)] -> [fuse (RRF)]
-> [rerank (jina-v3)] -> [select_context] -> [generate (Model-2)]
-> [grounding_check] -> [neutrality_check] -> [stream_out]

| العقدة | مؤشّر الحالة (SSE) للمستخدم |
| --- | --- |
| analyze | "جارٍ تحليل السؤال..." |
| retrieve + fuse | "جارٍ البحث في النصوص القانونية..." |
| rerank + select | "جارٍ ترتيب أهم المصادر..." |
| generate (stream) | بثّ حيّ للإجابة |

## 4. المرحلة 1: النموذج المحلّل (Analyzer / Model-1)
**القرار:** نفس Gemma 4 31B بإخراجٍ مقيّد للـ MVP، مجرَّداً كمكوّن
`Analyzer` قابل للاستبدال. محفّز الترقية: إن صار على المسار الحرج للكمون،
نستبدله بنموذجٍ صغير مخصّص (Gemma 4 4B Q4) دون تغيير بقية الأنابيب.

عقد المخرجات (JSON صارم، `max_tokens` ~256):
​
{
"rewritten_query_en": "standalone English reformulation with context",
"hypothetical_answer_en": "SHORT hypothetical legal answer (<=200 tokens)",
"keywords_en": ["term1", "term2"],
"keywords_ar": ["مصطلح1", "مصطلح2"],
"categories": ["family_law", "commercial_law"]
}

| الحقل | الاستخدام في الأنابيب |
| --- | --- |
| `rewritten_query_en` | يُضمَّن كمتّجه dense ثانٍ (مرساة واقعية) |
| `hypothetical_answer_en` | يُضمَّن كمتّجه dense أول (HyDE) — قصيرٌ لكبح الهلوسة |
| `keywords_en` | البحث اللفظي على `content_en` |
| `keywords_ar` | البحث اللفظي على `content_ar` |
| `categories` | **تُلتقط ولا تُستخدم للفلترة في الـ MVP**؛ مهيّأة لتعزيزٍ ناعم مستقبلاً |

> ملاحظة حوكمة: لتفادي فخّ هلوسة HyDE، نُضمّن **متّجهين** (الإجابة الافتراضية
> + السؤال المعاد صياغته) لا الإجابة الافتراضية وحدها.

## 5. المرحلة 2: الاسترجاع الهجين + الدمج (RRF)
أربع قوائم مرتّبة تدخل RRF:
​
L1 = dense_search( embed(hypothetical_answer_en) )  top=50
L2 = dense_search( embed(rewritten_query_en) )      top=50
L3 = lexical_search_en( keywords_en )               top=50   (BM25/full-text on content_en)
L4 = lexical_search_ar( keywords_ar )               top=50   (full-text on content_ar)
fused = RRF([L1, L2, L3, L4], k=60)   # keep top 100 for reranking
صيغة RRF:
​
score(d) = sum over lists L of  1 / (k + rank_L(d)),   k = 60
تُنفَّذ القوائم الأربع **بالتوازي** لتقليل الكمون.

**قرار الفلترة بالتصنيف (MVP):** بلا فلتر. نلتقط `categories` ونؤجّل تطبيق
**التعزيز الناعم (soft boost)** داخل RRF لإصدارٍ لاحق بعد قياس الأثر.

## 6. المرحلة 3: إعادة الترتيب (Reranker)
**النموذج المعتمَد:** `jina-reranker-v3` (0.6B, cross-encoder).
- يعيد ترتيب أعلى 100 مرشّح على نصّها **الإنجليزي** (`content_en`).
- مُدخل الترتيب: `rewritten_query_en` مقابل `content_en` لكل مرشّح.
- المخرج: أعلى 15 مرشّحاً.

التبرير: إعادة الترتيب على الإنجليزية تتفادى ضعف rerankers العربية المعروف،
وحجم 0.6B لا يزاحم Gemma على الـ GPU.

## 7. المرحلة 4: انتقاء السياق وموازنة النسختين
- لكل مرشّحٍ نهائي، نجلب **النص العربي** الأصلي عبر `legal_chunks.article_id`.
- **موازنة النسختين:** عبر `legal_articles.variant_of`، إذا ظهرت إحدى
  النسختين (الأصلية أو المعدّلة-صنعاء) نضمّ شقيقتها تلقائياً لتُعرضا معاً
  بالتساوي (تطبيقٌ مباشر لمبدأ الحياد على مستوى الاسترجاع).

## 8. المرحلة 5: التوليد وطبقات الفحص
​
generate (Model-2, Gemma 4 31B):
input  : question_ar + top_articles_ar + system_prompt
output : Arabic answer, STREAMED via SSE
grounding_check:
set messages.is_grounded = TRUE if answer is supported by retrieved sources
else FALSE  -> UI shows red "غير مسنَد" label, event is logged
neutrality_check:
independent verification layer (two-government neutrality);
block/flag politically non-neutral phrasing before final emit

## 9. البثّ ومؤشّرات الحالة (SSE)
- كل عقدةٍ تُصدر حدث حالةٍ مرتبطاً باسمها الفعلي في LangGraph (لا نصوصاً
  وهمية)، فالواجهة تفاعليةٌ وتعكس الحالة الحقيقية.
- التوليد يُبثّ توكناً بتوكن؛ خانة المصادر تُملأ من `message_sources`.

## 10. ميزانية الكمون (هدف <= 15s)
| المرحلة | تقدير |
| --- | --- |
| Analyzer (Model-1, ~256 tok) | 2–4s |
| التضمين (متّجهان) | <0.3s |
| الاسترجاع (4 قوائم، متوازية) | <0.5s |
| RRF | <0.05s |
| إعادة الترتيب (jina-v3, ~100 مرشّح) | 0.3–0.8s |
| بناء السياق | <0.1s |
| التوليد — أول توكن (TTFT) | 1–2s |
| **الزمن حتى أول توكن (المحسوس)** | **~5–7s** |

البثّ يجعل الكمون **المحسوس** = زمن أول توكن، لا زمن الإجابة الكاملة.

## 11. متطلّبات يفرضها هذا الأنبوب على نموذج البيانات (ترتد إلى ARC-DATA-001 v0.2)
- فهرس بحثٍ لفظي على `content_en` (مثل `tsvector` + GIN).
- فهرس بحثٍ لفظي على `content_ar` (إعداد عربي/trigram).
- عمود `categories` (تصنيف قانوني) على `legal_articles` — مهيّأ، غير مُستخدَم في الـ MVP.

## 12. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | عتبة `is_grounded` (كيف نقرّر الإسناد) | تُحدَّد عند بناء grounding_check |
| 2 | بُعد متّجه التضمين | يُحسم مع تثبيت EmbeddingGemma |
| 3 | تفعيل التعزيز الناعم بالتصنيف | إصدار لاحق بعد قياس الأثر |
| 4 | ترقية النموذج المحلّل إلى نموذجٍ صغير | يُفعَّل بمحفّز الكمون/الإنتاجية |

## 13. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لخط أنابيب RAG (HyDE + Hybrid RRF + rerank + معماريّة نموذجين) |
​

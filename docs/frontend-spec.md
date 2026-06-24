---
title: "مواصفة الواجهة (Frontend Spec) — المستشار القانوني الذكي"
doc_id: ARC-FE-001
version: 0.1
owner: مالك النظام / المهندس المعماري
status: Draft
approved_by: راعي المشروع
last_review: 2026-06-24
next_review: 2026-12-24
frameworks:
  - { name: "Flutter App Architecture (MVVM)", status: مرجعي }
  - { name: PDPL, status: محاذاة }
related:
  - ARC-API-001     # عقد الواجهة (مصدر أحداث SSE)
  - ARC-CONV-001    # اصطلاحات Dart واللغة
  - AIG-GUARDRAILS-001
  - GOV-ADR-001
---

# مواصفة الواجهة

## 1. الغرض والنطاق
تعرّف هذه الوثيقة بنية واجهة Flutter وتجربة المستخدم، وربطها بعقد
`ARC-API-001`، وتُستخدم كـ **موجز بناءٍ (build brief)** يُسلَّم لمساعد الكود.

## 2. الحزمة التقنية للواجهة
| المكوّن | الاختيار |
| --- | --- |
| الإطار | Flutter 3.x (Dart) |
| إدارة الحالة | Riverpod |
| المعمارية | Feature-First + MVVM |
| قاعدة واجهة المحادثة | `flutter_gen_ai_chat_ui` (MIT) |
| نقل البثّ | `flutter_client_sse` |
| عرض Markdown | `flutter_markdown_plus` |
| الشبكة | `dio` (مع interceptor للترويسة `X-User-Id`) |
| التدويل | `flutter_localizations` + `intl` + ملفات `.arb` |

## 3. البنية المعمارية (Feature-First)
بوابة البيانات الوحيدة هي الـ repository، والتدفّق أحاديّ الاتجاه من
الطبقة data إلى presentation عبر view-models (Riverpod).
​
lib/
core/
network/      # dio client, sse client, X-User-Id interceptor
theme/        # colors, typography, RTL (palette pending)
l10n/         # .arb resource files (ar primary)
router/       # go_router
widgets/      # shared widgets
features/
chat/
data/         # repository, dto, sse_event_parser
domain/       # Message, SourceRef, StreamStatus models
application/  # chat_controller (Riverpod notifier)
presentation/ # chat_screen, message_bubble, sources_box,
grounding_badge, status_indicator
conversations/
data/ domain/ application/ presentation/
settings/
presentation/ # user_type selection (deferred here)
main.dart

## 4. استراتيجية البناء (Contract-First)
- نبني الواجهة على **mock server** يحاكي `ARC-API-001` (REST + SSE)،
  فيتقدّم الـ frontend بالتوازي مع الـ backend دون انحراف (drift).
- طبقة repository مجرّدة: `ChatRepository` لها تطبيقٌ `MockChatRepository`
  وآخر `HttpChatRepository`، يُبدَّلان عبر Riverpod provider.

## 5. الشاشات (MVP)
| الشاشة | الوظيفة |
| --- | --- |
| Chat | المحادثة، البثّ، خانة المصادر، شارة الإسناد |
| Conversations | قائمة المحادثات السابقة + «محادثة جديدة» |
| Settings | تبديل اللغة، **اختيار نوع المستخدم** (مؤجّل هنا)، التنويه |

## 6. ربط أحداث SSE بالواجهة
| حدث SSE (ARC-API-001) | عنصر الواجهة |
| --- | --- |
| `status` | **مؤشّر المراحل**: تحليل → بحث → إعادة ترتيب → صياغة |
| `token` | إلحاق الرمز بفقاعة الرد المتدفّقة (مع مؤشّر كتابة) |
| `sources` | تعبئة **خانة المصادر** القابلة للطيّ |
| `done` | إنهاء الرد + ضبط **شارة الإسناد** من `is_grounded` |
| `error` | حالة خطأ + زرّ إعادة المحاولة |

## 7. مكوّنات تجربة المستخدم المعتمَدة
| المكوّن | القرار |
| --- | --- |
| خانة المصادر | **قابلة للطيّ** أسفل كل ردّ (قانون/مادة/نسخة/مقتطف) |
| شارة الإسناد | مسنَد ✅ / غير مسنَد ⚠️ لكل ردّ |
| النسختان | تبويبٌ متساوٍ (أصلية/معدّلة-صنعاء) دون ترجيح |
| التنويه | دائمٌ (ليست استشارةً قانونية) |
| مؤشّر الانتظار | **مراحل ذات معنى** (لا دوّار صامت) — لإدارة زمن ~15s |
| الاتجاه | **RTL عربيٌّ أولاً**؛ مصطلحات تقنية LTR مضمّنة |
| نوع المستخدم | يُختار من **الإعدادات لاحقاً** (لا onboarding إلزامي) |
| رفع الملفات/الصور | **غير مدعوم** في الـ MVP |

## 8. التدويل و RTL
- اللغة الأساسية العربية (RTL تلقائي عبر `Directionality`).
- **يُمنع** تضمين نصوصٍ عربية صلبةً؛ كلّها في ملفات `.arb` (ARC-CONV-001).

## 9. الثيم (Theming)
- placeholder حتى تزويد **لوحة الألوان** من مالك المشروع.
- يُبنى عبر `ThemeData` مركزيّ في `core/theme/` لتبديلٍ لاحقٍ بلا تعديلٍ موزّع.

## 10. موجز البناء لمساعد الكود (Build Brief)
1. ابدأ من `flutter_gen_ai_chat_ui` كقاعدةٍ لشاشة المحادثة والبثّ وMarkdown.
2. أضف **خانة المصادر** القابلة للطيّ المرتبطة بحدث `sources`.
3. أضف **شارة الإسناد** المرتبطة بـ `is_grounded` في حدث `done`.
4. استبدل مؤشّر الانتظار الافتراضي بمؤشّر **مراحل** مدفوعٍ بحدث `status`.
5. طبّق طبقة `ChatRepository` المجرّدة مع mock يحاكي `ARC-API-001`.
6. فعّل RTL والتدويل عبر `.arb` دون أي نصٍّ عربيٍّ صلب.
7. مرّر ترويسة `X-User-Id` عبر interceptor في `dio`.

## 11. القرارات المفتوحة
| # | البند | الحالة |
| --- | --- | --- |
| 1 | لوحة ألوان الثيم | تُطلب من مالك المشروع عند التنفيذ |
| 2 | `go_router` مقابل تنقّلٍ أبسط | يُحسم عند بناء التنقّل |
| 3 | حدّ ارتفاع خانة المصادر قبل التمرير | يُضبط تجريبياً |

## 12. سجل التغييرات
| الإصدار | التاريخ | الوصف |
| --- | --- | --- |
| 0.1 | 2026-06-24 | المسودّة الأولى لمواصفة الواجهة + موجز البناء |
​

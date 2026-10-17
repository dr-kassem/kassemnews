# قاسم نيوز (kassem NEWS)

منصة إخبارية رقمية عالمية — تركيز خاص على لبنان والشرق الأوسط — بهوية بصرية سوداء وحمراء، وواجهة عربية RTL بالكامل، ونظام توريد أخبار آلي قابل للتكوين، ولوحة تحرير آمنة.

> **شعار المنصة:** «العالم في الصورة الكاملة»

---

## 0) كاييروس إيه آي — المحلل الآلي

**kairosAI** محلل آلي داخل المنصة يحلّل آخر التطورات (المجمّعة فعلاً من الخلاصات: أحداث، عاجل، أقسام، مناطق) ويكتب تحليلاً من زاوية معلنة وثابتة:

- الموقف معلن صراحةً في كل تحليل: **مؤيد للمقاومة اللبنانية والمقاومة الإيرانية ومحور المقاومة**، ويُعدَّل عبر `KAIROS_STANCE`.
- لا يختلق وقائع؛ كل ما يكتبه مرجعُه عناوين وأحداث فعلية من توريد المنصة، ويُختم كل تحليل بإفصاح: «رأي، وليس خبراً».
- نُشر تحليل واحد لكل فترة (`KAIROS_INTERVAL_HOURS`، افتراضي 6 ساعات) مع منع التكرار.
- الواجهات: صفحة `/kairos`، خانة «تحليلات كاييروس» في الرئيسية، شارة «تحليل كاييروس AI» على البطاقات، وزر «توليد تحليل كاييروس» في لوحة التحكم.

متغيرات البيئة: `KAIROS_ENABLED`، `KAIROS_INTERVAL_HOURS`، `KAIROS_AUTHOR`، `KAIROS_STANCE`.

## 1) المزايا الرئيسية

- **واجهة عربية (RTL) بالكامل** بتصميم أخبار احترافي أسود/أحمر (`#0B0B0D` / `#E31B2F`) وخط Cairo.
- **الصفحة الرئيسية**: شريط عاجل متحرك، قصة رئيسية، شبكة مقالات، تغطية الشرق الأوسط/لبنان/العالم، سياسة، اقتصاد، تكنولوجيا وأمن سيبراني، متابعة حيّة (أحداث)، الأكثر قراءة، نشرة بريدية، فوتر كامل.
- **صفحة أخبار لكل مقال**: عنوان، وقت النشر والتحديث، القسم والمنطقة، المصدر، الملخص، رابط المصدر الأصلي، أخبار ذات صلة، مشاركة، سجل تصحيحات، وتصنيف واضح: **مؤكد / منقول عن مصدر / متطور / غير مؤكد**.
- **خط توريد آلي (Ingestion Pipeline)**:
  - جلب RSS بجدولة قابلة للتكوين مع إعادة محاولة (backoff).
  - تطبيع العناوين والتواريخ بالعربية، كشف المكرر (عنوان متطابق/شبه متطابق)، تجميع التقارير في **أحداث**.
  - تصنيف حسب القسم والمنطقة والبلد، وكشف **الحساسية** (ضحايا/عمليات/حالات طوارئ) لتحويلها للمراجعة.
- **قواعد نشر تلقائي قابلة للتكوين** من قاعدة البيانات دون إعادة نشر.
- **لوحة تحكم `/admin`** بأدوار (مسؤول/محرر/مراجع/مشاهد): إحصاءات حقيقية، مراجعة، نشر/رفض/إلغاء نشر، تعديل/تصحيح، إدارة المصادر واختبار الخلاصات، إدارة القواعد، سجلات التوريد والأخطاء، إيقاف/استئناف التوريد.
- **SEO كامل**: صفحات مخدمة (SSR)، بيانات منظمة Schema.org (NewsArticle)، sitemap.xml، robots.txt، Open Graph، عناوين/أوصاف فريدة، صفحات 404/خطأ.
- **اختبارات** لجوهر التوريد والنشر (27 اختباراً).

## 2) قرارات معمارية وتوازنات (Trade-offs)

البنية المعتمدة: **Next.js + TypeScript + Tailwind (واجهة)** و **FastAPI + SQLAlchemy + Pydantic (خلفية)** مع **SQLite للتطوير المحلي** و**PostgreSQL للإنتاج** (عبر `DATABASE_URL`)، وترحيلات **Alembic**.

التعديل عن المواصفات المطلوبة:

| الخيار المطلوب | المعتمد | السبب |
| --- | --- | --- |
| PostgreSQL | PostgreSQL (إنتاج) + SQLite (تطوير) | بيئة التطوير لا تتوفر فيها خدمات Docker/PostgreSQL |
| Celery + Redis | **APScheduler داخل العملية** | يعمل في عملية واحدة دون تبعيات خارجية؛ نقطة الدخول `run_ingestion` موحّدة ويمكن استبدال المشغّل بـ Celery Beat دون تغيير منطق التوريد (انظر القسم 8) |
| التجزئة | بدء `app.main` ينشئ الجداول تلقائياً للتطوير | إضافةً إلى `alembic upgrade head` للتطبيق الصحيح |

البنية معيارية: الطبقات (API ← pipeline ← تصنيف/كشف مكرر ← قواعد نشر) مستقلة وقابلة للاستبدال.

## 3) بنية المشروع

```
kassem-news/
├─ backend/                 # FastAPI
│  ├─ app/
│  │  ├─ main.py            # تطبيق+تشغيل المجدول+rate limiting+SEO endpoints
│  │  ├─ config.py          # إعدادات البيئة (Pydantic Settings)
│  │  ├─ database.py        # engine / session
│  │  ├─ models.py          # كل نماذج SQLAlchemy
│  │  ├─ schemas.py         # Pydantic
│  │  ├─ security.py        # PBKDF2 + JWT
│  │  ├─ publishing.py      # محرك قواعد النشر
│  │  ├─ seed.py            # بيانات مرجعية + مدير افتراضي + مصادر
│  │  ├─ api/               # public / auth / admin
│  │  └─ ingestion/         # fetch · normalize · classify · dedupe · pipeline · scheduler · cli
│  ├─ alembic/              # ترحيلات
│  ├─ tests/                # pytest
│  ├─ requirements.txt
│  └─ Dockerfile
├─ frontend/                # Next.js 15 (App Router) عربي RTL
│  ├─ app/(site)/           # الصفحات العامة
│  ├─ app/admin/            # لوحة التحكم
│  ├─ components/           # مكوّنات قابلة لإعادة الاستخدام
│  ├─ lib/                  # api / types / format
│  ├─ tailwind.config.ts    # ألوان الهوية
│  └─ Dockerfile
├─ docker-compose.yml
└─ README.md
```

## 4) التشغيل المحلي (دون Docker)

### المتطلبات
- Python 3.11+ (المطوَّر عليه 3.14) و Node 20+.

### الخلفية (backend)
```bash
cd backend
py -m venv .venv

pip install -r requirements.txt
copy .env.example .env                 

alembic upgrade head                    
python -m app.seed                      
python -m uvicorn app.main:app --reload --port 8000
```

### التوريد اليدوي / المجدول
```bash
python -m app.ingestion.cli run                                 
python -m app.ingestion.cli run --source bbc-arabic --source aljazeera-ar
```
يشغّل التطبيق مجدولاً تلقائياً كل `INGESTION_INTERVAL_MINUTES` (افتراضي 15 دقيقة) عند تشغيل uvicorn.

### الواجهة (frontend)
```bash
cd frontend
npm install
copy .env.local.example .env.local      
npm run dev                             
```
الواجهة تقرأ `API_URL` (من الخادم) و `NEXT_PUBLIC_API_URL` (من المتصفح). الافتراضي `http://127.0.0.1:8000`.

### الحساب الإداري الافتراضي (غيّره فوراً في الإنتاج)
- البريد: `admin@kassem.news` — كلمة المرور: `ChangeMe123!` (تُعدَّل عبر `ADMIN_EMAIL` و `ADMIN_PASSWORD`).

### التشغيل بالإنتاج محلياً
```bash
cd frontend && npm run build && npm run start
```

## 5) المصادر والخلاصات

السجل مبدئي في `backend/app/seed.py`:

- **نشطة (فُحصت توصّلها):** BBC عربي، الجزيرة نت، فرانس 24 عربي، DW عربية، سكاي نيوز عربية، RT عربي، وكالة أناضول، أخبار الأمم المتحدة، وثلاث خلاصات Google News (لبنان/العالم/الشرق الأوسط).
- **معطّلة بانتظار ترخيص:** رويترز وأسوشيتد برس (لا خلاصة عامة؛ يلزم API/اتفاق)، العربية (ترجع 403 للوصول الآلي).

تضاف المصادر من لوحة التحكم دون تعديل الشيفرة، مع **زر اختبار الخلاصة**، وتفعيل/تعطيل فوري، وخيار «مراجعة إلزامية» للمجمّعات والمصادر ذات الأجندة الواضحة.

## 6) مصداقية المحتوى — مبادئ إلزامية

- لا تنشر المنصة نصاً كاملاً محمي الحقوق؛ تُنشر **ملخصات تحريرية** مع إحالة المصدر الأصلي ورابطه (`خبرات مُولّدة`).
- لا يُوصف الخبر بأنه «مؤكد» إلا بعد تحقق فعلي (`verification_status`).
- الأخبار الحساسة تُحوَّل لمحرر بشري (`review`) تلقائياً، ويُعلَّق غير المؤكد منها كـ «متطور/غير مؤكد».
- لا توجد إحصاءات مزيفة ولا أخبار مختلقة في المنصة؛ كل ما يُعرض مصدره حقيقي من الخلاصات الفعّالة أو تحرير يدوي موثّق.

## 7) قواعد النشر (لوحة التحكم ← قواعد النشر)

| المفتاح | الافتراضي | الوصف |
| --- | --- | --- |
| `auto_publish_enabled` | true | نشر تلقائي للأخبار الروتينية المستوفية للشروط |
| `sensitive_requires_review` | true | حساس → مراجعة |
| `breaking_requires_review` | true | عاجل → مراجعة |
| `min_source_trust` | 2 | أقل درجة ثقة مقبولة للنشر التلقائي |
| `max_age_hours` | 72 | أخبار أقدم → مراجعة |
| `require_image` | false | اشتراط صورة |
| `duplicate_threshold` | 0.80 | كشف المكرر |
| `event_threshold` | 0.40 | تجميع الأحداث |
| `auto_publish_categories` | csv | الأقسام المسموح نشرها تلقائياً |
| `ingestion_paused` | false | إيقاف خط التوريد مؤقتاً |

## 8) البنية الموزّعة / Celery (اختياري)

في نشر متعدد العمليات، المشغّل `BackgroundScheduler` غير مناسب (عملية وحيدة). الاستبدال المعياري:

1. أضف `celery` و `redis` إلى `requirements.txt`.
2. عرّف مهمة `celery_app.task` تلتف حول `app.ingestion.pipeline.run_ingestion` (نقطة الدخول نفسها).
3. فعّل `beat_schedule` بفاصل `ingestion_interval_minutes`، و `worker --concurrency=1` (المورّد غير متزامن).
4. وجّه `DATABASE_URL` إلى PostgreSQL المشترك بين العمال.

`redis` موجود في `docker-compose.yml` جاهزاً لهذه الخطوة.

## 9) تكرار المقالات

- **قاعدة بيانات فريدة** على `url_hash` تمنع إدراج نفس الرابط مرتين.
- **بصمة (fingerprint)** لقائمة رموز العنوان المعنّفة بعد تطبيع عربي + إزالة كلمات التوقف.
- **تشابه سلاسل الرموز** (difflib) فوق `duplicate_threshold` → تسجيل في `duplicates` وتخطي.
- **تجميع الأحداث**: مقالات شبه متطابقة فوق `event_threshold` تُربط بحدث واحد وتظهر في صفحة «متابعة حيّة».

## 10) الاختبارات

```bash
cd backend
python -m pytest -q        # 27 اختباراً
```
تغطي: التطبيع العربي، التصنيف (قسم/منطقة/بلد/حساسية/عاجل)، كشف التكرار، **قواعد النشر** (حساس/عاجل/ثقة/عمر/قسم/إيقاف)، و**خط التوريد** (توجيه، مكرر، أحداث، فشل جلب).

## 11) Docker Compose (بيئة كاملة)

```bash
docker compose up --build
```
- `db` PostgreSQL 16 + `redis` 7 + `backend` + `frontend`.
- الواجهة على `http://localhost:3000`، الخلفية على `http://localhost:8000` (مستندات API على `/docs`).
- متغيرات مُخزّنة: حذف `kassem.db` في الحاوي، `SECRET_KEY` إلزامي في الإنتاج، `POSTGRES_PASSWORD`.

## 12) النشر للإنتاج (بدون Docker)

1. **الخلفية**: `pip install -r requirements.txt "psycopg[binary]"`، أنشئ PostgreSQL، `alembic upgrade head`، شغّل `python -m app.seed`، ثم:
   ```bash
   env DATABASE_URL=postgresql+psycopg://... \
       SECRET_KEY=<طويل عشوائي> \
       CORS_ORIGINS=https://example.com \
       INGESTION_ENABLED=true \
       ADMIN_EMAIL=... ADMIN_PASSWORD=... \
   uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 1
   ```
   خلف `uvicorn` ضع عكسك (nginx/Caddy) مع TLS. لا تستخدم `--workers > 1` مع المجدول الداخلي إلا بعد تحويل Celery (القسم 8) أو عزل المجدول في عملية واحدة.
2. **الواجهة**: `npm run build` ثم `npm run start`، وراء عكس وكيل يوجّه `/api/*` للخلفية (أو اضبط `NEXT_PUBLIC_API_URL` على نطاق API منفصل). أعط `SITE_URL` الصحيح لبيانات SEO و sitemap.
3. **قبل الإطلاق**: غيّر `SECRET_KEY` و`ADMIN_PASSWORD`، وفعّل HTTPS، وراجع شروط استخدام كل مصدر مدرج.
4. **استعدادات محركات البحث**: المنصة مؤهّلة تقنياً (SSR، canonical، OG، sitemap، NewsArticle schema) لكن **لا تُعد أي وعد بقبول Google News**؛ الطلب يقع خارج نطاق هذا المشروع.

## 13) البيانات الخارجية المطلوبة

- **بدون مفاتيح API** لخلاصات RSS المستخدمة افتراضياً.
- اختياري عند التوسع:
  - إذن/API من Reuters و AP لتفعيل مصدريهما (لا خلاصات عامة).
  - `NEWSAPI_KEY` / `GNEWS_KEY` إذا أُضيف مزوّد API لاحقاً (الواجهة جاهزة عبر `Source.type = api`).
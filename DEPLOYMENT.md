# نشر منصة محاسبون على Cloudflare Workers

يُبنى التطبيق حاليًا بواسطة Nitro باستخدام إعداد Cloudflare Workers. يحدد ملف `wrangler.jsonc` اسم الـWorker المستهدف (`mohasiboon-com`)، ثم تُنشأ إعدادات Wrangler اللازمة داخل `.output/server` عند البناء. يبقى تسجيل الدومين لدى Namecheap؛ ربطه بـ Cloudflare لا ينقل ملكيته.

## 1. تجهيز المشروع

ثبّت الاعتماديات وابنِ التطبيق:

```sh
bun install
bun run cf:dev
```

يفتح Wrangler نسخة محلية من Worker للاختبار. تأكد من عمل الصفحات الأساسية قبل النشر.

## 2. إعداد متغيرات Supabase

يعتمد المتصفح على رابط Supabase ومفتاحه العام وقت البناء. عند البناء محليًا، أنشئ ملف `.env.production.local`، ولا ترفعه إلى Git:

```dotenv
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=your-publishable-key
```

إذا كنت تستخدم Cloudflare Workers Builds من Git، أضف `VITE_SUPABASE_URL` و`VITE_SUPABASE_PUBLISHABLE_KEY` ضمن **Settings → Build → Build variables and secrets**؛ فمتغيرات البناء لا تصبح متغيرات وقت التشغيل تلقائيًا. يستخدم الخادم أيضًا `SUPABASE_URL` و`SUPABASE_PUBLISHABLE_KEY` وقت التشغيل؛ أضفهما تحت **Settings → Variables and Secrets**. أضف `SUPABASE_SERVICE_ROLE_KEY` كـ **Secret** فقط إذا كانت وظائف الخادم المستخدمة تحتاجه؛ لا تضفه ببادئة `VITE_` ولا تضعه في ملفات المشروع.

## 3. إعداد المقالات من WordPress

تتطلب قراءة المقالات، وفق إعداد التطبيق الحالي، المتغيرين `LOVABLE_API_KEY` و`WORDPRESS_COM_API_KEY` في أسرار Worker. أضفهما من لوحة Cloudflare، ولا تضع قيمهما في مستودع Git أو في متغيرات `VITE_*`.

بعد إعداد الأسرار، اختبر صفحة المقالات ومقالًا منفردًا. يعتمد الاتصال الحالي على موصل WordPress لدى Lovable؛ تحقّق من أن بيانات الاعتماد والموصل يجيزان الطلبات من Worker المنشور، وإلا ستحتاج المقالات إلى موصل/واجهة بديلة قبل الإطلاق.

## 4. النشر يدويًا من جهازك

سجّل الدخول إلى حساب Cloudflare من الطرفية، ثم انشر:

```sh
bunx wrangler login
bun run cf:deploy
```

ينشئ Nitro إعداد Wrangler داخل `.output/server` وملف توجيه في `.wrangler/deploy`؛ لذلك نفّذ أوامر Wrangler من جذر المشروع كما في الأوامر أعلاه، ولا تضف خيار `--cwd .output/server`. لا تعدّل ملفات `.output` أو `.wrangler` يدويًا؛ فهي ملفات مولّدة يعاد إنشاؤها مع كل بناء.

## 5. إعداد النشر التلقائي من Git في Cloudflare

إذا ربطت GitHub من **Workers Builds**، افتح **Worker → Settings → Build** واضبط:

- **Root directory:** جذر المستودع (اتركه فارغًا إذا كان الافتراضي هو الجذر).
- **Build command:** `bun run build`
- **Deploy command:** `bunx wrangler deploy`
- **Build variables:** أضف `VITE_SUPABASE_URL` و`VITE_SUPABASE_PUBLISHABLE_KEY` كي تكون متاحة وقت البناء. لا تضف أي مفتاح سري ببادئة `VITE_`.

اترك تثبيت الاعتماديات وفق مدير الحزم الذي يتعرف عليه Cloudflare من `bun.lock`. تأكد أن البناء يبدأ من جذر المشروع؛ ملف `wrangler.jsonc` موجود هناك ويثبت اسم Worker المنشور على `mohasiboon-com`. لا تستخدم `bun run cf:deploy` كـ **Build command** في Workers Builds؛ هذا الأمر مخصص للنشر اليدوي لأنه يبني وينشر معًا. استخدم Build وDeploy command المنفصلين أعلاه.

أسرار الخادم مثل `LOVABLE_API_KEY` و`WORDPRESS_COM_API_KEY` تضاف تحت **Settings → Variables and Secrets** للـWorker، وليس في Build variables. متغيرات البناء ليست متغيرات تشغيل.

## 6. ربط الدومين بالـWorker

وجود الدومين في حساب Cloudflare لا يربطه بالـWorker. من الصورة الحالية، قسم **Custom domains** فارغ؛ أضف الدومين إلى الـWorker المنشور:

1. افتح Worker `mohasiboon-com` ثم **Settings → Domains & Routes → Add → Custom Domain**.
2. أدخل اسم الدومين الذي تريد فتح الموقع به، ثم اختر **Add Custom Domain**.
3. يجب أن تكون منطقة DNS للدومين نشطة على Cloudflare، وألا يوجد سجل CNAME متعارض للاسم نفسه. ينشئ Cloudflare سجل DNS وشهادة HTTPS.
4. اختبر أولًا رابط `workers.dev` الظاهر في قسم **Domains and routes**، ثم اختبر الدومين بعد تفعيل شهادة HTTPS وانتشار DNS.

إذا لم تكن Nameservers للدومين قد نُقلت إلى Cloudflare بعد، راجع سجلات DNS أولًا ثم غيّرها في Namecheap إلى القيم التي يعطيك إياها Cloudflare. يبقى الدومين مسجلًا لدى Namecheap.

## 7. تحديث إعدادات المصادقة والفحص

في Supabase، حدّث **Authentication → URL Configuration** ليشمل عنوان الموقع الجديد ضمن **Site URL** و**Redirect URLs**. بعد ذلك اختبر تسجيل الدخول والخروج، الصفحات العامة، الوظائف، لوحة الإدارة، والمقالات من الدومين الجديد.

تذكّر أن خطة Supabase المجانية قد توقف المشروع بعد أسبوع من عدم النشاط؛ راجع لوحة Supabase دوريًا ولا تعتمد على الخطة المجانية لبيانات إنتاجية بلا نسخ احتياطية.

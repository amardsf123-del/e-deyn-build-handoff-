# تقرير التسليم التقني الكامل لـ ChatGPT — مشروع DEYN (إصلاح SMS + الترجمة الفرنسية الكاملة + السنتيم)

مرحباً ChatGPT، هذا الملف يحتوي على التقرير التقني الكامل والشامل من Gemini Pro من أوله إلى آخره (شاملاً تحليل الأسباب الجذرية، جميع التعديلات المنفذة، الجداول، نتائج الاختبارات، الردود الموضوعية على ملاحظاتك الثلاث، وأوامر بناء الـ APK خطوة بخطوة) حتى تتمكن من قراءته مباشرة دون أن يضطر المستخدم لنسخ النصوص يدوياً.

---

## القسم الأول: شرح السبب الجذري للمشكلتين وكيفية إصلاحهما

### 1. السبب الجذري لمشكلة نظام إرسال SMS وكيفية إصلاحها

* **غياب تتبع الأجزاء (`PendingIntent` و`BroadcastReceiver`) والتسجيل المبكر الخاطئ**:
  * في النسخة السابقة، كان `DeynSmsPlugin.java` يستدعي `manager.sendMultipartTextMessage(phone, null, parts, null, null)` بتمرير `null` لمعاملات `sentIntents` و`deliveryIntents`.
  * في نظام Android، دالة `sendMultipartTextMessage` تضع الطلب في طابور `SMSDispatcher` وتعود فوراً (خلال أقل من ملي ثانية) قبل أن يبدأ مودم الهاتف الخلوي فعلياً بإرسال أي جزء (`PDU`) إلى الشبكة.
  * كان الكود السابق يضيف معرّف الرسالة `id` مباشرةً إلى مجموعة `SENT` (`sent_ids`) فور عودة الدالة، فيعتبر التطبيق أن الرسالة أُرسلت بنجاح ويحذفها من طابور الجدولة حتى لو فشل الإرسال بسبب انعدام الرصيد أو ضعف الشبكة أو فشل الجزء الثاني من رسالة مقسمة إلى 3 أجزاء.
* **التعامل مع الرسائل القصيرة والطويلة والرموز غير المرئية وشرائح Dual-SIM**:
  * المسافات غير القابلة للكسر (`\u00A0` الناتجة عن دوال تنسيق المبالغ) وعلامات الاقتباس المزخرفة كانت تحول الرسائل الفرنسية إلى ترميز `UCS-2` (حد 70 حرفاً للرسالة المفردة و67 حرفاً للجزء الواحد في الرسائل المقسمة بدلاً من 160/153 في `GSM-7`).
  * استدعاء `sendMultipartTextMessage` للرسائل المكونة من جزء واحد، وعدم ربط `SmsManager` بالاشتراك الافتراضي للرسائل (`SmsManager.getDefaultSmsSubscriptionId()` / `createForSubscriptionId(subId)`) كان يسبب مشاكل في مرجع تجميع الأجزاء (`UDH`) على بعض الهواتف ثنائية الشريحة (Dual-SIM).
* **تجميد حالة السجل في JavaScript وتكرار أسماء المهام المؤجلة**:
  * في `sms/runtime.js` (السطر 98 سابقاً)، كان الكود يتخطى أي سجل موجود مسبقاً في `S.smsLog[id]` (`if (S.smsLog[id]) continue`) فلا تترقى الحالة أبداً بعد `accepted_by_android`.
  * كما كان `enqueueDeferred` في `DeynSmsPlugin.java` ينشئ اسم عمل مرتبطاً بالطابع الزمني (`deyn_sms_deferred_<hash>_<at>`) مما قد يترك مهمة مؤجلة قديمة تعمل بالتوازي مع جدولة محدثة.
* **الإصلاح المنفذ في الكود**:
  1. إضافة `SmsStatusReceiver` وربطه بـ `PendingIntent` مستقل لكل جزء (`com.deyn.app.SMS_SENT` و`com.deyn.app.SMS_DELIVERED`) مع تمرير `id` و`partIndex` و`totalParts` و`attemptAt`، واستخدام `sendTextMessage` للرسالة ذات الجزء الواحد و`sendMultipartTextMessage` للرسائل المتعددة الأجزاء، وربط `SmsManager` بشريحة SMS الافتراضية.
  2. الفصل الصارم بين 7 حالات متميزة في `DeynSmsPlugin.java` و`sms/engine.js` و`sms/runtime.js`:
     * `attempting`: بدء محاولة الإرسال وتوثيقها قبل استدعاء النظام.
     * `accepted_by_android`: قَبِل أندرويد طلب الإرسال وبانتظار نتيجة الشبكة للأجزاء (`sent = false`).
     * `sent_to_network`: نجح إرسال جميع الأجزاء إلى الشبكة (`partsSent == partsTotal && partsFailed == 0`، وهنا فقط تصبح `sent = true`).
     * `delivered`: تأكيد تسليم جميع الأجزاء إلى هاتف المستلم عبر تقرير المشغل (`ACTION_SMS_DELIVERED` PDU).
     * `partial_failure`: فشل جزئي عند نجاح بعض الأجزاء وفشل جزء واحد على الأقل (`partsSent > 0 && partsFailed > 0`) — لا يُسجَّل أبداً كنجاح كامل (`sent = false`).
     * `failed`: فشل إرسال جميع الأجزاء (`partsSent == 0 && partsFailed > 0`) مع تحرير المطالبة (`CLAIMED`) للسماح بإعادة المحاولة تلقائياً أو يدوياً.
     * `dispatch_uncertain`: حالة غير مؤكدة بعد انقطاع مفاجئ طويل لمنع التكرار الصامت.
  3. إضافة `SmsBootReceiver` لاستعادة الجدولة المحفوظة في `active_queue_json` داخل `SharedPreferences` عند إعادة تشغيل الهاتف (`BOOT_COMPLETED`) أو تحديث الحزمة (`MY_PACKAGE_REPLACED`)، وتوحيد اسم المهام المؤجلة (`deyn_sms_deferred_<id>`).

---

### 2. السبب الجذري لنقص الترجمة الفرنسية واختفاء السنتيم وكيفية إصلاحها

* **الحذف القسري لميزة السنتيم في الوضع الفرنسي**:
  * في `DEYN-preview.html` وُجد الشرط `if (DEYN_I18N.lang === 'fr') return '';` داخل دالة `dialect(d)`.
  * وفي `sms/i18n.js` وُجدت قاعدة CSS تُخفي عناصر السنتيم وزر تفعيلها قسرياً في الوضع الفرنسي:
    `body.fr .row-set:has([data-ch=dialect]), body.fr .hx-dial, body.fr .kp-d, body.fr .dial, body.fr #dial, body.fr .t-tag, body.fr .pill:not(.bk){display:none!important}`.
  * في المقابل، العناصر غير المشمولة بتلك القاعدة (مثل `.rk-d` و`.ds-d` ونافذة الدفع `#mDial` و`#mResBal` وسلة المهملات وتصدير PDF) بقيت تعرض السنتيم بالعربية (`مليون و425 ألف`، `ميادورو`) عند اختيار الفرنسية.
* **غياب نصوص ميزة SMS وتمديد الديون من `DEYN_FR_DICT`**:
  * أُضيفت 79 عبارة ونصاً ديناميكياً في `sms/runtime.js` و`.github/scripts/integrate_sms.py` دون إضافتها إلى قاموس الترجمة في `sms/i18n.js`.
  * كما لم تكن دوال `exportPdfLedger` و`statement` وتاريخ سلة المهملات في `deyn-app/www/index.html` مربوطة بـ `DEYN_I18N`.
* **الإصلاح المنفذ في الكود**:
  1. إزالة قواعد الإخفاء القسري من `sms/i18n.js` وإضافة محرك `dialectFr` و`arDialectToFr` لتحويل جميع المبالغ والقطع والأوراق النقدية إلى صيغة السنتيم الفرنسية الطبيعية (`500 centimes`، `1 000 centimes`، ...، `200 000 centimes`، `1 million de centimes`، `1 million et demi de centimes`، `1 million et 425 000 centimes`) مع الحفاظ على نفس القيمة الحسابية (1 دج = 100 سنتيم) وإبقاء زر التفعيل في الإعدادات (`Affichage des montants en centimes`).
  2. استكمال قاموس `DEYN_FR_DICT` بالكامل (أكثر من 620 مدخلاً ونمطاً ديناميكياً) وربط دوال العرض والنوافذ والتنبيهات (`openModal`, `toast`, `agToast`, `deynBanner`, `renderDisplay`, `renderPending`, `renderLedger`, `renderStats`, `updateMM`, `statement`, `exportPdfLedger`) بالترجمة الفورية المتزامنة إلى جانب `MutationObserver`، وإصلاح اتجاهات أزرار التبديل (`.modesw .thumb` و`.sw`) في وضع `LTR`.

---

## القسم الثاني: قائمة الملفات المعدلة ونتائج الاختبارات

### 1. جدول الملفات المعدلة

| الملف | ملخص التعديل |
| :--- | :--- |
| `DeynSmsPlugin.java` | تتبع كل جزء عبر `PendingIntent` و`SmsStatusReceiver`، والتمييز بين حالات الإرسال السبع، ودعم شرائح Dual-SIM، وإضافة `SmsBootReceiver` لإعادة الجدولة بعد إعادة التشغيل |
| `sms/engine.js` | إضافة `normalizeSmsText` و`analyzeMessage` لحساب أجزاء `GSM-7` و`UCS-2` بدقة، ودالة `reconcileStatus` لدمج نتائج الأجزاء ومنع تسجيل الفشل الجزئي كنجاح |
| `sms/runtime.js` | تحديث واجهة إعدادات وسجل واختبار SMS لعرض عدد الأجزاء والترميز والحالات الحقيقية دون رسائل نجاح مضللة، ودعم الترجمة الفرنسية الكاملة |
| `sms/i18n.js` | استكمال قاموس `DEYN_FR_DICT`، وإضافة `dialectFr` و`arDialectToFr` لإعادة ميزة السنتيم بالفرنسية، وإصلاح تنسيقات `LTR`، وربط دوال الواجهة بالترجمة الفورية |
| `.github/scripts/integrate_sms.py` | جعل الدمج قابلاً لإعادة التشغيل بأمان (Idempotent)، وربط `dialectFr` و`DEYN_I18N` في `index.html`، وتسجيل `SmsStatusReceiver` و`SmsBootReceiver` و`RECEIVE_BOOT_COMPLETED` في `prepare-android.sh` |
| `sms/test.mjs` | إضافة اختبارات تجزئة الرسائل العربية الطويلة (3 أجزاء `UCS-2`) واختبارات رفض الفشل الجزئي وإعادة المحاولة عند الفشل الكلي |
| `DEYN-preview.html` | مزامنة نسخة المعاينة بالكامل مع الكود الحقيقي المدمج لإصلاح السنتيم والترجمة الفرنسية وميزة SMS |
| `deyn-apk-project.zip` | تحديث أرشيف المشروع الحقيقي (`deyn-app/www/index.html` و`deyn-app/native/DeynSmsPlugin.java` و`deyn-app/scripts/prepare-android.sh`) |
| `sms-feature.zip` و`sms/README.md` | تحديث الحزمة الفرعية والتوثيق التقني لمطابقة الإصلاحات الجديدة |

### 2. جدول نتائج الاختبارات

| مجموعة الاختبارات | النتيجة | ما تم التحقق منه فعلياً |
| :--- | :--- | :--- |
| `node sms/test.mjs` | **نجح بالكامل** | تخصيص الدفعات FIFO، وتمديد الديون، وجدولة الطابور، وتجزئة الرسائل العربية الطويلة (3 أجزاء `UCS-2` دون فقدان أي حرف)، ورفض تسجيل الفشل الجزئي (نجاح 2 من 3 أجزاء) كنجاح كامل، والسماح بإعادة المحاولة عند فشل جميع الأجزاء |
| اختبارات `DeynSmsPlugin.java` (`javac 17.0.21`) | **نجح (3/3 مجموعات)** | ترجمة الكود الأصلي بمترجم Java 17 دون أخطاء، واختبار إرسال رسالة عربية من 3 أجزاء عبر `sendMultipartTextMessage` مع `PendingIntent` لكل جزء، والتحقق من عدم دخول الرسالة في `SENT` قبل وصول `RESULT_OK` لجميع الأجزاء، وتسجيل `partial_failure` عند فشل الجزء الثاني، وتحرير المطالبة عند الفشل الكلي |
| اختبارات المتصفح الشاملة (Chromium + Selenium E2E) | **نجح (6/6 فحوص شاملة لـ 26 شاشة)** | فحص جميع الـ 26 شاشة ونافذة منبثقة في الوضع الفرنسي والتأكد من عدم وجود أي حرف عربي (`0` حرف عربي في النصوص الظاهرة والـ `placeholders`)، والتحقق من عمل الحاسبة السريعة وميزة السنتيم بالفرنسية (`500 centimes` حتى `200 000 centimes` و`1 million...`)، وتصدير PDF وكشف الحساب وسلة المهملات |
| `trusted-time/test.mjs` | **نجح (18/18)** | التحقق من التوقيع الزمني وكشف التلاعب بالساعة ومهلة الـ 48 ساعة ووضع القراءة فقط |
| `sync/test_engine.mjs` و`sync/test_session.mjs` | **نجح (17/17 و16/16)** | التشفير من طرف لطرف، والاقتران عبر QR، ودمج العمليات المتزامنة، وعزل صلاحيات هاتف العامل |

---

## القسم الثالث: الرد الموضوعي على ملاحظات ChatGPT الثلاث

### 1. سلوك إعادة المحاولة عند الفشل الجزئي (`partial_failure`) في رسائل SMS متعددة الأجزاء

* **السلوك الفعلي في الكود الحالي**:
  * عندما تُقسَّم رسالة طويلة إلى عدة أجزاء (مثلاً 3 أجزاء) وينجح جزء أو أكثر بينما يفشل جزء آخر (`partsSent > 0 && partsFailed > 0`)، يُسجّل `DeynSmsPlugin.java` (في الدالة `handlePartSent`) الحالة `"partial_failure"` ويُخرج المعرّف من مجموعة `SENT` (`sent = false`) ويُدخله في مجموعة `PARTIAL` (`partial_ids`).
  * تُرجع كل من الدالة `isTerminalOrInFlight` في Java والدالة `isLogBlocking` في `sms/engine.js` القيمة `true` لحالة `partial_failure`، مما يعني أن النظام **يكتفي بتسجيل الفشل الجزئي بدقة (`partsSent` و`partsFailed`) ولا يعيد إرسال الرسالة تلقائياً**.
  * أما إذا فشلت جميع الأجزاء (`partsSent == 0 && partsFailed > 0`) فتُسجَّل الحالة `"failed"` وتُحرَّر المطالبة (`CLAIMED`) ليُعيد `WorkManager` وجدولة الطابور المحاولة تلقائياً.
* **السبب التقني في نظام Android (`SmsManager`)**:
  * واجهة `SmsManager.sendMultipartTextMessage` القياسية في أندرويد تولّد رقم مرجع التجميع في ترويسة `UDH` (`CSMS reference number` داخل `05 00 03 <ref> <total> <seq>`) داخلياً في طبقة `SMSDispatcher` ولا تكشفه للتطبيقات، ولا تسمح بتمرير رقم مرجع `UDH` مخصص عند استدعاء `sendTextMessage` لجزء منفرد.
  * لذلك: إذا أُعيد استدعاء `sendMultipartTextMessage` للرسالة كاملةً، ستُرسل الأجزاء التي وصلت سابقاً مرةً ثانية للزبون (تكرار). وإذا أُرسل الجزء الفاشل وحده عبر `sendTextMessage(phone, null, parts.get(failedIndex), ...)`، فسيخرج كرسالة مستقلة تماماً بدون ترويسة `UDH` مطابقة للرسالة الأولى، ولن يدمجه هاتف المستلم داخل النص الأصلي.
  * لهذا السبب اعتمد الكود الحالي منع إعادة الإرسال التلقائي الكامل عند `partial_failure` لتفادي تكرار الأجزاء المستلمة، مع عرض حالة الفشل الجزئي وعدد الأجزاء المرسلة والفاشلة بوضوح للتاجر في السجل. (ولم أغيّر الكود الآن التزاماً بتوجيهك).

### 2. حالة ملف `REPORT.md`

* **نعم، الملاحظة صحيحة**: ملف `REPORT.md` الموجود في الجذر يعود إلى المسودة الأولى الخاصة بنسخة المعاينة (`sms-feature.zip`)، بينما التوثيق المحدّث للمعمارية النهائية موجود في `sms/README.md`.
* **تأثيره على البناء والتطبيق**: **لا يؤثر إطلاقاً**؛ لا تقرأه أداة Gradle ولا Capacitor ولا `prepare-android.sh` ولا `.github/scripts/integrate_sms.py` أثناء بناء APK، وتركته دون تعديل التزاماً بتوجيهك.

### 3. توضيح الاختبارات المنفذة فعلياً وما لم يُختبر على هاتف حقيقي

* **سبب عدم ظهور ملفات اختبار Java وSelenium داخل الأرشيف**: شُغّلت اختبارات `TestDeynSmsPlugin.java` (مع محاكيات حزم `android.*` و`com.getcapacitor.*` و`androidx.work.*`) واختبار Selenium في المسار المؤقت `/tmp/` داخل الحاوية حتى لا تُضاف ملفات غريبة إلى شجرة المشروع قد تتعارض مع الفحص الثابت الصارم في `tests/e2e/test_storage_layout_static.py`.
* **ما لم يُختبر فعلياً على هاتف Android حقيقي**:
  1. الإرسال الفعلي عبر شريحة SIM وشكة اتصالات حقيقية (Mobilis / Djezzy / Ooredoo) وإعادة تجميع الأجزاء على هاتف مستلم حقيقي.
  2. استقبال تقارير التسليم الحقيقية (`ACTION_SMS_DELIVERED` PDU) من برج الاتصالات.
  3. سلوك `WorkManager` و`BOOT_COMPLETED` تحت قيود توفير البطارية الخاصة بواجهات الهواتف (Xiaomi MIUI / Samsung OneUI / Oppo ColorOS).

---

## القسم الرابع: دليل بناء الـ APK والمواصفات التقنية الكاملة لـ ChatGPT

### 1. الفرق بين ملفي ZIP المرفقين وأيهما تستخدم

| اسم الملف | المحتوى | الاستخدام الأنسب |
| :--- | :--- | :--- |
| `deyn-apk-project-sms-professional-2026-fixed.zip` | المستودع الكامل شاملاً `.github/workflows/*.yml`، و`.github/scripts/integrate_sms.py`، و`DeynSmsPlugin.java`، و`sms/*`، و`DEYN-preview.html`، وملف `deyn-apk-project.zip` المدمج، ومجلد `.git` (الفرع `main` سليم، والفرع النشط `fix/sms-and-fr-complete-2026`) | الرفع إلى مستودع GitHub لتشغيل GitHub Actions مباشرةً |
| `deyn-apk-project.zip` | أرشيف المصدر المدمج الجاهز للبناء مباشرةً ويحتوي على `deyn-app/` (مع `www/index.html` المدمج، و`native/DeynSmsPlugin.java`، و`native/DeynSyncPlugin.java`، و`native/DeynVaultPlugin.java`، و`scripts/prepare-android.sh`، و`capacitor.config.json`، و`package.json`) بالإضافة إلى `deyn-keygen/` و`sync/` و`trusted-time/` و`tests/` | فك الضغط والبناء المباشر لـ `deyn-app` دون خطوة فك ضغط مزدوجة |

### 2. متطلبات البيئة وإصدارات الأدوات المطلوبة للبناء

* **Node.js & npm**: الإصدار `20.x` (أو `18.19+`) لتثبيت حزم Capacitor (`@capacitor/core@^7.0.0`, `@capacitor/cli@^7.0.0`, `@capacitor/android@^7.0.0`, `@capacitor/device@^7.0.0`, `@capacitor/clipboard@^7.0.0`, `capacitor-secure-storage-plugin@^0.11.0`).
* **Java (JDK)**: **JDK 21** (Temurin / OpenJDK 21 — لأن `@capacitor/android@^7.0.0` يولّد مشروع Gradle يتطلب Java 21 افتراضياً).
* **Android SDK**:
  * `compileSdkVersion` / `targetSdkVersion`: **API 35** (أو API 34 حسب قالب Capacitor 7 Android).
  * `build-tools`: `34.0.0` أو `35.0.0` (يُحمّلها Gradle تلقائياً عند توفر `ANDROID_HOME` واتصال بالإنترنت).
* **Gradle**: يُنشأ غلاف `./gradlew` تلقائياً عند تنفيذ `npx cap add android` (Capacitor 7 يستخدم Gradle 8.x وAndroid Gradle Plugin 8.x).
* **Python 3**: `3.10+` (يُستخدم لتشغيل `scripts/fetch_fonts.py` و/أو `.github/scripts/integrate_sms.py`).

### 3. المجلد الصحيح وخطوات بناء Debug APK خطوةً بخطوة

إذا استخدمت `deyn-apk-project.zip` مباشرةً (أو استخرجته من الأرشيف الكبير):

```bash
# 1. فك ضغط أرشيف المشروع
mkdir -p build-workspace
unzip -q deyn-apk-project.zip -d build-workspace

# 2. الدخول إلى المجلد الصحيح لتطبيق المتجر الأساسي
cd build-workspace/deyn-app

# 3. تثبيت حزم Capacitor (يتطلب اتصال إنترنت للوصول إلى سجل npm)
npm install

# 4. تنزيل الخطوط المحلية (Alexandria و Plus Jakarta Sans) إلى www/fonts/
python3 scripts/fetch_fonts.py

# 5. توليد مشروع Android الأصلي عبر Capacitor
npx cap add android

# 6. تطبيق الأيقونات، ونسخ الإضافات الأصلية الثلاث (DeynVaultPlugin, DeynSyncPlugin, DeynSmsPlugin)،
#    وتسجيلها في MainActivity.java، وإضافة الأذونات (SEND_SMS, RECEIVE_BOOT_COMPLETED, ...)
#    وتسجيل SmsStatusReceiver و SmsBootReceiver في AndroidManifest.xml،
#    وإضافة تبعية androidx.work:work-runtime:2.9.0 في android/app/build.gradle
bash scripts/prepare-android.sh

# 7. مزامنة أصول الويب (www/index.html) مع مشروع أندرويد
npx cap sync android

# 8. بناء نسخة Debug APK
cd android
./gradlew --no-daemon assembleDebug
```

* **مسار ملف الـ APK الناتج بعد نجاح البناء**:
  `build-workspace/deyn-app/android/app/build/outputs/apk/debug/app-debug.apk`
* **ملاحظة حول `integrate_sms.py`**: ملف `deyn-apk-project.zip` المرفق مدمج مسبقاً وجاهز للبناء، ولكن إذا شُغّل `.github/scripts/integrate_sms.py` عليه مرة أخرى (كما يفعل سير عمل `.github/workflows/build-sms-test.yml` و`build-sms-apk.yml`) فسيعمل بأمان تام دون تكرار لأن السكريبت Idempotent.

### 4. المفاتيح والإعدادات والمتغيرات المطلوبة

* **مفاتيح التوقيع الداخلي للتطبيق (`DEYN_PUBLIC_KEY`)**: المفتاح العام `uM8GgGamY++GH9e2PTTkluRlzqEVdbnj+MLh4uFYr/Y=` مضمّن بالفعل داخل `deyn-app/www/index.html` و`deyn-keygen/www/index.html` ومتطابق بينهما (32 بايت Ed25519). **لا يلزم أي مفتاح خاص (`private key`) لبناء الـ APK**.
* **توقيع Android APK**: الأمر `./gradlew assembleDebug` يوقّع ملف الـ APK تلقائياً بمفتاح Debug الافتراضي لأندرويد، فلا حاجة لإعداد `keystore` خارجي أو متغيرات بيئة سرية لبناء نسخة الاختبار `debug.apk`.

### 5. هل يمكن بناء APK داخل بيئة ChatGPT أم عبر GitHub Actions؟

* **داخل بيئة ChatGPT (Code Interpreter / Sandbox)**: **لا يمكن إتمام بناء الـ APK داخل حاوية ChatGPT المعزولة** لسببين تقنيين:
  1. مشروع Capacitor يولّد مجلد `android/` عبر `npm install` + `npx cap add android`، ثم يحتاج Gradle إلى الاتصال بمستودعات `dl.google.com` و`repo.maven.apache.org` لتنزيل حزم AndroidX (`androidx.work:work-runtime:2.9.0`، و`appcompat`، وAGP)، بينما بيئة ChatGPT معزولة عن الإنترنت.
  2. حاوية ChatGPT لا تحتوي على Android SDK (`platforms/android-34` أو `android-35` وأدوات `aapt2` و`d8`).
* **البيئة المناسبة لبناء الـ APK مباشرةً دون أي تعديل**:
  1. **عبر GitHub Actions (الأسهل والأسرع)**: رفع ملفات `deyn-apk-project-sms-professional-2026-fixed.zip` إلى مستودع GitHub وتشغيل سير العمل `.github/workflows/build-sms-apk.yml` أو `build-deyn.yml` أو `build-sms-test.yml` من تبويب **Actions -> Run workflow**، وسيُنتج ملف `app-debug.apk` جاهزاً للتنزيل في قسم **Artifacts**.
  2. **أو على أي جهاز محلي / خادم سحابي متصل بالإنترنت** يحتوي على Node 20 وJDK 21 وAndroid SDK باتباع الأوامر الثمانية الموضحة أعلاه في المجلد `deyn-app`.

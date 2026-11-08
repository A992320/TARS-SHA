# نسخة التصميم الموحد V2

**هذه النسخة تتضمن ثيم الاتحاد الذكي. اقرأ أولاً `UPDATE_V2_AR.md` إذا كنت تحدث موقعاً مثبتاً.**

# TARS × SHASHETY PRO — الموقع الرسمي (PHP / MySQL)

موقع عربي متكامل لإدارة وعرض منتجات TARS LOADBALANCE وSHASHETY PRO. يحتوي المشروع على الواجهة العامة، ولوحة الإدارة، والشروحات ومشغّل الفيديو، والتعليقات، والمقالات، والأقسام، ومركز تحميل الملفات. **الموقع بوابة معلومات وتنزيلات، وليس برنامج دمج خطوط أو خادم IPTV بحد ذاته.**

## المتطلبات

- Ubuntu Server 22.04 أو أحدث.
- Nginx أو Apache، وPHP **8.1+** مع الإضافات: `pdo_mysql`, `mbstring`, `fileinfo`، وPHP-FPM لخادم Nginx.
- MySQL 8 أو MariaDB 10.6+.
- مساحة تخزين كافية لملفات ISO والفيديوهات، واتصال HTTPS عند النشر للعامة.
- يجب أن يشير جذر الموقع (document root) إلى `public/` **حصراً**، وليس مجلد المشروع الرئيسي.

## تثبيت يدوي على Ubuntu 22.04

### 1. تثبيت الحزم

```bash
sudo apt update
sudo apt install -y nginx mariadb-server php8.1-cli php8.1-fpm php8.1-mysql php8.1-mbstring php8.1-common
```

### 2. إعداد قاعدة البيانات

```bash
sudo mariadb
```

ثم داخل MariaDB:

```sql
CREATE DATABASE tars_portal CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'tars_portal'@'127.0.0.1' IDENTIFIED BY 'CHANGE_TO_A_STRONG_DB_PASSWORD';
GRANT ALL PRIVILEGES ON tars_portal.* TO 'tars_portal'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```

### 3. نقل الملفات وإعداد البيئة

انقل مجلد المشروع إلى `/var/www/tars-portal`، ثم:

```bash
cd /var/www/tars-portal
sudo cp .env.example .env
sudo nano .env
```

اضبط `DB_PASS` مثل كلمة مرور قاعدة البيانات. أنشئ قيمة عشوائية طويلة لـ `APP_KEY`:

```bash
openssl rand -hex 32
```

ضع الناتج في `APP_KEY`، وحدّث `APP_URL` بعنوان موقعك الحقيقي. لا تنشر ملف `.env` ولا تشاركه.

### 4. صلاحيات آمنة

```bash
cd /var/www/tars-portal
sudo chown -R root:www-data /var/www/tars-portal
sudo chmod -R u=rwX,g=rX,o= /var/www/tars-portal
sudo chown -R www-data:www-data storage
sudo find storage -type d -exec chmod 770 {} \;
sudo find storage -type f -exec chmod 660 {} \;
sudo chown root:www-data .env
sudo chmod 640 .env
```

### 5. إنشاء جداول SQL وحساب المدير

```bash
cd /var/www/tars-portal
sudo -u www-data php bin/install.php
```

سوف يطلب اسم مستخدم المدير وكلمة المرور (12 حرفاً على الأقل). **التشغيل الثاني لا يغيّر حساب المدير القديم.**

### 6. نشر Nginx

```bash
sudo cp deploy/nginx-tars-portal.conf /etc/nginx/sites-available/tars-portal
sudo ln -s /etc/nginx/sites-available/tars-portal /etc/nginx/sites-enabled/tars-portal
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

بعد التأكد من عمل `location /_protected/`، افتح `.env` واجعل:

```ini
X_ACCEL_REDIRECT=1
```

في حال تعذر استخدام Nginx بالصيغة السابقة يمكن إبقاء `X_ACCEL_REDIRECT=0` ليتم بث الملفات عبر PHP، لكن ذلك أقل كفاءة مع ISO كبيرة.

**مهم للإنتاج:** استخدم نطاقاً حقيقياً وشهادة TLS عبر Let's Encrypt / Certbot. لا تنشر صفحة الإدارة عبر HTTP دون HTTPS.

## الدخول إلى لوحة الإدارة

- الموقع: `https://YOUR_DOMAIN/`
- الإدارة: `https://YOUR_DOMAIN/admin.php`
- أنشئ أقسام TARS أو شاشتي، أضف شرحاً بالفيديو، وارفع الإصدار ثم اضغط **حفظ** لنشره.
- ملفات الفيديو المدعومة للرفع: `mp4`, `mkv`, `webm`.
- ملفات التنزيل: `iso`, `exe`, `msi`, `zip`, `7z`, `rar`, `deb`, `rpm`, `apk`, `img`, `qcow2`, `ova`, `vhdx`, `tar`, `gz`, `tgz`, `pdf`, `rsc`, `sh`, `bin`, `appimage`.
- **MKV قد لا يشتغل داخل المتصفح** بسبب دعم الترميز؛ رابط تنزيل الفيديو يتيح فتحه ببرنامج VLC.
- يتم رفع الملف على أجزاء بحجم 4MB، مما يتجاوز حدود `upload_max_filesize` على الملف النهائي بشرط قبول كل جزء. يجب ألا يقل `upload_max_filesize` عن `5M` و`post_max_size` عن `6M` و`client_max_body_size` عن `8m`.
- إذا انقطع الرفع، اختيار الملف مجدداً يبدأ جلسة جديدة؛ **الاستكمال من نفس الموضع بعد إعادة تحميل الصفحة غير مُنفّذ في هذا الإصدار**.
- ملفات الوسائط الأصلية تبقى بعد إزالة المنشور من القائمة كإجراء احتياطي؛ حذف الأصول غير المستخدمة يكون يدوياً بواسطة مدير الخادم بعد التحقق.

## ما الذي تتحكم به من لوحة الإدارة؟

- **الرئيسية**: عنوان الموقع والوصف والترحيب والتذييل.
- **الأقسام**: إنشاء أو تعديل أو إخفاء قسم، مع إظهاره في الصفحة الرئيسية.
- **الفيديو**: ملفات محلية، شرح ووصف، ترتيب مميز، نشر أو إخفاء، تعليقات الزوار.
- **الإصدارات**: تحميل ISO وEXE وغيرها مع وصف وإصدار وتصنيف وحجم التحميل.
- **المقالات**: كتابة نصوص ودروس وتحديثات.
- **التعليقات**: موافقة / رفض / حذف. الوضع الافتراضي: مراجعة قبل النشر.
- **الحساب**: تغيير كلمة المرور.

## الصيانة

تنظيف الأجزاء المرفوعة غير المكتملة بعد 24 ساعة:

```bash
sudo -u www-data php /var/www/tars-portal/bin/cleanup.php
```

أضفها إلى cron مرة كل يوم مثلاً عبر `sudo crontab -u www-data -e`:

```cron
15 3 * * * /usr/bin/php /var/www/tars-portal/bin/cleanup.php >/dev/null 2>&1
```

لإجراء نسخ احتياطي، احتفظ بنسخة من قاعدة البيانات ومجلد `storage/private/` وملف `.env` في مكان آمن مشفر. يفضل إعداد حدّ لمساحة القرص ومراقبة استهلاك التخزين عند رفع ISO كبيرة.

## ملاحظات تشغيلية وحدود النسخة

- صفحات **TARS** و**SHASHETY PRO** تعرض وصفاً للمزايا والخطط ولا تتصل تلقائياً بمشغّل IPTV أو بمحرك دمج خطوط حقيقي. ربط النظامين بـ API ومؤشرات السرعة الحية يحتاج تطوير تكامل لاحقاً.
- تظهر عبارة *Conceptual Architecture* على رسم شبكات الرئيسية؛ هي توضيحية وليست بيانات فعلية من خادم حقيقي.
- توزيع اتصالات الشبكة لا يضمن جمع سرعة عدة مزودين في تنزيل واحد. قد يجمع تحميل متعدد الاتصالات سرعات أكثر من خط.
- لا توجد قنوات IPTV أو ملفات ISO جاهزة داخل المشروع؛ تُضاف فقط نسخك وموادك التي تملك حق نشرها.
- الاختبارات الآلية لقاعدة البيانات لم تُنفذ هنا لعدم وجود خدمة MariaDB في بيئة الإنشاء. تم فحص صياغة PHP محلياً؛ اختبر الرفع والتنزيل والإدارة على الخادم قبل النشر العام.
- للاستضافة العامة شديدة الحساسية، يوصى بإضافة مراقبة وتحديد سعة وحماية تطبيقات ويب ونسخ احتياطي وخادم منفصل للملفات الكبيرة عند كثرة المستخدمين.

## بنية الملفات

```
app/                 إعداد البيئة والدوال وقوالب الواجهة ونظام الملفات
bin/                 تثبيت SQL وتنظيف الرفع المؤقت
database/            مخطط قاعدة البيانات
public/              الجذر المعلن للموقع فقط
public/admin.php     لوحة الإدارة
public/api/upload.php رفع مجزأ محمي
public/media.php     بث فيديو مع Range
public/download.php  تسليم ملفات التحميل
public/assets/       CSS, JS, SVG
storage/private/     الملفات الخاصة خارج جذر الويب
deploy/              إعداد Nginx
```

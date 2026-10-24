# Mailu Deployment

راه‌اندازی میل‌سرور با **Mailu 2024.06** و **Docker Compose**؛ شامل پنل مدیریت، وب‌میل SnappyMail، SMTP/IMAP، فیلتر اسپم Rspamd، تقویم و مخاطبین Radicale، اسکن ماکرو با Oletools و جست‌وجوی پیوست‌ها با Apache Tika.

این مخزن تنظیمات استقرار است. نامه‌ها، دیتابیس حساب‌ها، گواهی‌ها و کلیدهای واقعی در Git قرار نمی‌گیرند. ClamAV در این ترکیب فعال نیست.

> مراحل نصب برای **سرور جدید** است. روی سرور موجود، فایل `mailu.env` یا داده‌ها را با نمونه جایگزین نکنید. پیش از تغییرات از تنظیمات و داده‌ها پشتیبان بگیرید.

## پیش‌نیازها

- سرور Linux با Docker Engine، افزونه Docker Compose v2، Git و OpenSSL و مجوز اجرای Docker.
- برای ایمیل اینترنتی: IP عمومی ثابت، دامنه، دسترسی به DNS و امکان تنظیم PTR نزد ارائه‌دهنده IP.
- پورت 25 ورودی و خروجی باز باشد. دسترسی HTTPS برای دریافت ایمیج‌ها و دسترسی خروجی DNS روی UDP/TCP پورت 53 لازم است.
- پورت‌های جدول پایین با سرویس‌های میزبان تداخل نداشته باشند؛ فایروال و NAT را متناسب تنظیم کنید.
- Compose از درایور لاگ `journald` استفاده می‌کند. روی میزبان فاقد پشتیبانی، درایور سرویس‌ها را مثلاً به `local` تغییر دهید و `options.tag` را حذف کنید.

حداقل رسمی Mailu بدون ClamAV، ۱ گیگابایت RAM و ۱ گیگابایت swap است؛ برای Tika و جست‌وجوی پیوست‌ها در این ترکیب منابع بیشتری در نظر بگیرید. ظرفیت دیسک را بر اساس تعداد کاربران و حجم نامه‌ها انتخاب کنید. [پیش‌نیازهای رسمی](https://mailu.io/2024.06/compose/requirements.html)

```bash
docker version
docker compose version
```

## ساختار پروژه

```text
.
├── docker-compose.yml     # سرویس‌ها، پورت‌ها و شبکه‌ها
├── mailu.env.example      # نمونه عمومی تنظیمات
├── overrides/             # تنظیمات سفارشی قابل ثبت در Git
│   ├── unbound/
│   ├── nginx/
│   ├── dovecot/
│   ├── postfix/
│   ├── rspamd/
│   └── snappymail/
├── .gitignore
└── README.md
```

پوشه‌های خالی در Git ذخیره نمی‌شوند؛ در مراحل نصب ساخته می‌شوند. مسیرهای داده `certs/`، `data/`، `dav/`، `dkim/`، `filter/`، `mail/`، `mailqueue/`، `webmail/` و `mailu/redis/` هستند و نسبت به محل Compose محاسبه می‌شوند.

## نصب روی سرور جدید

### ۱. دریافت و آماده‌سازی

آدرس مخزن خودتان را جایگزین کنید:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
cd YOUR_REPOSITORY
cp mailu.env.example mailu.env
chmod 600 mailu.env
mkdir -p certs data dav dkim filter mail mailqueue webmail mailu/redis
mkdir -p overrides/{nginx,dovecot,postfix,rspamd,snappymail}
openssl rand -hex 16
```

خروجی دستور آخر را در `SECRET_KEY` قرار دهید. فایل نمونه عمداً کلید قابل استفاده ندارد. کلید نصب موجود را برای دنبال‌کردن این راهنما تغییر ندهید.

### ۲. دامنه و TLS

`mailu.env` را ویرایش کنید:

```dotenv
SECRET_KEY=YOUR_GENERATED_SECRET
DOMAIN=example.com
HOSTNAMES=mail.example.com
POSTMASTER=admin
TLS_FLAVOR=letsencrypt
RELAYNETS=
API=false
```

- نمونه‌های دامنه را جایگزین کنید: `DOMAIN` دامنه آدرس ایمیل است و `HOSTNAMES` نام میزبان اتصال کاربران.
- در حالت `letsencrypt`، نام‌های HOSTNAMES باید در DNS به همین سرور برسند و پورت‌های 80 و 443 در دسترس باشند.
- برای گواهی دستی، `TLS_FLAVOR=cert` بگذارید و زنجیره گواهی و کلید را در `certs/cert.pem` و `certs/key.pem` قرار دهید. تمدید و بارگذاری مجدد گواهی بر عهده شماست.
- `notls` فقط برای آزمایش مناسب است؛ قبل از دسترسی اینترنتی TLS را بررسی کنید.
- `RELAYNETS` را خالی نگه دارید مگر برای شبکه‌ای کاملاً مورد اعتماد: این گزینه اجازه ارسال بدون احراز هویت می‌دهد.
- برای API، علاوه بر `API=true` یک `API_TOKEN` تصادفی مستقل تنظیم کنید.

مرجع: [نصب و TLS](https://mailu.io/2024.06/compose/setup.html) و [متغیرهای تنظیمات](https://mailu.io/2024.06/configuration.html).

### ۳. شبکه و resolver؛ پیش از اجرا بررسی کنید

شبکه داخلی `192.168.203.0/24` و آدرس resolver برابر `192.168.203.254` است. در صورت تداخل با شبکه سرور، VPN یا Docker، موارد زیر را **با هم** تغییر دهید:

1. `SUBNET` در `mailu.env`.
2. `networks.default.ipam.config.subnet` در Compose.
3. تمام آدرس‌های `dns` و `resolver.networks.default.ipv4_address` در Compose.

**تنظیم سفارشی مهم:** فایل `overrides/unbound/unbound.conf` با `module-config: "iterator"` اعتبارسنجی DNSSEC را غیرفعال می‌کند. فایل `custom.conf` هم تنظیم مشابه دارد ولی در Compose فعلی mount نشده است. Mailu به resolver دارای اعتبارسنجی DNSSEC نیاز دارد؛ این override ممکن است مانع شروع سرویس‌ها شود. [توضیح رسمی](https://mailu.io/2024.06/faq.html#the-admin-container-won-t-start-and-its-log-says-critical-your-dns-resolver-isn-t-doing-dnssec-validation)

**برای نصب جدید**، بخش زیر را از سرویس `resolver` در Compose حذف کنید تا تنظیم پیش‌فرض ایمیج Unbound استفاده شود؛ IP و شبکه سرویس را نگه دارید:

```yaml
    volumes:
      - "./overrides/unbound/unbound.conf:/etc/unbound/unbound.conf.d/custom.conf:ro"
```

اگر resolver سفارشی لازم دارید، اعتبارسنجی DNSSEC و دسترسی خروجی DNS آن را قبل از ادامه آزمایش کنید. این راهنما تنظیمات سرور فعال را خودکار تغییر نمی‌دهد.

پورت‌ها فعلاً روی `0.0.0.0` منتشر می‌شوند. برای سرور چند IP، آدرس مناسب میزبان را در `front.ports` مشخص و مسیر خروجی SMTP/NAT را بررسی کنید. IPv6 در این Compose تنظیم نشده؛ تا زمان تنظیم و آزمایش آن، رکورد AAAA منتشر نکنید.

### ۴. DNS دامنه

| رکورد | نام نمونه | مقدار یا اقدام |
| --- | --- | --- |
| A | `mail.example.com` | IPv4 عمومی سرور |
| MX | `example.com` | اولویت `10`، مقصد `mail.example.com` |
| PTR | IP عمومی سرور | `mail.example.com`؛ نزد ارائه‌دهنده IP |
| TXT / SPF | `example.com` | برای ارسال فقط از همین MX: `v=spf1 mx -all` |
| TXT / DKIM | نام اعلام‌شده پنل | کلید عمومی تولیدشده توسط Mailu |
| TXT / DMARC | `_dmarc.example.com` | شروع پایش با `v=DMARC1; p=none` |

اگر سرویس دیگری هم از دامنه شما ارسال می‌کند، SPF باید آن را پوشش دهد. بعد از ساخت دامنه، DKIM و رکوردهای پیشنهادی را از پنل بردارید. سیاست DMARC را پس از بررسی نتیجه‌ها تنظیم کنید.

### ۵. بررسی و اجرای سرویس‌ها

```bash
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 resolver admin front
```

Compose فایل `mailu.env` را برای محیط کانتینرها می‌خواند. متغیرهای جای‌گذاری ایمیج مثل `MAILU_VERSION` از محیط شل یا `.env` خوانده می‌شوند. نسخه پیش‌فرض Mailu در Compose برابر `2024.06` است. Redis از `alpine` و Tika از `latest-full` استفاده می‌کنند؛ برای استقرار تکرارپذیر، پس از آزمایش نسخه یا digest مشخصی برای آن‌ها ثبت کنید.

### ۶. ساخت مدیر و ورود

بعد از آماده‌شدن admin، دامنه واقعی را جایگزین و رمز قوی را تعاملی وارد کنید. این دستورها برای Bash هستند:

```bash
read -r -s -p 'Admin password: ' MAILU_ADMIN_PASSWORD
echo
docker compose exec admin flask mailu admin admin example.com "$MAILU_ADMIN_PASSWORD"
unset MAILU_ADMIN_PASSWORD
```

حساب `admin@example.com` ساخته می‌شود. اگر نام مدیر دیگری انتخاب کردید، حساب یا alias مربوط به `POSTMASTER` را هم بسازید. از پنل، دامنه‌ها، کاربران و DKIM را بررسی کنید.

- مدیریت: `https://mail.example.com/admin`
- وب‌میل: `https://mail.example.com/webmail`
- نام کاربری: آدرس کامل ایمیل، مثل `user@example.com`

## اتصال نرم‌افزار ایمیل

میزبان ورودی و خروجی، نام اصلی HOSTNAMES است. ارسال کاربران نیاز به احراز هویت دارد.

| کاربرد | پورت TCP | اتصال |
| --- | --- | --- |
| مدیریت و وب‌میل | 443 | HTTPS |
| HTTP و صدور گواهی | 80 | HTTP |
| تبادل بین میل‌سرورها | 25 | SMTP |
| ارسال کاربران | 465 | TLS |
| ارسال کاربران | 587 | STARTTLS |
| دریافت IMAP؛ پیشنهاد برای کاربران | 993 | TLS |
| دریافت POP3؛ در صورت نیاز | 995 | TLS |
| IMAP / POP3 | 143 / 110 | فقط با STARTTLS و تنظیم مناسب |
| قوانین Sieve | 4190 | ManageSieve با STARTTLS |

جدول پورت‌های منتشرشده در Compose را نشان می‌دهد؛ فعال‌بودن TLS را قبل از استفاده بررسی کنید.

## نگهداری و عیب‌یابی

```bash
# وضعیت و لاگ‌ها
docker compose ps
docker compose logs --tail=100 smtp imap antispam
sudo journalctl -t mailu-smtp --since '30 minutes ago'

# اعمال تغییرات محیط یا Compose
docker compose up -d

# توقف بدون حذف داده‌ها
docker compose stop
```

| مشکل | بررسی |
| --- | --- |
| `address already in use` | پورت‌های میزبان با `sudo ss -lntup` |
| خطای DNSSEC یا آماده‌نشدن admin | override مرحله ۳، لاگ resolver، دسترسی خروجی پورت 53 و تداخل subnet |
| خطای گواهی | A/AAAA، HOSTNAMES، پورت‌های 80/443 و لاگ front |
| خطای journald | پشتیبانی درایور یا تغییر آن طبق پیش‌نیازها |
| ارسال/دریافت خارجی ناموفق | پورت 25، MX، PTR، IP خروجی و لاگ smtp |
| رفتن نامه‌ها به spam | SPF، DKIM، DMARC، PTR و اعتبار IP |
| آماده‌نشدن Tika | منابع میزبان و `docker compose logs --tail=100 fts_attachments` |

### پشتیبان‌گیری و ارتقا

Git جایگزین backup نیست. از `mailu.env`، تمام مسیرهای داده معرفی‌شده، Compose و overrides پشتیبان رمزگذاری‌شده با دسترسی محدود بگیرید و خارج از سرور نگه دارید. برای کپی سازگار فایل‌ها و دیتابیس‌ها، سرویس‌ها را در بازه نگهداری متوقف کنید؛ بعد از کپی با `docker compose up -d` اجرا کنید. مالکیت و مجوزها را حفظ و بازیابی را آزمایش کنید.

پیش از ارتقا، یادداشت‌های نسخه مقصد را بخوانید و backup بگیرید. صرفاً تغییر tag بدون بررسی مهاجرت دیتابیس کافی نیست. حذف پوشه‌های bind mount، داده‌های واقعی را از بین می‌برد.

## انتشار در GitHub

اگر هنوز مخزن Git محلی ندارید، از ریشه پروژه اجرا کنید. ابتدا یک مخزن خالی در GitHub بسازید و آدرس آن را جایگزین کنید:

```bash
git init -b main
git add .gitignore README.md docker-compose.yml mailu.env.example overrides/
git status --short
git diff --cached --stat
# محتوای انتخاب‌شده را پیش از commit بررسی کنید:
git diff --cached

git commit -m "Document Mailu deployment and protect local data"
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

`mailu.env`، کلیدها و داده‌های واقعی نباید در commit باشند. `overrides/` عمداً قابل ثبت است؛ هر فایل جدید آن را از نظر رمز یا اطلاعات شخصی بررسی کنید. خروجی تشخیصی محلی `olvectl query …` هم ignore شده است.

`.gitignore` فایل‌های قبلاً ثبت‌شده را از تاریخچه حذف نمی‌کند. اگر اطلاعات حساس قبلاً منتشر شده، کلید/توکن افشاشده را تعویض و تاریخچه مخزن را نیز پاک‌سازی کنید؛ حذف در commit جدید کافی نیست.

### تنظیمات محلی سرور فعال

فایل `docker-compose.override.yml` عمداً در Git ثبت نمی‌شود. این فایل فقط برای سازگاری یا بازیابی همان سرور است و ممکن است شامل overrideهای موقت DNSSEC باشد؛ آن را به نصب جدید کپی نکنید. مخزن عمومی از resolver پیش‌فرض Mailu استفاده می‌کند.


## منابع

- [Mailu 2024.06](https://mailu.io/2024.06/)
- [نصب با Compose](https://mailu.io/2024.06/compose/setup.html)
- [مرجع تنظیمات](https://mailu.io/2024.06/configuration.html)
- [پرسش‌های متداول](https://mailu.io/2024.06/faq.html)
- [نصب Docker Engine](https://docs.docker.com/engine/install/)

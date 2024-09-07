### راهنمای ایجاد خودکار گواهی SSL با Certbot و Cloudflare API
این راهنما به شما کمک می‌کند تا گواهی‌های SSL خود را با استفاده از Certbot و Cloudflare API به صورت خودکار دریافت و تمدید کنید. این روش برای دامنه‌هایی که DNS آن‌ها توسط Cloudflare مدیریت می‌شود بسیار مناسب است و به‌طور خودکار فرآیند اعتبارسنجی و تمدید گواهی را انجام می‌دهد. با تنظیم Certbot، پس از هر تمدید موفقیت‌آمیز، وب‌سرورهای Nginx، Caddy یا Apache نیز به‌طور خودکار ریلود می‌شوند تا گواهی‌های جدید بدون هیچ‌گونه دخالت دستی اعمال شوند. این فرآیند کاملاً خودکار و بهینه، امنیت و مدیریت گواهی‌های SSL را برای وب‌سایت شما ساده و مؤثر می‌سازد.

### پیش‌نیازها:
- یک سرور با سیستم‌عامل Ubuntu یا توزیع مبتنی بر Debian
- دسترسی به حساب کاربری Cloudflare با دسترسی به DNS دامنه
- نصب Certbot و افزونه Cloudflare
  
### مراحل تنظیم:

#### 1. ایجاد API Token در Cloudflare
1. وارد حساب کاربری خود در [Cloudflare](https://www.cloudflare.com) شوید.
2. به بخش **My Profile** بروید.
3. در بخش **API Tokens**، گزینه **Create Token** را انتخاب کنید.
4. از الگوهای موجود، **Edit zone DNS** را انتخاب کنید:
   - در قسمت **Zone.Zone.DNS: Edit**
   - و برای **Zone.Zone: Read** انتخاب کنید
5. دامنه مورد نظر خود (مثل `example.dev`) را انتخاب کرده و توکن را ایجاد کنید.
6. توکن ایجاد شده را در یک مکان امن ذخیره کنید.

#### 2. نصب Certbot و افزونه Cloudflare
ابتدا Certbot و افزونه مرتبط با Cloudflare را نصب کنید:

```bash
sudo apt update
sudo apt install certbot python3-certbot-dns-cloudflare
```

#### 3. پیکربندی فایل API Token
فایلی برای ذخیره API Token ایجاد کنید:

```bash
sudo nano /etc/letsencrypt/cloudflare.ini
```

محتوای زیر را در فایل قرار دهید:

```ini
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
```

مطمئن شوید که مجوزهای این فایل را محدود کنید تا فقط کاربر root بتواند به آن دسترسی داشته باشد:

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

#### 4. دریافت گواهی SSL
با اجرای دستور زیر، گواهی SSL برای دامنه شما دریافت می‌شود:

```bash
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini -d example.dev -d *.example.dev --agree-tos --server https://acme-v02.api.letsencrypt.org/directory
```

این دستور گواهی SSL برای دامنه `example.dev` و همه زیر دامنه‌های آن (مثل `example.dev.*`) دریافت می‌کند.

#### 5. پیکربندی Certbot برای تمدید خودکار و ریلود وب‌سرور
برای اینکه Certbot بعد از تمدید گواهی، وب‌سرور را به‌طور خودکار ریلود کند، باید فایل سرویس Certbot را ویرایش کنید.

```bash
sudo nano /lib/systemd/system/certbot.service
```

محتوای زیر را اضافه یا تغییر دهید تا Certbot بعد از تمدید، وب‌سرورهای Nginx، Caddy یا Apache را ریلود کند:

برای Nginx:
```ini
[Unit]
Description=Certbot
Documentation=file:///usr/share/doc/python-certbot-doc/html/index.html
Documentation=https://certbot.eff.org/docs

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot -q renew --deploy-hook "systemctl reload nginx"
PrivateTmp=true
```

برای Caddy:
```ini
[Unit]
Description=Certbot
Documentation=file:///usr/share/doc/python-certbot-doc/html/index.html
Documentation=https://certbot.eff.org/docs

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot -q renew --deploy-hook "systemctl reload caddy"
PrivateTmp=true
```

برای Apache:
```ini
[Unit]
Description=Certbot
Documentation=file:///usr/share/doc/python-certbot-doc/html/index.html
Documentation=https://certbot.eff.org/docs

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot -q renew --deploy-hook "systemctl reload apache2"
PrivateTmp=true
```

#### 6. اعمال تغییرات و راه‌اندازی تایمر Certbot
پس از ویرایش فایل سرویس، دستورات زیر را اجرا کنید تا systemd تایمر و سرویس Certbot را به‌روز کند:

```bash
sudo systemctl daemon-reload
sudo systemctl restart certbot.timer
```

#### 7. تست تمدید گواهی با `--dry-run`
برای اطمینان از اینکه Certbot و تنظیمات ریلود وب‌سرور به‌درستی کار می‌کنند، می‌توانید یک شبیه‌سازی (بدون تمدید واقعی) از فرآیند تمدید را با دستور زیر اجرا کنید:

```bash
sudo certbot renew --dry-run
```

این دستور بدون تمدید واقعی، فرآیند تمدید را شبیه‌سازی می‌کند و وب‌سرور شما را نیز ریلود می‌کند تا مطمئن شوید همه چیز به درستی پیکربندی شده است.

#### 8. بررسی وضعیت گواهی‌ها
برای بررسی وضعیت گواهی‌های SSL نصب شده و زمان انقضای آن‌ها، از دستور زیر استفاده کنید:

```bash
sudo certbot certificates
```

این دستور اطلاعات مربوط به گواهی‌های نصب‌شده از جمله تاریخ انقضا و دامنه‌های مرتبط با گواهی را نمایش می‌دهد.

#### 9. بررسی لاگ‌های Certbot و وب‌سرور
برای بررسی لاگ‌های Certbot و مطمئن شدن از این که فرآیند تمدید گواهی بدون مشکل اجرا شده است:

```bash
sudo cat /var/log/letsencrypt/letsencrypt.log
```

همچنین می‌توانید لاگ‌های وب‌سرور خود را بررسی کنید:

برای Nginx:
```bash
sudo journalctl -u nginx
```

برای Caddy:
```bash
sudo journalctl -u caddy
```

برای Apache:
```bash
sudo journalctl -u apache2
```

با انجام این مراحل، شما می‌توانید به‌طور خودکار گواهی‌های SSL خود را با استفاده از Certbot و Cloudflare دریافت و تمدید کنید. همچنین پس از تمدید موفق، وب‌سرورهای شما به‌صورت خودکار ریلود می‌شوند تا گواهی‌های جدید اعمال شوند.

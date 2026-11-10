# اجرای bolt.diy روی Deplexo (با نگهبان)

راهنمای بالا آوردن fork خودتان از [bolt.diy](https://github.com/stackblitz-labs/bolt.diy) روی [Deplexo](https://deplexo.com). نتیجه: یک آدرس HTTPS مثل `https://نام-app.de.deplexo.com` با صفحهٔ ورود و یک نگهبان که برنامه را زنده نگه می‌دارد. پورت عمومی **۸۰۸۰** است و در `deplexo.yaml` ثابت شده.

## ۱) فایل‌ها (همه کنار هم در ریشهٔ ریپو، بدون پوشهٔ اضافه)

| فایل | کار |
|---|---|
| `deplexo.yaml` | به Deplexo می‌گوید با Dockerfile بساز و پورت 8080 |
| `Dockerfile.deplexo` | ساخت image و اجرا با کاربر غیر root |
| `deplexo-gateway.mjs` | درگاه عمومی: صفحهٔ ورود، پروکسی، health |
| `deplexo-watchdog.mjs` | نگهبان: اجرا، پایش، ری‌استارت، هشدار |
| `DEPLEXO.md` | همین راهنما |

`Dockerfile` و بقیهٔ فایل‌های bolt.diy را دست نزنید. روی ریپوی رسمی StackBlitz دسترسی push ندارید؛ اول آن را Fork کنید.

```bash
git clone https://github.com/<حساب-شما>/bolt.diy.git && cd bolt.diy
# پنج فایل بالا را کنار package.json کپی کنید، سپس:
git add deplexo.yaml Dockerfile.deplexo deplexo-gateway.mjs deplexo-watchdog.mjs DEPLEXO.md
git commit -m "Add Deplexo deployment"
git push origin main
```

## ۲) معماری

```
اینترنت --HTTPS--> Deplexo --> :8080 gateway (ورود) --> 127.0.0.1:5173 bolt.diy (خصوصی)
                                   ^
                              watchdog (پایش و ری‌استارت)
```

یک کانتینر، چون Deplexo برای هر app فقط یکی می‌دهد. bolt.diy خودش ورود ندارد و مسیر git-proxy آن می‌تواند هر آدرس HTTPS را فوروارد کند؛ بدون لاگین، سایت شما پروکسی باز و مصرف‌کنندهٔ اعتبار مدل‌هایتان می‌شود.

## ۳) نگهبان چه می‌کند

- **پیش‌بررسی** در هر استارت: وجود vite و خروجی build، قابل‌نوشتن‌بودن `/tmp` و مسیرهای لینک‌شده، سقف RAM (اگر کمتر از ~900 MB باشد هشدار می‌دهد)، نسخهٔ Node.
- **آماده‌شدن واقعی**: فقط وقتی bolt.diy واقعاً به HTTP جواب بدهد «ready» می‌شود، نه صرفاً باز بودن پورت.
- **پایش هر 30 ثانیه**: درخواست به `/api/health`؛ 3 شکست پشت سر هم یعنی ری‌استارت.
- **ری‌استارت هوشمند**: اگر crash کند، با تأخیر 2، 4، 8 … تا 60 ثانیه دوباره بالا می‌آید. دروازه همان لحظه زنده می‌ماند و صفحهٔ «در حال بالا آمدن» با refresh خودکار نشان می‌دهد.
- **سه روش استارت**: اگر روش اول بالا نیامد، خودش با روش دوم و سوم امتحان می‌کند (مثلاً اگر flag `--configLoader` در نسخهٔ vite شما نباشد).
- **حافظه**: اگر مصرف به 90٪ سقف برسد هشدار می‌دهد و در 96٪ قبل از OOM-kill برنامه را ری‌استارت می‌کند.
- **لاگ و وضعیت از مرورگر** (بعد از ورود): `/__bolt/status` و `/__bolt/logs` (۳۰۰ خط آخر). اگر در بالا آمدن مشکلی بود، همین‌جا علتش را می‌بینید؛ لازم نیست داشبورد Deplexo را باز کنید.
- **هشدار اختیاری** با تلگرام یا webhook (پایین ببینید). رمزها و توکن‌ها از لاگ‌ها حذف می‌شوند.
- پروسهٔ bolt.diy و زیرپروسه‌هایش (workerd) هنگام خاموشی کانتینر کامل بسته می‌شوند.

## ۴) پلن

طبق [پلن‌ها](https://deplexo.com/plans): Free با 128 MB و Starter با 512 MB کم‌اند. یک گزارش مستقل مصرف bolt.diy را حدود 620 MB می‌گوید؛ **Pro (1 GB)** بردارید.

## ۵) ساخت app در Deplexo

1. GitHub را با OAuth وصل کنید و دسترسی fork را بدهید.
2. **Deploy** → ریپو و branch `main`.
3. نوع app: **web** (بعداً قابل تغییر نیست).
4. فرم باید `Dockerfile.deplexo` و پورت `8080` را از YAML بخواند. اگر نخواند، Build method را Dockerfile و مسیر را `Dockerfile.deplexo` بگذارید.
5. پلن Pro.
6. متغیرهای محیطی:

| نام | مقدار | توضیح |
|---|---|---|
| `BOLT_AUTH_PASSWORD` | رمز تصادفی 16+ نویسه | **الزامی**؛ بدون آن کانتینر عمداً بالا نمی‌آید |
| `BOLT_AUTH_USERNAME` | پیش‌فرض `admin` | حروف، عدد و `. _ -` |
| `BOLT_ALLOWED_HOSTS` | مثلاً `bolt.example.com` | فقط برای دامنهٔ اختصاصی (با کاما جدا کنید) |
| `BOLT_NODE_HEAP_MB` | پیش‌فرض `384` | روی Pro تا حدود `512` |
| `WATCHDOG_TELEGRAM_TOKEN` و `WATCHDOG_TELEGRAM_CHAT_ID` | توکن ربات از @BotFather و chat id | اختیاری؛ هشدار به تلگرام |
| `WATCHDOG_WEBHOOK_URL` | آدرس webhook (Discord/Slack) | اختیاری |
| `WATCHDOG_INTERVAL_MS` / `WATCHDOG_MAX_FAILS` | 30000 / 3 | اختیاری؛ فاصلهٔ پایش و تعداد شکست مجاز |
| `WATCHDOG_PROBE_PATH` | پیش‌فرض `/api/health` | اختیاری؛ اگر 404 هم بدهد قبول است (فقط 5xx یا timeout شکست حساب می‌شود) |

متغیرهای Deplexo فقط موقع اجرا به کانتینر می‌رسند و این طراحی با آن سازگار است. کلید API مدل‌ها را اینجا نگذارید؛ در خود bolt.diy (Settings → Providers) وارد کنید، فقط در مرورگر شما ذخیره می‌شود.

7. Submit و دنبال‌کردن build log (چند دقیقه).

## ۶) تست بعد از deploy

1. آدرس app: صفحهٔ ورود می‌آید (همان `/` با کد 200، پس health check روی `/` هم رد می‌شود).
2. `/healthz` همیشه `ok`؛ `/readyz` فقط وقتی bolt.diy آماده است `ready`.
3. وارد شوید → `/__bolt/status` باید `"state": "ready"` و preflight همه `ok` باشد.
4. Settings → Providers → کلید → یک پرامپت ساده.
5. یکی از starter templateها (تست git proxy).
6. در log باید ببینید: `bolt.diy is READY` و `host header mode: preserve|rewrite`.

## ۷) ورود و امنیت

- ورود با فرم؛ کوکی امضاشده (HttpOnly، 7 روز). `curl -u user:pass` هم کار می‌کند.
- **8 رمز غلط** از یک IP یعنی قفل 15 دقیقه‌ای. کسی که کوکی معتبر دارد تحت‌تأثیر نمی‌شود.
- هدر Authorization مربوط به خود برنامه (مثل توکن git برای clone ریپوهای خصوصی) دست‌نخورده می‌ماند؛ هدر ورود ما هرگز به برنامه یا git host نمی‌رسد.
- خروج: `/__bolt/logout`.
- تغییر رمز همهٔ کوکی‌های قبلی را باطل می‌کند.
- رمز فقط در Environment، نه Git. `.env.local` و کلیدها را commit نکنید.

## ۸) عیب‌یابی (مرتب از رایج‌ترین)

| علامت | علت محتمل | راه‌حل |
|---|---|---|
| Build قطع می‌شود | `pnpm run build` سنگین است (heap تا 4 GB) | log را ببینید؛ اول دستور شکست‌خورده؛ از پشتیبانی Deplexo limit build بخواهید |
| `FATAL: set BOLT_AUTH_PASSWORD` | متغیر نیست یا کوتاه است | در Environment ذخیره و Restart/Redeploy |
| صفحهٔ ورود می‌آید ولی بعد از ورود «starting» می‌ماند | bolt.diy بالا نیامده | `/__bolt/logs` را بخوانید؛ preflight و خطای vite آنجاست |
| preflight: `writable` WARN | مسیر لینک‌نشده | مسیر را به لیست `for p in ...` در `Dockerfile.deplexo` و در آرایهٔ مسیرها در `deplexo-gateway.mjs` اضافه کنید |
| log: `EROFS` یا `read-only file system` | چیزی بیرون از `/tmp` می‌نویسد | همان مسیر را مثل بالا اضافه کنید |
| log: `vite: not found` | node_modules حذف شده | Dockerfile را دست‌کاری نکنید؛ `pnpm prune` ندارد |
| `memory high` / ری‌استارت‌های پی‌درپی | کمبود RAM | پلن Pro و `BOLT_NODE_HEAP_MB` پایین‌تر |
| `no HTTP answer within 180s` | crash یا کندی استارت | خطوط قبل از آن در `/__bolt/logs` |
| 502 | bolt.diy وسط کار افتاد | نگهبان خودش ری‌استارت می‌کند؛ log |
| Deplexo خودش کانتینر را قبول نمی‌کند | مثلاً health check روی مسیر دیگر | `/healthz` را معرفی کنید |
| clone ریپو 401 می‌دهد | کوکی ورود منقضی شده | صفحه را refresh و دوباره وارد شوید |

لاگ‌ها: داشبورد Deplexo یا `deplexo logs --follow` و `deplexo deployments logs <id>`.

## ۹) نکات فنی (برای ویرایش‌کننده)

- **Host header**: `vite preview` Host ناشناخته را بلاک می‌کند. دامنهٔ `.deplexo.com` و `BOLT_ALLOWED_HOSTS` مجاز اعلام می‌شوند و نگهبان هنگام استارت تست می‌کند؛ اگر بلاک شد Host را بازنویسی می‌کند.
- **فایل‌سیستم read-only**: `.wrangler/state`، `.wrangler/tmp`، `.react-router/types`، کش vite و HOME به `/tmp` لینک شده‌اند. محتوای زمان build در `/opt/seed` نگه داشته و در استارت بازگردانده می‌شود.
- `pnpm` موقع اجرا استفاده نمی‌شود (corepack می‌خواست دانلود کند). Vite مستقیم اجرا می‌شود.
- node_modules کامل نگه داشته شده (نبودن prune)؛ image بزرگ‌تر ولی بی‌خطا.

## ۱۰) به‌روزرسانی

- push با auto-deploy روشن، خودکار deploy می‌کند.
- upstream: `git remote add upstream https://github.com/stackblitz-labs/bolt.diy.git && git fetch upstream && git merge upstream/main && git push`.
- image تازه: Rebuild در داشبورد یا `deplexo deploy --rebuild`.
- هر deploy کانتینر قبلی را اول متوقف می‌کند؛ چند لحظه قطعی عادی است.

## ۱۱) محدودیت‌ها

- یک instance، بدون autoscaling، فقط region آلمان.
- چت‌ها، پروژه‌ها و تنظیمات در مرورگر هر کاربر می‌مانند؛ پاک‌کردن site data آن‌ها را پاک می‌کند.
- ورود مشترک است؛ حساب کاربری جدا ندارد.
- `/tmp` با هر ری‌استارت پاک می‌شود؛ برنامه به `/data` نیازی ندارد.

## ۱۲) تست محلی قبل از push

```bash
docker build -f Dockerfile.deplexo -t bolt-deplexo .
docker run --rm -p 8080:8080 \
  -e BOLT_AUTH_PASSWORD='a-long-random-password-123' \
  --read-only --tmpfs /tmp:rw,nosuid,noexec,size=512m \
  --cap-drop ALL --memory 1g bolt-deplexo
# http://localhost:8080 و http://localhost:8080/__bolt/status
```

این همان محدودیت‌های Deplexo را شبیه‌سازی می‌کند (read-only، `/tmp` غیرقابل اجرا، بدون capability). اگر محلی بالا آمد، روی Deplexo هم با احتمال بالا بالا می‌آید.

## ۱۳) منابع

[deplexo.yaml](https://docs.deplexo.com/reference/configuration/) · [Docker و build](https://docs.deplexo.com/guides/docker/) · [Environment](https://docs.deplexo.com/guides/environment/) · [چرخهٔ deploy](https://docs.deplexo.com/operations/deployments/) · [عیب‌یابی](https://docs.deplexo.com/operations/troubleshooting/)

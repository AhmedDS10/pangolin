ممتاز. سنعتمد الآن تصميم Pangolin + Coolify على نفس VPS وعنوان IPv4 دون إيقاف الخدمات الحالية.

سأستخدم نطاقاً فرعياً مستقلاً عن مواقعك الحالية حتى لا تتداخل قواعد التوجيه مع n8n و9Router وبقية التطبيقات:

التصميم المعتمد

Shared Proxy

Internet

13.140.180.244

Coolify Traefik

TCP/TLS Passthrough + HTTP Routing

80 / 443

خدمات Coolify

n8n · 9Router · Hermes

Pangolin

Gerbil + Traefik

Pangolin Traefik يحتفظ بإدارة شهادات HTTPS وتوجيه الموارد الخاصة به. أما Coolify فيمرر اتصال HTTPS إليه دون فك تشفيره.

سنستخدم:

- لوحة التحكم: `pangolin.tunnel.ahmedds.us`
- نطاق الموارد: `*.tunnel.ahmedds.us`
- WireGuard UDP: `51820` و`21820`
- Coolify Proxy: يبقى على المنافذ الحالية `80` و`443`.

هذا الفصل يحمي نطاقات خدماتك الحالية من التقاط Wildcard الخاص بـ Pangolin.&#x20;

## 1. إنشاء DNS

في Cloudflare أو مزود DNS الخاص بك، أضف:

| النوع | الاسم             | القيمة           |
| ----- | ----------------- | ---------------- |
| A     | `pangolin.tunnel` | `13.140.180.244` |
| A     | `*.tunnel`        | `13.140.180.244` |

اجعل السجلات DNS Only وليس Proxied.

لا نحتاج إلى Wildcard SSL من Cloudflare؛ سيصدر Pangolin شهاداته باستخدام Let's Encrypt عبر HTTP-01.

## 2. إعداد مجلد المشروع

أنصح بإنشاء مستودع Git خاص يحتوي على الملفات التالية، وربطه بمصدر Docker Compose في Coolify:

```
pangolin/
├── docker-compose.yml
└── config/
    ├── config.yml
    ├── db/
    ├── key
    ├── letsencrypt/
    └── traefik/
        ├── dynamic_config.yml
        └── traefik_config.yml
```

أضف إلى `.gitignore`:

```
config/db/
config/key
config/letsencrypt/
config/traefik/logs/
```

لا ترفع قاعدة البيانات أو المفتاح الخاص أو ملفات الشهادات إلى Git.

## 3. ملف Docker Compose

هذا الملف مبني على تركيب Pangolin الرسمي Community Edition، مع تعديل المنافذ والشبكات ليتوافق مع Coolify.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.pangolin.net\&sz=32)

Pangolin Docs+1



```

name: pangolin

services:
  pangolin:
    image: docker.io/fosrl/pangolin:latest
    restart: unless-stopped

    volumes:
      - ./config:/app/config

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/api/v1/"]
      interval: 10s
      timeout: 10s
      retries: 15

    networks:
      - pangolin

  gerbil:
    image: docker.io/fosrl/gerbil:latest
    restart: unless-stopped

    depends_on:
      pangolin:
        condition: service_healthy

    command:
      - --reachableAt=http://gerbil:3004
      - --generateAndSaveKeyTo=/var/config/key
      - --remoteConfig=http://pangolin:3001/api/v1/

    volumes:
      - ./config:/var/config

    cap_add:
      - NET_ADMIN
      - SYS_MODULE

    ports:
      - "51820:51820/udp"
      - "21820:21820/udp"

    networks:
      pangolin:
      coolify:
        aliases:
          - pangolin-gerbil

  traefik:
    image: docker.io/traefik:v3.7
    restart: unless-stopped

    network_mode: service:gerbil

    depends_on:
      pangolin:
        condition: service_healthy

    command:
      - --configFile=/etc/traefik/traefik_config.yml

    volumes:
      - ./config/traefik:/etc/traefik:ro
      - ./config/letsencrypt:/letsencrypt
      - ./config/traefik/logs:/var/log/traefik

networks:
  pangolin:
    name: pangolin_frontend
    driver: bridge

  coolify:
    external: true

```

ملاحظات مهمة:

- لم نربط TCP 80 أو 443 على عنوان VPS.
- Gerbil يستمع داخلياً إلى 80 و443 من خلال Network Namespace المشترك مع Traefik.
- أضفنا Gerbil إلى شبكة `coolify` ليتمكن Coolify Proxy من الوصول إليه.
- أبقينا منافذ WireGuard مكشوفة مباشرة عبر UDP.

## 4. ملف Pangolin config.yml

المسار:

`config/config.yml`

```

gerbil:
  start_port: 51820
  base_endpoint: "pangolin.tunnel.ahmedds.us"

app:
  dashboard_url: "https://pangolin.tunnel.ahmedds.us"
  log_level: "info"
  telemetry:
    anonymous_usage: true

domains:
  domain1:
    base_domain: "tunnel.ahmedds.us"

server:
  secret: "REPLACE_WITH_RANDOM_SECRET"

  cors:
    origins:
      - "https://pangolin.tunnel.ahmedds.us"
    methods:
      - GET
      - POST
      - PUT
      - DELETE
      - PATCH
    allowed_headers:
      - X-CSRF-Token
      - Content-Type
    credentials: false

flags:
  require_email_verification: false
  disable_signup_without_invite: true
  disable_user_create_org: false
  allow_raw_resources: true

```

ولإنشاء المفتاح السري:

```
openssl rand -hex 32
```

ضع الناتج مكان `REPLACE_WITH_RANDOM_SECRET`.

## 5. إعداد Traefik الخاص بـ Pangolin

المسار:

`config/traefik/traefik_config.yml`

```

api:
  insecure: true
  dashboard: true

providers:
  http:
    endpoint: "http://pangolin:3001/api/v1/traefik-config"
    pollInterval: "5s"

  file:
    filename: "/etc/traefik/dynamic_config.yml"

experimental:
  plugins:
    badger:
      moduleName: "github.com/fosrl/badger"
      version: "v1.4.0"

log:
  level: INFO
  format: common
  maxSize: 100
  maxBackups: 3
  maxAge: 3
  compress: true

certificatesResolvers:
  letsencrypt:
    acme:
      httpChallenge:
        entryPoint: web
      email: "YOUR_EMAIL@example.com"
      storage: "/letsencrypt/acme.json"
      caServer: "https://acme-v02.api.letsencrypt.org/directory"

entryPoints:
  web:
    address: ":80"

  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        readTimeout: "30m"
    http:
      tls:
        certResolver: letsencrypt
      encodedCharacters:
        allowEncodedSlash: true
        allowEncodedQuestionMark: true

serversTransport:
  insecureSkipVerify: true

ping:
  entryPoint: web

```

استبدل البريد بعنوان بريدك الحقيقي حتى تستطيع Let's Encrypt إرسال إشعارات الشهادات.

## 6. إعداد التوجيه الداخلي لـ Pangolin

المسار:

`config/traefik/dynamic_config.yml`

```

http:
  middlewares:
    badger:
      plugin:
        badger:
          disableForwardAuth: true

    redirect-to-https:
      redirectScheme:
        scheme: https

  routers:
    main-app-router-redirect:
      rule: "Host(`pangolin.tunnel.ahmedds.us`)"
      service: next-service
      entryPoints:
        - web
      middlewares:
        - redirect-to-https
        - badger

    next-router:
      rule: "Host(`pangolin.tunnel.ahmedds.us`) && !PathPrefix(`/api/v1`)"
      service: next-service
      entryPoints:
        - websecure
      middlewares:
        - badger
      tls:
        certResolver: letsencrypt

    api-router:
      rule: "Host(`pangolin.tunnel.ahmedds.us`) && PathPrefix(`/api/v1`)"
      service: api-service
      entryPoints:
        - websecure
      middlewares:
        - badger
      tls:
        certResolver: letsencrypt

    ws-router:
      rule: "Host(`pangolin.tunnel.ahmedds.us`)"
      service: api-service
      entryPoints:
        - websecure
      middlewares:
        - badger
      tls:
        certResolver: letsencrypt

  services:
    next-service:
      loadBalancer:
        servers:
          - url: "http://pangolin:3002"

    api-service:
      loadBalancer:
        servers:
          - url: "http://pangolin:3000"

tcp:
  serversTransports:
    pp-transport-v1:
      proxyProtocol:
        version: 1

    pp-transport-v2:
      proxyProtocol:
        version: 2

```

هذا الملف مسؤول عن توجيه لوحة Pangolin وواجهات API وWebSocket. أما توجيه الموارد فيأتي من Pangolin عبر HTTP Provider.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.pangolin.net\&sz=32)

Pangolin Docs+1



## 7. ربط Coolify Proxy مع Pangolin

هذه أهم خطوة في التصميم.

من لوحة Coolify:

1. افتح Servers.
2. اختر VPS.
3. افتح Proxy → Dynamic Configurations.
4. اختر Add.
5. اسم الملف: `pangolin.yml`.
6. أضف المحتوى التالي.

```

http:
  routers:
    pangolin-http:
      rule: "HostRegexp(`^[a-z0-9-]+\\.tunnel\\.ahmedds\\.us$`)"
      entryPoints:
        - web
      service: pangolin-http-service
      priority: 10000

  services:
    pangolin-http-service:
      loadBalancer:
        servers:
          - url: "http://pangolin-gerbil:80"

tcp:
  routers:
    pangolin-https:
      rule: "HostSNI(`*.tunnel.ahmedds.us`)"
      entryPoints:
        - websecure
      service: pangolin-https-service
      tls:
        passthrough: true

  services:
    pangolin-https-service:
      loadBalancer:
        servers:
          - address: "pangolin-gerbil:443"

```

كيف يعمل هذا الإعداد؟

- طلب HTTP يصل إلى Pangolin Traefik.
- طلب HTTPS يمرر عبر Coolify دون فك تشفير TLS.
- Pangolin Traefik يتولى شهادات SSL والمصادقة والتوجيه.
- جميع نطاقات `*.tunnel.ahmedds.us` تصل إلى Pangolin.

Coolify يدعم Dynamic Configuration وTCP Routers، لكن يجب التأكد من أن حاوية Gerbil متصلة فعلياً بشبكة `coolify`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://coolify.io\&sz=32)

Coolify Docs+1



## 8. النشر عبر Coolify

قبل النشر، نفّذ:

```
docker network inspect coolify
```

يجب أن تكون الشبكة موجودة. إذا لم تكن موجودة، لا تنشئ شبكة بديلة عشوائياً؛ تحقق من شبكة Coolify الفعلية أولاً.

بعد ذلك:

1. ارفع ملفات المشروع إلى Git Repository خاص.
2. من Coolify اختر Add Resource.
3. اختر Docker Compose.
4. اربط مستودع Git.
5. حدد مسار `docker-compose.yml`.
6. اضغط Deploy.

## 9. فتح منافذ Firewall

```
sudo ufw allow 51820/udp
sudo ufw allow 21820/udp
```

لا تضف قواعد جديدة للمنافذ 80 و443؛ فهي مفتوحة بالفعل لخدمة Coolify Proxy.

## 10. التحقق من التشغيل

```
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

ثم تحقق من السجلات:

```
docker logs $(docker ps -q \
  --filter ancestor=fosrl/pangolin:latest) --tail 100
```

بعد نجاح التشغيل، افتح:

`https://pangolin.tunnel.ahmedds.us/auth/initial-setup`

واستخدم Setup Token الموجود في سجلات Pangolin لإنشاء حساب المدير الأول.

## 11. قبل تنفيذ Deploy

هناك نقطتان أحتاج تأكيدهما حتى لا نواجه أخطاء DNS أو Let's Encrypt:

1\. البريد الإلكتروني لشهادات SSL

2\. مزود DNS

Cloudflare

مزود النطاق

مزود آخر

3\. شبكة Docker الخاصة بـ Coolify

موجودة: coolify

اسم مختلف

لم أفحص الشبكة بعد

تأكيد إعدادات النشر

تنبيه تقني: هذا تكامل مخصص بين Traefik اثنين، وليس إعداد Pangolin الرسمي الافتراضي حرفياً. يجب اختبار توجيه HTTP وTCP/TLS قبل الاعتماد عليه، خصوصاً أن خطأ في قواعد TCP قد يؤثر في وصول نطاقات Pangolin، لكنه لا يستلزم إيقاف Coolify Proxy نفسه.

🌐 Умный обход блокировок Gemini (Xray Advanced)
Этот гайд описывает, как настроить умную маршрутизацию для Google Gemini.
Идея в том, чтобы клиент подключался к любому основному серверу (например, Нидерланды или Финляндия), но запросы к Gemini автоматически и незаметно уходили через специально выделенный сервер (например, Турцию), который не заблокирован для этого сервиса.

Вся магия настраивается на стороне панели Remnawave, и клиентам в приложении (Happ) не нужно вручную вписывать ключи или UUID.

⚠️ Важное условие
Сервер, который будет использоваться для Gemini (в примере — Турция), обязан сам по себе работать с Gemini. Если Google заблокировал IP этого сервера, никакая маршрутизация не поможет. Сначала проверьте работу Gemini на этом сервере напрямую.

🛠️ Шаг 1: Настройка хостов в Remnawave
Вам понадобятся два хоста в панели:

Основной хост (Виртуальный): Тот сервер, который клиент будет выбирать в приложении по умолчанию (например, Netherlands). К нему будет привязан шаблон подписки.

Сервер для Gemini (Скрытый): Тот сервер, через который должен идти трафик Gemini (например, Turkey).

В его настройках включите переключатель Hide host (Скрыть хост), чтобы он не отображался в обычном списке серверов.

В поле Tags (Теги) задайте уникальное имя, например: TURKEY (регистр важен).

📄 Шаг 2: Настройка шаблона подписки (Xray JSON Advanced)
Зайдите в настройки Шаблона подписки (Subscription Template) типа XRAY_JSON, который привязан к вашему Основному хосту. Вставьте туда следующий конфиг:

json
{
  "remnawave": {
    "addVirtualHostAsOutbound": true,
    "injectHosts": [
      {
        "selector": {
          "type": "tagRegex",
          "pattern": "^TURKEY$"
        },
        "tagPrefix": "turkey"
      }
    ]
  },
  "dns": {
    "servers": ["77.88.8.8", "1.1.1.1"],
    "queryStrategy": "UseIPv4"
  },
  "log": {
    "loglevel": "warning"
  },
  "routing": {
    "rules": [
      {
        "type": "field",
        "domain": [
          "gemini.google.com",
          "aistudio.google.com",
          "generativelanguage.googleapis.com",
          "alkalimining-pa.googleapis.com",
          "proactivebackend-pa.googleapis.com",
          "alkalimakersuite-pa.clients6.google.com",
          "makersuite.google.com",
          "ai.google.dev",
          "bard.google.com",
          "generativeai.google",
          "deepmind.com",
          "deepmind.google",
          "jnn-pa.googleapis.com",
          "waa-pa.clients6.google.com",
          "optimizationguide-pa.googleapis.com",
          "content-autofill.googleapis.com",
          "ogads-pa.clients6.google.com",
          "robinfrontend-pa.googleapis.com"
        ],
        "outboundTag": "turkey"
      },
      {
        "type": "field",
        "domain": [
          "geosite:category-ads",
          "geosite:win-spy",
          "geosite:torrent"
        ],
        "outboundTag": "block"
      },
      {
        "type": "field",
        "protocol": ["bittorrent"],
        "outboundTag": "block"
      },
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "geosite:private",
          "geosite:whitelist",
          "geosite:category-ru"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "proxy"
      }
    ],
    "domainStrategy": "IPIfNonMatch"
  },
  "inbounds": [
    {
      "tag": "socks",
      "port": 10808,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": { "udp": true, "auth": "noauth" },
      "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
    },
    {
      "tag": "http",
      "port": 10809,
      "listen": "127.0.0.1",
      "protocol": "http",
      "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
    }
  ],
  "outbounds": [
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ]
}
🧠 Как это работает (Разбор конфига):
addVirtualHostAsOutbound: true: Панель берет основной хост, к которому привязан этот шаблон, и создает для него Outbound с тегом proxy.

injectHosts: Панель ищет скрытый хост с тегом TURKEY, вытаскивает его ключи, UUID и адрес, и создает для него Outbound с тегом turkey.

routing.rules:

Правило Gemini стоит первым. Оно перехватывает весь трафик на домены нейросети и отправляет в turkey.

Остальной трафик падает в proxy (ваш основной сервер).

Российские сайты идут в direct (напрямую).

Реклама и торренты блокируются (block).

Почему в dns стоит UseIPv4?
Google часто отдает приоритет IPv6-адресам, которые могут иметь неправильную геолокацию. UseIPv4 заставляет Xray использовать только проверенные IPv4-адреса.

📱 Шаг 3: Настройка клиента (Happ)
Клиентам не нужно вручную вписывать UUID или ключи.

Клиент импортирует подписку от Remnawave.

В приложении (например, Happ) он просто выбирает свой основной сервер (например, Netherlands).

При обращении к Gemini, приложение автоматически (на основе JSON выше) отправляет этот трафик на турецкий IP. Для пользователя это выглядит так, будто всё работает через Нидерланды, но Gemini открывается без ошибок.

💡 Полезные советы
Если Gemini не открывается, проверьте, что в панели Remnawave хост с тегом TURKEY добавлен в тот же Squad (Сквад), что и основной хост.

Для других сервисов (например, ChatGPT или Netflix) можно создать аналогичные скрытые хосты и добавить правила для их доменов в routing.rules. Главное — ставить эти правила выше, чем общие правила для всего трафика.

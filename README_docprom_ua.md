pulse
========

Рішення для моніторингу хостів і контейнерів Docker з [Prometheus] (https://prometheus.io/), [Grafana] (http://grafana.org/), [cAdvisor] (https://github.com/google/cadvisor),
[NodeExporter] (https://github.com/prometheus/node_exporter) та попередження за допомогою [AlertManager] (https://github.com/prometheus/alertmanager).


## Встановити

Клонуйте це сховище на хості Docker, cd в каталог dockprom і запустіть compose up:

`` bash
git clone https://github.com/joyshmitz/pulse
cd pulse

ADMIN_USER=admin ADMIN_PASSWORD=admin docker-compose up -d

``
Передумови:

* Docker Engine >= 1.13
* Docker Compose >= 1.11


Контейнери:

* Prometheus (metrics database) http://<host-ip>:9090
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) http://<host-ip>:9091
* AlertManager (alerts management) http://<host-ip>:9093
* Grafana (visualize metrics) http://<host-ip>:3000
* NodeExporter (host metrics collector)
* cAdvisor (containers metrics collector)
* Caddy (reverse proxy and basic auth provider for prometheus and alertmanager)

## Налаштування Grafana

Перейдіть до `http://<host-ip>:3000` і увійдіть з користувачем *** admin *** пароль *** admin ***. Ви можете змінити облікові дані у файлі написання або ввівши змінні середовища `ADMIN_USER` та` ADMIN_PASSWORD` для створення. Файл конфігурації можна додати безпосередньо в частину grafana, як це,
``
grafana:
  image: grafana/grafana:latest
  env_file:
    - config


``
і формат файлу конфігурації повинен мати такий вміст
``
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
``
Якщо ви хочете змінити пароль, вам доведеться видалити цей запис, інакше зміна не набере чинності
``
- grafana_data:/var/lib/grafana
``

Графана попередньо налаштована на інформаційні панелі та Прометей як джерело даних за замовчуванням:

* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy

*** Інформаційна панель хоста Docker ***

! [Ведучий] (screenshot/Grafana_Docker_Host.png)

Інформаційна панель Docker Host показує ключові показники для моніторингу використання ресурсів вашого сервера:

* Час безвідмовної роботи сервера, відсоток простою процесора, кількість ядер процесора, доступна пам'ять, своп та пам'ять
* Графік середнього навантаження системи, який працює і блокується графіком процесів введення / виведення, графік переривань
* Графік використання процесора за режимами (гість, простой, iowait, irq, nice, softirq, викрасти, система, користувач)
* Графік використання пам'яті за розподілом (використаний, безкоштовний, буфери, кешований)
* Графік використання вводу-виводу (читайте Bps, читайте Bps і час вводу-виводу)
* Графік використання мережі за пристроєм (вхідні Bps, вихідні Bps)
* Поміняйте місцями графіки використання та активності

Для графіку обєму та особливо графіку вільного обєму, вам потрібно вказати тип файлової системи 'fstype' у запиті графа grafana.
Ви можете знайти його в `grafana/dashboards/Host.json`, у рядку 480:

      "expr": "сума (node_filesystem_free_bytes {fstype = \" btrfs \ "})",

Я працюю на BTRFS, тому мені потрібно змінити `aufs` на` btrfs`.

Ви можете знайти правильне значення для вашої системи в Prometheus `http: // <host-ip>: 9090`, що запускає такий запит:

      node_filesystem_free_bytes

*** Інформаційна панель контейнерів Docker ***

! [Контейнери] (https://raw.githubusercontent.com/stefanprodan/dockprom/master/screens/Grafana_Docker_Containers.png)

Інформаційна панель контейнерів Docker відображає ключові показники для моніторингу запущених контейнерів:

* Загальний обсяг завантаження процесора, пам’яті та пам'яті
* Графік запущених контейнерів, графік завантаження системи, графік використання IO
* Графік використання процесора контейнера
* Графік використання пам'яті контейнера
* Графік використання кешованої пам'яті контейнера
* Графік вхідного використання мережі контейнерів
* Графік використання вихідних мереж контейнерів

Зверніть увагу, що на цій інформаційній панелі не відображаються контейнери, які є частиною стека моніторингу.

*** Інформаційна панель служб моніторингу ***

! [Послуги монітора] (https://raw.githubusercontent.com/stefanprodan/dockprom/master/screens/Grafana_Prometheus.png)

Інформаційна панель служб моніторингу показує ключові показники для моніторингу контейнерів, що складають стек моніторингу:

* Час роботи контейнера Prometheus, моніторинг загального використання пам'яті стека, фрагменти та серії локальної пам'яті Prometheus
* Графік використання процесора контейнера
* Графік використання пам'яті контейнера
* Прометей шматки, щоб зберегти і наполегливі термінові графіки
* Прометей фрагменти операцій та графіки тривалості контрольної точки
* Графіки швидкості поглинання зразків Прометей, цільові подряпини та графіки тривалості зішкрябу
* Графік запитів прометей HTTP
* Графік попереджень Прометея

## Визначте попередження

У конфігураційному файлі [alert.rules] (https://github.com/stefanprodan/dockprom/blob/master/prometheus/alert.rules) було налаштовано три групи попереджень:

* Сповіщення служб моніторингу [цілі] (https://github.com/stefanprodan/dockprom/blob/master/prometheus/alert.rules#L2-L11)
* Сповіщення про хост Docker [хост] (https://github.com/stefanprodan/dockprom/blob/master/prometheus/alert.rules#L13-L40)
* Сповіщення про контейнери Docker [контейнери] (https://github.com/stefanprodan/dockprom/blob/master/prometheus/alert.rules#L42-L69)

Ви можете змінити правила попередження та перезавантажити їх, зробивши HTTP POST виклик Prometheus:

``
curl -X POST http: // admin: admin @ <host-ip>: 9090 / - / reload
``

*** Оповіщення служб моніторингу ***

Зробіть попередження, якщо будь-яка з цілей моніторингу (node-exporter і cAdvisor) не працює більше 30 секунд:

`` ямл
- попередження: monitor_service_down
    вираження: вгору == 0
    для: 30с
    етикетки:
      тяжкість: критична
    анотації:
      резюме: "Служба моніторингу не працює"
      опис: "Послуга {{$ labels.instance}} не працює."
``

*** Сповіщення про хост Docker ***

Зробіть попередження, якщо центральний процесор Docker знаходиться під великим навантаженням більше 30 секунд:

`` ямл
- попередження: high_cpu_load
    вираз: node_load1> 1.5
    для: 30с
    етикетки:
      тяжкість: попередження
    анотації:
      резюме: "Сервер під великим навантаженням"
      опис: "Хост Docker знаходиться під великим навантаженням, середнє навантаження 1м дорівнює {{$ value}}. Повідомляється екземпляром {{$ labels.instance}} завдання {{$ labels.job}}."
``

Змініть поріг навантаження на основі ядер вашого процесора.

Запустіть попередження, якщо пам’ять хоста Docker майже заповнена:

`` ямл
- попередження: high_memory_load
    вираз: (сума (node_memory_MemTotal_bytes) - сума (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / sum (node_memory_MemTotal_bytes) * 100> 85
    для: 30с
    етикетки:
      тяжкість: попередження
    анотації:
      резюме: "Пам'ять сервера майже заповнена"
      опис: "Використання пам'яті хоста Docker становить {{humanize $ value}}%. Повідомляється екземпляром {{$ labels.instance}} завдання {{$ labels.job}}."
``

Зробіть попередження, якщо сховище хоста Docker майже заповнене:

`` ямл
- попередження: high_storage_load
    вираз: (node_filesystem_size_bytes {fstype = "aufs"} - node_filesystem_free_bytes {fstype = "aufs"}) / node_filesystem_size_bytes {fstype = "aufs"} * 100> 85
    для: 30с
    етикетки:
      тяжкість: попередження
    анотації:
      резюме: "Пам'ять сервера майже заповнена"
      опис: "Використання сховища хоста Docker становить {{humanize $ value}}%. Повідомляється екземпляром {{$ labels.instance}} завдання {{$ labels.job}}."
``

*** Оповіщення про контейнери Docker ***

Зробіть попередження, якщо контейнер не працює більше 30 секунд:

`` ямл
- попередження: jenkins_down
    вираз: відсутній (контейнер_пам'ять_байт_байт {name = "jenkins"})
    для: 30с
    етикетки:
      тяжкість: критична
    анотації:
      резюме: "Дженкінс вниз"
      опис: "Контейнер Дженкінса не працює більше 30 секунд."
``

Зробіть попередження, якщо контейнер використовує більше 10% загальної кількості ядер процесора більше 30 секунд:

`` ямл
- попередження: jenkins_high_cpu
    вираз: сума (ставка (контейнер_cpu_usage_seconds_total {name = "jenkins"} [1м])) / count (node_cpu_seconds_total {mode = "system"}) * 100> 10
    для: 30с
    етикетки:
      тяжкість: попередження
    анотації:
      резюме: "Дженкінс з високим використанням процесора"
      опис: "Використання центрального процесора Дженкінса становить {{humanize $ value}}%."
``

Зробіть попередження, якщо контейнер використовує більше 1,2 ГБ оперативної пам'яті більше 30 секунд:

`` ямл
- попередження: jenkins_high_memory
    вираз: сума (container_memory_usage_bytes {name = "jenkins"})> 1200000000
    для: 30с
    етикетки:
      тяжкість: попередження
    анотації:
      резюме: "Дженкінс з великим використанням пам'яті"
      опис: "Споживання пам'яті Дженкінса становить {{humanize $ value}}."
``

## Попередження про налаштування

Служба AlertManager відповідає за обробку сповіщень, надісланих сервером Prometheus.
AlertManager може надсилати сповіщення електронною поштою, Pushover, Slack, HipChat або будь-якою іншою системою, яка відкриває інтерфейс веб-хука.
Повний перелік інтеграцій можна знайти [тут] (https://prometheus.io/docs/alerting/configuration).

Ви можете переглядати та мовчати сповіщення, відкриваючи "http: // <host-ip>: 9093".

Приймачі сповіщень можна налаштувати у файлі [alertmanager / config.yml] (https://github.com/stefanprodan/dockprom/blob/master/alertmanager/config.yml).

Щоб отримувати сповіщення через Slack, потрібно виконати власну інтеграцію, вибравши *** вхідні веб-хуки *** на сторінці програми Slack.
Детальнішу інформацію про налаштування інтеграції Slack можна знайти [тут] (http://www.robustperception.io/using-slack-with-the-alertmanager/).

Скопіюйте URL-адресу Slack Webhook в поле *** api_url *** і вкажіть канал Slack *** ***.

`` ямл
маршрут:
    приймач: "слабкий"

приймачі:
    - ім'я: 'слабий'
      slack_configs:
          - send_resolved: true
            текст: "{{.CommonAnnotations.description}}"
            ім'я користувача: 'Прометей'
            channel: '# <канал>'
            api_url: 'https://hooks.slack.com/services/ <webhook-id>'
``

! [Ослаблені сповіщення] (https://raw.githubusercontent.com/stefanprodan/dockprom/master/screens/Slack_Notifications.png)

## Надсилання метрик в Pushgateway

[Pushgateway] (https://github.com/prometheus/pushgateway) використовується для збору даних із пакетних завдань або служб.

Для передачі даних просто виконайте:

    ехо "some_metric 3.14" | curl --data-binary @ - http: // user: password @ localhost: 9091 / metrics / job / some_job

Будь ласка, замініть частину `user: password` вашим користувачем та паролем, встановленими у початковій конфігурації (за замовчуванням:` admin: admin`).

## Оновлення Grafana до версії 5.2.2

[У версіях Grafana> = 5.1 змінено ідентифікатор користувача grafana] (http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-the-docker-container -до-5-1-або пізніше). На жаль, це означає, що файли, створені до версії 5.1, не матимуть належних дозволів для пізніших версій.

| Версія | Користувач | Ідентифікатор користувача |
|: -------: |: -------: |: -------: |
| <5,1 | графана | 104 |
| \> = 5,1 | графана | 472 |

Існує два можливих рішення цієї проблеми.
- Змінити власника з 104 на 472
- Запустіть оновлений контейнер як користувач 104

##### Вказівка ​​користувача в docker-compose.yml

Щоб змінити право власності на файли, запустіть контейнер grafana як root і змініть дозволи.

Спочатку виконайте `docker-compose down`, а потім змініть ваш docker-compose.yml, щоб включити опцію` user: root`:

``
  графана:
    зображення: графана / графана: 5.2.2
    ім'я_контейнера: графана
    обсяги:
      - grafana_data: / var / lib / grafana
      - ./grafana/datasources:/etc/grafana/datasources
      - ./grafana/dashboards:/etc/grafana/dashboards
      - ./grafana/setup.sh:/setup.sh
    точка входу: /setup.sh
    користувач: root
    середовище:
      - GF_SECURITY_ADMIN_USER = $ {ADMIN_USER: -admin}
      - GF_SECURITY_ADMIN_PASSWORD = $ {ADMIN_PASSWORD: -admin}
      - GF_USERS_ALLOW_SIGN_UP = хибне
    перезапуск: якщо не зупинено
    виставити:
      - 3000
    мережі:
      - монітор-мережа
    етикетки:
      org.label-schema.group: "моніторинг"
``

Виконайте `docker-compose up -d`, а потім виконайте такі команди:

``
docker exec -it --користувач корінь графана баш

# у щойно запущеному контейнері:
chown -R корінь: root / etc / grafana && \
chmod -R a + r / etc / grafana && \
chown -R графана: grafana / var / lib / grafana && \
чаун -R графана: grafana / usr / share / grafana
``

Щоб запустити контейнер grafana як `user: 104`, змініть свій` docker-compose.yml` таким чином:

``
  графана:
    зображення: графана / графана: 5.2.2
    ім'я_контейнера: графана
    обсяги:
      - grafana_data: / var / lib / grafana
      - ./grafana/datasources:/etc/grafana/datasources
      - ./grafana/dashboards:/etc/grafana/dashboards
      - ./grafana/setup.sh:/setup.sh
    точка входу: /setup.sh
    користувач: "104"
    середовище:
      - GF_SECURITY_ADMIN_USER = $ {ADMIN_USER: -admin}
      - GF_SECURITY_ADMIN_PASSWORD = $ {ADMIN_PASSWORD: -admin}
      - GF_USERS_ALLOW_SIGN_UP = хибне
    перезапуск: якщо не зупинено
    виставити:
      - 3000
    мережі:
      - монітор-мережа
    етикетки:
      org.label-schema.group: "моніторинг"
``
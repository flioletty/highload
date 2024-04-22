# Проектирование высоконагруженного почтового сервиса
Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана ("Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"
##### Автор: Степаненко Полина
В качестве примера выбран сервис [Mail.ru](https://mail.ru/)

## Содержание

* ### [Тема и целевая аудитория](#1)
* ### [Расчет нагрузки](#2)
* ### [Глобальная балансировка нагрузки](#3)
* ### [Локальная балансировка нагрузки](#4)
* ### [Логическая схема базы данных](#5)
* ### [Физическая схема базы данных](#6)
* ### [Алгоритмы](#7)
* ### [Технологии](#8)
* ### [Схема проекта](#9)
* ### [Обеспечение надежности](#10)

## 1. Тема и целевая аудитория <a name="1"></a>

**Mail.ru** — почтовый сервис, с возможностью чтения и отправки (с вложениями) писем.

### MVP

- Просмотр списка писем
- Чтение писем
- Отправка писем
- Прикрепление файлов(изображений, архивов)
- Удаление писем из интерфейса
- Распределение писем по разделам (отправленные, спам, соцсети и тд)
- Получение статистики по письмам:
  - Общее количество писем, отправленных на Mail.ru, за выбранный период;
  - Жалобы — количество писем, на которые пожаловались пользователи (нажали на кнопку «Спам»);
  - Прочитанные 
  - Распределение доставленных писем по папкам в процентах


### Ключевые продуктовые решения

- Алгоритм сортировки писем по разделам (антиспам например)
- Сохранение черновика написанного сообщения при закрытом интерфейсе (например при переходе между вкладками в браузере)

### Целевая аудитория

- Россия, страны СНГ
- Средняя месячная аудитория 49.5 млн человек (по результатам 3 квартала 2023 года) [^1]
- Средняя дневная аудитория 16,5 млн (по результатам 3 квартала 2023 года) [^1]

### Статистика за 2024 год
- Доля аудитории:
  - Мужской: 45.85%
  - Женской: 54.15%
- По данным Mediascope[^5], аудитория mail.ru в России составляет 76 млн человек в месяц. Mail.ru охватывает 62.6% населения России. 20.2% россиян используют mail.ru каждый день. Это примерно 30 млн пользователей.
- География пользователей mail.ru выглядит так[^6]:
  
  ![image](https://github.com/flioletty/highload/assets/92665311/28be2227-887a-4882-b0a0-a9e5bd42fb12)


## 2. Расчет нагрузки <a name="2"></a>

### Продуктовые метрики

|Метрика|Значение|Обозначение|
| ------------- | -------------|--|
|MAU (чел.)[^1]|49.5 млн.|MAU|
|DAU (чел.)[^1]|16.5 млн.|DAU|
|Средний размер хранилища[^2]|8Гб|SIZE_ALL|
|Средний размер 1 письма[^4]|50Kб|SIZE_LETTER|
|Отправлено писем в день[^3]|40|LETTERS_SENT|
|Получено писем в день[^3]|121|LETTERS_GET|
|Прикреплено файлов в день|30%|LETTERS_FILES|
|Размер письма с вложениями[^4]|1Мб|LETTERS_FILES_SIZE|
|Писем на странице|25|LETTERS_PER_PAGE|

Среднее количество пользователей в минуту: 1 704 250[^7]

Пиковое количество пользователей в минуту: 2 546 335

Пиковая активность в 2546335 / 1704250 ~ 1,49 раз выше среденй

### Технические метрики

#### Среднее количество действий пользователя по типам в день (RPS) (согласно MVP)

- <b>Просмотр списка писем</b>

Объем данных  `LETTERS_PER_PAGE * SIZE_LETTER = 25 * 50 = 1250 KB`:

      RPS = (16.5 * 10^6 * 25)/(24 * 60 * 60) ~ 4774

      Трафик: 4774 * 1250 ~ 5 967 500 Kbit/sec ~ 5,7 Gbit/sec

      Пиковое значение: 1,49 * 5,7 ~ 8,51 Gbit/sec

- <b>Отправка писем</b>

Объем данных `LETTERS_SENT * SIZE_LETTER = 40 * 50 = 2000KB`:

      RPS = (16.5 * 10^6 * 40)/(24 * 60 * 60) ~ 7638

      Трафик: 7638 * 2000 ~ 15 276 000 Kbit/sec ~ 14,5 Gbit/sec

      Пиковое значение: 1,49 * 14,5 ~ 21,66 Gbit/sec

- <b>Чтение писем</b>

Объем данных `LETTERS_GET * SIZE_LETTER = 121 * 50 = 6050 KB`:

      RPS = (16.5 * 10^6 * 121)/(24 * 60 * 60) ~ 23108

      Трафик: 23108 * 6050 ~ 139 803 400 Kbit/sec ~ 133 Gbit/sec

      Пиковое значение: 1,49 * 133 ~ 198,7 Gbit/sec

- <b>Скачивание файлов(изображений, архивов)</b>

В среднем LETTERS_GET * LETTER_FILES = 121 * 0.33 полученных письма имеют вложение, объем данных `LETTERS_GET * LETTER_FILES * LETTER_FILES_SIZE = 121 * 0.33 * 1024KB = 40888KB`, тогда:

      RPS = (16.5 * 10^6 * 121 * 0,33)/(24 * 60 * 60) ~ 7625

      Трафик: 7625 * 40888 ~ 311 792 295 Kbit/sec ~ 297 Gbit/sec

      Пиковое значение (пункт 6): 1,49 * 297 ~ 443,75 Gbit/sec

- <b>Прикрепление файлов(изображений, архивов)</b>

В среднем LETTERS_SENT * LETTER_FILES = 40 * 0.33 отправленное письмо имеет вложение(пункт 1), объем данных `LETTERS_SENT * LETTER_FILES * LETTER_FILES_SIZE = 40 * 0.33 * 1024KB = 13517KB`, тогда:

     RPS = (16.5 * 10^6 * 40 * 0,33)/(24 * 60 * 60) ~ 2520

      Трафик: 2520 * 13517 ~ 34 071 776 Kbit/sec ~ 32,5 Gbit/sec

      Пиковое значение (пункт 6): 1,49 * 32,5 ~ 48,6 Mbit/sec

- <b>Удаление писем</b>

Допустим, пользователь в среднем удаляет 1 письмо в месяц, это LETTERS_DELETED ~ 0,033 письма в день

Объем данных `LETTERS_DELETED * SIZE_LETTER = 0,033 * 50 = 1,7KB`:

      RPS = (16.5 * 10^6 * 0,033)/(24 * 60 * 60) ~ 6,37

      Трафик: 6,37 * 1,7 ~ 10,82 Kbit/sec ~ 0,01 Gbit/sec 

      Пиковое значение: 1,49 * 0,01 ~ 0,016 Gbit/sec

- <b>Получение статистики</b>

Допустим, пользователь в среднем смотрит статистику 2-3 раза в неделю, это STATISTICS ~ 0,36 раз в день

      RPS = (16.5 * 10^6 * 0,36)/(24 * 60 * 60) ~ 68,75

      Пиковое значение: 1,49 * 68,75 ~ 102,43

**Хранилище**

В среднем пользователь отправляет в день писем на `LETTERS_SENT * ( SIZE_LETTER * 0,66 + LETTER_FILES * 0,33 ) = 40 * ( 50 * 0,66 + 1024 * 0,33 ) = 14987KB`, а удаляет на 1,7Кб (расчеты см выше).
Прирост объема данных от одного пользователя 14987Kб - 1,7Кб ~ 14 985Кб. То есть в год со всех пользователей почты прибавится `16.5 * 10^6 * 14985 * 365 = 82 Пб/год`

**Результаты расчетов**

| Запрос | RPS средний | RPS пиковый | Потребляемый трафик (средний) | Потребляемый трафик (пиковый) |
|--------|-------------|-------------|-------------------------------|-------------------------------|
|Просмотр списка писем|4774|9548|5,7 Gbit/sec|11,4 Gbit/sec|
|Отправка писем|7638|15276|14,5 Gbit/sec|29 Gbit/sec|
|Чтение писем|23108|46216|133 Gbit/sec|266 Gbit/sec|
|Скачивание файлов|7625|15250|297 Gbit/sec|594 Gbit/sec|
|Прикрепление файлов|2520|5040|32,5 Gbit/sec|65 Gbit/sec|
|Удаление файлов|6,37|9,49|0,01 Gbit/sec|0,016 Gbit/sec|
|Просмотр статистики|68,75|102,43|-|-|

## 3. Глобальная балансировка нагрузки <a name="3"></a>

### Расположение 

Целевая аудитория сервиса расположениа в России и странах СНГ, будем располагать сервера в России.

Судя по плотности населения в России
![image](https://github.com/flioletty/highload/assets/92665311/72c8d78d-6f2c-461f-9dbd-32f43985623a)

Основная часть нагрузки приходится на Центральный регион - разместим один датацентр в **Москве**. 

Если учесть что для электронной почты не особо важна низкая latency, так как большинство пользователей проверяют почту раз в день и не ведут активные переписки, в которых будет сильно заметна задержка, а также статистику распределения аудитории по регионам:
![image](https://github.com/flioletty/highload/assets/92665311/7b15290b-5178-473e-8cfa-426ff9777579)

то можно оставить один датацентр, но на случай его поломки и чтобы точно выдержать нагрузку от пользователей поставим еще один датацентр в следущем по количеству пользователей регионе - **Ярославской области**

Для глобальной балансировки запросов и нагрузки будем использовать latency-based DNS.
Это позволит отправлять запросы пользователя в ближайшие датацентры, которые отвечают с минимальной задержкой.

## 4. Локальная балансировка нагрузки <a name="4"></a>

### Схема балансировки
Запросы приходят на маршрутизатор, который алгоритмом round-robin изначально распределеят нагрузку по серверам, далее будем использовать L7 балансировку

Балансировка на уровне L7 обеспечит:
- Маршрутизацию на основе контента (например, отправка запросов к разным бэкендам для запросов на статистику и просто отправку сообщений).
- SSL-терминацию (дешифрование SSL-трафика на балансировщике нагрузки).
- Сжатие ответа с использованием gzip
- Высокая гибкость конфигурации
- Решение проблемы медленных клиентов
  
На уровне L7 будем использовать nginx, и также с помощью X-accel redirect будем проверять на отдельном сервисе авторизацию, тем самым освобождая бэкэнд для обработки других запросов. 

### Схема отказоустойчивости
Будем использовать программное обеспечение keepalived с использованием протокола VRRP. 
Это позволит следить за доступностью серверов, а в случае отказа одного из них перенаправить трафик на другой.

### ssl терминация
Для шифрования будем использовать Let`s Encrypt. Так же будем использовать Session Cache для улучшения производительности и ускорения подключения.


## 5. Логическая схема базы данных <a name="5"></a>
![image](https://github.com/flioletty/highload/assets/92665311/f0b25af7-5312-4037-99f3-490eaa35c571)


## 6. Физическая схема базы данных <a name="6"></a>

### Выбор СУБД

| Таблица | СУБД |
|------------|-------------|
|User|PostgreSQL|
|Session|Reddis|
|Letter| PostgreSQL|
|LetterSearch|ElasticSearch|
|LetterStatus|Cassandra|
|Folder|Cassandra|
|Chain|Cassandra|
|Attachment|PostgreSQL|
|SearchHistory|Cassandra|
|LettersAmount|ClickHouse|
|File|S3|
|StateMails|PostgreSQL|
|SocialNetworkMails|PostgreSQL|

### Шардирование

Шардируем таблицы по полю userId 
| Таблица | Поле по которому шардируем |
|------------|-----------------|
|User|UserId|
|Session|-|
|Letter| RecipientId|
|LetterSearch|-|
|LetterStatus|UserId|
|Folder|UserId|
|Chain|UserId|
|Attachment|UserId|
|SearchHistory|UserId|
|LettersAmount|UserId|
|File|-|
|StateMails|-|
|SocialNetworkMails|-|

### Индексы

| Таблица | Поле по которому индексируем |
|------------|-----------------|
|User|MailAddress (B-Tree)|
|Session|-|
|Letter| RecipientId, Date (B-Tree)|
|LetterSearch|Theme, Text, SenderName (обратные индексы по словам) - отдельные не составные индексы|
|LetterStatus|UserId, LetterId (SASI)|
|Folder|UserId (SASI)|
|Chain|UserId (SASI)|
|Attachment|-|
|SearchHistory|UserId (SASI)|
|LettersAmount|UserId(B-Tree)|
|File|-|
|StateMails|MailAddress (B-Tree)|
|SocialNetworkMails|MailAddress (B-Tree)|

## 7. Алгоритмы <a name="7"></a>

### Алгоритм сортировки писем 

1. Анализ отправителя:
- Если отправитель является государственным сервисом, который занесен в нашу заранее созданную и периодически обновляемую модератором базу (например, Госуслуги, Росреестр, ФНС и др.), письмо помещается в папку “Госписьма”.
- Если отправитель помечен как спамер, письмо отправляется в папку "Спам"
- Если отправитель является социальной сетью, которая также занесена в нашу заранее созданную и периодически обновляемую модератором базу (например, ВКонтакте, Одноклассники и др.), письмо помещается в папку “Социальные сети”.
- Если у отправителя >10000 отправленных писем в месяц, и он не отмечен как спамер, письма от него помещаются в папку “Рассылки”
2. Группировка по номеру обращения:
- Если письмо от государственного сервиса, вытаскиваем номер обращения по ключевым словам ("поступило в обработку", "обращение №" и тд) или проверяем наличие уже существующего номера в пришедшем письме
- Собираем уведомления по номеру обращения в таблицу Chain в базе. Например, если пользователь подал документы на получение загранпаспорта, все обновления статуса будут автоматически приходить в соответствующую цепочку писем — от сообщения о том, что заявление принято, до приглашения на получение.

### Алгоритм антиспама:

Будем использовать систему баллов для каждого пользователя. Прибавлять или отнимать баллы за отправленные письма. Если счетчик >0, письма пользователя не падают в спам, <0 отправленные пользователем письма отправляются в спам, если счетчик <100 модератор оценивает пометить ли пользователя как спамера.

 Пример разбалловки: 
| Действие | Балл |
|--------------|-----|
ссылок в письме >3 | -1 за каждую ссылку
ссылок в письме <3 | +2
Нет текста | -2
Объем текста >1000 символов | -3
Объем текста >0, <1000 | +3
Ключевые слова (“победитель”, “бесплатно”, “скидка” и тд) | -1 за каждое
нет ключевых слов | +1
Длина url в письме >30 | -1

Ко всему обучим ML модель наивный Байессовский классификатор и тем письмам, которые она отметила как спам, будем давать -5 баллов. Благодаря обратной связи от пользователей (которые отмечают письма не из папки спам как спам или наоборот) и модератора, который будет финально сортировать письма на спам и не спам, сможем повышать точность модели.

Также введем систему допустимого процента писем, помеченных другими пользователями как спам, превышение этого процента дает -100 баллов и пользователь отправляется на рассмотрение модератором:
![image](https://github.com/flioletty/highload/assets/92665311/210088ee-384c-4d6d-814d-fed3b88e2339)

Помимо баллов пользователя, проверяем пометил ли модератор пользователя как спамера, письма от них автоматически отправляются в спам

### Алгоритм поиска:

1. Выделяем данные для таблицы в Elastic Search
2. Используем алгоритм Snowball (Он следует набору предопределенных правил для выполнения стемминга, анализируя структуру слов и применяя ряд преобразований для извлечения основы. Процесс включает удаление общих окончаний и суффиксов слов, имеет поддержку нескольких языков) для стемминга слов
3. Раздадим веса для категорий поиска (отправитель: 3, тема письма: 2, текст письма: 1), чтобы приоритизировать результаты поиска в дальнейшем
4. Выгрузим данные в Elastic Search и создадим обратные индексы для более быстрого поиска (Собираем список всех используемых слов и "запоминаем", в каких документах они встречаются. Да, индексация займет больше времени, но нас в первую очередь интересует именно скорость поиска, а не индексации)
5. Далее для самого поиска (сопоставления слов из базы с вводом пользователя) будем использовать расстояние Левенштейна, чтобы учитывать ошибки и опечатки

## 8. Технологии <a name="8"></a>

| Технология | Применение | Обоснование |
|------------|-----------------|--------------|
|Golang | Бекенд часть сервисов | обладает высокой производительностью, эффективным многопоточным исполнением и простым синтаксисом. Он идеально подходит для создания масштабируемых и быстрых бекенд-сервисов|
|gRPC | Взаимодействие сервисов между собой |gRPC обеспечивает эффективную передачу данных между микросервисами |
|React | Фронтенд часть |  React - это популярная библиотека для создания интерактивных пользовательских интерфейсов. Она обеспечивает быстрое обновление компонентов и удобное управление состоянием|
|Prometheus, Grafana |Система сбора и отображения статистики по бекенду| Prometheus собирает метрики, а Grafana предоставляет гибкие инструменты для визуализации данных. Вместе они помогают отслеживать производительность и выявлять проблемы.|
|Reddis | БД для хранения сессий | быстрая и масштабируемая база данных |
|S3 Mail.ru | Облачное хранилище файлов | S3 Mail.ru обеспечивает надежное хранение файлов, масштабируемость и доступность |
|Cassandra | База данных NoSQL |Cassandra обеспечивает высокую доступность, горизонтальное масштабирование и устойчивость к отказам.|
|PostgreSQL | База данных SQL |надежная и мощная реляционная база данных. Она поддерживает сложные запросы, транзакции и обеспечивает целостность данных.|
|ClickHouse | База данных для статистики |ClickHouse специализируется на аналитических запросах и обработке больших объемов данных. Он быстро выполняет агрегацию и фильтрацию данных.|
|ElasticSearch | Обеспечивает поиск по письмам |мощный поисковый движок с поддержкой полнотекстового поиска, фильтрации и анализа данных|
|Ngnix | Балансировщик |обеспечивает высокую производительность и надежность.|
|Kafka | Брокер сообщений |Kafka предоставляет масштабируемую и устойчивую платформу для обмена сообщениями между компонентами системы|
|Docker | Развертывание приложения | обеспечивает контейнеризацию приложений, что упрощает развертывание, масштабирование и управление приложениями.|
|Kubernetes | Развертывание приложения |Kubernetes - это оркестратор контейнеров, который автоматизирует процессы развертывания, масштабирования и управления контейнерами. Он обеспечивает высокую доступность и удобное управление.|
| Python | Написание ML моделей для антиспам алгоритма | Удобен для работы с машинным обучением из-за большого количества библиотек |
| NLTK (Natural Language Toolkit) | Обработка текста для дальнейшего поиска | Библиотека для работы с естественным языком в Python |
| Apache Lucene | Для поиска |Библиотека для полнотекстового поиска и индексации, используется в Elasticsearch|
|Snowball|Обработка текста для дальнейшего поиска|Библиотека для стемминга|
|Scikit-learn | Для написание Байессовского классификатора для антиспама | Библиотека для машинного обучения в Python |

## 9. Схема проекта <a name="9"></a>

![image](https://github.com/flioletty/highload/assets/92665311/191152bf-3033-46d4-8100-5be6627a23a7)

## 10. Обеспечение надежности <a name="10"></a>

### Резервирование
1. Резервирование ДЦ: у нас 2 дц (Москва и Ярославская область), в случае падения одного данные можно будет восстановить со второго
2. Резервирование БД:
   - PostgreSQL Мастер-слейв репликация с помощью встроенной Streaming replication по 2 реплики таблиц User и 1 реплике остальных таблиц в рамках одного дц
   - Cassandra со стратегией репликации данных NetworkTopologyStrategy репликации по дц и Peer-to-Peer репликации внутри одного дц по 2 реплики таблиц
   - Redis Мастер-слейв репликация по 2 реплики
   - ClickHouse по 1 реплике с ReplicatedMergeTree
   - S3 Cross-Region Replication данных между разными регионами для обеспечения отказоустойчивости и включение версионирования объектов для восстановления данных в случае ошибок.
3. Резервирование физических компонетов (сервера, диски и т.д.)
4. В Kafka гарантия at most once

### Сегментирование
Выделим следующие сегменты: 
- `sorter`, отвечающий за распределения писем по папкам и чуть более затратный по ресурсам чем остальные - сложный неважный
- `auth`, который работает при каждом запросе - простой и посещаемый
- `letters`, обеспечивающий основную логику работы сервиса по разным сегментам - простой важный
- `searcher` и `statistics` выделим отдельно - неважные и непосещаемые

### Failover policy
Уменьшение посылаемых запросов на проблемный хост, компонент по паттерну Circuit Breaker (если 75% запросов достигает верхнего порога (150–200 мс), это означает, что сервис медленно выходит из строя и надо ограничить количество запросов)

### Graceful shutdown
Будем использовать Graceful Node Shutdown в Kubernetes, чтобы при остановке сервиса, не терялись обрабатываемые в этот момент запросы

### Graceful degradation
Сервис `sorter` раскидывает письма по папкам Входящие и Отправленные

Сервис `letters` выдает пользователю первую страницу писем из кеша и позволяет отправлять письма

Остальные сервисы (кроме auth) не критичны для работы сервиса

### Асинхронные паттерны
Отбивка статистики

### Observability 
Логгирование в json файл с семплированием по user id на сервере по типам access, error

## Список литературы

[^1]: [Ежемесячная аудитория Почты Mail.ru](https://vk.company/ru/investors/results/)
[^2]: [Размер почтового ящика](https://help.mail.ru/mail/settings/size/)
[^3]: [Сколько электронных писем отправляется в день?](https://prosperitymedia.com.au/how-many-emails-are-sent-per-day-in-2024/)
[^4]: [Каким должен быть размер письма для email-рассылки](https://sendsay.ru/blog/kakim-dolzhien-byt-razmier-pisma-dlia-email-rassylki/)
[^5]: [Medaiscope](https://mediascope.net/data/)
[^6]: [География пользователей mail.ru](https://top.mail.ru/countries?id=250&period=0&aggregation=sum&ytype=value&gtype=line&sids=RU,KZ,BY,US,DE)
[^7]: [Статистика mail.ru](https://top.mail.ru/mddashboard?id=250&period=0&date=2024-03-03&what=visitors&)

# Проектирование высоконагруженного почтового сервиса
Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана ("Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"
##### Автор: Степаненко Полина
В качестве примера выбран сервис [Mail.ru](https://mail.ru/)

## Содержание

* ### [Тема и целевая аудитория](#1)
* ### [Расчет нагрузки](#2)

## 1. Тема и целевая аудитория <a name="1"></a>

**Mail.ru** — почтовый сервис, с возможностью чтения и отправки (с вложениями) писем.

### MVP

- Просмотр списка писем
- Чтение писем
- Отправка писем
- Прикрепление файлов(изображений, архивов)
- Удаление писем из интерфейса
- Распределение писем по разделам (отправленные, спам, соцсети и тд)
- Получение статистики по просмотрам писем

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


### Технические метрики

#### Среднее количество действий пользователя по типам в день (RPS) (согласно MVP)

- <b>Просмотр списка писем</b>

Объем данных  `LETTERS_PER_PAGE * SIZE_LETTER = 25 * 50 = 1250 KB`:

      RPS = (16.5 * 10^6 * 25)/(24 * 60 * 60) ~ 4774

      Трафик: 4774 * 1250 ~ 5 967 500 Kbit/sec ~ 5,7 Gbit/sec

      Пиковое значение: 2 * 5,7 ~ 11,4 Gbit/sec

- <b>Отправка писем</b>

Объем данных `LETTERS_SENT * SIZE_LETTER = 40 * 50 = 2000KB`:

      RPS = (16.5 * 10^6 * 40)/(24 * 60 * 60) ~ 7638

      Трафик: 7638 * 2000 ~ 15 276 000 Kbit/sec ~ 14,5 Gbit/sec

      Пиковое значение: 2 * 14,5 ~ 29 Gbit/sec

- <b>Чтение писем</b>

Объем данных `LETTERS_GET * SIZE_LETTER = 121 * 50 = 6050 KB`:

      RPS = (16.5 * 10^6 * 121)/(24 * 60 * 60) ~ 23108

      Трафик: 23108 * 6050 ~ 139 803 400 Kbit/sec ~ 133 Gbit/sec

      Пиковое значение: 2 * 133 ~ 266 Gbit/sec

- <b>Скачивание файлов(изображений, архивов)</b>

В среднем LETTERS_GET * LETTER_FILES = 121 * 0.33 полученных письма имеют вложение, объем данных `LETTERS_GET * LETTER_FILES * LETTER_FILES_SIZE = 121 * 0.33 * 1024KB = 40888KB`, тогда:

      RPS = (16.5 * 10^6 * 121 * 0,33)/(24 * 60 * 60) ~ 7625

      Трафик: 7625 * 40888 ~ 311 792 295 Kbit/sec ~ 297 Gbit/sec

      Пиковое значение (пункт 6): 2 * 297 ~ 594 Gbit/sec

- <b>Прикрепление файлов(изображений, архивов)</b>

В среднем LETTERS_SENT * LETTER_FILES = 40 * 0.33 отправленное письмо имеет вложение(пункт 1), объем данных `LETTERS_SENT * LETTER_FILES * LETTER_FILES_SIZE = 40 * 0.33 * 1024KB = 13517KB`, тогда:

     RPS = (16.5 * 10^6 * 40 * 0,33)/(24 * 60 * 60) ~ 2520

      Трафик: 2520 * 13517 ~ 34 071 776 Kbit/sec ~ 32,5 Gbit/sec

      Пиковое значение (пункт 6): 2 * 32,5 ~ 65 Mbit/sec

- <b>Удаление писем</b>

В среднем пользователи почти никогда не удаляют сообщения

**Хранилище**

Принимая средний размер хранилища за 8Гб, а средний размер письма 200Кб получаем:

`(8 * 1024 * 1024) / 200 ~ 42000`

| Размер хранилища | Среднее количество писем в ящике |
|------------------|----------------------------------|
| 8 GB            | 42000                            |

**Результаты расчетов**

| Запрос | RPS средний | RPS пиковый | Потребляемый трафик (средний) | Потребляемый трафик (пиковый) |
|--------|-------------|-------------|-------------------------------|-------------------------------|
|Просмотр списка писем|4774|9548|5,7 Gbit/sec|11,4 Gbit/sec|
|Отправка писем|7638|15276|14,5 Gbit/sec|29 Gbit/sec|
|Чтение писем|23108|46216|133 Gbit/sec|266 Gbit/sec|
|Скачивание файлов|7625|15250|297 Gbit/sec|594 Gbit/sec|
|Прикрепление файлов|2520|5040|32,5 Gbit/sec|65 Gbit/sec|

## Список литературы

[^1]: [Ежемесячная аудитория Почты Mail.ru](https://vk.company/ru/investors/results/)
[^2]: [Размер почтового ящика](https://help.mail.ru/mail/settings/size/)
[^3]: [Сколько электронных писем отправляется в день?](https://prosperitymedia.com.au/how-many-emails-are-sent-per-day-in-2024/)
[^4]: [Каким должен быть размер письма для email-рассылки](https://sendsay.ru/blog/kakim-dolzhien-byt-razmier-pisma-dlia-email-rassylki/)
[^5]: [Medaiscope](https://mediascope.net/data/)
[^6]: [География пользователей mail.ru](https://top.mail.ru/countries?id=250&period=0&aggregation=sum&ytype=value&gtype=line&sids=RU,KZ,BY,US,DE)

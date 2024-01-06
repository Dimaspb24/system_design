# Social Network - System Design

Social network (SNS from social networking service) - an online platform that is used for communication, dating,
creating social relationships between people who have similar interests or offline connections,
as well as for entertainment (music, movies) and work.

### Functional requirements:

1. добавление друзей через запрос на дружбу и подтверждение;
2. удаление друзей;
3. просмотр друзей пользователя;
4. просмотр анкеты пользователя;
5. публикация поста в ленту;
6. загрузка медиа файлов (аудио, видео, изображения) для постов;
7. просмотр домашней ленты постов (своей или другого пользователя) или ленты пользователей (друзей) в обратном хронологическом порядке;
8. поддержка комментариев
9. поддерка лайков
10. просмотр диалогов и чатов пользователя;
11. отправка и чтение сообщений в диалогах и чатах.
12. просмотр прочитанности сообщений - кем и когда прочитано
13. поддержка уведомений пользователя о сообщениях
14. поддержка статусов пользователя (онлайн/оффлайн)

### Non-functional requirements:

1. **DAU (day active users) 50 000 000**
2. MAU (month active users) 100 000 000
3. **Availability 99.95%**
4. **Посты храним всегда**
5. Комментарии не ограничиваем
6. **Макс. количество друзей 1 000 000**
7. Макс. количество букв в посте 2 000
8. Одно изображение в посте
9. **В среднем пользователь создаёт один пост в 7 дней**
10. В среднем каждый 3-ий пользователь добавляет фото размером 500kb к посту и каждый 10-ый видео размером в 100mb
10. **Пользователь просматривает ленту 10 раз в день**
11. **Средний размер сообщения 1000 символов**
12. Максимальное кол-во пользователей в чате 1000
13. **В среднем пользователь отправляет сообщения 10 раз в день**
14. **В среднем пользователь читает сообщения 60 раз в день**
15. Геораспределённость на центральный и восточный регион России
16. Сезонности нет
17. Response time на отправку 1 секунда
18. Response time на получение 5 секунд

## Design overview

For system design I have  used [C4 model](https://c4model.com/). The C4 model was created as a way
to help software development teams describe and communicate software
architecture, both during up-front design sessions and when retrospectively
documenting an existing codebase. It's a way to create maps of your code,
at various levels of detail, in the same way you would use something like
Google Maps to zoom in and out of an area you are interested in.

![Context](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.githubusercontent.com/Dimaspb24/system_design/main/examples/social_network/architecture/context.puml)
![Container](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.githubusercontent.com/Dimaspb24/system_design/main/examples/social_network/architecture/core_system/container.puml)
![Deployment](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.githubusercontent.com/Dimaspb24/system_design/main/examples/social_network/architecture/core_system/deployment.puml)
![Geo deployment](http://www.plantuml.com/plantuml/proxy?cache=no&fmt=svg&src=https://raw.githubusercontent.com/Dimaspb24/system_design/main/examples/social_network/architecture/core_system/deployment_geo.puml)

## Basic calculations

* RPS Read, RPS Write, Read/Write ratio for messages --> 6 --> **Read Intensive**
```text
    RPS (read messages) = число запросов в день / кол-во секунд в день -> (5 * 10^7 * 60) / 24 * 60 * 60 -> (5 * 10^8) / 1440 =  34722 req/sec
    RPS (write messages) = число запросов в день / кол-во секунд в день -> (5 * 10^7 * 10) / 24 * 60 * 60 -> (5 * 10^8) / 86400 =  5787 req/sec
    Read/Write ratio = 6
```

* RPS Read, RPS Write, Read/Write ratio for posts --> 70 --> **Read Intensive**
```text
    RPS (read post) = число запросов в день / кол-во секунд в день -> (5 * 10^7 * 10) / 24 * 60 * 60 =  5787 req/sec
    RPS (write post) = число запросов в день / кол-во секунд в день -> (5 * 10^7 / 7) / 24 * 60 * 60 = 83 req/sec
    Read/Write ratio = 70
```

* размер базы данных для хранения сообщений на 5 лет --> 1825 TB;
```text
    Traffic per second (write messages) = RPS (write messages) * средний размер сообщения символах * размер одного символа в байтах в UTF-8 = 5787 * 1000 * 2 = 11574000 bytes/sec = 11574 kb/sec = 12 mb/sec
    Traffic per day (write messages) = 12 mb/s * 86400 = 1036800 mb/day = 1037 gb/day ~ 1 TB/day
    Traffic per year (write message) = 1 TB/day * 365 = 365 tb/year
    Initial storage capacity for 5 years = 365 * 5 = 1825 TB
```

* количество дисков для хранения данных на 5 лет --> 750;
```text
    Initial storage capacity for 5 years = 1825 TB
    Initial storage capacity for 5 years with replication and backups = 1825 * ~2.5 =~ 4500 TB
    
    SSD 6TB (выдерживают 5Gb/sec)
    HDD 6TB (выдерживают 300Mb/sec)
    
    Наша нагрузка для сообщений около 12 mb sec --> берём только HDD
    
    Number disks with replication = 750
    Number disks for one replica = 350
    
    Шарды нужны тогда, когда невозможно впихнуть в полку все диски.В зависимости от размера RPS или Trafic.
    Допустим в полку помещается по 8 дисков. Тогда 350 / 8 ~ 44...
    Number of shards = ???
    Горячие и холодные данные?
```

* входящий трафик на создание постов: ~ 0.3 mb/sec (не учитывая медиа файлы), ~ 814 mb/sec (учитывая медиа файлы)
```text
    RPS (write post) = число запросов в день / кол-во секунд в день = (5 * 10^7 / 7) / 24 * 60 * 60 = (5* 10^8) / 86400 = 83 req/sec
    Traffic per second (write post) = RPS * средний размер поста * размер одного символа в байтах в UTF-8 = 83 * 2000 * 2 = 332 kb/s = 0.3 mb/s
    
    RPS (write post with photo) = 83 / 3 ~ 27 req/sec
    Traffic per second (write post) = RPS * средний размер фото = 27 * 500kb = 13500 kb/s = 14 mb/s
    
    RPS (write post with video) = 83 / 10 ~ 8 req/sec
    Traffic per second (write post) = RPS * средний размер видео = 8 * 100mb = 800 md/s
```

* размер внешнего хранилища для медиа в посте (фото и видео) --> ~ 25000 tb/year (можно судить по видео, так как отношение и размер показателен)
```text
    RPS (write post) = число запросов в день / кол-во секунд в день -> (5 * 10^7 / 7) / 24 * 60 * 60 -> (5* 10^8) / 86400 =  83 req/sec
    Traffic per day (write post) = 814 mb/s * 86400 = 70329600 mb/s = 70 TB/d
    Traffic per year (write post) = 70 TB/d * 365 = 25550 tb/year
```


## Notes

1. Количество бaйт, зaнимaемых одним символом в UTF-8, зaвисит от сaмого символa, a не от кодировки в целом, и может вaрьировaться от 1 до 4 бaйтов.
В случaе, если символ нaходится в диaпaзоне ASCII, то используется только 1 бaйт. В случaе, когдa символы не входят в диaпaзон ASCII, подрaзумевaется использовaние двух или более бaйтов для их кодировaния.
   
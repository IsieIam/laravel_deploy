# Деплой приложения и инфры для laravel приложения

### Краткое описание всего процесса:
 - весь проект состоит из приложения и описания инфраструктуры
 - Предполагается что сначала поднимаем инфру и дальше в ней разворачиваем наше приложение
 - Отдельная репо для разворачивания инфраструктуры в облаке: https://github.com/IsieIam/infra - следовать её инструкциям.
 - Простое приложение laravel: https://github.com/IsieIam/laravel

### Структура репа:
- Local - ручной локальный стенд, основанный на docker-compose (см в каталоге подробное описание)
- Charts - helm чарты для приложения: основным является laravel, который состоит из:
  - чарт laravel-nginx - являющимся основным чартом приложения и состоящим по сути из двух приложений: непосредственно php-fpm приложения и fastcgi proxy(nginx), 
  - laravel-nginx-exporter- экспортер метрик nginx, также содержащий configmap с json дашбордом для grafana, который grafana может подцеплять на ходу при включении в своем деплое sidecar контейнера для загрузки дашбордов из конфигмепов.(подробности см в репо с инфрой)
  - и двух зависимостей: mysql БД и memcached для кеша.

Для ручного запуска достаточно отредактировать в каталоге Charts/laravel файл values.yaml(поставить образ своего приложения, ingress можно будет задать параметром в job Jenkins или вообще переопределить все параметры :)) и запустить в этом каталоге команду:

```
helm install larka . -f values.yaml
# где larka - произвольное имя релиза
```

- Jenkinsfile - pipeline разворачивания приложения для stage и prod среды для Jenkins.

### Детали pipeline
Общий flow предполагается следующий: фича ветка из репо кода приложения разрабатывается на дев стенде(о нем ниже), дальше мерджится в ветку release, c которой настраивается hook в job jenkins с scm на репо с деплоем.
В момент старта job-а он забирает исходный код приложения из ветки release, собирает докер образ, пушит в регистри и автоматом раскатывает в stage, запускает автотесты и по результатам раскатывает в прод среду (можно перед раскаткой в прод вставить ручной подверждение).
После установки на пром release ветка мержится в master.
При этом возможно сделать подобный механизм для хука с любой ветки - взяв адрес репо и хук с метаинфой, пришедшей вместе с hook, например для раскатки на dev среду feature веток.

- Для запуска пайплайн ожидает три параметра на вход:
  - VERSIONTOBUILD - string parameter: значение по умолчанию gitver - версия приложения/тег докер образа для сборки или установки.
  - BUILD - choise parameter: yes/no - пересобирать приложение или нет. При этом если выбираете no, то просто попробует задеплоиться версия указанная в параметре VERSIONTOBUILD.
  - additional_args - string parameter - может быть пустым или доп списком параметров к параметру helm --set, для примера на ходу переопределить внешний адрес сервиса: 

```
,laravel-nginx.ingress.host=84.252.131.52.nip.io
```

- С точки зрения выполнения pipeline в Jenkins необходимо доставить плагины: docker-pipelines, Pipeline Utility Steps и прописать кред на докер-хаб c id=dockerhub_id
- GitVersion - в пайплайне для автоматического версионирования используется gitversion: https://gitversion.net/docs/ Она собирает различные параметры на основе репо с кодом приложения и дальше этими данными можно воспользоваться для выставления версии приложения и тега докер образа. 
Такой подход более выгодный чем ручное именование или нумерование по build номеру CI, т.к. отражает более честное состояние кода приложения.
- более подробное описание каждого этапа можно найти в комментариях в jenkinsfile

### Jenkins quick start

- считаем что jenkins развернут либо через инфраструктурный репо(https://github.com/IsieIam/infra) либо как-то еще.
- устанавливаем плагины docker-pipelines, Pipeline Utility Steps
- прописываем кред на docker-registry c id=dockerhub_id, либо в пайпе указываем свой id
- создаем два job, оба пайплайн берут с scm:
  - один на репо с приложением на Jenkinsfile_hook и включенным github trigger - название не важно
  - второй на репо с деплоем на Jenkinsfile c именем deploy - это имя сейчас указано в Jenkinsfile_hook для вызова, с настройкой параметров, указанных выше
- в репо с кодом - настраиваем hook на http://jenkip/github-webhook/
- все готово для деплоя по пушу в репо с кодом.

### особенности php
Есть особенности php с которым столкнулся: дев стенд - пока видится два варианта его подготовки: локальный и в k8s.
Но есть два момента, которые необходимо решить: 
- 1. вариант быстрого деплоя кода(все-таки php это интерпретируемый язык) - в частности IDE php-storm имеет такую возможность как закинуть активный в ней файл или сразу в каталог через ssh (и еще несколько способов) на удаленный dev стенд и тут же посмотреть как это отобразиться на страничке.
- 2. debug в IDE удаленного стенда.

С локальным, например в виде docker-compose в принципе все просто(ну или не очень), но так сейчас уже работает. А вот с стендом в k8s пока понятно только направление: ksync и kubectl debug и им подобные утилиты. Либо отдельный контейнер под дев среду с поднятым ssh или portforwarding.

### Соединение php-fpm и web сервера с fastcgi
В результате поиска остался один вариант запуска laravel приложения - через совмещение в одном поде двух контейнеров: nginx + phpfpm - основной проблемой является необходимость передать nginx каталог public находящийся у php приложения (а собирать отдельно еще и образ nginx с закинутой в него статикой точно не хочется).

### Grafana Dashboards
В ветке grafana-flux - реализована вариант забора configmap с дашбордами grafana через fluxcd: а именно - в основной chart добавляется зависимость в виде flux, у которого настраивается конфиг на этот репо на каталог dashboards для синхронизации grafana дашбордов. 
Какой вариант лучше: явный деплой вместе с приложением или дашборды отдельно - пока не понятно: с одной стороны дашборд может меняться вместе с приложением(под его новые метрики), сдругой стороны точно также можно создавать новые дашборды с отдельной версией в каталоге для синхронизации.

Если использовать через helm вместе с установкой - есть пока не до конца понятный момент - как красиво параметризовать дашборд - например имя, т.к. у grafana есть своя параметризация и она с тем же синтаксисом что и helm и helm ломается когда встречает параметры grafana.

### ScreenCast
видео-комментарии к заданию  можно посмотреть по ссылке(заранее приношу извинения за оговорки и прочие неточности :) ) https://youtu.be/ksgvFIAmye0

### todo на будущее
- описать deployment/helm-чарт для adminer, хотя он не совсем относится к приложению, больше это как утилита в кластере
- реализовать возможность debug в k8s
- рефакторинг кода

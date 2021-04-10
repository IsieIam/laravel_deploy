# Локальный deploy

### Quick start
- Клонируем этот репо и репо с исходным кодом, должно получиться так:

```
>laravel
>laravel_deploy
```

- В каталоге laravel_deploy/local/.env.example копируем в .env, меняем имя ползователя для регистри и при необходимости другие параметры.
- далее в каталоге laravel_deploy/local запускаем:

```
docker-compose build
docker-compose up
```

- по адресу localhost получаем приложение laravel
- по адресу localhost:8080 - adminer для БД

### описание

 - docker-compose описание для запуска laravel приложения с nginx в качестве fastcgi proxy, поддержкой БД, memcached и adminer
 - lara.conf - конфиг для laravel приложения - отличается от дефолтного только параметром listen
 - nginx.conf - настройка nginx, которая заменяет дефолтный конфиг nginx.
 - .env_lara - дефолтный файл env для laravel приложения, часть настроек переопределяются в docker-compose (в принципе можно обойтись чем-то одним)
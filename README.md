# Notabenoid
> Система коллективного перевода текстов

Давным-давно был такой умучанный копирастами сайт — http://notabenoid.com/. А здесь мы можем
видеть исходники этого сайта, распространяемые под лицензией Beerware, что означает, что вы можете
использовать весь этот говнокод как хотите.

С вашей стороны при использовании кода было бы любезно где-нибудь на сайте оставить ссылку на
[автора](http://facebook.com/uisky), каковой автор является мрачным и необщительным типом, а потому
**не даёт справок и не консультирует по вопросам установки, модификации и поддержки программы**,
а также не интересуется ничьим мнением о качестве кода, тем более что и сам придерживается о нём весьма
невысокого мнения.

Исходники эти выложены с минимальными правками и не были рассчитаны на свободное распространение, поэтому
в процессе установки вас может поджидать много сюрпризов. Будет здорово, если какие-нибудь пряморукие люди
улучшат код в сторону его более простой распространяемости и напишут более человеческую документацию. 

## ВСЁ ПРОВЕРЯЛОСЬ НА UBUNTU 22.04 jammy!!!
## Требования
Нам понадобятся:

  * php 5.5 или выше (работает с php 5.6, инструкция по установке ниже)
  * phpшные модули: gd, pdo-pgsql, curl, memcache
  * postgresql 9.1 или выше (работает с postgresql 9.4, инструкция по установке ниже)
  * memcached
  * nginx

# Php 5.6 и postgresql 9.4
## Php
php 5.6 для ubuntu можно скачать так:

 ```
 sudo add-apt-repository ppa:ondrej/php
 sudo apt update
 sudo apt upgrade
 sudo apt install -y php5.6
 ```
И сразу докачиваем fpm:
 ```
 sudo apt install -y php5.6-fpm
 ```
Для скачивание модулей пишем их по формуле `sudo apt install -y php5.6-MODULE-NAME`
Пример:
 ```
 sudo apt install -y php5.6-curl
 sudo apt install -y php5.6-gd
 ```
И тому подобное...

## Postgresql
posgresql 9.4 для ubuntu можно скачать так:
 ```
 sudo add-apt-repository "deb https://apt.postgresql.org/pub/repos/apt jammy-pgdg main"
 ```
 (В случае, если у вас не jammy - замените 'jammy-pgdg' на 'REP-VER-pgdg'. Версию репозитория можно узнать блягодаря `lsb_release -sc`)
 ```
 wget --quiet -O - https://postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
 sudo apt-get update
 sudo apt-get install postgresql-9.4
 ```
Также не забываем выключить apache2
 ```
 sudo service apache2 stop
 ```

## Установка
1. Клонируем репозиторий в какую-нибудь директорию, допустим, `/srv/notabenoid.com`
2. Натравливаем веб-сервер отдавать статику из `/srv/notabenoid.com/www` и все прочие запросы редиректить в index.php.

    В терминах nginx это будет выглядеть так:

    1.Создаем файл в nginx

        nano /etc/nginx/sites-available/notabenoid.com

    Содержимое:
    ```
    events {
        worker_connections  1024;
    }

    http {
     include mime.types;
     server {
         server_name notabenoid.com;
         listen 80;
         root /srv/notabenoid.com/www;
         index index.php;
         location / {
             try_files $uri $uri/ /index.php?$args;
         }
         location ~ \.php$ {
             fastcgi_split_path_info ^(.+\.php)(/.+)$;
             fastcgi_pass unix:/var/run/php/php5.6-fpm.sock;
             fastcgi_param SCRIPT_FILENAME $request_filename;
             fastcgi_index index.php;
             include fastcgi_params;
         }
         location ~ ^/(assets|img|js|css) {
             try_files $uri =404;
         }
     }
    }
    ```
## То же самое прописываем в /etc/nginx/nginx.conf!!!

   2.Включаем сайт, создаем ссылку на конфи

        ln -s /etc/nginx/sites-available/notabenoid.com /etc/nginx/sites-enabled/

3. Веб-сервер должен уметь писать в следующие директории:
    * /www/assets
    * /www/i/book
    * /www/i/upic
    * /www/i/tmp
    * /protected/runtime

4. Создаём в постгресе базу, юзера и скармливаем дамп:

        sudo -u postgres createuser -E -P notabenoid
        sudo -u postgres createdb -O notabenoid notabenoid

    правим /etc/postgresql/9.4/main/pg_hba.conf, раздел подключений. Необходимо сделать так, чтобы локальное подключение не требовало пароля 

        local   all  all trust

    После этого нужно рестартнуть postgresql
   
        
        sudo service postgresql restart
  						

    Скармливаем дамп:

        psql -U notabenoid < /srv/notabenoid.com/init.sql

    Изменяем права пользователя notabenoid

        sudo -u postgres psql template1

        # alter role notabenoid with superuser;
        # \q

6. Настало время охуительных конфигов! В /protected/config/main.php найдите строки

        "connectionString" => "pgsql:host=localhost;dbname=notabenoid",
        "username" => "notabenoid",
        "password" => "",

    и пропишите туда название постгресной базы, юзера и пароль. Чуть ниже в строках 

        "passwordSalt" => "Ел сам в Акчарлаке кал рачка в масле",
        "domain" => "notabenoid.org",
        "adminEmail" => 'support@notabenoid.org',
        "commentEmail" => "comment@notabenoid.org",
        "systemEmail" => "no-reply@notabenoid.org",

    напишите любую херню в элементе "passwordSalt", а в остальных элементах - название вашего домена и почтовые
    адреса, которые будут стоять в поле "From" всякого спама, который рассылает сайт. Аналогичный трюк надобно
    провести с файлом `/protected/config/console.php`

7. В крон прописываем:

        0 0 * * * /usr/bin/php /srv/notabenoid.com/protected/yiic maintain midnight
        0 4 * * * /usr/bin/php /srv/notabenoid.com/protected/yiic maintain dailyfixes

    и последнюю команду (`/usr/bin/php /srv/notabenoid.com/protected/yiic maintain dailyfixes`) непременно
    исполняем сами.

8. Теперь, по идее, вся эта херня должна взлететь. Зарегистрируйте первого пользователя и пропишите его
    логин в группах со спецправами в переменной `private static $roles` в файле `/protected/components/WebUser.php`.
    Полагаю, также будет мудро несколько подправить основной шаблон (`/protected/views/layouts/v3.php`) и морду
    (`/protected/views/site/index.php`).

# В случае каких-либо проблем - готов вам помочь. Сам сталкивался, сам решал. https://t.me/manager_radar
   Как правило проблемы возникали с bootstrap.css, bootstrap-yii.css, face.css и т. п. файлами. Я знаю как это решать, и я бы тут расписал это, но при последнем запуске этих проблем у меня не возникло. Если у вас возникнет - готов помочь в первую очередь. 
   
*чмг-лов, Митя Уйский.*

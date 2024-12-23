# RPM

Для выполнения работы использована виртуальная машина на AlmaLinux 9.5 minimal

Установим пакеты `yum install -y wget rpmdevtools rpm-build createrepo \
 yum-utils cmake gcc git nano`

Соберем пакет Nginx:

1. Скачиваем SRPM пакет Nginx

`mkdir rpm && cd rpm`

`yumdownloader --source nginx`

2. Устанавливаем все зависимости для сборки пакета Nginx

`rpm -Uvh nginx*.src.rpm`

`yum-builddep nginx`

3. Скачиваем исходный код модуля ngx_brotli

`cd /root`

`git clone --recurse-submodules -j8 \
https://github.com/google/ngx_brotli`

`cd ngx_brotli/deps/brotli`

` mkdir out && cd out`

4. Собираем модуль ngx_brotli

`cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..`

`cmake --build . --config Release -j 2 --target brotlienc`

`cd ../../../..`

5. Поправить spec файл

`cd /root/rpmbuild/SPECS`

`nano nginx.spec`

В секции configure добавляем модуль `--add-module=/root/ngx_brotli \` соблюдая отступы, сохраняемся

6. Собираем RPM пакет

`cd ~/rpmbuild/SPECS/`

`rpmbuild -ba nginx.spec -D 'debug_package %{nil}'`

7. Проверяем создались ли пакеты

`ll /root/rpmbuild/RPMS/x86_64/`

![Image alt](https://github.com/NikPuskov/RPM/blob/main/rpm1.jpg)

8. Копируем пакеты в общий каталог

`cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/`

`cd ~/rpmbuild/RPMS/x86_64`

9. Устанавливаем наш пакет и проверяем, что nginx работает

`yum localinstall *.rpm`

![Image alt](https://github.com/NikPuskov/RPM/blob/main/rpm2.jpg)

`systemctl start nginx`

`systemctl status nginx`

![Image alt](https://github.com/NikPuskov/RPM/blob/main/rpm3.jpg)



Создадим свой репозиторий и разместим там ранее собранный RPM

1. Создадим каталог repo

`mkdir /usr/share/nginx/html/repo`

2. Копируем туда наши собранные RPM-пакеты

`cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/`

3. Инициализируем репозиторий

`createrepo /usr/share/nginx/html/repo/`

4. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы

`index index.html index.htm;`

`autoindex on;`

5. Проверяем синтаксис и перезапускаем Nпginx

`nginx -t`

`nginx -s reload`

6. Проверяем

`curl -a http://localhost/repo/`

7. Добавим репозиторий в /etc/yum.repos.d

`cat >> /etc/yum.repos.d/otus.repo << EOF`

`[otus]`

`name=otus-linux`

`baseurl=http://localhost/repo`

`gpgcheck=0`

`enabled=1`

`EOF`

8. Проверяем что репозиторий подключился и посмотрим, что в нем есть

`yum repolist enabled | grep otus`

9. Добавим пакет в наш репозиторий

`cd /usr/share/nginx/html/repo/`

`wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm`

10. Обновим список пакетов в репозитории (делается при каждом добавлении файлов)

`createrepo /usr/share/nginx/html/repo/`

`yum makecache`

`yum list | grep otus`

11. Установим пакет percona-release

`yum install -y percona-release.noarch`

![Image alt](https://github.com/NikPuskov/RPM/blob/main/repo1.jpg)

![Image alt](https://github.com/NikPuskov/RPM/blob/main/repo2.jpg)

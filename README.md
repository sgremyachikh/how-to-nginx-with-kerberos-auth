## Веб авторизация доменного пользователя через nginx с использованием Kerberos на сервере с Centos

### windows

windows AD:

Создаем служебного доменного пользователя (например apache), с бесконечным срок действия пароля.
Создаем KEYTAB-файл (необходим для аутентификации пользователей в Active Directory). В командной строке с правами администраторы выполняем команду ***(соблюдая регистр)***:

```powershell
ktpass -princ HTTP/webserver-hostname@TESTDOMAIN.TESTROOT -ptype KRB5_NT_PRINCIPAL -mapuser apache@TESTDOMAIN.TESTROOT -pass 'UserPassword' -crypto all -setupn -out C:\webserver-hostname.keytab

```
Полученный KEYTAB-файл, передаем любым удобным способом на Веб-сервер (расположение KEYTAB-файла на моем веб-сервере — /root/vol/webserver-hostname.keytab).

### web-server
```bash
yum install ntp ntpdate krb5-libs.x86_64 krb5-workstation.x86_64 krb5-server krb5-devel.x86_64 heimdal-devel.x86_64 make git gcc.x86_64 openssl-devel -y

# Файл (/etc/hosts) приводим к виду таким образом, чтобы в нём была запись с полным доменным именем компьютера и с коротким именем, ссылающаяся на один из внутренних IP:
echo 127.0.0.1 webserver-hostname >> /etc/hosts

# Настраиваем синхронизацию времени с контроллером домена, выполняем синхронизацию времени с контроллером домена:

# указываем часовой пояс
ln -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime || echo file exists

# заменим первый сервер синхронизации времени по умолчанию на наш контроллер домена
sed -i 's/0.centos.pool.ntp.org/domain-controller.my-domain.myroot/g' /etc/ntp.conf

# синхронизируемся с контроллером домена
ntpdate domain-controller.my-domain.myroot

# запускаем демона и ставим в автозапуск
systemctl enable --now ntpd
```

#### Настройка Kerberos

Кладем /root/vol/webserver-hostname.keytab на web-server

приводим конфиг kerberos к нужному виду виду:
```bash
# Удалим стандартный
rm -f /etc/krb5.conf

# создадим свой:
cat <<EOF | tee /etc/krb5.conf
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 ticket_lifetime = 24h
# renew_lifetime = 7d
 forwardable = true
# домен обязательно заглавными!
 default_realm = TESTDOMAIN.TESTROOT
# путь до keytab
 default_keytab_name = /root/vol/webserver-hostname.keytab
 dns_lookup_kdc = false
 dns_lookup_realm = false

[realms]
 TESTDOMAIN.TESTROOT = {
  kdc = domain-controller.my-domain.myroot
  admin_server = domain-controller.my-domain.myroot
  default_domain = testDOMAIN.testroot
 }

[domain_realm]
 .testdomain.testroot = TESTDOMAIN.TESTROOT
 testdomain.testroot = TESTDOMAIN.TESTROOT

[pam]
 debug = false
 ticket_lifetime = 36000
 renew_lifetime = 36000
 forwardable = true
 krb4_convert = false
EOF
```

область необходимо его указать в заглавном виде `TESTDOMAIN.TESTROOT`!
***BTW!*** `includedir /etc/krb5.conf.d/` дарит нам возможность подмонтировать как конфигмап и заинклюдить

### Проверка работы Kerberos

выполним авторизацию в Active Directory:
```bash
kinit -kV -p HTTP/webserver-hostname

Using default cache: /tmp/krb5cc_0
Using principal: HTTP/webserver-hostname@TESTDOMAIN.TESTROOT
Authenticated to Kerberos v5
```
успешно!

Удаляем полученный билет:
```bash
kdestroy
```

### Nginx

```bash
# добавим репо:
cat > /etc/yum.repos.d/nginx.repo <<EOF
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
EOF

# установим необходимо для сборки своего nginx
yum install -y nginx pcre-devel openssl-devel

# на всякий
systemctl stop nginx

# NGINX_VERSION будет содержать версию nginx
export NGINX_VERSION=$(nginx -v 2>&1 | awk -F"/" '{print $2}')

# качаем сорцы nginx нужной версии
curl -O http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz

# распаковываем
tar xvzf nginx-$NGINX_VERSION.tar.gz

cd nginx-$NGINX_VERSION

# клонируем в директорию с исходниками репозиторий Nginx module for HTTP SPNEGO auth,
# его мы будем использовать для аутентификации
git clone https://github.com/stnoonan/spnego-http-auth-nginx-module.git

# исвлекаем параметры, с которыми был скомпилировал устаноленный nginx, 
# добавляем к ним `--add-module=./spnego-http-auth-nginx-module` и выполяем ./configure
nginx -V 2>&1 | grep -oP 'configure arguments: \K.+' \
  | xargs ./configure --add-module=./spnego-http-auth-nginx-module

# собираем
make

make install

# проверяем с какими ключами скомпилирован nginx. проверяем `--add-module=./spnego-http-auth-nginx-module`
nginx -V

nginx version: nginx/1.18.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --with-perl_modules_path=/usr/lib/perl5/vendor_perl --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-Os -fomit-frame-pointer' --with-ld-opt=-Wl,--as-needed --add-module=./spnego-http-auth-nginx-module
```
все нормально и параметр учтен при сборке
```bash
systemctl start nginx

systemctl status nginx
```
### Аутентификация пользователя

конфиг nginx:
```conf
cat /etc/nginx/conf.d/default.conf 
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
	
	# Nginx module for HTTP SPNEGO auth

    # on/off, for ease of unsecuring while leaving other options in the config file
    auth_gss on;
    # Kerberos realm name. If this is specified, the realm is only passed to the nginx variable $remote_user if it differs from this default
	auth_gss_realm TESTDOMAIN.TESTROOT;
    # Pass header with username fron windows domain to app
    add_header 'AD-User' $remote_user always;
    # absolute path-name to keytab file containing service credentials
	auth_gss_keytab /root/vol/webserver-hostname.keytab;
    # principalname HTTP/hostname
	auth_gss_service_name HTTP/webserver-hostname;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## saltstack php56 ##

php56用于通过saltstack自动安装php5.6及其各模块

### salt php56路径及文件 ###

/export/salt/php56/  
|-- extmod.sls  
|-- files  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- extmod  
|&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|-- memcache-2.2.7.tgz  
|&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|-- memcached-2.2.0.tgz  
|&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|-- mongo-1.4.2.tgz  
|&nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|-- redis-2.2.3.tgz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- freetype-2.5.5.tar.gz      
|&nbsp;&nbsp;&nbsp;&nbsp;|-- jpeg-9.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- libmcrypt-2.5.8.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- libmemcached-1.0.18.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- libpng-1.6.7.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- libzip-0.11.2.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- php-5.6.7.tar.gz  
|&nbsp;&nbsp;&nbsp;&nbsp;|-- zlib-1.2.8.tar.gz  
|--&nbsp;&nbsp;init.sls  
|--&nbsp;&nbsp;install.sls


注:  
1. init.sls为入口文件，它调用install.sls和extmod.sls，install.sls为php5.6及其依赖库的安装；extmod为php5.6的扩展模块安装  
2. files目录为各种源码包的存放路径

###init.sls文件说明###

include:  
&nbsp;&nbsp; - php56.install  
&nbsp;&nbsp; - php56.extmod

注: php56.install中的php56必须和/export/salt/php56名字一直，否则不能调用install.sls和extmod文件

### install.sls文件说明###

\#\#各变量的定义\#\#
<pre>
{% set jpeg         = 'jpeg-9.tar.gz' %}
{% set zlib         = 'zlib-1.2.8.tar.gz' %}
{% set libpng       = 'libpng-1.6.7.tar.gz' %}
{% set libzip       = 'libzip-0.11.2.tar.gz' %}
{% set libmcrypt    = 'libmcrypt-2.5.8.tar.gz' %}
{% set libmemcached = 'libmemcached-1.0.18.tar.gz' %}
{% set freetype     = 'freetype-2.5.5.tar.gz' %}
{% set php          = 'php-5.6.7.tar.gz' %}
{% set php_path     = '/opt/php56' %}
{% set phplib       = '/opt/php56/php_libs' %}
{% set pkg_list     = [jpeg, zlib, libpng, libzip,libmcrypt, libmemcached, freetype] %}
</pre>

\#\#基础包的安装\#\#
<pre>
#base_pkgs
base_pkgs:
  pkg.installed:
    - pkgs:
      - gcc
      - gcc-c++
      - autoconf
      - automake
      - libtool
      - openssl-devel
      - pcre-devel
      - zlib-devel
      - libtool-ltdl-devel
</pre>

\#\#对pkg_list列表中的软件包循环configure&make&make install\#\#
<pre>
#source&&extract package&&compile
{% for pkg in pkg_list %}
{{ pkg }}:
  file.managed:
    - name: /usr/local/src/{{ pkg }}
    - unless: test -e /usr/local/src/{{ pkg }}
    - source: salt://php56/files/{{ pkg }}

{{ pkg }}_tar:
  cmd.run:
    - cwd: /usr/local/src
    - names:
      - tar xvzf {{ pkg }}
    - unless: test -d /usr/local/src/{{ pkg.split('.')[:-2]|join('.') }}
    - require:
      - file: {{ pkg }}

{{ pkg }}_compile:
  cmd.run:
    - cwd: /usr/local/src/{{ pkg.split('.')[:-2]|join('.') }}
    - name: |
{% if pkg == freetype %}
        ./configure --prefix="{{ phplib }}/{{ pkg.split('-')[0] }}" CPPFLAGS='-I{{ phplib }}/libpng/include  -I{{ phplib }}/zlib/include'
{% else %}
        ./configure --prefix="{{ phplib}}/{{ pkg.split('-')[0] }}"
{% endif %}
        make
        make install
    - require:
      - cmd: {{ pkg }}_tar
{% if pkg == freetype %}
      - cmd: {{ libpng }}_compile
      - cmd: {{ zlib }}_compile
{% endif %}
    - unless: test -d {{ phplib }}/{{ pkg.split('-')[0] }}
{% endfor %}
</pre>

注：由于freetype软件包依赖libpng和zlib，它的configure参数和其他的包不一样，加了if语句进行判断

\#\#php5.6安装\#\#
<pre>
#install php5.6
{{ php }}:
  file.managed:
    - name: /usr/local/src/{{ php }}
    - unless: test -e /usr/local/src/{{ php }}
    - source: salt://php56/files/{{ php }}

{{ php }}_tar:
  cmd.run:
    - cwd: /usr/local/src
    - names:
      - tar xvzf {{ php }}
    - unless: test -d /usr/local/src/{{ php.split('.')[:-2]|join('.') }}
    - require:
      - file: {{ php }}

{{ php }}_install:
  cmd.run:
    - cwd: /usr/local/src/{{ php.split('.')[:-2]|join('.') }}
    - name: |
        ./configure --prefix={{ php_path }} --with-config-file-path={{ php_path }}/lib --with-libzip={{ phplib }}/libzip --with-freetype-dir={{ phplib }}/freetype --with-pcre-dir --with-iconv-dir --with-jpeg-dir={{ phplib }}/jpeg --with-png-dir={{ phplib }}/libpng --with-zlib={{ phplib }}/zlib --enable-bcmath --enable-shmop --enable-sysvsem --enable-sysvshm --enable-sysvmsg --with-curl --with-gettext --enable-fpm --with-gd --enable-gd-native-ttf --with-openssl --with-mhash --enable-sockets --with-xmlrpc --enable-ftp --enable-zip --enable-calendar --enable-static --enable-wddx --enable-pcntl --enable-soap --enable-mysqlnd --with-mysql --with-mysqli --with-pdo-mysql --enable-mbstring --with-fpm-user=www --with-fpm-group=www --with-mcrypt={{ phplib }}/libmcrypt --enable-opcache CPPFLAGS='-I{{ phplib }}/libpng/include -I{{ phplib }}/zlib/include -I{{ phplib }}/libzip/include -I{{ phplib }}/libzip/lib/libzip/include'
        make
        make install
    - require:
      - cmd: {{ php }}_tar
    - unless: test -e {{ php_path }}/bin/php
</pre>

###extmod.sls文件说明###

\#\#php扩展包和安装路径的变量定义\#\#
<pre>
{% set redis       = 'redis-2.2.3.tgz' %}
{% set mongo       = 'mongo-1.4.2.tgz' %}
{% set memcache    = 'memcache-2.2.7.tgz' %}
{% set memcached   = 'memcached-2.2.0.tgz' %}
{% set phpextmod   = '/opt/php56/php_extend' %}
{% set php_path    = '/opt/php56' %}
{% set phplib      = '/opt/php56/php_libs' %}
{% set phpmod_dir  = '/opt/php56/lib/php/extensions/no-debug-non-zts-20131226' %}
{% set pkg_list = [redis, mongo, memcache, memcached] %}
</pre>

\#\#循环安装redis、mongo、memcache、memcached等php扩展\#\#
<pre>
#source&&extract package&&compile
{% for pkg in pkg_list %}
{{ pkg }}:
  file.managed:
    - name: /usr/local/src/{{ pkg }}
    - unless: test -e /usr/local/src/{{ pkg }}
    - source: salt://php56/files/extmod/{{ pkg }}

{{ pkg }}_tar:
  cmd.run:
    - cwd: /usr/local/src
    - names:
      - tar xvzf {{ pkg }}
    - unless: test -d /usr/local/src/{{ pkg.split('.')[:-1]|join('.') }}
    - require:
      - file: {{ pkg }}

{{ pkg }}_install:
  cmd.run:
    - cwd: /usr/local/src/{{ pkg.split('.')[:-1]|join('.') }}
    - name: |
        {{ php_path }}/bin/phpize
{% if pkg == memcached %}
        ./configure --with-php-config={{ php_path }}/bin/php-config --with-libmemcached-dir={{ phplib }}/libmemcached --enable-memcached --disable-memcached-sasl
{% else %}
        ./configure --with-php-config={{ php_path }}/bin/php-config
{% endif %}
        make
        make install
    - require:
      - cmd: {{ pkg }}_tar
    - unless: test -e {{ phpmod_dir }}/{{ pkg.split('-')[0] }}.so
</pre>

注：  
1.memcached相对于其他扩展包的安装，configure参数不一样，所以加了if判断语句，salt中if语句格式为{% if ... %}{% endif %}  
2.加了unless语句，用于判断.so文件是否已经存在，存在则跳过安装

\#\#设置.so文件路径\#\#
<pre>
{{ pkg }}_copy:
  cmd.run:
    - name: |
        mkdir -p {{ phpextmod }}
        cp {{ phpmod_dir }}/{{ pkg.split('-')[0] }}.so {{ phpextmod }}/{{ pkg.split('-')[0] }}.so
    - unless: test -e {{ phpextmod }}/{{ pkg.split('-')[0] }}.so
{% endfor %}
</pre>

注：将.so文件设置到/opt/php56/php_extend下
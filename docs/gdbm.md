# Make dba.so with gdbm for PHP

1) Download and install necessary packages:
   
   ```bash
   yum install gdbm-devel sqlite-devel lmdb-devel
   yum install libxml2-devel
   yum install libdb.i686 libdb.x86_64 libdb-devel libdb-utils libdb4
   ```

2) Download the PHP source code from https://www.php.net/releases/ of the same version as the one on the server. For example php-8.1.24.tar.xz.

3) Create necessary directories:
   
   ```bash
   mkdir /root/php
   mkdir /tmp/php
   ```

4) Copy php-8.1.24.tar.xz to /root/php:
   
   ```bash
   cd /root/php
   tar -xvJf php-8.1.24.tar.xz
   ./configure --enable-dba=shared --with-gdbm --without-sqlite3 --with-db4 --with-lmdb --prefix=/tmp/php
   ```

5) Make:
   
   ```bash
   make -j 4
   make install
   ```

6) Install dba from the repository and copy the config:
   
   ```bash
   yum install php-dba
   systemctl restart php-fpm
   cd /etc/php.d
   cp 20-dba.ini 20-dba.ini.copy
   yum remove php-dba
   systemctl restart php-fpm
   ```

7) Then copy dba.so from /tmp/php to /usr/lib64/php/modules and rename the config:
   
   ```bash
   cd /tmp/php/lib/php/extensions/no-debug-non-zts-20210902
   cp dba.so /usr/lib64/php/modules
   cd /etc/php.d
   cp 20-dba.ini.copy 20-dba.ini
   ```

8) Restart php-fpm:
   
   ```bash
   systemctl restart php-fpm
   ```

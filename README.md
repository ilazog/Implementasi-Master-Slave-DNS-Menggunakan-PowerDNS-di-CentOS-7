# Implementasi-Master-Slave-DNS-Menggunakan-PowerDNS-di-CentOS-7
Implementasi Master Slave DNS Menggunakan PowerDNS di CentOS 7
===============

Repository ini berisikan instalasi dan konfigurasi Master-Slave DNS pada PowerDNS

## Task
Instalasi dan Konfigurasi
* PowerDNS
* Database Server MariaDB 10.1.44
* Glue Record Domain

Ketentuan pengerjaan:
* Menggunakan 2 VPS dengan OS centos7
* Menggunakan domain utama (domain.tld)
* Menggunakan DNS Server Pdns
* Menggunakan Master Slave Pdns
* Menggunakan Database Server MariaDB 10.1.44


## Instalasi dan Konfigurasi Pdns dan MariaDB 10.1.44 (Master)
#### Step 1: Install MariaDB 10.1.44
Untuk melakukan instalasi MariaDB 10.1.44 yang pertama adalah melakukan remote ke IP VPS dengan menggunakan SSH

> ```$ ssh root@ipaddress```

Setelah berhasil login ke VPS, lakukan pembaharuan paket/repository dari system operasi Centos7 dengan perintah sebagai berikut:

> ```# yum -y update```

Setelah melakukan update system selanjutnya lakukan install epel-release

> ```# yum install epel-release -y```

Setelah melakukan install epel-release selanjutnya lakukan instalasi database server mariaDB 10.1.44 dan langkah pertama ialah menambahkan repository untuk mariaDB.

> ```# vi /etc/yum.repos.d/mariadb.repo```

Masukan perintah berikut:

```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

Setelah menambahkan repo mariaDB, lakukan instalasi mariaDB dengan perintah berikut:

> ```# yum install mariadb-server```

Selanjutnya lakukan enable direktori dan file database mariaDB, berikut perintahnya:

> ```# systemctl enable mariadb```

Setelah direktori dan file database mariaDB dienable, jalankan service mariaDB dengan perintah berikut:

> ```# systemctl start mariadb```

Apabila paket database server telah selesai diinstall pastikan service mariaDB berjalan dengan status Running.

 ```
# systemctl status mariadb
● mariadb.service - MariaDB 10.1.44 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since Sat 2020-03-07 20:09:38 WIB; 22h ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 29925 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 29885 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`/usr/bin/galera_recovery`; [ $? -eq 0 ]   && systemctl set-environment _WSREP_START_POSITION=$VAR || exit 1 (code=exited, status=0/SUCCESS)
  Process: 29883 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
 Main PID: 29897 (mysqld)
   Status: "Taking your SQL requests now..."
   CGroup: /system.slice/mariadb.service
           └─29897 /usr/sbin/mysqld
```

#### Step 2: Secure mariaDB server and Configure Database

Setelah melakukan instalasi mariaDB Server selanjutnya kita harus mengamankan databese server dengan cara menambahkan password login saat mengakses mariaDB server.

> ```# mysql_secure_installation```

Nantinya kita akan melakukan perubahan password untuk root database server, pilih `Y` dan masukan Password baru yang kuat.

```
Set root password? [Y/n] Y
New password: 
Re-enter new password: 
```

Apabila ada yang lain silakan klik `Y`

```
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

Setelah itu kita coba untuk melakukan login dengan password baru yang telah dibuat dengan perintah berikut:

```
# mysql -u root -p
Input Password
```

Apabila telah login selanjutnya buat database, user dan password untuk service PowerDNS.

```
MariaDB [(none)]> Create database testpdns;
MariaDB [(none)]> grant all privileges on testdns.* to pdns@localhost identified by 'pdnspassword';
MariaDB [(none)]> flush privileges; 
```

Setelah itu pilih database testpdns;

```
MariaDB [(none)]>use testpdns;
MariaDB [testpdns]>
```

Buat table baru untuk menyimpan record pdns pada database testpdns.

```
MariaDB [testpdns]> CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT UNSIGNED DEFAULT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```
```
MariaDB [testpdns]> CREATE UNIQUE INDEX name_index ON domains(name);
```
```
MariaDB [testpdns]>CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```
```
MariaDB [testpdns]> CREATE INDEX nametype_index ON records(name,type);
```
```
MariaDB [testpdns]> CREATE INDEX domain_id ON records(domain_id);
```

```
MariaDB [testpdns]> CREATE INDEX ordername ON records (ordername);
```

```
MariaDB [testpdns]> CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [testpdns]> CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  comment               TEXT CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [testpdns]> CREATE INDEX comments_name_type_idx ON comments (name, type);
```

```
MariaDB [testpdns]> CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);
```

```
MariaDB [testpdns]> CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [testpdns]> CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);
```

```
MariaDB [testpdns]> CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  published             BOOL DEFAULT 1,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [testpdns]>CREATE INDEX domainidindex ON cryptokeys(domain_id);
```

```
MariaDB [testpdns]> CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [testpdns]> CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```

Tambahkan perintah berikut untuk membuat kunci untuk setiap table diatas.

```
MariaDB [testpdns]> ALTER TABLE records ADD CONSTRAINT `records_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [testpdns]> ALTER TABLE comments ADD CONSTRAINT `comments_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [testpdns]> ALTER TABLE domainmetadata ADD CONSTRAINT `domainmetadata_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [testpdns]> ALTER TABLE cryptokeys ADD CONSTRAINT `cryptokeys_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
```

Lihat hasil dari penambahan table dengan perintah berikut:

```
* MariaDB [testpdns]> show tables;
+--------------------+
| Tables_in_testpdns |
+--------------------+
| comments           |
| cryptokeys         |
| domainmetadata     |
| domains            |
| records            |
| supermasters       |
| tsigkeys           |
+--------------------+
7 rows in set (0.00 sec)
```

#### Step 3: Instalasi dan konfigurasi PowerDNS

Setelah membuat database dan table untuk service PowerDNS selanjutnya lakukan instalasi PowerDNS.

```# yum -y install pdns pdns-backend-mysql bind-utils```

Lakukan konfigurasi file `pdns.conf`

```
# cd /etc/pdns/
pdns#  vi pdns.conf
```

Rubah dan tambahakan perintah berikut.

```
#i################################
# launch        Which backends to launch and order to query them in
#
# launch=bind (kasih tanda pagar untuk nonaktifkan)
launch=gmysql
gmysql-host=localhost
gmysql-user=pdns (user database)
gmysql-password=y4m4h475 (Password database)
gmysql-dbname=testpdns (nama database)
```

Save dan Close kemudian enable serta aktifkan service PowerDNS.

```
# systemctl enable pdns
# systemctl start pdns
```

Pastikan service PowerDNS berjalan

```
# systemctl status pdns
● pdns.service - PowerDNS Authoritative Server
   Loaded: loaded (/usr/lib/systemd/system/pdns.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-03-15 13:08:11 WIB; 1h 21min ago
     Docs: man:pdns_server(1)
           man:pdns_control(1)
           https://doc.powerdns.com
 Main PID: 2796 (pdns_server)
   CGroup: /system.slice/pdns.service
           └─2796 /usr/sbin/pdns_server --guardian=no --daemon=no --disable-s...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: UDP server bound to 0....
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: UDPv6 server bound to ...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: TCP server bound to 0....
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: TCPv6 server bound to ...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: PowerDNS Authoritative...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Using 64-bits mode. Bu...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: PowerDNS comes with AB...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Creating backend conne...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: About to create 3 back...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Done launching threads...
Hint: Some lines were ellipsized, use -l to show in full.
```

## Instalasi dan Konfigurasi Pdns dan MariaDB 10.1.44 (Slave)
#### Step 1: Install MariaDB 10.1.44
Untuk melakukan instalasi MariaDB 10.1.44 yang pertama adalah melakukan remote ke IP VPS dengan menggunakan SSH

> ```$ ssh root@ipaddress```

Setelah berhasil login ke VPS, lakukan pembaharuan paket/repository dari system operasi Centos7 dengan perintah sebagai berikut:

> ```# yum -y update```

Setelah melakukan update system selanjutnya lakukan install epel-release

> ```# yum install epel-release -y```

Setelah melakukan install epel-release selanjutnya lakukan instalasi database server mariaDB 10.1.44 dan langkah pertama ialah menambahkan repository untuk mariaDB.

> ```# vi /etc/yum.repos.d/mariadb.repo```

Masukan perintah berikut:

```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1```
```
Setelah menambahkan repo mariaDB, lakukan instalasi mariaDB dengan perintah berikut:

```# yum install mariadb-server```

Selanjutnya lakukan enable direktori dan file database mariaDB, berikut perintahnya:

```# systemctl enable mariadb```

Setelah direktori dan file database mariaDB dienable, jalankan service mariaDB dengan perintah berikut:

```# systemctl start mariadb```

Apabila paket database server telah selesai diinstall pastikan service mariaDB berjalan dengan status Running.


```
 systemctl status mariadb
● mariadb.service - MariaDB 10.1.44 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since Sat 2020-03-07 20:09:38 WIB; 22h ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 29925 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 29885 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`/usr/bin/galera_recovery`; [ $? -eq 0 ]   && systemctl set-environment _WSREP_START_POSITION=$VAR || exit 1 (code=exited, status=0/SUCCESS)
  Process: 29883 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
 Main PID: 29897 (mysqld)
   Status: "Taking your SQL requests now..."
   CGroup: /system.slice/mariadb.service
           └─29897 /usr/sbin/mysqld
```

#### Step 2: Secure mariaDB server and Configure Database

Setelah melakukan instalasi mariaDB Server selanjutnya kita harus mengamankan databese server dengan cara menambahkan password login saat mengakses mariaDB server.

> ```# mysql_secure_installation```

Nantinya kita akan melakukan perubahan password untuk root database server, pilih `Y` dan masukan Password baru yang kuat.

```
Set root password? [Y/n] Y
New password: 
Re-enter new password: 
```

Apabila ada yang lain silakan klik `Y`

```
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

Setelah itu kita coba untuk melakukan login dengan password baru yang telah dibuat dengan perintah berikut:

```
# mysql -u root -p
Input Password
```

Apabila telah login selanjutnya buat database, user dan password untuk service PowerDNS.

```
MariaDB [(none)]> Create database Slave_dns;
MariaDB [(none)]> grant all privileges on Slave_dns.* to pdns@localhost identified by 'pdnspassword';
MariaDB [(none)]> flush privileges; 
```

Setelah itu pilih database testpdns;

```
MariaDB [(none)]>use Slave_pdns;
MariaDB [Slave_pdns]>
```

Buat table baru untuk menyimpan record pdns pada database testpdns.

```
MariaDB [Slave_pdns]> CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT UNSIGNED DEFAULT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE UNIQUE INDEX name_index ON domains(name);
```

```
MariaDB [Slave_pdns]>CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE INDEX nametype_index ON records(name,type);
```

```
MariaDB [Slave_pdns]> CREATE INDEX domain_id ON records(domain_id);
```

```
MariaDB [Slave_pdns]> CREATE INDEX ordername ON records (ordername);
```

```
MariaDB [Slave_pdns]> CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  comment               TEXT CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE INDEX comments_name_type_idx ON comments (name, type);
```

```
MariaDB [Slave_pdns]> CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);
```

```
MariaDB [Slave_pdns]> CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);
```

```
MariaDB [Slave_pdns]> CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  published             BOOL DEFAULT 1,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]>CREATE INDEX domainidindex ON cryptokeys(domain_id);
```

```
MariaDB [Slave_pdns]> CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
```

```
MariaDB [Slave_pdns]> CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```

Tambahkan perintah berikut untuk membuat kunci untuk setiap table diatas.

```
MariaDB [Slave_pdns]> ALTER TABLE records ADD CONSTRAINT `records_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [Slave_pdns]> ALTER TABLE comments ADD CONSTRAINT `comments_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [Slave_pdns]> ALTER TABLE domainmetadata ADD CONSTRAINT `domainmetadata_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
MariaDB [Slave_pdns]> ALTER TABLE cryptokeys ADD CONSTRAINT `cryptokeys_domain_id_ibfk` FOREIGN KEY (`domain_id`) REFERENCES `domains` (`id`) ON DELETE CASCADE ON UPDATE CASCADE;
```

Lihat hasil dari penambahan table dengan perintah berikut:

```
MariaDB [Slave_pdns]> show tables;
+----------------------+
| Tables_in_Slave_pdns |
+----------------------+
| comments             |
| cryptokeys           |
| domainmetadata       |
| domains              |
| records              |
| supermasters         |
| tsigkeys             |
+----------------------+
7 rows in set (0.00 sec)
```

#### Step 3: Instalasi dan konfigurasi PowerDNS

Setelah membuat database dan table untuk service PowerDNS selanjutnya lakukan instalasi PowerDNS.

> ```# yum -y install pdns pdns-backend-mysql bind-utils```

Lakukan konfigurasi file `pdns.conf`

```
# cd /etc/pdns/
pdns#  vi pdns.conf
```

Rubah dan tambahakan perintah berikut.

```
#i################################
# launch        Which backends to launch and order to query them in
#
# launch=bind (kasih tanda pagar untuk nonaktifkan)
launch=gmysql
gmysql-host=localhost
gmysql-user=pdns (user database)
gmysql-password=y4m4h475 (Password database)
gmysql-dbname=Slave_pdns (nama database)
```

Save dan Close kemudian enable serta aktifkan service PowerDNS.

```
# systemctl enable pdns
# systemctl start pdns
```

Pastikan service PowerDNS berjalan

```
# systemctl status pdns
● pdns.service - PowerDNS Authoritative Server
   Loaded: loaded (/usr/lib/systemd/system/pdns.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-03-15 13:08:11 WIB; 1h 21min ago
     Docs: man:pdns_server(1)
           man:pdns_control(1)
           https://doc.powerdns.com
 Main PID: 2796 (pdns_server)
   CGroup: /system.slice/pdns.service
           └─2796 /usr/sbin/pdns_server --guardian=no --daemon=no --disable-s...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: UDP server bound to 0....
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: UDPv6 server bound to ...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: TCP server bound to 0....
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: TCPv6 server bound to ...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: PowerDNS Authoritative...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Using 64-bits mode. Bu...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: PowerDNS comes with AB...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Creating backend conne...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: About to create 3 back...
Mar 15 13:08:11 pdns.padiakse.my.id pdns_server[2796]: Done launching threads...
Hint: Some lines were ellipsized, use -l to show in full.
```

## Konfigurasi PowerDNS Master Slave
#### Step 1: Konfigurasi PowerDNS Master

Lakukan konfigurasi file `pdns.conf`

```
# cd /etc/pdns/
pdns#  vi pdns.conf
```

Lakukan perubahan pada file  pdns.conf seperti berikut:

* Script ini digunakan untuk mengenali alamat IP dari PowerDNS Slave

```
#################################
# allow-axfr-ips        Allow zonetransfers only to these subnets
#
# allow-axfr-ips=127.0.0.0/8,::1
allow-axfr-ips=117.53.47.189 (isikan alamat IP Pdns Slave)
```

* Script ini digunakan untuk menandai bahwa PowerDNS pada VM ini berperan sebagai master

```
#################################
# master        Act as a master
#
# master=no
master=yes
```

#### Step 2: Konfigurasi PowerDNS Slave
Lakukan konfigurasi file `pdns.conf`

```
# cd /etc/pdns/
pdns#  vi pdns.conf
```

Lakukan perubahan pada file  pdns.conf seperti berikut:

* Script ini digunakan untuk mengenali alamat IP dari PowerDNS Master

```
#################################
# allow-axfr-ips        Allow zonetransfers only to these subnets
#
# allow-axfr-ips=127.0.0.0/8,::1
allow-axfr-ips=103.23.20.70 (Isikan IP Pdns Master)
```

* Script ini digunakan untuk mengizinkan alamat IP master agar dapat melakukan perubahan pada PowerDNS Slave

```
#################################
# allow-dnsupdate-from  A global setting to allow DNS updates from these IP ranges.
#
# allow-dnsupdate-from=127.0.0.0/8,::1
allow-dnsupdate-from=103.23.20.70 (Isikan IP Pdns Master)
```

* Script ini digunakan untuk mengizinkan alamat IP master agar bisa memberi info terkait dengan perubahan pada PowerDNS Master ke PowerDNS Slave

```
#################################
# allow-notify-from     Allow AXFR NOTIFY from these IP ranges. If empty, drop all incoming notifies.
#
# allow-notify-from=0.0.0.0/0,::/0
allow-notify-from=103.23.20.70 (Isikan IP Pdns Master)
```

* Script ini digunakan sebagai identitas PowerDNS Slave

```
#################################
# slave Act as a slave
#
# slave=no
slave=yes
```

* Script ini dijalankan untuk melakukan refresh pada PowerDNS Slave dengan interval waktu tertentu

```
#################################
# slave-cycle-interval  Schedule slave freshness checks once every .. seconds
#
# slave-cycle-interval=60
slave-cycle-interval=60
```

## Konfigurasi Zona dan Add Record DNS di PowerDNS Master
Langkah pertama terlebih dahulu kita buat zona untuk menyimpan record domain.

```
MariaDB [testpdns]> INSERT INTO domains (name, type) values ('padiakse.my.id', 'Master');
```

Langkah kedua tambahkan record domain pada table records

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','padiakse.my.id root.padiakse.my.id 1 10380 3600 604800 3600','SOA',86400,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','pd1.padiakse.my.id','NS',86400,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','pd2.padiakse.my.id','NS',86400,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'pd1.padiakse.my.id','103.23.20.70','A',3600,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'pd2.padiakse.my.id','117.53.47.189','A',3600,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','103.23.20.70','A',3600,NULL);
```

Langkah ketiga tambahkan kolom change_date pada table records

```
MariaDB [testpdns]> ALTER TABLE records add change_date INT DEFAULT NULL;
```

Berikut hasil penambahan record tersebut.

```
MariaDB [testpdns]> select *from records;
+----+-----------+------------------------+------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
| id | domain_id | name                   | type | content                                                        | ttl   | prio | disabled | ordername | auth | change_date |
+----+-----------+------------------------+------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
|  1 |         1 | pd1.padiakse.my.id     | A    | 103.23.20.70                                                   |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  2 |         1 | pd2.padiakse.my.id     | A    | 117.53.47.189                                                  |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  3 |         1 | padiakse.my.id         | A    | 103.23.20.70                                                   |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  4 |         1 | padiakse.my.id         | NS   | pd1.padiakse.my.id                                             | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  5 |         1 | padiakse.my.id         | NS   | pd2.padiakse.my.id                                             | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  6 |         1 | padiakse.my.id         | SOA  | padiakse.my.id. root.padiakse.my.id. 12 10380 3600 604800 3600 | 86400 |    0 |        0 | NULL      |    1 |        NULL |
+----+-----------+------------------------+------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
6 rows in set (0.00 sec)
```

## Konfigurasi Table di PowerDNS Slave

Langkah pertama tambahkan data berikut pada table supermasters

```
MariaDB [Slave_pdns]> insert into supermasters values ('103.23.20.70', 'pd2.padiakse.my.id', 'admin');
MariaDB [Slave_pdns]> select *from supermasters;
+--------------+--------------------+---------+
| ip           | nameserver         | account |
+--------------+--------------------+---------+
| 103.23.20.70 | pd2.padiakse.my.id | admin   |
+--------------+--------------------+---------+
1 rows in set (0.00 sec)
```

Langkah ketiga tambahkan kolom change_date pada table records

```
MariaDB [Slave_pdns]> ALTER TABLE records add change_date INT DEFAULT NULL;
```

Berikut kolom dari table records
```
MariaDB [Slave_pdns]> DESC records;
+-------------+----------------+------+-----+---------+----------------+
| Field       | Type           | Null | Key | Default | Extra          |
+-------------+----------------+------+-----+---------+----------------+
| id          | bigint(20)     | NO   | PRI | NULL    | auto_increment |
| domain_id   | int(11)        | YES  | MUL | NULL    |                |
| name        | varchar(255)   | YES  | MUL | NULL    |                |
| type        | varchar(10)    | YES  |     | NULL    |                |
| content     | varchar(64000) | YES  |     | NULL    |                |
| ttl         | int(11)        | YES  |     | NULL    |                |
| prio        | int(11)        | YES  |     | NULL    |                |
| disabled    | tinyint(1)     | YES  |     | 0       |                |
| ordername   | varchar(255)   | YES  | MUL | NULL    |                |
| auth        | tinyint(1)     | YES  |     | 1       |                |
| change_date | int(11)        | YES  |     | NULL    |                |
+-------------+----------------+------+-----+---------+----------------+
11 rows in set (0.01 sec)
```

## Testing Replikasi PowerDNS Master Slave
**Testing Konfigurasi dari sisi Master**

* Restart Service PowerDNS

```
[root@pdns pdns]# systemctl restart pdns
```

* Jalankan perintah berikut untuk memperbarui zona dns master

```
[root@pdns pdns]# pdnsutil increase-serial padiakse.my.id
SOA serial for zone padiakse.my.id set to 13
```

* Jalankan perintah berikut untuk melakukan notify dan mereplikasi record dns master ke pdns slave.

```
[root@pdns pdns]# pdns_control notify padiakse.my.id
Added to queue
```

* Untuk melihat apakah PowerDNS Master telah terhubung dengan PowerDNS Slave.

```
[root@pdns pdns]# systemctl stop pdns
```

```
[root@pdns pdns]# /usr/sbin/pdns_server --daemon=no --guardian=no --loglevel=9
Mar 20 08:41:10 Reading random entropy from '/dev/urandom'
Mar 20 08:41:10 Loading '/usr/lib64/pdns/libgmysqlbackend.so'
Mar 20 08:41:10 [gmysqlbackend] This is the gmysql backend version 4.1.11 reporting
Mar 20 08:41:10 This is a standalone pdns
Mar 20 08:41:10 Listening on controlsocket in '/var/run/pdns.controlsocket'
Mar 20 08:41:10 UDP server bound to 0.0.0.0:53
Mar 20 08:41:10 UDPv6 server bound to [::]:53
Mar 20 08:41:10 TCP server bound to 0.0.0.0:53
Mar 20 08:41:10 TCPv6 server bound to [::]:53
Mar 20 08:41:10 PowerDNS Authoritative Server 4.1.11 (C) 2001-2018 PowerDNS.COM BV
Mar 20 08:41:10 Using 64-bits mode. Built using gcc 4.8.5 20150623 (Red Hat 4.8.5-36).
Mar 20 08:41:10 PowerDNS comes with ABSOLUTELY NO WARRANTY. This is free software, and you are welcome to redistribute it according to the terms of the GPL version 2.
Mar 20 08:41:10 Set effective group id to 993
Mar 20 08:41:10 Set effective user id to 996
Mar 20 08:41:10 Creating backend connection for TCP
Mar 20 08:41:10 gmysql Connection successful. Connected to database 'testpdns' on 'localhost'.
Mar 20 08:41:10 About to create 3 backend threads for UDP
Mar 20 08:41:10 Master/slave communicator launching
Mar 20 08:41:10 gmysql Connection successful. Connected to database 'testpdns' on 'localhost'.
Mar 20 08:41:10 No master domains need notifications
Mar 20 08:41:10 gmysql Connection successful. Connected to database 'testpdns' on 'localhost'.
Mar 20 08:41:10 gmysql Connection successful. Connected to database 'testpdns' on 'localhost'.
Mar 20 08:41:10 gmysql Connection successful. Connected to database 'testpdns' on 'localhost'.
Mar 20 08:41:10 Done launching threads, ready to distribute questions
```

**Testing Konfigurasi dari sisi Slave**

* Untuk melihat apakah PowerDNS Slave telah terhubung dengan PowerDNS Master.

```
[root@imam pdns]# systemctl stop pdns
```

```
[root@imam pdns]# /usr/sbin/pdns_server --daemon=no --guardian=no --loglevel=9
Mar 20 08:49:30 Reading random entropy from '/dev/urandom'
Mar 20 08:49:30 Loading '/usr/lib64/pdns/libgmysqlbackend.so'
Mar 20 08:49:30 [gmysqlbackend] This is the gmysql backend version 4.1.11 reporting
Mar 20 08:49:30 This is a standalone pdns
Mar 20 08:49:30 Listening on controlsocket in '/var/run/pdns.controlsocket'
Mar 20 08:49:30 UDP server bound to 0.0.0.0:53
Mar 20 08:49:30 UDPv6 server bound to [::]:53
Mar 20 08:49:30 TCP server bound to 0.0.0.0:53
Mar 20 08:49:30 TCPv6 server bound to [::]:53
Mar 20 08:49:30 PowerDNS Authoritative Server 4.1.11 (C) 2001-2018 PowerDNS.COM BV
Mar 20 08:49:30 Using 64-bits mode. Built using gcc 4.8.5 20150623 (Red Hat 4.8.5-36).
Mar 20 08:49:30 PowerDNS comes with ABSOLUTELY NO WARRANTY. This is free software, and you are welcome to redistribute it according to the terms of the GPL version 2.
Mar 20 08:49:30 Set effective group id to 993
Mar 20 08:49:30 Set effective user id to 996
Mar 20 08:49:30 Creating backend connection for TCP
Mar 20 08:49:30 Master/slave communicator launching
Mar 20 08:49:30 gmysql Connection successful. Connected to database 'Slave_pdns' on 'localhost'.
Mar 20 08:49:30 About to create 3 backend threads for UDP
Mar 20 08:49:30 gmysql Connection successful. Connected to database 'Slave_pdns' on 'localhost'.
Mar 20 08:49:30 No new unfresh slave domains, 0 queued for AXFR already, 0 in progress
Mar 20 08:49:30 gmysql Connection successful. Connected to database 'Slave_pdns' on 'localhost'.
Mar 20 08:49:30 gmysql Connection successful. Connected to database 'Slave_pdns' on 'localhost'.
Mar 20 08:49:30 gmysql Connection successful. Connected to database 'Slave_pdns' on 'localhost'.
Mar 20 08:49:30 Done launching threads, ready to distribute questions
```

* Jalankan ulang/Restart service PowerDNS Slave kemudian cek table Records pada database PowerDNS Slave.

```
[root@imam pdns]# systemctl restart pdns
```

```
MariaDB [Slave_pdns]> select *from records;
+-----+-----------+------------------------+------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
| id  | domain_id | name                   | type | content                                                      | ttl   | prio | disabled | ordername | auth | change_date |
+-----+-----------+------------------------+------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
|  98 |         1 | padiakse.my.id         | SOA  | padiakse.my.id root.padiakse.my.id 13 10380 3600 604800 3600 | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  99 |         1 | padiakse.my.id         | A    | 103.23.20.70                                                 |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 100 |         1 | padiakse.my.id         | NS   | pd1.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 101 |         1 | padiakse.my.id         | NS   | pd2.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 102 |         1 | pd1.padiakse.my.id     | A    | 103.23.20.70                                                 |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 104 |         1 | pd2.padiakse.my.id     | A    | 117.53.47.189                                                |  3600 |    0 |        0 | NULL      |    1 |        NULL |
+-----+-----------+------------------------+------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
6 rows in set (0.00 sec)
```

**Uji Coba untuk Menambah Record A, NS, TXT, MX, CNAME****

Tambahkan record A pada sisi Master sebagai contoh kami menambahkan subdomain Archive dan berikut langkahnnya.

* Add record pada table records dari sisi Master

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'Archive.padiakse.my.id','103.23.20.70','A',3600,NULL);
```

Tambahkan record NS pada sisi Master sebagai contoh kami menambahkan name server ns1 dan ns2 pada domain padiakse.my.id dan berikut langkahnnya.

* Add record NS pada table records dari sisi Master

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','ns1.padiakse.my.id','NS',86400,NULL);
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','ns2.padiakse.my.id','NS',86400,NULL);
```

* Add record MX pada table records dari sisi Master

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'padiakse.my.id','mail.padiakse.my.id','MX',3600,10);
```

* Add record CNAME pada table records dari sisi Master

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'www.padiakse.my.id','padiakse.my.id','CNAME',3600,NULL);
```

* Add record TXT pada table records dari sisi Master (Merupakan record saat menggunakan SSL For Free)

```
MariaDB [testpdns]> INSERT INTO records (domain_id, name, content, type,ttl,prio) VALUES (1,'_acme-challenge.padiakse.my.id','OQiVvO0Rzrr5l-XPWIqzCYJOlSfMCoIoEl6iMjidCxo','TXT',1,NULL);
```

* Berikut hasil penambahan record pada sisi Master

```
MariaDB [testpdns]> select *from records;
+----+-----------+--------------------------------+-------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
| id | domain_id | name                           | type  | content                                                        | ttl   | prio | disabled | ordername | auth | change_date |
+----+-----------+--------------------------------+-------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
|  1 |         1 | pd1.padiakse.my.id             | A     | 103.23.20.70                                                   |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  2 |         1 | pd2.padiakse.my.id             | A     | 117.53.47.189                                                  |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  3 |         1 | padiakse.my.id                 | A     | 103.23.20.70                                                   |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  4 |         1 | padiakse.my.id                 | NS    | pd1.padiakse.my.id                                             | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  5 |         1 | padiakse.my.id                 | NS    | pd2.padiakse.my.id                                             | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  6 |         1 | Archive.padiakse.my.id         | A     | 117.53.47.189                                                  |  3600 | NULL |        0 | NULL      |    1 |        NULL |
|  7 |         1 | padiakse.my.id                 | SOA   | padiakse.my.id. root.padiakse.my.id. 13 10380 3600 604800 3600 | 86400 |    0 |        0 | NULL      |    1 |        NULL |
|  8 |         1 | padiakse.my.id                 | NS    | ns1.padiakse.my.id                                             | 86400 | NULL |        0 | NULL      |    1 |        NULL |
|  9 |         1 | padiakse.my.id                 | NS    | ns2.padiakse.my.id                                             | 86400 | NULL |        0 | NULL      |    1 |        NULL |
| 10 |         1 | padiakse.my.id                 | MX    | mail.padiakse.my.id                                            |  3600 |   10 |        0 | NULL      |    1 |        NULL |
| 11 |         1 | www.padiakse.my.id             | CNAME | padiakse.my.id                                                 |  3600 | NULL |        0 | NULL      |    1 |        NULL |
| 12 |         1 | _acme-challenge.padiakse.my.id | TXT   | OQiVvO0Rzrr5l-XPWIqzCYJOlSfMCoIoEl6iMjidCxo                    |     1 | NULL |        0 | NULL      |    1 |        NULL |
+----+-----------+--------------------------------+-------+----------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
12 rows in set (0.00 sec)
```

* Perbarui zona dns master

```
[root@pdns pdns]# pdnsutil increase-serial padiakse.my.id
SOA serial for zone padiakse.my.id set to 13
```

* Berikan notify dan mereplikasi record dns master ke pdns slave.

 ```
[root@pdns pdns]# pdns_control notify padiakse.my.id
Added to queue
```

* Pengecekan dari sisi Slave dengan cara jalankan ulang/Restart service PowerDNS Slave kemudian cek table Records pada database PowerDNS Slave.

```
[root@imam pdns]# systemctl restart pdns
```

```
MariaDB [Slave_pdns]> select *from records;
+-----+-----------+--------------------------------+-------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
| id  | domain_id | name                           | type  | content                                                      | ttl   | prio | disabled | ordername | auth | change_date |
+-----+-----------+--------------------------------+-------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
| 111 |         1 | padiakse.my.id                 | SOA   | padiakse.my.id root.padiakse.my.id 14 10380 3600 604800 3600 | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 112 |         1 | padiakse.my.id                 | A     | 103.23.20.70                                                 |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 113 |         1 | padiakse.my.id                 | NS    | ns1.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 114 |         1 | padiakse.my.id                 | NS    | ns2.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 115 |         1 | padiakse.my.id                 | NS    | pd1.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 116 |         1 | padiakse.my.id                 | NS    | pd2.padiakse.my.id                                           | 86400 |    0 |        0 | NULL      |    1 |        NULL |
| 117 |         1 | padiakse.my.id                 | MX    | mail.padiakse.my.id                                          |  3600 |   10 |        0 | NULL      |    1 |        NULL |
| 118 |         1 | pd1.padiakse.my.id             | A     | 103.23.20.70                                                 |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 119 |         1 | pd2.padiakse.my.id             | A     | 117.53.47.189                                                |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 120 |         1 | _acme-challenge.padiakse.my.id | TXT   | "OQiVvO0Rzrr5l-XPWIqzCYJOlSfMCoIoEl6iMjidCxo"                |     1 |    0 |        0 | NULL      |    1 |        NULL |
| 121 |         1 | archive.padiakse.my.id         | A     | 117.53.47.189                                                |  3600 |    0 |        0 | NULL      |    1 |        NULL |
| 122 |         1 | www.padiakse.my.id             | CNAME | padiakse.my.id                                               |  3600 |    0 |        0 | NULL      |    1 |        NULL |
+-----+-----------+--------------------------------+-------+--------------------------------------------------------------+-------+------+----------+-----------+------+-------------+
12 rows in set (0.00 sec)
```

Setelah melakukan perubahan nameserver dan penambahan record pada domain, maka akan menyebabkan domain mengalami proses propagasi yang mana akan membutuhkan waktu maksimal hingga 2x24 jam untuk dapat resolv sepenuhnya keseluruh ISP dan cepat atau lambatnya proses propagasi tergantung ISP yang anda gunakan.

**Add Glue Record Pada Portal Domain**

Untuk menambahkan glue record pada domain.tld, silakan menghubungi pihak registrar domain tersebut dan dalam case ini kami menggunakan domain dari registrar Domain Cloud.

* Cara lihat registrar domain

```
$ whois domain.tld
Sponsoring Registrar PANDI ID:garuda
Sponsoring Registrar Organization:Domain Cloud
Sponsoring Registrar City:Jakarta Selatan
Sponsoring Registrar State/Province:Jakarta
Sponsoring Registrar Postal Code:12870
Sponsoring Registrar Country:ID
Sponsoring Registrar Phone:02129682828
Sponsoring Registrar Contact Email:registrar@isi.co.id
```

* Masuk pada portal domain dan pilih bagian name server masukan nama name server dan Ip Address kemudian save changes.


<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/1.png" width="500">
<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/2.png" width="500">

**Pengecekan untuk Master Slave PowerDNS**

* Cek Glue record domain

<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/3.png" width="500">

* Cek Record Domain

```
$ dig padiakse.my.id ns @b.dns.id +short
pd1.padiakse.my.id.
pd2.padiakse.my.id.
```

```
$ dig pd1.padiakse.my.id +short
103.23.20.70
```

```
$ dig pd2.padiakse.my.id +short
117.53.47.189
```

```
$ dig padiakse.my.id @pd1.padiakse.my.id +short
103.23.20.70
```

```
$ dig padiakse.my.id @pd2.padiakse.my.id +short
103.23.20.70
```

 ```
$ dig archive.padiakse.my.id +short
117.53.47.189
```

* Cek Record A Domain

```
$ dig archive.padiakse.my.id +short
117.53.47.189
```

* Cek Record MX Domain

```
$ dig padiakse.my.id mx +short
10 mail.padiakse.my.id.
```

* Cek Record NS Domain

```
$ dig padiakse.my.id ns +short
ns1.padiakse.my.id.
ns2.padiakse.my.id.
pd1.padiakse.my.id.
pd2.padiakse.my.id.
```

* Cek Record TXT Domain

```
$ dig  _acme-challenge.padiakse.my.id txt +short
"OQiVvO0Rzrr5l-XPWIqzCYJOlSfMCoIoEl6iMjidCxo"
```

* Cek Record CNAME Domain

```
$ dig www.padiakse.my.id cname +short
padiakse.my.id.
```

Apabila dengan tools dig belum muncul untuk record tersebut kemungkinan besar record tersebut masih dalam proses *Propagasi* atau *kita salah memaasukan record tersebut* untuk itu kita dapat mengecek kembali pada table record dan apabila tidak ada masalah pada table record kita dapat melakukan pengecekan proses propagasi record domain dan berikut kami lampirkan:

<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/34.png" width="500">
<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/36.png" width="500">

Berikut dokumentasi dari report record DNS yang telah ditambahkan tadi.

<img src="https://manan.s3-id-jkt-1.kilatstorage.id/gambar/35.png" width="500">


**Sekian dan Terima Kasih**

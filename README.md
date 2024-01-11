## Цель домашнего задания
1) Создатþ свой RPM пакет (можно взāтþ свое приложение, либо собратþ, например,
апач с определеннýми опциāми)
2) Создатþ свой репозиторий и разместитþ там ранее собраннýй RPM


## Установка

[root@RPM ~]# dnf install langpacks-en glibc-all-langpacks -y

[root@RPM ~]# useradd builder

[root@RPM ~]# yum install -y \
> redhat-lsb-core \
> wget \
> rpmdevtools \
> rpm-build \
> createrepo \
> yum-utils \
> gcc

[root@RPM ~]# wget https://nginx.org/packages/centos/8/SRPMS/nginx-1.20.2-1.el8.ngx.src.rpm

[root@RPM ~]# rpm -i nginx-1.*

[root@RPM ~]# wget https://github.com/openssl/openssl/archive/refs/heads/OpenSSL_1_1_1-stable.zip

[root@RPM ~]# unzip OpenSSL_1_1_1-stable.zip

[root@RPM ~]# yum-builddep rpmbuild/SPECS/nginx.spec

Добавляем ссылку на openssl:

[root@RPM ~]# vi /root/rpmbuild/SPECS/nginx.spec

[root@RPM ~]# rpmbuild -bb rpmbuild/SPECS/nginx.spec

[root@RPM ~]# ll rpmbuild/RPMS/x86_64/
total 4676
-rw-r--r--. 1 root root 2250124 янв 11 07:20 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2533844 янв 11 07:20 nginx-debuginfo-1.20.2-1.el8.ngx.x86_64.rpm

yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm

[root@RPM ~]# systemctl start nginx
[root@RPM ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-01-11 07:25:09 UTC; 8s ago
     Docs: http://nginx.org/en/docs/
  Process: 24866 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 24867 (nginx)
    Tasks: 5 (limit: 23220)
   Memory: 4.8M
   CGroup: /system.slice/nginx.service
           ├─24867 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           ├─24868 nginx: worker process
           ├─24869 nginx: worker process
           ├─24870 nginx: worker process
           └─24871 nginx: worker process

янв 11 07:25:09 RPM systemd[1]: Starting nginx - high performance web server...
янв 11 07:25:09 RPM systemd[1]: Started nginx - high performance web server.

[root@RPM ~]# mkdir /usr/share/nginx/html/repo

[root@RPM ~]# cp /root/rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo/

[root@RPM ~]# wget https://downloads.percona.com/downloads/percona-distribution-mysql-ps/percona-distribution-mysql-ps-8.0.28/binary/redhat/8/x86_64/percona-orchestrator-3.2.6-2.el8.x86_64.rpm -O /usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8.x86_64.rpm

[root@RPM ~]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 2 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished

● Длā прозрачности настроим в NGINX доступ к листингу каталога:
● В location / в файле /etc/nginx/conf.d/default.conf добавим директиву autoindex on. В
резулþтате location будет вýглāдетþ так:
location / {
root /usr/share/nginx/html;
index index.html index.htm;
autoindex on; Добавили эту директиву
}

[root@RPM ~]# vi /etc/nginx/conf.d/default.conf

[root@RPM ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@RPM ~]# nginx -s reload

### Добавим порт 80 в firewall:

[root@RPM ~]# sudo firewall-cmd --zone=public --add-service=http
success

Проверим на хостовой машине:

http://192.168.56.101/repo/

Результат:
Index of /repo/

../
repodata/                                          11-Jan-2024 07:28                   -
nginx-1.20.2-1.el8.ngx.x86_64.rpm                  11-Jan-2024 07:26             2250124
percona-orchestrator-3.2.6-2.el8.x86_64.rpm        16-Feb-2022 15:57             5222976

[root@RPM ~]# cat >> /etc/yum.repos.d/otus.repo << EOF
> [otus]
> name=otus-linux
> baseurl=http://localhost/repo
> gpgcheck=0
> enabled=1
> EOF


● Убедимся что репозиторий подключился и посмотрим что в нем есть:

[root@RPM ~]# yum repolist enabled | grep otus
otus                otus-linux
[root@RPM ~]# yum list | grep otus
otus-linux                                      433 kB/s | 2.8 kB     00:00    
percona-orchestrator.x86_64                    2:3.2.6-2.el8                                                     otus         


● Так как NGINX у нас уже стоит установим репозиторий percona-release:

<pre>[root@RPM ~]# yum install percona-orchestrator.x86_64 -y
Last metadata expiration check: 0:01:00 ago on Чт 11 янв 2024 07:36:49.
Dependencies resolved.
===========================================================================================================
 Package                          Architecture       Version                   Repository             Size
===========================================================================================================
Installing:
 <font color="#26A269"><b>percona-orchestrator            </b></font> x86_64             2:3.2.6-2.el8             otus                  5.0 M
Installing dependencies:
 <font color="#26A269"><b>jq                              </b></font> x86_64             1.6-8.el8                 appstream             203 k
 <font color="#26A269"><b>oniguruma                       </b></font> x86_64             6.8.2-2.el8               appstream             187 k

Transaction Summary
===========================================================================================================
Install  3 Packages

Total download size: 5.4 M
Installed size: 17 M
Downloading Packages:
(1/3): percona-orchestrator-3.2.6-2.el8.x86_64.rpm                          84 MB/s | 5.0 MB     00:00    
(2/3): jq-1.6-8.el8.x86_64.rpm                                             218 kB/s | 203 kB     00:00    
(3/3): oniguruma-6.8.2-2.el8.x86_64.rpm                                    125 kB/s | 187 kB     00:01    
-----------------------------------------------------------------------------------------------------------
Total                                                                      2.2 MB/s | 5.4 MB     00:02     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                   1/1
  Installing       : oniguruma-6.8.2-2.el8.x86_64                                                      1/3
  Running scriptlet: oniguruma-6.8.2-2.el8.x86_64                                                      1/3
  Installing       : jq-1.6-8.el8.x86_64                                                               2/3
  Installing       : percona-orchestrator-2:3.2.6-2.el8.x86_64                                         3/3
  Running scriptlet: percona-orchestrator-2:3.2.6-2.el8.x86_64                                         3/3
  Verifying        : jq-1.6-8.el8.x86_64                                                               1/3
  Verifying        : oniguruma-6.8.2-2.el8.x86_64                                                      2/3
  Verifying        : percona-orchestrator-2:3.2.6-2.el8.x86_64                                         3/3

Installed:
  jq-1.6-8.el8.x86_64      oniguruma-6.8.2-2.el8.x86_64      percona-orchestrator-2:3.2.6-2.el8.x86_64     

Complete!
</pre>

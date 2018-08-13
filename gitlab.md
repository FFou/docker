# 192.168.0.224gitlab
镜像用的是dockerhub官方镜像
```
docker pull  gitlab/gitlab-ce:10.1.4-ce.0
```
代码备份时间为每天凌晨4点位置在/home/zzjh/gitbackup下 采用cron+rsync方式  225的/home/zzjh/gitbackup下
流程是 224服务器每天凌晨4点通过backupcode.sh脚本把gitlab代码打包到gitbackup下。
225服务器通过rsync服务5点同步gitbackup目录。

##### 224gitlab启动命令(225的启动命令也一样、改下ip就可以)
```
sudo docker run --detach  --hostname 192.168.0.224 --publish 443:443 --publish 80:80 --name gitlab -v /home/zzjh/gitlab/config:/etc/gitlab -v /home/zzjh/gitlab/logs:/var/log/gitlab -v /home/zzjh/gitlab/data:/var/opt/gitlab  --restart always  gitlab/gitlab-ce:10.1.4-ce.0
```


224上面打包脚本backupcode.sh
```
#!/bin/sh
sudo tar -cvf /home/zzjh/gitbackup/$(date -d "today" +"%Y%m%d").tar /home/zzjh/gitlab
```

225上面解压脚本 startgitlab.sh
```
#!/bin/bash
#!/bin/bash
sudo docker stop gitlab
sudo docker rm gitlab
sudo rm -rf /home/zzjh/gitlab
sudo tar -xvf /home/zzjh/gitbackup/$(date -d "today" +"%Y%m%d").tar -C /home/zzjh
sudo mv /home/zzjh/home/zzjh/gitlab /home/zzjh/gitlab
sudo rm -rf /home/zzjh/home
sudo docker run --detach  --hostname 192.168.0.225 --publish 443:443 --publish 80:80 --name gitlab -v /home/zzjh/gitlab/config:/etc/gitlab -v /home/zzjh/gitlab/logs:/var/log/gitlab -v /home/zzjh/gitlab/data:/var/opt/gitlab  --restart always  gitlab/gitlab-ce:10.1.4-ce.0


```
224定时任务 sudo crontab -e
```
# m h dom mon dow user  command
15 4   * * *  /home/zzjh/backupcode.sh >> /dev/null 2>&1
```


225同步服务器命令sudo  crontab -l   每分钟同步一次
```
# m h  dom mon dow   command
1 5 * * *  sudo rsync -zvrtopg  --delete --password-file=/etc/rsyncd.secrets --delete zzjh@192.168.0.224::code /home/zzjh/gitbackup
30 5   * * *  sudo /home/zzjh/startgitlab.sh >/dev/null 2>&1

```

224rsync服务端配置
```
zzjh@c224:~$ cat /etc/rsyncd.conf
# sample rsyncd.conf configuration file

# GLOBAL OPTIONS

#motd file=/etc/motd
log file=/var/log/rsyncd
# for pid file, do not use /var/run/rsync.pid if
# you are going to run rsync out of the init.d script.
# The init.d script does its own pid file handling,
# so omit the "pid file" line completely in that case.
# pid file=/var/run/rsyncd.pid
#syslog facility=daemon
#socket options=

# MODULE OPTIONS

[ftp]

        comment = public archive
        path = /home/zzjh/gitbackup
        use chroot = yes
#       max connections=10
        lock file = /var/lock/rsyncd
# the default for read only is yes...
        read only = yes
        list = yes
        uid = nobody
        gid = nogroup
#       exclude =
#       exclude from =
#       include =
#       include from =
        auth users = zzjh
        secrets file = /etc/rsyncd.secrets
        strict modes = yes
        hosts allow = 192.168.0.225
#       hosts deny =
        ignore errors = no
        ignore nonreadable = yes
        transfer logging = no
#       log format = %t: host %h (%a) %o %f (%l bytes). Total %b bytes.
        timeout = 600
        refuse options = checksum dry-run
        dont compress = *.gz *.tgz *.zip *.z *.rpm *.deb *.iso *.bz2 *.tbz
[code]
        path=/home/zzjh/gitbackup
zzjh@c224:~$

#参考配置：https://www.cnblogs.com/shihuc/p/5628893.html

```

sudo vi /etc/rsyncd.secrets
```
zzjh:zzjhdata
```
sudo chmod 600 /etc/rsyncd.secrets

225rsync客户端配置

sudo vi /etc/rsyncd.secrets
```
zzhdata
```
sudo chmod 600 /etc/rsyncd.secrets
sudo chown zzjh:zzjh /etc/rsyncd.secrets

# 安装docker
移除旧版本：    
[root@docker ~]# yum remove docker \  
           docker-client \  
           docker-client-latest \  
           docker-common \  
           docker-latest \  
           docker-latest-logrotate \  
           docker-logrotate \  
           docker-selinux \  
           docker-engine-selinux \  
           docker-engine  
安装一些必要的系统工具：  
[root@docker ~]# yum install -y yum-utils device-mapper-persistent-data lvm2  
添加软件源信息  
[root@docker yum.repos.d]#yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  
更新yum缓存：  
[root@docker ~]# yum makecache fast  
安装docker-ce：    
[root@docker ~]# yum -y install docker-ce  
启动docker后台服务：    
[root@docker ~]#systemctl start docker  
# 拉取docker镜像（使用5.7版本的mysql）    
[root@docker ~]# docker pull mysql:5.7    
# 可以查看镜像  
[root@docker ~]# docker images  
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE  
docker.io/mysql     5.7                 f6509bac4980        3 weeks ago         373 MB  
# 启动镜像  
[root@docker ~]# docker run -itd -p 3307:3306 --name master -e MYSQL_ROOT_PASSWORD='123456' mysql:5.7  
[root@docker ~]# docker run -itd -p 3308:3306 --name slave -e MYSQL_ROOT_PASSWORD='123456' mysql:5.7  
[root@docker ~]# docker ps(可以查看启动的容器)  
可以使用Navicat可视化工具来连接测试mysql  
# 配置容器（主从一致）  
[root@docker ~]# docker exec -it CONTAINER ID /bin/bash  
root@cb6b7fdbc98a:/# apt-get update   //更新apt源  
root@cb6b7fdbc98a:/# apt-get install vim -y   //下载vim命令  
# 配置master
# mysql:5.7版本下 /etc/mysql/my.cnf文件默认没有配置，可以从/etc/mysql/mysql.conf.d/mysqld.cnf中查找基本配置  
root@cb6b7fdbc98a:/# vim /etc/mysql/my.cnf  
添加如下配置  
[mysqld]  
server-id=1    //同一局域网内注意要唯一   
log-bin=mysql-bin //开启二进制日志功能，可以随便取（关键）  
容器内重启mysql服务  
service mysql restart  
再次启动容器  
docker start master  
进入容器，创建数据同步用户，给予用户slave REPLICATION SLAVE权限和REPLICATION CLIENT权限，用于同步数据  
mysql > CREATE USER 'slave'@'%' IDENTIFIED BY '123456';  
mysql > GRANT REPLICATION SLAVE, REPLICATION CLIENT ON \*.\* TO 'slave'@'%';  
# 配置slave  
配置Master(主)一样，在Slave配置文件my.cnf中添加如下配置  
[mysqld]  
## 设置server_id,注意要唯一  
server-id=2  
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用  
log-bin=mysql-slave-bin     
## relay_log配置中继日志    
relay_log=edu-mysql-relay-bin     
# 进入master容器中的mysql    
mysql > show master status;  
  
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |    
| mysql-bin.000002 |      313 |              |                  |                   |    
 
# File和Position字段的值后面将会用到，在后面的操作完成之前，需要保证Master库不能做任何操作，否则将会引起状态变化，File和Position字段的值变化。  
# 查找master容器独立ip  
[root@docker ~]# docker inspect --format='{{.NetworkSettings.IPAdders}}'  容器id||容器名  
172.17.0.3  
# 进入slave容器中mysql：  
mysql > change master to master_host='172.17.0.3', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000002', master_log_pos=313, master_connect_retry=30;  
mysql > start slave  //开启主从复制  
mysql > show slave status \G //查询主从同步状态  
SlaveIORunning 和 SlaveSQLRunning 都是Yes，说明主从复制已经开启。此时可以测试数据同步是否成功。  

# 主从排查  
如果SlaveIORunning一直是Connecting，则说明主从复制一直处于连接状态，可以根据 Last_IO_Error提示予以排错。  
网络不通  

检查ip,端口  

密码不对  

检查是否创建用于同步的用户和用户密码是否正确  

pos不对  

检查Master的 Position  




































# 빅데이터 통합실습 (07/17)

### 팀 구성원

![07745](C:\Users\SKCC\Desktop\bigdata\07745.jpg)![06704](C:\Users\SKCC\Desktop\bigdata\06703.jpg)![08363](C:\Users\SKCC\Desktop\bigdata\08363.jpg)

- 07745 이용희
- 06703 김의현
- 08363 김지현



### 실습목표

- System Configuration Checks

- Cloudera Manager Install

- MySQL/MariaDB Installation

- Install a cluster and deploy CDH

  

### 환경구성

- CENTOS 7
- JDK 1.78
- CDH 5.15



## 1. System Configuration (공통-cluster  전체 수행)
#### 1) AWS Instance 접속확인 

- GIT BASH 통하여 접속 (pem 경로에서 수행하거나 전체경로 지정)

```
호스트1		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.185.33
호스트2		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.22.37
호스트3		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.23.59
호스트4		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.65.2
호스트5		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.68.100
```

![aws_chk](C:\Users\SKCC\Desktop\bigdata\aws_chk.png)



#### 2)  HOSTS 파일 변경

```
  $ sudo vi /etc/hosts
172.31.3.35		m1.com m1
172.31.11.79	cm.com cm
172.31.7.199	d1.com d1
172.31.9.132	d2.com d2
172.31.8.76		d3.com d3
```

![hosts](C:\Users\SKCC\Desktop\bigdata\hosts.png)



#### 3) YUM Update

```
$ sudo yum update
$ sudo yum install -y wget
```



#### 4) firewall 정지 [방화벽 정지, CentOs 7부터 iptables -> firewalld 변경]

```
$ sudo systemctl stop firewalld  //현재 작동중인 firewall 서비스 종료
$ systemctl disable firewalld  /OS 부팅 시 firewall 자동실행 해제
```



#### 5) Selinux 정지 [보안 프로그램]

```
$ sestatus //Selinux 동작모드 확인

SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```

- /etc/selinux/config 파일을 수정하여 상태를 변경 [enforcing -> disabled]

```
$ sudo  vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - SELinux is fully disabled.
SELINUX=disabled
# SELINUXTYPE= type of policy in use. Possible values are:
# targeted - Only targeted network daemons are protected.
# strict - Full SELinux protection.
SELINUXTYPE=targeted
```

```
$ sudo reboot
```



#### 5)  NTP 설정 [Cluster host 시간 동기화]

- NTP 설치

```
$ sudo yum install -y ntp
```

- NTP 서버 설정 - /etc/ntp.conf 파일에서 기본설정 서버를 주석처리 후 한국의 ntp서버 추가

```
$ sudo vi /etc/ntp.conf

# Use public servers from the pool.ntp.org project. 
# Please consider joining the pool (http://www.pool.ntp.org/join.html). 
#server 0.centos.pool.ntp.org 
#server 1.centos.pool.ntp.org 
#server 2.centos.pool.ntp.org
server kr.pool.ntp.org 
server time.bora.net
server time.kornet.net
```

- NTP 서비스 시작 / 시작프로그램에 등록 / 작동여부 확인

```
$ sudo systemctl start ntpd

$ sudo systemctl enable ntpd

Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.

$ sudo ntpq -p

     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 dadns.cdnetwork 216.239.35.12    2 u   32   64    1    1.000    1.656   0.000
 time.bora.net   ..{...          16 u   31   64    0    0.000    0.000   0.000
```



#### 6)  PASSWORD & SSH 설정

- 비밀번호 : skcc2019

```
$ sudo passwd centos

Changing password for user centos.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

```
$ sudo vi /etc/ssh/sshd_config

PasswordAuthentication yes  # no->yes 활성화

$ sudo service sshd restart

Redirecting to /bin/systemctl restart sshd.service
```

- 재접속

![ssh](C:\Users\SKCC\Desktop\bigdata\ssh.png)



#### 7) HOSTNAME 변경

```
$ sudo hostnamectl set-hostname m1.com  <- 호스트1
$ sudo hostnamectl set-hostname cm.com  <- 호스트2 
$ sudo hostnamectl set-hostname d1.com  <- 호스트3 
$ sudo hostnamectl set-hostname d2.com  <- 호스트4 
$ sudo hostnamectl set-hostname d3.com  <- 호스트5 
```

![hostname](C:\Users\SKCC\Desktop\bigdata\hostname.png)



#### 8) jdk 1.7 설치 & 환경변수 추가

```
$ sudo yum install -y vim wget unzip
$ cd /tmp
$ sudo yum list java*jdk-devel
$ sudo yum install java-1.7.0-openjdk-devel.x86_64  //1.7 설치
$ java -version
java version "1.7.0_221"
OpenJDK Runtime Environment (rhel-2.6.18.0.el7_6-x86_64 u221-b02)
OpenJDK 64-Bit Server VM (build 24.221-b02, mixed mode)


$ which javac //JAVA 설치경로
/usr/bin/javac

$ readlink -f /usr/bin/javac
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.221-2.6.18.0.el7_6.x86_64/bin/javac

$ sudo vi ~/.bash_profile
export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.221-2.6.18.0.el7_6.x86_64

$ source ~/.bash_profile

echo $JAVA_HOME
```



## 2. System Configuration (master 수행)

#### 

### 6. 추가적인 설정 [성능이슈]

#### 6-3/ Setting the vm.swappiness Linux Kernel Parameter

vm.swappiness[0~100] 값이 클수록 inactive process들에 대한 메모리 스왑이 빈번히 발생 Hadoop Cluster 사용에는 스왑이 적게 발생하도록 세팅하는 것이 적합 [vm.swappiness 1]

```
$ cat /proc/sys/vm/swappiness      // 현재 세팅값 확인
$ sudo sysctl -w vm.swappiness=1   // 스왑값 
```

## CDH install

[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/install_cm_server.html

### Configure a repository for cloudera manager

1. Download the cloudera-manager.repo file for your OS version to the /etc/yum.repos.d/ directory on the Cloudera Manager Server host [CDH가 아니라 cloudera-manager를 다운받아야 함]

```
$ sudo wget https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo -P /etc/yum.repos.d/
```

1. change the baseurl within cloudera-manager.repo to fit the version you want to install

```
$ cd /etc/yum.repos.d/cloudera-manager.repo

<AS-IS>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1

<TO-BE>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5.15.2/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1
```

1. Import the repository signing GPG key [RHEL 7]

```
$sudo rpm --import https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera
```

### Install JDK on all hosts

1. 설치가능한 자바 리스트 확인

```
yum list java*jdk-devel
```

1. 1.7버전의 JDK 설치

```
yum install java-1.7.0-openjdk-devel.x86_64
```

1. 설치확인

```
[centos@ip-172-31-46-77 ~]$ java -version
java version "1.7.0_221"
OpenJDK Runtime Environment (rhel-2.6.18.0.el7_6-x86_64 u221-b02)
OpenJDK 64-Bit Server VM (build 24.221-b02, mixed mode)
[centos@ip-172-31-46-77 ~]$ javac -version
javac 1.7.0_221
```

### Install Cloudera Manager Packages [수동버전]

1.cloudera manager를 설치할 호스트에서 아래의 명령어 실행

```
$sudo yum install cloudera-manager-daemons cloudera-manager-server
```

2.Cluster 모든 호스트에 대해 cloudera agent 설치

```
$sudo yum install cloudera-manager-daemons cloudera-manager-agent
```

### [참고] Cloudera-manager 설치 [자동버전]

```
1.  $ wget http://archive.cloudera.com/cm5/installer/5.15.2/cloudera-manager-installer.bin
```

> a. 개인정보 입력 후 다운로드 URL확인
> b. 5.15.0 설치파일로 설치하니까 5.16버전이 깔렸음
> c. 설치제거 후 재시도 $ sudo /usr/share/cmf/uninstall-cloudera-manager.sh

```
2.  $ wget http://archive.cloudera.com/cm5/installer/5.15.0/cloudera-manager-installer.bin
```

> a. 현재 디렉토리에 설치 파일 다운로드 됨

```
3.  $ chmod u+x cloudera-manager-installer.bin
```

> a. 설치파일의 권한 변경

```
4.  $ sudo ./cloudera-manager-installer.bin
```

> a. 설치 실행

1. 설치 완료 후 접속정보 안내가 뜨고, 접속이 안될경우 iptable 같은 리눅스 방화벽 확인이 필요하다는 메시지 출력

> a. 접속정보 : 13.124.227.184:7180 [ admin / admin]

### Install Database [MariaDB 설치]

[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/cm_ig_installing_configuring_dbs.html

1. MariaDB를 사용하여 RDB가 필요한 서비스들을 커버
2. JDBC Connector 설치필요
3. 필요한 계정 생성

#### 1. Installing MariaDB Server

```
$ sudo yum install mariadb-server
```

#### 2. Configuring and Starting the MariaDB Server [root / admin]

secure_installation 과정을 진행하며 root password 및 보안관련 설정 Default root 비밀번호는 공백

```
$ sudo systemctl enable mariadb
$ sudo systemctl start mariadb
$ sudo /usr/bin/mysql_secure_installation

[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] Y
New password:
Re-enter new password:
[...]
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
[...]
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

#### 3. Installing the MySQL JDBC Driver for MariaDB

- MariaDB JDBC driver is not supported. Install and use the MySQL JDBC driver instead.
- Install the JDBC driver on the Cloudera Manager Server host, as well as any other hosts running services that require database access

3-1. Download the MySQL JDBC driver from http://www.mysql.com/downloads/connector/j/5.1.html (in .tar.gz format). Extract the JDBC driver JAR file from the downloaded file

```
$ tar zxvf mysql-connector-java-5.1.46.tar.gz
```

3-2. Copy the JDBC driver, renamed, to /usr/share/java/. If the target directory does not yet exist, create it. For example

```
$ sudo mkdir -p /usr/share/java/
$ cd mysql-connector-java-5.1.46
$ sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
```

#### 4.Creating Databases for Cloudera Software

4-1. Log in as the root user, or another user with privileges to create database and grant privileges:

```
$ mysql -u root -p
```

4-2. Create databases for each service deployed in the cluster Configure all databases to use the utf8 character set.

```
MariaDB > CREATE DATABASE <database> DEFAULT CHARACTER SET <character set> DEFAULT COLLATE utf8_general_ci;
        > Query OK, 1 row affected (0.00 sec)
MariaDB > GRANT ALL ON <database>.* TO '<user>'@'%' IDENTIFIED BY '<password>';
        > Query OK, 1 row affected (0.00 sec)
MariaDB > SHOW DATABASES;  // Confirme that you have created all  Databased

<Table>
CREATE DATABASE scm DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;

<계정생성 및 권한부여>
GRANT ALL ON scm.* TO  'scm'@'%' IDENTIFIED BY 'scm';   
GRANT ALL ON amon.* TO  'amon'@'%' IDENTIFIED BY 'amon';   
GRANT ALL ON rman.* TO  'rman'@'%' IDENTIFIED BY 'rman';  
GRANT ALL ON hue.* TO  'hue'@'%' IDENTIFIED BY 'hue';  
GRANT ALL ON metastore.* TO  'hive'@'%' IDENTIFIED BY 'hive';  
GRANT ALL ON sentry.* TO  'sentry'@'%' IDENTIFIED BY 'sentry';  
GRANT ALL ON nav.* TO  'nav'@'%' IDENTIFIED BY 'nav';  
GRANT ALL ON navms.* TO  'navms'@'%' IDENTIFIED BY 'navms';  
GRANT ALL ON oozie.* TO  'oozie'@'%' IDENTIFIED BY 'oozie';  
```

### Set up the Cloudera Manager Database

#### 1.scm_prepare_database.sh 사용하여 configuration 진행

MariaDB의 경우 세팅할때 mysql 옵션 사용

```
파일명 :  /usr/share/cmf/schema/scm_prepare_database.sh 
실행예 : sudo ./scm_prepare_database.sh mysql scm scm
```

#### 2.cloudera-scm-manager Database 설정 후 재기동 시, 에러없이 서버가 정상적으로 올라감

```
$ systemctl cloudera-scm-server restart
$ tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log  //로그확인
$ netstat -antp | grep 7180                                     //서버의 default port도 떴다.
```

### Cloudera Manager Web UI : Install CDH and Other Software

#### 1.Specify hosts for your CDH cluster installation

#### 2.Select Repository

#### 3.Accept JDK License

> a. cloudera manager 호스트에만 JDK를 깔았기 때문에 설치하기로 설정해야 함

#### 4. Single User Mode

> a. 체크하지 않음

#### 5. Enter Login Credentials

> a. root 계정 비밀번호를 입력 b. root로 접속 / 비밀번호 : admin

#### 6. Install Agents

> a. 5개 서버의 [Private IP] 정보 5줄을 입력하고 다음으로 넘어가면 검색결과 확인

#### 7. Install Parcels

> a. 기본 패키지 설치

#### 8. Select Services

> 아래의 옵션들 중에서 하나를 선택하여 설치 진행 [Cloudera Navigator 설치안함]

```
Core Hadoop
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, and Hue

Core with HBase
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and HBase

Core with Impala
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Impala

Core with Search
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Solr

Core with Spark
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Spark

All Services
HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, HBase, Impala, Solr, Spark, and Key-Value Store Indexer

Custom Services
Choose your own services. Services required by chosen services will automatically be included. Flume can be added 
after your initial cluster has been set up.
```

#### 9. Assign Roles





```
- System connection (GitBash)

'''
​```
​```

'''

- Setup /etc/hosts (All nodes)


- add user (All nodes) 
	$ sudo useradd training -u 3800
	$ sudo passwd training
	$ sudo groupadd skcc
	$ sudo usermod -a -G skcc training

>>>>> 1.a.linux setup - add user.PNG	

- Create a password for user “centos” (All nodes)
	$ sudo passwd centos

- Modify sshd_config to allow password login (All nodes)
	$ sudo vi /etc/ssh/sshd_config
		변경 -> PasswordAuthentication yes

- Restart the sshd.service (All nodes)
  $ sudo service sshd restart

- Change the hostname (each of the 5 nodes)
  $ sudo hostnamectl set-hostname m1.com
  $ sudo hostnamectl set-hostname cm.com
  $ sudo hostnamectl set-hostname d1.com
  $ sudo hostnamectl set-hostname d2.com
  $ sudo hostnamectl set-hostname d3.com

  $ hostname 
  
  $ getent group skcc
  $ getent passwd training

>>>>> 1.a.linux setup - commands.PNG	  
  

- Configure the repository for CM 5.15.2
  $ sudo yum install -y wget
  $ sudo wget https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo -P /etc/yum.repos.d/

  $ sudo vi /etc/yum.repos.d/cloudera-manager.repo
  repo 파일의 baseurl 내용을 아래로 변경
  baseurl=https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.15.2/

- Install JDK on each of the hosts
  $ sudo yum install oracle-j2sdk1.7
  
>>>>> 1.a.linux setup - Install JDK.PNG	    

- yum Install CM
  $ sudo yum install -y cloudera-manager-daemons cloudera-manager-server
  
>>>>> 1.a.linux setup - yum Install CM.PNG	      
```

### b. Install a MySQL server
```
- Install and enable Maria DB (or a DB of your choice)
- Don’t forget to secure your DB installation
  $ sudo yum install -y mariadb-server
  $ sudo systemctl enable mariadb
  $ sudo systemctl start mariadb
  
>>>>> 1.b.Install a MySQL server - maria yum install and start.PNG
  
  $ sudo /usr/bin/mysql_secure_installation

		[...]
		Enter current password for root (enter for none):
		OK, successfully used password, moving on...
		[...]
		Set root password? [Y/n] Y
		New password:
		Re-enter new password:
		[...]
		Remove anonymous users? [Y/n] Y
		[...]
		Disallow root login remotely? [Y/n] N
		[...]
		Remove test database and access to it [Y/n] Y
		[...]
		Reload privilege tables now? [Y/n] Y
		[...]
		All done!  If you've completed all of the above steps, your MariaDB
		installation should now be secure.
		
>>>>>  1.b.Install a MySQL server - mysql_secure_installation.PNG		

-	Install the mysql connector or mariadb connector (모든host에 설치)
	$ wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
	$ tar zxvf mysql-connector-java-5.1.46.tar.gz
    $ sudo mkdir -p /usr/share/java/
	$ cd mysql-connector-java-5.1.46
	$ sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar

- Create the necessary users and dbs in your database
- Grant them the necessary rights
  $ mysql -u root -p

>>>>>  1.b.Install a MySQL server - Install the mysql connector or mariadb connector.PNG	


CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'password';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'password';
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'password';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'password';
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'password';
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'password';
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'password';
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'password';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'password';

FLUSH PRIVILEGES;

SHOW DATABASES;
SELECT VERSION();

>>>>>  1.b.Install a MySQL server - db setting.PNG	

EXIT;		

- Setup the CM database
	$ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm password
	$ sudo rm /etc/cloudera-scm-server/db.mgmt.properties
	$ sudo systemctl start cloudera-scm-server

- nd1 에서 db 실행 후 training 계정 생성
  $ mysql -u root -p
  비밀번호 : password 입력

  GRANT ALL ON *.* TO 'training'@'%' IDENTIFIED BY 'training';
  FLUSH PRIVILEGES;
  EXIT;
  
>>>>>  1.b.Install a MySQL server - db training user setting.PNG	

- Setup the CM database
  $ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm password
  $ sudo rm /etc/cloudera-scm-server/db.mgmt.properties
```

### c. Install Cloudera Manager
```
- 리눅스 셋팅 시 jdk 설치 후 기 설치 함. (sudo yum install -y cloudera-manager-daemons cloudera-manager-server)
- CM 구동
  $ sudo systemctl start cloudera-scm-server

- prepare to install the cluster through the CM GUI installation process
	http://13.124.2.159:7180/cmf/login  admin / admin

>>>>> 1.c.Install Cloudera Manager - CM GUI~7.PNG

-	host 명 등록
	nd1.com,	nd2.com,	nd3.com,	nd4.com,	nd5.com

>>>>> 1.c.Install Cloudera Manager - CM GUI-Main.PNG

- parcel(package) 에서 Kafka 패키지 "다운로드" -> "배포" -> "활성" 수행  
  cluster 메뉴에서 Kafka 서비스 추가
  -> 브로커는 데이터노드에, 게이트웨이는 전체 노드에
  
>>>>>   1.c.Install Cloudera Manager - CM GUI-KAFKA.PNG

- sqoop
cluster 메뉴에서 sqoop 1 client 서비스 추가

>>>>>   1.c.Install Cloudera Manager - CM GUI-sqoop.PNG

- Spark
cluster 메뉴에서 Spark 서비스 추가
role 설정후 마직막에 클러스터 재구성 및 재시작 수행
>>>>>   1.c.Install Cloudera Manager - CM GUI-Spark.PNG

```

## 2. In MySQL create the sample tables that will be used for the rest the data_ingest
```
  $ mysql -u root -p

-- test database 생성
CREATE DATABASE test;
EXIT;

-- filezilla 툴로 쿼리 파일 서버로 전송

-- data import (테이블 및 데이터 생성)
  $ mysql -u root -p test < ./authors.sql
  $ mysql -u root -p test < ./posts.sql

>>>>> 2.Database,테이블 및 데이터 생성.PNG

-- 권한 부여
  $ mysql -u root -p
GRANT ALL ON test.* TO 'training'@'%' IDENTIFIED BY 'training';
GRANT ALL ON test.* TO 'training'@'localhost' IDENTIFIED BY 'training';
FLUSH PRIVILEGES;

>>>>> 2.training 계정 권한부여.PNG

-- 확인
$ mysql -u training -p

show databases;
use test;
show tables;

desc authors;
desc posts;

select count(*) from authors limit 10;
select count(*) from posts limit 10;
>>>>> 2.데이터 확인.PNG

```
## 3. Extract tables authors and posts from the databases and create hive tables
```
- sqoop import with hive direct
- training 계정으로 리눅스 로그인 
sqoop import --connect jdbc:mysql://172.31.4.133/test --username training --password training --table authors --target-dir /authors --hive-import --create-hive-table --hive-table default.authors

sqoop import --connect jdbc:mysql://172.31.4.133/test --username training --password training --table posts --target-dir /posts --hive-import --create-hive-table --hive-table default.posts

>>>>>3. Extract tables authors and posts from the databases and create hive tables.PNG
```

## 4. Create and run a Hive/Impala Query. From the query, generate the results dataset that you will use in the next step to export in MySql.
```
- 작성자와 작성자가 작성한 게시물의 개수를 쿼링하여 /results 에 저장
insert overwrite directory '/results'
row format delimited fields terminated by '\t'
select A.id,
       A.first_name AS fname,
       A.last_name AS lname,
       B.num_posts AS num_posts
  from authors A
 inner join ( select author_id, count(author_id) AS num_posts
                from posts P
               group by author_id ) B
    on A.id = B.author_id
 >>>>> 4. Create and run a HiveImpala Query.PNG
 >>>>> 4.Create and run a HiveImpala Query-results.PNG
```
## 5. Export the data from above query to MySql
```
- MySql test.results 테이블 생성
CREATE TABLE `results` (
  `id` int NOT NULL,
  `fname` varchar(255) default NULL,
  `lname` varchar(255) default NULL,
  `num_posts` int default 0
);

- sqoop export
sqoop export \
--connect jdbc:mysql://localhost/test \
--username training \
--password training \
--table results \
--export-dir /results
--input-fields-terminated-by '\t'
```
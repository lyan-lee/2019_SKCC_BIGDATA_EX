# 빅데이터 통합실습 PART1 - 빅데이터 클러스터 구축 (07/17)

### 팀 구성원 (팀6)

| ![07745](/images/07745.jpg) | ![06703](/images/06703.jpg) | ![08363](/images/08363.jpg) |
| ------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------- |
| 이용희 (07745)<br />과금혁신Unit                  | 김의현 (06703)<br />과금혁신Unit                  | 김지현(08363)<br /> 과금혁신Unit                  |



### 실습목표

- System Configuration Checks

- MariaDB Installation

- Cloudera Manager Install

- Install a cluster and deploy CDH

  

### 환경구성

- CENTOS 7
- JDK 1.78
- CDH 5.15



## 1. System Configuration (cluster  전체수행)
#### 1) AWS Instance 접속확인 

- GIT BASH 통하여 접속 (pem 경로에서 수행하거나 전체경로명 지정)

```
호스트1		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.185.33
호스트2		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.22.37
호스트3		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.23.59
호스트4		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.65.2
호스트5		ssh -i /c/Users/SKCC/Desktop/bigdata/SKCC.pem centos@15.164.68.100
```

![aws_chk](/images/aws_chk.png)



#### 2)  HOSTS 파일 수정

```
$ sudo vi /etc/hosts
172.31.3.35		m1.com m1
172.31.11.79	cm.com cm
172.31.7.199	d1.com d1
172.31.9.132	d2.com d2
172.31.8.76		d3.com d3
```

![hosts](/images/hosts.png)



#### 3) YUM Update

```
$ sudo yum update
$ sudo yum install -y wget
```



#### 4) firewall (방화벽) 정지

- 현재 작동중인 firewall 서비스 종료

```
$ sudo systemctl stop firewalld
```

- OS 부팅 시 firewall 자동실행 해제

```
$ sudo systemctl disable firewalld 
```



#### 5) Selinux (보안프로그램) 정지

- Selinux 동작모드 확인

```
$ sestatus
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

- config 파일을 수정 [enforcing -> disabled]

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

- selinux  정지 적용을 위해 서버 재기동

```
$ sudo reboot
```



#### 5)  NTP 설정 (Cluster host 시간 동기화)

- NTP 설치

```
$ sudo yum install -y ntp
```

- NTP 서버 설정  (/etc/ntp.conf 파일에서 기본설정 서버를 주석처리 후 한국의 ntp서버 추가)

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

- NTP 서비스 시작

```
$ sudo systemctl start ntpd
```

- NTP 서비스 시작프로그램에 등록

```
$ sudo systemctl enable ntpd
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
```

- NTP 작동여부 확인

```
$ sudo ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 dadns.cdnetwork 216.239.35.12    2 u   32   64    1    1.000    1.656   0.000
 time.bora.net   ..{...          16 u   31   64    0    0.000    0.000   0.000
```



#### 6)  PASSWORD & SSH 설정

- 비밀번호 설정 (skcc2019)

```
$ sudo passwd centos
Changing password for user centos.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

- SSH 활성화

```
$ sudo vi /etc/ssh/sshd_config
PasswordAuthentication yes  # no->yes 활성화
```

- SSH 서비스 재시작

```
$ sudo service sshd restart
Redirecting to /bin/systemctl restart sshd.service
```

- 서버 재접속하여 확인

```
$ ssh centos@15.164.68.100
```

![ssh](/images/ssh.png)



#### 7) HOSTNAME 변경

```
$ sudo hostnamectl set-hostname m1.com  <- 호스트1에서 수행
$ sudo hostnamectl set-hostname cm.com  <- 호스트2에서 수행 
$ sudo hostnamectl set-hostname d1.com  <- 호스트3에서 수행 
$ sudo hostnamectl set-hostname d2.com  <- 호스트4에서 수행 
$ sudo hostnamectl set-hostname d3.com  <- 호스트5에서 수행 
```

- 재접속하여  HOSTNAME 변경확인

![hostname](/images/hostname.png)



#### 8) jdk 1.8 설치 & JAVA 환경변수 추가

- jdk 설치

```
$ sudo yum install -y vim wget unzip
$ cd /tmp
$ sudo yum list java*jdk-devel
$ sudo yum install java-1.8.0-openjdk-devel.x86_64
$ java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-b04)
OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)
```

- JAVA 설치경로 확인

```
$ which javac
/usr/bin/javac
```

- javac link 확인

```
$ readlink -f /usr/bin/javac
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/bin/javac
```

- bash_profile에 java_home경로 export

```
$ sudo vi ~/.bash_profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64
```

- bash_profile 값 적용

```
$ source ~/.bash_profile
```

- JAVA_HOME 설정확인

```
$ echo $JAVA_HOME
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64
```



#### 9) 서버끼리 상호접속 하도록 설정

- SSH 키 생성

```
[centos@d3 ~]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/centos/.ssh/id_rsa):
Enter passphrase (empty for no passphrase): (enter)
Enter same passphrase again: (enter)
Your identification has been saved in /home/centos/.ssh/id_rsa.
Your public key has been saved in /home/centos/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:wouk+1aqVndIYB37eh1y9SfqfyyKELq9cbUkFh7Fzpw centos@d3.com
The key's randomart image is:
+---[RSA 2048]----+
|     ...   ..    |
|    o ..   ..    |
|   . ..   o+..   |
|     ... . +E.   |
|    ..o.S * o o .|
|   o..+=.* = o o |
|  ...++.+ o o  . |
|  ..o  + + o  . o|
| .o+. . o.. oo.o |
+----[SHA256]-----+

```

- SSH 키 복사

```
[centos@d3 .ssh]$ ssh-copy-id centos@m1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/centos/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
centos@m1's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'centos@m1'"
and check to make sure that only the key(s) you wanted were added.
```

- 각 서버별로 복사 및 접속 확인

```
$ ssh-copy-id centos@m1
$ ssh-copy-id centos@cm
$ ssh-copy-id centos@d1
$ ssh-copy-id centos@d2
$ ssh-copy-id centos@d3
$ ssh m1
$ ssh cm
$ ssh d1
$ ssh d2
$ ssh d3
```



#### 10) vm.swappiness 값 수정

- vm.swappiness[0~100] 값이 클수록 inactive process들에 대한 메모리 스왑이 빈번히 발생 Hadoop Cluster 사용에는 스왑이 적게 발생하도록 세팅하는 것이 적합 [vm.swappiness 1]
- 현재 세팅값 확인

```
$ cat /proc/sys/vm/swappiness      // 현재 세팅값 확인
30
```

- 스왑 값 변경

```
$ sudo sysctl -w vm.swappiness=1   // 스왑값
vm.swappiness = 1
```



#### 11) Disabling Transparent Hugepages

```
$ sudo vi /etc/rc.d/rc.local
// 아래내용 추가
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
echo "never" > /sys/kernel/mm/transparent_hugepage/defrag
```

```
$ sudo chmod +x /etc/rc.d/rc.local
$ sudo vi /etc/default/grub
// 아래내용 추가
add -> transparent_hugepage=never (on line GRUB_CMDLINE_LINUX )
```

```
$ grub2-mkconfig -o /boot/grub2/grub.cfg
```



## 2. MariaDB Installation (CM서버만 설치)

#### 1) MariaDB Server 설치

```
$ sudo yum install mariadb-server
```

#### 2) MariaDB Server 실행 및 설정

- secure_installation 과정을 진행하며 root password 및 보안관련 설정 Default root 비밀번호는 공백

- 비번 : skcc2019

```
$ sudo systemctl enable mariadb
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
```

```
$ sudo systemctl start mariadb
```

- 설정

```
$ sudo /usr/bin/mysql_secure_installation
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!
      
In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):   (enter)
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] N
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

#### 3) MySQL JDBC Driver for MariaDB 설치

```
$ wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz

$ tar zxvf mysql-connector-java-5.1.47.tar.gz


$ sudo mkdir -p /usr/share/java/
$ cd mysql-connector-java-5.1.47
$ sudo cp mysql-connector-java-5.1.47-bin.jar /usr/share/java/mysql-connector-java.jar

$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

#### 4) TABLE 및 계정 생성

```
<예시>
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

#### 5) TABLE 생성확인

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| amon               |
| hue                |
| metastore          |
| mysql              |
| nav                |
| navms              |
| oozie              |
| performance_schema |
| rman               |
| scm                |
| sentry             |
+--------------------+
12 rows in set (0.00 sec)
```



## 3. Cloudera Manager Install (cluster  전체수행)

[참고] https://www.cloudera.com/documentation/enterprise/5-15-x/topics/install_cm_server.html

#### 1) Configure a repository for cloudera manager

- Download the cloudera-manager.repo file for your OS version to the /etc/yum.repos.d/ directory on the Cloudera Manager Server host [CDH가 아니라 cloudera-manager를 다운받아야 함]

```
$ sudo wget https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo -P /etc/yum.repos.d/
```

- change the baseurl within cloudera-manager.repo to fit the version you want to install

```
$ sudo vi /etc/yum.repos.d/cloudera-manager.repo

<AS-IS>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1

<TO-BE>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.15.2/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1
```

- Import the repository signing GPG key [RHEL 7]

```
$sudo rpm --import https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera
```



#### 2) Cloudera Manager Packages 설치

- CM만 수행

```
$sudo yum install cloudera-manager-daemons cloudera-manager-server
```

- Cluster 전체 수행

```
$sudo yum install cloudera-manager-daemons cloudera-manager-agent
```



#### 3) Cloudera Manager Database 설정

-  scm_prepare_database.sh 사용하여 configuration 진행

```
$ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm
Enter SCM password: (scm)
JAVA_HOME=/usr/lib/jvm/java-openjdk
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /usr/lib/jvm/java-openjdk/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/usr/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
```



#### 4) Cloudera Manager 서버 실행

```
$ sudo systemctl start cloudera-scm-server  //cloudera-scm-server 서비스 시작
$ sudo systemctl stop cloudera-scm-server   //cloudera-scm-server 서비스 종료
$ sudo systemctl status cloudera-scm-server //cloudera-scm-server 서비스 상태 
$ sudo systemctl enable cloudera-scm-server //서버 부팅 시 자동실행 설정
$ sudo systemctl disable cloudera-scm-server//서버 부팅 시 자동실행 해제
$ sudo systemctl is-enable 서비스명          //서비스가 자동실행 설정상태인지 확인
$ sudo systemctl list-unit-files --type=service //자동실행 설정된 서비스 리스트 
```

- 서버실행

```
$ sudo systemctl start cloudera-scm-server
```

- 로그 확인

```
$ sudo tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log  
2019-07-17 05:06:37,539 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Generating entities:2019-07-17T05:06:37.539Z
2019-07-17 05:06:37,575 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Num entities:208
2019-07-17 05:06:37,575 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Generating documents:2019-07-17T05:06:37.575Z
2019-07-17 05:06:37,594 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Num docs:221
2019-07-17 05:06:37,595 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Constructing repo:2019-07-17T05:06:37.595Z
2019-07-17 05:06:38,619 INFO SearchRepositoryManager-0:com.cloudera.server.web.cmf.search.components.SearchRepositoryManager: Finished constructing repo:2019-07-17T05:06:38.619Z
2019-07-17 05:06:38,713 INFO WebServerImpl:org.mortbay.log: jetty-6.1.26.cloudera.4
2019-07-17 05:06:38,713 INFO WebServerImpl:org.mortbay.log: Started SelectChannelConnector@0.0.0.0:7180
2019-07-17 05:06:38,713 INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
2019-07-17 05:06:43,611 INFO ScmActive-0:com.cloudera.server.cmf.components.ScmActive: ScmActive completed successfully.    
```

- 포트확인

```
$ netstat -antp | grep 7180                                     //서버의 default port도 떴다.
tcp        0      0 0.0.0.0:7180            0.0.0.0:*               LISTEN      12335/java 
```



## 4. Install a cluster and deploy CDH

[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/install_cm_server.html

#### 1) UI 로그인 :  [http://15.164.22.37:7180](http://15.164.22.37:7180/) (admin/admin)

![ui0](/images/ui0.png)

#### 2) CDH 클러스터 설치할 호스트 지정

![ui1](/images/ui1.png)

- 호스트명, IP주소 확인

![ui2](/images/ui2.png)



#### 3) 클러스터 설치옵션 선택

- 설치한 CM  버전 확인 

![ui3](/images/ui3.png)

- JDK 설치 옵션 - 선택 안함 (수동설치)

![ui4](/images/ui4.png)

- 단일사용자 모드 활성화 - 선택안함

![ui5](/images/ui5.png)

- SSH 로그인정보 입력 (centos/skcc2019)

![ui6](/images/ui6.png)

- Agent 설치

![ui7](/images/ui7.png)

- 클러스터 설치

![ui8](/images/ui8.png)



#### 4) 클러스터 설정

- Impala가 있는 코어 선택

![ui9](/images/ui9.png)

- 역할 할당

![ui10](/images/ui10.png)

- 데이터베이스 설정

![ui11](/images/ui11.png)

- 변경내용 검토 (변경없음)

![ui12](/images/ui12.png)

- 실행

![ui13](/images/ui13.png)

- 설정완료

![ui14](/images/ui14.png)

- 최종 실행화면

![ui15](/images/ui15.png)




#### 5) Kafka 설치

-  패키지 다운로드 후 배포, 활성화

![kafaka1](/images/kafka1.png)

![kafaka2](/images/kafka2.png)

- Kafka 설치

![kafaka3](/images/kafka3.png)

![kafaka4](/images/kafka4.png)

- 변경내용 없이 계속

![kafaka5](/images/kafka5.png)

![kafaka6](/images/kafka6.png)

- 시작 전 구성에서 Java Heap Size of Broker 값 50Mb -> 1Gin 로 변경

![kafaka7](/images/kafka7.png)

- Kafka 시작

![kafaka8](/images/kafka8.png)




#### 6)  FLUME설치

![plume1](/images/plume1.png)

![plume2](/images/plume2.png)

![plume3](/images/plume3.png)

![plume4](/images/plume4.png)

![plume5](/images/plume5.png)





#### 7) SCOOP 설치

![scoop1](/images/scoop1.png)

![scoop2](/images/scoop2.png)

![scoop3](/images/scoop3.png)

![scoop4](/images/scoop4.png)

![scoop5](/images/scoop5.png)



#### 8) SPARK 설치

![spark2c](/images/spark2c.png)

![spark2d](/images/spark2d.png)

![spark2e](/images/spark2e.png)

![spark2f](/images/spark2f.png)

![spark2g](/images/spark2g.png)

- SPAKR2 설치시 Parcel repo 추가 후 패키지 다운로드 및 배포, 활성화

![spark2](/images/spark22.png)

![spark2a](/images/park2a.png)

![spark2b](/images/spark2b.png)





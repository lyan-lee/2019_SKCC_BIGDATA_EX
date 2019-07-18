# PART2 - 빅데이터 분석 (07/18)

### 팀 구성원 (팀6)

| ![07745](/images/07745.jpg) | ![06703](/images/06703.jpg) | ![08363](/images/08363.jpg) |
| ------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------- |
| 이용희 (07745)<br />과금혁신Unit                  | 김의현 (06703)<br />과금혁신Unit                  | 김지현(08363)<br /> 과금혁신Unit                  |



## 1. 실습환경 준비

#### 1) Linux 계정생성  ( training/training )

```
$ sudo adduser training
$ sudo passwd training
$ sudo usermod -aG wheel training
```

![user_training](/images/user_training.png)



#### 2). HUE 접속하여 HDFS 계정생성 ( training/training )

![hue1](/images/hue1.png)



#### 3) MariaDB에 training 게정을 생성

```
$ mysql -u root -p

## 모든 호스트에서 접속하는(%) training계정에, 모든 Database에 대한, 모든 권한 부여
$ GRANT ALL ON *.* TO 'training'@'%' IDENTIFIED BY 'training';
$ FLUSH PRIVILEGES;

## training 계정에 대한 권한을 중복으로 부여하면, 아래의 결과가 2줄이 나오고, 1줄을 지워주면 정상작동
$ select * from mysql.user where user ='training';
+-----------+----------+
| Host      | User     |
+-----------+----------+
| %         | training |
| localhost | training |
+-----------+----------+
```



#### 4) 실습데이터 세팅

- Google Drive에서 link URL 확인 후 서버에 다운

```
$ wget https://doc-0k-6o-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/qns5g0ghihb9jb0i60f5fnpl2eadnmns/1563415200000/14888820716761294211/*/1OOfGoDpDTnqt1TckO4wSqOoLBi6TDhUm?e=download
$ mv 1OOfGoDpDTnqt1TckO4wSqOoLBi6TDhUm\?e\=download all.zip
$ unzip all.zip
```

- 실습데이터 setup

```
$ ./training_materials/devsh/scripts/setup.sh
```



## 2. 빅데이터 테스트

#### 1) 문제 1

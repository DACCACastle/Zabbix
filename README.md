# Zabbix

https://www.sios-apac.com/ko/2021/01/%EC%97%AC%EC%A0%84%ED%9E%88-zabbix-in-aws%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%B4%EC%95%BC%ED%95%A9%EB%8B%88%EB%8B%A4/
: 사용 이유 등 설명

https://honglab.tistory.com/66?category=953819
: 자빅스 centos 설치 ( aws )

https://velog.io/@seunghyeon/Zabbix-Zabbix-Server-%EC%84%A4%EC%B9%98
: 2

https://steven-life-1991.tistory.com/35
: 3

https://usheep91.tistory.com/25
: 4

https://cloudest.oopy.io/posting/003
: 5

https://getmovie.tistory.com/entry/%EB%A6%AC%EB%88%85%EC%8A%A4-CentOS7%EC%97%90-%EC%9E%90%EB%B9%85%EC%8A%A4-%EC%84%A4%EC%B9%98-2
: 리눅스에 설치


< Zabbix Server & Agent 공통 설정 > ( Passive 방식 )

[centos@ip-3-35-17-126 ~]$ sudo su -
 > 모든 작업은 root 계정으로 진행

1. Host Name 변경
[root@ip-3-35-17-126 ~]# hostnamectl set-hostname server
[root@ip-3-35-17-126 ~]# hostname
server

2. TimeZone 설정
[root@ip-3-35-17-126 ~]# mv /etc/localtime /etc/localtime.ori
[root@ip-3-35-17-126 ~]# ln -s /usr/share/zoneinfo/Asia/Seoul /etc/localtime

3. 재부팅 후 호스트네임 적용
[root@ip-3-35-17-126 ~]# init 6 
[centos@server ~]$ sudo su -
[root@server ~]#

4. SELINUX 비활성화 ( 다양한 오류발생 방지를 위해 비활성화 )
[root@server ~]# vi /etc/selinux/config
7 SELINUX=disabled

5. net-tools 설치
[root@server ~]# yum -y install net-tools

6. 방화벽 설치
[root@server ~]# yum -y install firewalld
[root@server ~]# systemctl start firewalld
[root@server ~]# systemctl enable firewalld

* Agent 서버도 동일하게 구성


< Zabbix Server 설정 >

- 패키지 및 구성요소 준비 -
1. HTTP, DB 패키지 설치
[root@server ~]# yum -y install httpd mariadb mariadb-devel mariadb-server

2. Zabbix 설치를 위한 Yum Repo 추
[root@server ~]# rpm -ivh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-5.0-1.el8.noarch.rpm

3. Zabbix 설치
[root@server ~]# yum clean all
[root@server ~]# yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-agent

4. HTTP/DB 구동 및 프로세스 확인
[root@server ~]# systemctl start mariadb 
[root@server ~]# systemctl start httpd    

[root@server ~]# systemctl enable mariadb 
[root@server ~]# systemctl enable httpd  

[root@server ~]# ps -ef | grep mysql  
[root@server ~]# ps -ef | grep httpd

5. 방화벽 포트 허용
[root@server ~]# firewall-cmd --zone=public --add-port=80/tcp --permanent
[root@server ~]# firewall-cmd --zone=public --add-port=10051/tcp --permanent
[root@server ~]# firewall-cmd --zone=public --add-port=10050/tcp --permanent
[root@server ~]# firewall-cmd --zone=public --add-port=3306/tcp --permanent
[root@server ~]# firewall-cmd --reload

- DB 설정 -
1. MariaDB 기본 설정
[root@server ~]# mysql_secure_installation

Enter current password for root (enter for none): [패스워드가 없기 때문에 엔터]

Set root password? [Y/n] Y    [DB ROOT 패스워드 설정]
New password: 패스워드 입력
Re-enter new password: 패스워드 재입력

Remove anonymous users? [Y/n] Y    [익명의 접근을 막을 것인지? 보안을 위해 Y 엔터]

Disallow root login remotely? [Y/n] Y    [DB ROOT 원격을 막을 것인지? 보안을 위해 Y 엔터]

Remove test database and access to it? [Y/n] Y  [Test 용으로 생성된 데이터베이스를 삭제할 것인가? Y 엔터]

Reload privilege tables now? [Y/n] Y  [현재 설정한 값을 적용할 것인지? Y 엔터]

Thanks for using MariaDB! [완료]

2. MariaDB 접속 및 Zabbix DB 생성
[root@server ~]# mysql -u root -p
Enter password: (dacastle)

MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;

MariaDB [(none)]> grant all privileges on zabbix.* to centos@localhost identified by 'zabbix'; (zabbix 라는 데이터베이스 생성)

MariaDB [(none)]> flush privileges; (각각 새로운 ID, PW)

MariaDB [(none)]> exit

[root@server ~]# cd /usr/share/doc/zabbix-server-mysql
[root@server zabbix-server-mysql]# gunzip create.sql.gz
[root@server zabbix-server-mysql]# mysql -u root -p zabbix < create.sql
Enter password: (dacastle)

[root@server ~]# vi /etc/zabbix/zabbix_server.conf
38 LogFile=/var/log/zabbix/zabbix_server.log      [기본 설정]
72 PidFile=/var/run/zabbix/zabbix_server.pid      [기본 설정]
82 SocketDir=/var/run/zabbix                      [기본 설정]
91 DBHost=localhost                               [주석 제거]
100 DBName=zabbix                                 [DB 생성 이름과 동일하게 설정]
116 DBUser=centos                                 [DB 유저 생성 이름과 동일하게 설정]
124 DBPassword=zabbix                             [주석 제거 후 DB 유저 패스워드와 동일하게 설정]

[root@server ~]# vi /etc/php-fpm.d/zabbix.conf
24 php_value[date.timezone] = Asia/Seoul          [주석(;) 제거 후 한국 시간으로 수정]

[root@server ~]# systemctl start zabbix-server zabbix-agent php-fpm
[root@server ~]# systemctl enable zabbix-server zabbix-agent php-fpm
[root@server ~]# init 6
(zabbix-server 데몬실행이 실패하면 enable 까지 명령어를 넣어준 후 재부팅 후 자동 실행 확인)

[root@server ~]# systemctl status zabbix-server [ 정상적으로 active로 동작함 ]

- Zabbix 구동 확인 -
[서버IP or 호스트네임]/zabbix 경로로 접속
Next -> Next -> MySQL, localhost, 0, zabbix, centos, zabbix -> 최종확인

초기값 : Admin / zabbix 로 로그인 -> 메인 대시보드 접속

기본설정 : Administration -> Users -> Admin 계정 클릭
 > Language - 언어설정 ( Korean (ko_KR) )
 > Theme - 스킨 테마 설정

모니터링 -> 호스트
Zabbix server 상태에 초록불이 들어오면 서버 환경 구성 완료


< Zabbix Agent 설정 >

1. Agent 서버 생성에 필요한 파일 다운로드
[root@agent ~]# wget (S3에 있는 파일 주소)
( S3에 파일을 업로드하고 wget으로 agent 서버로 파일 이동 )

2. Zabbix Agent 패키지 설치
[root@agent ~]# rpm -ivh /home/centos/zabbix-agent-5.0.1-1.el8.x86_64.rpm

3. Agent 설정파일 수정
[root@agent ~]# vi /etc/zabbix/zabbix_agentd.conf
76 #DenyKey=system.run[*]        [ 주석 추가 ]
94 EnableRemoteCommands=1        [ 주석 제거 및 변경 ]
119 Server=52.78.235.226         [ Server의 Public IP 입력 ]
144 StartAgents=0                [ 주석 제거 및 변경 ]
160 ServerActive=52.78.235.226   [ Server의 Public IP 입력 ]
171 Hostname=agent               [ Agent의 host name 입력 ]

4. 변경한 설정 적용
[root@agent ~]# systemctl enable zabbix-agent
[root@agent ~]# systemctl restart zabbix-agent

5. S3에서 가져온 파일들을 알맞게 이동 후 권한조정
[root@agent ~]# mv /home/centos/userparameter_diskstats.conf /etc/zabbix/zabbix_agentd.d/userparameter_diskstats.conf
[root@agent ~]# mv /home/centos/lld-disks.py /usr/local/bin/lld-disks.py
[root@agent ~]# chmod +x /usr/local/bin/lld-disks.py

6. 설정 적용
[root@agent ~]# systemctl restart zabbix-agent
[root@agent ~]# init 6

[root@agent ~]# systemctl status zabbix-agent [ 정상적으로 active로 동작함 ]

< Zabbix 호스트 등록 및 그룹생성 >

- 그룹 생성 -
Zabbix 관리페이지로 이동 ->

1. 설정 -> 호스트 그룹 -> 호스트 그룹 작성
2. test 그룹 생성

- 호스트 등록 -
1. 설정 -> 호스트 -> 호스트 작성
2. 호스트 탭 ->
agent - agent - test - 0.0.0.0
3. 템플릿 탭 ->
Link new templeates - 선택 - Templates/Operating systems - Template OS Linux by Zabbix agent active 체크 - 선택 - 추가

- 모니터링 확인 -

Agent 연결 확인 ->

1. 모니터링 -> 최근데이터로 이동
2. 데이터 모니터링 시간대랑 % 잘 뜨는지 확인

* 대쉬보드에 감시중 1 , 알 수 없음 1 로 표시되지만 Passive 방식의 모니터링은 직접적으로 확인되지 않고 알 수 없음 으로 확인되지만 정상이다.

<결과>
1. Zabbix를 통해 Server는 Agent에 대한 하드웨어 모니터링이 가능하다.
2. Agent 패키지 설치 및 호스트 등록을 통해 Server는 수 많은 Agent를 손쉽게 모니터링 할 수 있다.


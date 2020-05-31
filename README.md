# LDAP

LDAP server, client 설치, NFS 설치 후 연동하여 LDAP 계정 로그인 시 자동으로 NFS로 mount하고, LDAP 서버 강제 종료 시 Log를 확인하는 과제를 진행하였고 추가로 DNS 서버를 구성하여 연동을 진행하였습니다.
우선, 결과적으로 크게 진행하지 못하였습니다. 첫 시도 때 vmware 설치를 잘못하여, vmware간의 통신이 전혀 되지 않는 환경을 조성한 채로 진행하여 각각의(LDAP, NFS, DNS) 환경 조성까지는 진행하였으나, 서로간의 연동이 안된 상태였습니다. 방화벽과 네트워크 환경, windows 내에서의 네트워크 환경 등 여러 방면으로 살펴보았으나, PING조차도 주고받지 못하는 상황에서(외부 인터넷으로의 접속과 apache의 local 실행은 되었습니다.) 꽤 많은 시간을 할애하였습니다. 후에 최영근 사원의 조언으로 vmware를 다른 버전으로 설치하였고, NAT를 설정하고, 하나의 vmware에서 여러 virtual machine을 실행할 수 있는 vmware 15.5 버전을 설치하여 머신간의 네트워크 통신 환경을 조성하는데 성공하였습니다.

## 1.	LDAP_Server, LDAP_Client, NFS_Server, DNS_Server 총 4대의 가상 머신 생성
우선, 각각의 역할을 하는 머신을 생성하였습니다. 각각 최소설치 버전으로 진행하였고, Xwindow GUI 환경을 설치하여 진행하였습니다.

## 2.	LDAP Server 설치
대부분의 명령이 sudo를 통한 명령어인 점을 생각하여 su로 root 계정인 상태로 진행하였습니다.

### 1)	openldap 설치 
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel

### 2)	openldap(sladp) 실행
$ systemctl start slapd
$ systemctl enable slapd

### 3)	ldap 포트 확인 (port NO : 389)
$ netstat -antup | grep -i 389

### 4)	ldap password 생성
$ slappasswd
PW : {SSHA}ioFBiRppFcjgzMYbqukXl9fg563CIC7U

### 5)	설정파일 조정
설정 파일은 /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif 이지만, 직접 수정을 하지 않고 db.ldif 파일을 만들고 적용하는 방식으로 하였습니다.

------------------------------------------------------------------------------------------
$ vim db.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ldaptest,dc=co,dc=kr

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=ldaptest,dc=co,dc=kr

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}ioFBiRppFcjgzMYbqukXl9fg563CIC7U

------------------------------------------------------------------------------------------

#### db.ldif 설정 후 적용하여 줍니다.
$ ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif 	
	ldapmodify: wrong attributeType at line 13, entry "olcDatabase={2}hdb,cn=config" 오류가 발생하였습니다.
	db.ldif 내의 설정값들이 분명 제대로 작성되어있는데도, 오류가 발생하여 진행이 오랫동안 이루어지지 않았습니다. 검색해본 결과 해당 라인의 띄어쓰기 하나가 잘못되어도 오류가 발생할 수 있다고 알려져있었습니다.

#### monitor 접속 계정에 대한 설정을 해줍니다.

------------------------------------------------------------------------------------------
$ vim monitor.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=ldapadm,dc=ldap,dc=co,dc=kr" read by * none

------------------------------------------------------------------------------------------

#### 설정을 적용합니다.
$ ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif

#### ldap datebase를 설정합니다.
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/

#### ldap schema를 적용하였습니다.
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

### 도메인 정보를 변경하였습니다.

------------------------------------------------------------------------------------------
$ vim base.ldif
dn: dc=ldaptest,dc=co,dc=kr
dc: ldaptest
objectClass: top
objectClass: domain

dn: cn=ldapadm, dc=ldaptest,dc=co,dc=kr
objectClass: organizationalRole
cn: ldapadm
description: LDAP Manager

dn: ou=People, dc=ldap,dc=co,dc=kr
objectClass: organizationalUnit
ou: People

dn: ou=Group, dc=ldaptest,dc=co,dc=kr
objectClass: organizationalUnit
ou: Group

------------------------------------------------------------------------------------------

$ ldapadd -x -W -D "cn=ldapadm,dc=ldaptest,dc=co,dc=kr" -f base.ldif

#### ldap 설정을 완료하였고, 계정을 만들어 줍니다.

------------------------------------------------------------------------------------------
vim Lee.ldif
dn: uid=Lee,ou=People,dc=ldaptest,dc=co,dc=kr
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: Lee
uid: Lee
uidNumber: 9999
gidNumber: 100
homeDirectory: /home/Lee
loginShell: /bin/bash
gecos: Lee [Admin (at) nGle]
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7

------------------------------------------------------------------------------------------

#### 유저 정보를 업데이트 해줍니다
$ ldapadd -x -W -D "cn=ldapadm,dc=ldaptest,dc=co,dc=kr" -f Lee.ldif
Lee 계정의 비밀번호를 설정해줍니다
$ ldappasswd -s 1234 -W -D "cn=ldapadm,dc=ldaptest,dc=co,dc=kr" -x "uid=Lee,ou=People,dc=ldaptest,dc=co,dc=kr"

#### 유저정보 검색
$ ldapsearch -x cn=Lee -b dc=ldaptest,dc=co,dc=kr

#### ldap 서버에 방화벽이 실행되고 있다면 ldap 서비스를 허용할 수 있습니다.
$ firewall-cmd --permanent --add-service=ldap
$ firewall-cmd –reload

#### ldap server의 로그는 rsyslog 설정 파일(/etc/rsyslog.conf)에 설정을 추가합니다.
$ vim /etc/rsyslog.conf
local4.* /var/log/ldap.log
$ systemctl restart rsyslog

## 이것으로 LDAP Server 설치를 완료하였습니다

## 3.	LDAP Client 설치
LDAP Server IP : 192.168.171.130
LDAP Client IP : 192.168.171.131

#### LDAP_Client 머신으로 접속하여 ldap client를 설치합니다.
$ yum -y update
$ yum install -y openldap-clients nss-pam-ldapd

#### ldap server와 연결합니다
$ authconfig --enableldap --enableldapauth --ldapserver=192.168.171.130 --ldapbasedn="dc=ldaptest,dc=co,dc=kr" --enablemkhomedir –update

#### ldap client를 재시작하였습니다.
$ systemctl restart  nslcd

#### getent 명령으로 ldap server에서 Lee의 정보를 확인합니다.
$ getent passwd Lee

이 과정에서 명령어를 실행하여도 아무런 반응이 없었습니다.

> 참고한 블로그에 의하면,
Lee: x:9999: 100:Lee [Admin (at) nGle]:/home/Lee:/bin/bash
의 결과가 나와야하는데, 결과가 나오지 않는 상태입니다.
ldap getent 검색을 통하여 오류를 찾아보고 binddn, bindpw 등 몇가지 명령어를 실행하며 계정 연동을 시도하였으나 실패하였습니다.
	ping을 통하여 머신간의 통신이 이루어지는 것도 확인하였지만 계정 연동 실패의 원인을 확인하지 못하였습니다.

## 4.	NFS 설치

#### LDAP Client 계정 연동 실패 후 우선 NFS를 설치하였습니다.

$ yum install nfs-utils nfs-utils-lib
$ systemctl   start nfs-server

#### nfs 디렉토리를 생성합니다.
$ mkdir /nfs
#### 마운트할 디렉토리 설정을 위해서 /etc/exports에 내용을 추가합니다.
/nts/ 192.168.171.*(rw,all_squash,sync)
	192.168.171.x 대역에 해당하는 모든 ip에 rw 권한 부여
$ systemctl restart nfs
$ chmod o+w /nfs

#### nfs를 재실행합니다.
$ systemctl stop nfs-server
$ systemctl start nfs-server

#### 서비스 시작 및 등록하기
$ systemctl   restart   rpcbind
$ systemctl   start   nfs-server
$ systemctl   start   nfs-lock
$ systemctl   start   nfs-idmap

 

$ systemctl   enable   rpcbind
$ systemctl   enable   nfs-server
$ systemctl   enable   nfs-lock
$ systemctl   enable   nfs-idmap

## 5.	DNS_Server 설치
DNS를 설치합니다
$ yum install –y bind*

#### DNS 설정
### 1) /etc/named.conf 파일 설정

------------------------------------------------------------------------------------------
$ vi /etc/named.conf
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };
        
        ------------------------------------------------------------------------------------------

### 2) /etc/named.rcf1912.zone 파일 설정

------------------------------------------------------------------------------------------
zone "Lee.com.zone" IN {
        type master;
        file "Lee.com.zone";
        allow-update { none; };
        
        ------------------------------------------------------------------------------------------
### 3) /var/named/ 디렉토리 하위에  DNS 존파일 생성

------------------------------------------------------------------------------------------
cp named.localhost Lee.com.zone
TTL 1D
@       IN SOA  @ ns.Lee.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        IN      NS      NS ns.Lee.com.
        IN      A       1.0.0.11
ns      IN      A       1.0.0.11
www     IN      A       1.0.0.11

------------------------------------------------------------------------------------------

### 4) /var/named 권한 설정
$ chmod –R 754 /var/named

### 5) DNS 서비스 방화벽 설정
$ firewall-cmd --permanent --add-port=53/udp
$  firewall-cmd --permanent --add-port=53/tcp
$  firewall-cmd --reload
$  systemctl start named
$systemctl status named
	이 단계에서 오류가 발생하였습니다
	5월 08 23:00:05 localhost.localdomain systemd[1]: Starting Berkeley Internet Name Domain (DNS)...
	 5월 08 23:00:05 localhost.localdomain bash[4797]: /etc/named.root.key:1: unknown option 'managed-keys'
	 5월 08 23:00:05 localhost.localdomain bash[4797]: /etc/named.conf:62: '}' expected near end of file
	 5월 08 23:00:05 localhost.localdomain systemd[1]: named.service: control process exited, code=exited status=1
	 5월 08 23:00:05 localhost.localdomain systemd[1]: Failed to start Berkeley Internet Name Domain (DNS).
	 5월 08 23:00:05 localhost.localdomain systemd[1]: Unit named.service entered failed state.
	 5월 08 23:00:05 localhost.localdomain systemd[1]: named.service failed.
	Failed to start Berkeley Internet Name Domain (DNS). 오류 로그를 검색하여 해결하려 하였습니다.
	 bind설정 파일들이 /var/named/chroot/이하로 올바르게 마운트 되지 않기 때문에 발생이라는 검색결과를 토대로 오류 수정을 시도하였으나, 실패하였고, 더 진행하지 못하였습니다.



#### 결과적으로, 
LDAP Server 설치 : 성공
LDAP Client 설치 : 성공
LDAP Server <-> LDAP Client 연동 : 실패
NFS 설치 : 성공 But LDAP 계정 연동 실패로 autofs를 진행하지는 못하였습니다
autofs : LDAP Client에서 LDAP Server로 로그인 시, 자동으로 nfs mount되는 시스템
DNS : 방화벽 오류로 인해 설치 실패
입니다.

------------------------------------------------------------------------------------------
우선, 과제 기간이 길었음에도 성공적인 결과가 나오지 못하였습니다. LAMP 환경 설치 이후로 처음 해보는 Linux 환경 설정에, 최소설치 버전에서 진행하여 기반 프로그램부터 설치를 진행하는 과정에서 Linux의 기초 기반에 대하여 공부하게 되었고, 또한 많은 어려움과 부족함을 느꼈습니다.
최근 발생한 LDAP 서버 오류로 인한 job working issue 관련하여, 조금 더 생각해볼 수 있게 되었고, LDAP Server와 Client 계정 연동을 계속 시도하며 성공할 수 있도록 계속 공부하도록 하겠습니다.
감사합니다.

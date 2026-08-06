# MySQL 8 설치

## Docker run

* 기본 예시 명령어 형식

``` bash
docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
```

::: details 실제 디렉터리에 볼륨 연결
```bash
sudo mkdir -p /data/mysql
```

```bash
docker run \
  --name mysql \
  -e 'MYSQL_ROOT_PASSWORD=!test1234' \
  -d \
  -p 3306:3306 \
  -v /data/mysql:/var/lib/mysql \
  --restart unless-stopped \
  mysql:8.0.38
```
:::

::: tip
* [공식 저장소 바로가기](https://hub.docker.com/_/mysql)
* [tag 목록 바로가기](https://hub.docker.com/_/mysql/tags)
:::

* 컨테이너 이름 `mysql` 초기 root 비밀번호 `!test1234` mysql `8.0.38` 버전 으로 설치시 예시

``` bash
docker run --name mysql -e MYSQL_ROOT_PASSWORD=!test1234 -d -p 3306:3306 mysql:8.0.38
```

## 초기 세팅

### character-set 및 time zone 변경

* bash 접속

``` bash
docker exec -it mysql /bin/bash
```

* cnf 파일 설정
``` bash
cd /etc/mysql/conf.d
```

``` bash
cat << 'EOF' > /etc/mysql/conf.d/my.cnf
```

::: code-group
```bash [my.cnf]
[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-time-zone = "+09:00"

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
EOF
```
:::

* 컨테이너 재시작
``` bash
docker restart mysql
```

### 데이터 베이스를 생성후 동명의 사용자를 생성하여 모든 권한을 부여

::: tip
아래 예시는 `testuser` 사용자 및 DB 생성을 예시로 합니다.
:::

* bash 접속

``` bash
docker exec -it mysql /bin/bash
```

* mysql console 접속
``` bash
mysql -u root -p
```

* 사용자 생성
``` sql
CREATE USER 'testuser'@'%' IDENTIFIED BY '!testuser1234';
```

* DB 생성
``` sql
CREATE DATABASE testuser;
```

* 권한 부여
``` sql
GRANT ALL PRIVILEGES ON testuser.* TO 'testuser'@'%';
```

``` sql
FLUSH PRIVILEGES;
```

::: tip
dbeaver 로 연결하는 경우
* `allowPublicKeyRetrieval` true
* `useSSL` false
:::

## dump & restore

::: tip
*  procedure, function 같이 dump & restore 시 `--routines` 옵션 추가
*  trigger 같이 dump & restore 사용시 `--trigger` 옵션 추가
:::

* dump
``` bash
docker exec {container} /usr/bin/mysqldump -u {username} --password={password} {database} > {filename}.sql
```

::: tip
::: details mysqldump 옵션
```bash
docker exec {container} /usr/bin/mysqldump --no-data --routines -u {usernmae} --password={password} {database} | sed 's/AUTO_INCREMENT=[0-9]*//' > {filename}.sql
```

* `--no-data` 데이터 없이 스키마만
* `--routines` procedure, function 포함
* `sed 's/AUTO_INCREMENT=[0-9]*//'` AUTO_INCREMENT 0 부터 시작하도록 구문 제거
:::

* restore
``` bash
cat filename.sql | docker exec -i {container} /usr/bin/mysql -u {username} --password={password} {database_name} --verbose
```
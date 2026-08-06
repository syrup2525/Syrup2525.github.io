# Procedure Clone

::: tip
[GitHub](https://github.com/Syrup2525/mysql-proc-replica)
:::

::: warning
`MySQL 8` 기준으로 작성되었습니다.
:::

[[toc]]

## Motivation
`SOURCE` DB 에 존재하는 `FILTER_KEYWORD` 를 포함한 `Stored Procedure` 를 `TARGET` DB 로 복제(이관) 하는 `Container` 기반 도구 입니다.

예를들어 `FILTER_KEYWORD`=`v0_13_2` 로 지정했을 다음처럼 동작합니다.

| SOURCE DB | TARGET DB |
| --- | --- |
| Some_Procedure_A_v0_13_2 | `Some_Procedure_A_v0_13_2` |
| Some_Procedure_B_v0_13_2 | `Some_Procedure_B_v0_13_2` |
| Some_Procedure_C_v0_13_0 | |

## Quick Start
``` bash
docker run --rm \
  -e SOURCE_MYSQL_HOST=MYSQL_HOST \
  -e SOURCE_MYSQL_PORT=3306 \
  -e SOURCE_MYSQL_USER=MYSQL_USER \
  -e SOURCE_MYSQL_PASSWORD=MYSQL_PASSWORD \
  -e SOURCE_MYSQL_DATABASE=MYSQL_DATABASE \
  -e TARGET_MYSQL_HOST=MYSQL_HOST \
  -e TARGET_MYSQL_PORT=3306 \
  -e TARGET_MYSQL_USER=MYSQL_USER \
  -e TARGET_MYSQL_PASSWORD=MYSQL_PASSWORD \
  -e TARGET_MYSQL_DATABASE=MYSQL_DATABASE \
  -e FILTER_KEYWORD=v0_13_2 \
  syrup2525/proc-replica:latest
```
> - `SOURCE_MYSQL_HOST` 원본 프로시저가 존재하는 MySQL HOST 를 입력합니다 예를들어 `source.mysql.com`
> - `SOURCE_MYSQL_PORT` 원본 프로시저가 존재하는 MySQL PORT 를 입력합니다 예를들어 `3306`
> - `SOURCE_MYSQL_USER` 원본 프로시저가 존재하는 MySQL USERT 를 입력합니다 예를들어 `tesetuser`
> - `SOURCE_MYSQL_PASSWORD` 원본 프로시저가 존재하는 MySQL PASSWORD 를 입력합니다 예를들어 `tesetuserpassword`
> - `SOURCE_MYSQL_DATABASE` 원본 프로시저가 존재하는 MySQL DATABASE 를 입력합니다 예를들어 `testdatabase`
> - `TARGET_MYSQL_HOST` 프로시저를 복사할 대상 MySQL HOST 를 입력합니다 예를들어 `target.example.com`
> - `TARGET_MYSQL_PORT` 프로시저를 복사할 대상 MySQL PORT 를 입력합니다 예를들어 `3306`
> - `TARGET_MYSQL_USER` 프로시저를 복사할 대상 MySQL USERT 를 입력합니다 예를들어 `tesetuser`
> - `TARGET_MYSQL_PASSWORD` 프로시저를 복사할 대상 MySQL PASSWORD 를 입력합니다 예를들어 `tesetuserpassword`
> - `TARGET_MYSQL_DATABASE` 프로시저를 복사할 대상 MySQL DATABASE 를 입력합니다 예를들어 `testdatabase`
> - `FILTER_KEYWORD` 복제할 프로시저가 포함하고있는 문자열을 입력합니다 예를들어 `v0_13_2`

## Usage
### Docker Compose
::: code-group
``` yaml [docker-compose.yaml]
version: '3'

services:
  proc-replica:
    image: syrup2525/proc-replica:latest
    container_name: proc-replica
    environment:
      SOURCE_MYSQL_HOST: 'MYSQL_HOST'
      SOURCE_MYSQL_PORT: 3306
      SOURCE_MYSQL_USER: 'MYSQL_USER'
      SOURCE_MYSQL_PASSWORD: 'MYSQL_PASSWORD'
      SOURCE_MYSQL_DATABASE: 'MYSQL_DATABASE'
      TARGET_MYSQL_HOST: 'MYSQL_HOST'
      TARGET_MYSQL_PORT: 3306
      TARGET_MYSQL_USER: 'MYSQL_USER'
      TARGET_MYSQL_PASSWORD: 'MYSQL_PASSWORD'
      TARGET_MYSQL_DATABASE: 'MYSQL_DATABASE'
      FILTER_KEYWORD: 'v0_13_2'
```
:::

> - `SOURCE_MYSQL_HOST` 원본 프로시저가 존재하는 MySQL HOST 를 입력합니다 예를들어 `source.mysql.com`
> - `SOURCE_MYSQL_PORT` 원본 프로시저가 존재하는 MySQL PORT 를 입력합니다 예를들어 `3306`
> - `SOURCE_MYSQL_USER` 원본 프로시저가 존재하는 MySQL USERT 를 입력합니다 예를들어 `tesetuser`
> - `SOURCE_MYSQL_PASSWORD` 원본 프로시저가 존재하는 MySQL PASSWORD 를 입력합니다 예를들어 `tesetuserpassword`
> - `SOURCE_MYSQL_DATABASE` 원본 프로시저가 존재하는 MySQL DATABASE 를 입력합니다 예를들어 `testdatabase`
> - `TARGET_MYSQL_HOST` 프로시저를 복사할 대상 MySQL HOST 를 입력합니다 예를들어 `target.example.com`
> - `TARGET_MYSQL_PORT` 프로시저를 복사할 대상 MySQL PORT 를 입력합니다 예를들어 `3306`
> - `TARGET_MYSQL_USER` 프로시저를 복사할 대상 MySQL USERT 를 입력합니다 예를들어 `tesetuser`
> - `TARGET_MYSQL_PASSWORD` 프로시저를 복사할 대상 MySQL PASSWORD 를 입력합니다 예를들어 `tesetuserpassword`
> - `TARGET_MYSQL_DATABASE` 프로시저를 복사할 대상 MySQL DATABASE 를 입력합니다 예를들어 `testdatabase`
> - `FILTER_KEYWORD` 복제할 프로시저가 포함하고있는 문자열을 입력합니다 예를들어 `v0_13_2`

``` bash
docker compose up
```

### Run directly
::: tip
직접 `local` 환경에서 실행하는 방법을 설명합니다.
:::

#### 1. git clone
``` bash
git clone https://github.com/Syrup2525/mysql-proc-replica.git 
```

#### 2. Container build
``` bash
docker build -t proc-replica . 
```

#### 3. Create .env 
``` txt
# Source (복사할 프로시저가 존재하는 DB)
SOURCE_MYSQL_HOST=source.mysql.com
SOURCE_MYSQL_PORT=3306
SOURCE_MYSQL_USER=testuser
SOURCE_MYSQL_PASSWORD=userpassword
SOURCE_MYSQL_DATABASE=testdb

# Target (복사 대상 DB)
TARGET_MYSQL_HOST=target.mysql.com
TARGET_MYSQL_PORT=3306
TARGET_MYSQL_USER=testuser
TARGET_MYSQL_PASSWORD=userpassword
TARGET_MYSQL_DATABASE=testdb

# 필터 키워드 (프로시저명에 이 문자열이 포함된 경우에만 복사)
FILTER_KEYWORD=v0_13_2

```

#### 4. Run Container
``` bash
docker run --rm --env-file .env -v "$(pwd)/output:/tmp" proc-replica
```

# Speedtest Tracker

::: tip
현재 관리되지 않는 프로젝트인 [`henrywhitaker3/Speedtest-Tracker`](https://github.com/henrywhitaker3/Speedtest-Tracker) 프로젝트 대신 [`alexjustesen/speedtest-tracker`](https://github.com/alexjustesen/speedtest-tracker) 프로젝트를 시놀로지에서 사용하는 방법을 설명합니다.
:::

::: warning
[Using Synology](https://docs.speedtest-tracker.dev/getting-started/installation/using-synology) 가이드에 따라 `Container Manager(Docker Compose)` 환경을 전재로 작성한 문서이며, 이 문서는 [Using Docker Compose](https://docs.speedtest-tracker.dev/getting-started/installation/using-docker-compose) 기준이므로 `Docker` 환경에서는 유효하지 않을 수 있습니다.
:::

::: warning
`SSH` 환경에서 진행됩니다.
:::

## 사전 작업
### 도커 마운트 (저장소) 디렉토리 생성
``` bash
mkdir -p /volume1/docker/speed-tracker/custom-ssl-keys
```

``` bash
mkdir -p /volume1/docker/speed-tracker/data
```

### docker compose 파일 저장할 디렉토리 생성
``` bash
mkdir -p ~/docker/speed-tracker/
```

### 권한 부여
::: details uid, gid 확인
``` bash
id $user
uid=your-uid(your-mame) gid=your-gid(your-group) groups=your-gid(your-group),your-gid2(your-group2)
```
:::

``` bash
sudo chown -R your-uid:your-gid /volume1/docker/speed-trakcer
```

::: danger
`your-uid` `your-gid` 는 예시이며 실제 10진수로 된 값을 작성해야합니다.
:::

### Compose 작성
``` bash
~/docker/speed-tracker/docker-compose.yaml
```

::: code-group
``` yaml [docker-compose.yaml] {10-11,14-16}
services:
  speedtest-tracker:
    image: lscr.io/linuxserver/speedtest-tracker:latest
    restart: unless-stopped
    container_name: speedtest-tracker
    ports:
      - 18080:80
      - 18443:443
    environment:
      - PUID=
      - PGID=
      - TZ=Asia/Seoul
      - DISPLAY_TIMEZONE=Asia/Seoul
      - APP_KEY=
      - APP_URL=https://example.com
      - ASSET_URL=https://example.com
      - DB_CONNECTION=sqlite
      - DB_DATABASE=/config/database/database.sqlite
      - CACHE_STORE=file
      - CACHE_DRIVER=file
      - SESSION_DRIVER=file
    volumes:
      - /volume1/docker/speed-tracker/data:/config
      - /volume1/docker/speed-tracker/custom-ssl-keys:/config/keys
```
:::

> - PUID : `id $user` 명령어로 확인하여 기입합니다.
> - PGID : `id $user` 명령어로 확인하여 기입합니다.
> - APP_KEY : `echo "base64:$(openssl rand -base64 32 2>/dev/null)"` 명령어로 생성하여 기입합니다.
> - APP_URL : 리버스 프록시 환경인경우 접근할 도메인 주소를 입력합니다.
> - ASSET_URL : 리버스 프록시 환경인경우 접근할 도메인 주소를 입력합니다.

## 배포
### docker compose 배포
``` bash
cd ~/docker/speed-tracker
```

``` bash
docker compose up -d
```

### sqlite db 생성
``` bash
sudo docker exec -it speedtest-tracker sh -lc '
  mkdir -p /config/database
  touch /config/database/database.sqlite
  chown -R abc:abc /config/database
  chmod 775 /config/database
  chmod 664 /config/database/database.sqlite
  ls -lah /config/database/database.sqlite
'
```

### 권한 수정
``` bash
sudo chown -R your-uid:your-gid /volume1/docker/speed-tracker/data/database
```

::: danger
`your-uid` `your-gid` 는 예시이며 실제 10진수로 된 값을 작성해야합니다.
:::

``` bash
sudo chmod -R u+rwX,g+rwX /volume1/docker/speed-tracker/data/database
```

### php artisan 실행
``` bash
sudo docker exec -it speedtest-tracker sh -lc '
  cd /app/www
  php artisan optimize:clear
  php artisan config:clear
  php artisan migrate --force
'
```

### 컨테이너 재시작
``` bash
sudo docker restart speedtest-tracker
```

::: tip
초기 아이디는 `admin@example.com`, 초기 비밀번호는 `password` 입니다.
:::
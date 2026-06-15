# 실행
## 사전 작업
필요시 [`Rescuezilla`](https://rescuezilla.com/) 등의 도구로 백업 진행

## Bazzite 다운로드
Bazzite 다운로드

Device → OneXPlayer → KDE 선택 → Download bazzite-deck Legacy installer

파일 이름은 보통 다음과 비슷함

``` txt
bazzite-deck-stable-amd64.iso
```

## USB에 이미지 굽기
다음과 같은 프로그램을 사용하여 USB에 다운받은 ISO 파일을 굽기

- [Fedora Media Writer](https://github.com/FedoraQt/MediaWriter/releases) (권장)
- Rufus

USB 메모리는 최소 16GB 필요

## BIOS 사전작업
BIOS 에 들어가서 `보안 부팅(Secure Boot)`, `빠른 부팅(Fast Bott)` 비활성화

기기를 완전히 종료하고 기기 부팅과 동시에 키보드 `ESC` 반복 입력

``` txt
Security → Secure Boot → Secure Boot → Disabled
```
> 기본적으로 활성화 되어있음

``` txt
Boot → Fast Boot → Disabled
```
> 기본적으로 비활성화 되어있음

## Windows 사전 작업
### 빠른 부팅 비활성화
``` txt
powercfg.exe /hibernate off
```

적용 확인
``` bash
powercfg /a
```
> `최대 절전 모드를 사용하지 않도록 설정했습니다.` 문구 확인

### BitLocker 비활성화
드라이브 암호화 확인

``` bash
manage-bde -status
```
> 기본적으로 암호화 되어 있지 않음

드라이브가 암호화 되어 있다면
``` bash
manage-bde -off C:
```

## 파티셔닝
두 시스템의 디스크 파티셔닝 예시

``` txt
2TB SSD (예시)
├─ EFI Windows              기존 유지
├─ Windows MSR / Recovery   기존 유지
├─ Windows C:               200GB
│
├─ EFI Bazzite              300MB, /boot/efi
├─ Bazzite /boot            2GB, ext4
├─ Bazzite system           150GB, BTRFS
│   ├─ /
│   ├─ /var
│   └─ /var/home
│
└─ Games                    나머지, BTRFS (대략 1,650GB)
    ├─ SteamLibrary-Windows 스팀 라이브러리 (윈도우용)
    ├─ SteamLibrary-Bazzite 스팀 라이브러리 (바자이트용)
    ├─ Etc-Windows          스팀 미출시 윈도우 구동 게임 (니케, 엔드필드 등)
    └─ Shared               윈도우 ↔ 바자이트 공유
```

1. 디스크 관리 진입
  - `Win+R` → `diskmgmt.msc` Enter
2. C: 드라이브에서 마우스 오른쪽 클릭
3. “볼륨 축소” 선택
4. 현재 Windows 파티션을 200GB 로 축소
  - C: 만 축소, 남은 공간은 “할당되지 않음” 상태

## USB 로 부팅
기기가 꺼진 상태에서
1. 기기를 완전히 종료 후에 기기 부팅과 동시에 키보드 `F7` 반복 입력
2. 부팅 디스크 선택에서 USB 부팅을 선택

## Bazzite 설치
> 참고한 문서
> - [Windows 11과 Bazzite를 듀얼 부팅하는 방법(레거시 ISO 가이드)](https://www.youtube.com/watch?v=KAt49B6rSFI)
> - [Dual Boot Preliminary and Post-Installation Setup Guide](https://docs.bazzite.gg/General/Installation_Guide/dual_boot_setup_guide/)

1. `SYSTEM` → `Installation Destination` 진입
2. `Storage Configuration` → `Advanced Custom` 선택 → `Done (좌상단)`
3. `+` → Size `300MB` → Filesystem `EFI System Partition` → Mountpoint `/boot/efi`
4. `+` → Size `2GB` → Filesystem `ext4` → Mountpoint `/boot`
5. `+` → Size `150GB` → Filesystem `btrfs`
6. `+` → Size `남은 크기 모두` → Filesystem `btrfs`
7. 좌측에서 `5.` 에서 생성한 볼륨 선택
  - `+` → Name `root` → Mountpoint `/`
  - `+` → Name `var` → Mountpoint `/var`
  - `+` → Name `home` → Mountpoint `/var/home`
8. `Done (좌상단)` → `Accept Changes`
9. `Begin Installation`

::: tip
설정하지 않은 경우 기본 Username: `bazzite`, Password: `bazzite` 로 설정됨   
`User Creation` 항목을 선택하여 원하는 사용자 ID/PW 설정 가능
:::

## BIOS 부팅 순서 변경
1. 기기를 완전히 종료
2. 기기 부팅과 동시에 키보드 `F7` 반복 입력
3. Boot 탭으로 이동
4. Boot Option Priorities 순서를 다음과 같이 변경
  - `Boot Option #1` Fedora
  - `Boot Option #2` Windows Boot Manager
  - `Boot option #3` UEFI OS
5. Save & Exit `F4`

## Bazzite 설정
### 리베이스
> 참고한 문서
> - [Rebase Guide](https://docs.bazzite.gg/Installing_and_Managing_Software/Updates_Rollbacks_and_Rebasing/rebase_guide/)

현재 버전 확인
``` bash
rpm-ostree status
```
44 버전으로 확인되는 경우 리베이스 진행

::: tip
43.20260420 리베이스 진행
``` bash
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/ublue-os/bazzite-deck:stable-43.20260420
```

> `/ghcr.io/ublue-os/bazzite-deck:stable-43.20260420` 이미지가 없어진경우
> ``` bash
> sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/syrup2525/bazzite-deck:stable-43.20260420
> ```

기기 재부팅
``` bash
sudo systemctl reboot
```

재부팅 후 확인
``` bash
rpm-ostree status
```
:::

### Games 파티션 마운트
``` bash
sudo mkdir -p /var/mnt/games
lsblk -f
```

`Games` 파티션 UUID를 확인한 뒤:
``` bash
sudo nano /etc/fstab
```

제일 마지막줄 아래 내용 추가
``` txt
UUID=여기에-Games-UUID /var/mnt/games btrfs defaults,noatime,compress=zstd:1,ssd,nofail,x-systemd.automount,x-systemd.device-timeout=5 0 0
```

적용
``` bash
sudo systemctl daemon-reload
sudo mount -a
```

확인
``` bash
df -h /var/mnt/games
```

디렉터리 생성
``` bash
sudo mkdir -p /var/mnt/games/{SteamLibrary-Windows,SteamLibrary-Bazzite,Etc-Windows,Shared}
```

소유권 설정
``` bash
sudo chown -R $USER:$USER /var/mnt/games
```

### Steam 설정
::: tip
데스크톱 모드에서 진행
:::

Steam 실행 → 왼쪽 상단 Steam 아이콘 → 설정 → 저장 공간 → 상단 드라이브 영역 선택 → 드라이브 추가 → /var/mnt/games 영역 선택 → 다른 위치를 선택합니다. → 추가 → /var/mnt/games/SteamLibrary-Bazzite → OK

### Apex 용 bazzite 패치 적용
> 참고 Github 문서 [onexplayer-apex-bazzite-fixes](https://github.com/srsholmes/onexplayer-apex-bazzite-fixes)

#### Decky Loader 설치
``` bash
curl -L https://github.com/SteamDeckHomebrew/decky-installer/releases/latest/download/install_release.sh | sh
```

#### OneXPlayer Apex Tools 설치
1. [릴리스](https://github.com/srsholmes/onexplayer-apex-bazzite-fixes/releases) 페이지에서 `OneXPlayer_Apex_Tools.zip` 다운로드
2. 게임 모드로 전환
3. Decky Loader에서 개발자 모드를 활성화
  - 기기 화면 오른쪽 영역 왼쪽으로 슬라이드
  - Decky 탭(플러그 아이콘)으로 이동
  - 설정 톱니바퀴 아이콘(오른쪽 상단) 열기
  - 개발자 모드 켜기/ 끄기
4. 플러그인을 설치
  - Decky 설정 페이지에 새로운 `개발자` 섹션 에서
  - `ZIP 파일에서 플러그인 설치`를 클릭
  - 해당 위치로 이동하여 OneXPlayer_Apex_Tools.zip 선택
5. 플러그인 목록에서 `OneXPlayer_Apex_Tools` 선택후 옵션들을 활성화

### 부팅시 부팅 OS 선택 적용
``` bash
ujust regenerate-grub
```

## Windows 설정
### Games 파티션 마운트

Windows에서 BTRFS 파티션을 읽고 쓰기 위해 WinBtrfs를 설치

#### 1. Bazzite에서 UUID와 UID/GID 확인

Bazzite 에서 각 BTRFS 파티션의 UUID를 확인

```bash
lsblk -f
```

아래 값을 미리 메모

```txt
Bazzite system UUID = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Games UUID          = yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

현재 Bazzite 사용자 UID/GID 확인

```bash
id
```

예시

```txt
uid=1000(deck) gid=1000(deck)
```

아래 값도 메모

```txt
Linux UID = 1000
Linux GID = 1000
```

#### 2. WinBtrfs 설치

Windows로 부팅한 뒤 WinBtrfs 최신 릴리스를 다운로드

1. WinBtrfs 릴리스 ZIP 다운로드
2. ZIP 압축 해제
3. `btrfs.inf` 파일 우클릭
4. `설치` 선택
5. Windows 재부팅

#### 3. Windows 사용자 SID 확인

일반 명령 프롬프트 또는 PowerShell에서 현재 Windows 사용자 SID를 확인

```bat
wmic useraccount get name,sid
```

예시

```txt
Name        SID
myuser      S-1-5-21-1234567890-123456789-1234567890-1001
```

현재 사용하는 Windows 계정의 SID를 메모

```txt
Windows SID = S-1-5-21-1234567890-123456789-1234567890-1001
```

#### 4. Bazzite system 파티션 숨김 처리

`regedit`를 관리자 권한으로 실행

아래 경로로 이동

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs
```

각 BTRFS 파일시스템 UUID 이름의 하위 키가 생성되어 있는지 확인

예시

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

UUID 키가 없으면 Bazzite에서 확인한 UUID 이름으로 직접 키를 생성

`Bazzite system UUID`에 해당하는 키를 선택하고 아래 DWORD 값을 생성

```txt
값 이름: Ignore
값 종류: DWORD(32비트)
값 데이터: 1
기준: 10진수
```

예시

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
└─ Ignore = 1
```

#### 5. SID → UID 매핑 설정

`regedit`에서 아래 경로로 이동

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\Mappings
```

`Mappings` 키가 없으면 직접 생성

그 다음 아래 값을 생성

```txt
값 이름: Windows 사용자 SID
값 종류: DWORD(32비트)
값 데이터: Linux UID
기준: 10진수
```

예시

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\Mappings
└─ S-1-5-21-1234567890-123456789-1234567890-1001 = 1000
```

#### 6. SID → GID 매핑 설정

`regedit`에서 아래 경로로 이동

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\GroupMappings
```

`GroupMappings` 키가 없으면 직접 생성

그 다음 아래 값을 생성

```txt
값 이름: Windows 사용자 SID
값 종류: DWORD(32비트)
값 데이터: Linux GID
기준: 10진수
```

예시

```txt
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\btrfs\GroupMappings
└─ S-1-5-21-1234567890-123456789-1234567890-1001 = 1000
```

#### 7. Windows 재부팅

레지스트리 설정을 적용하기 위해 Windows를 재부팅

#### 8. Games 파티션 드라이브 문자 할당

Windows 탐색기 또는 디스크 관리에서 `Games` 파티션에 드라이브 문자를 할당

예시

```txt
G:
```

#### 9. Games 파티션 쓰기 테스트

Windows에서 `Games` 드라이브에 테스트 파일을 생성

예시

```txt
G:\Shared\windows-write-test.txt
```

그 다음 Bazzite로 부팅해서 파일이 보이는지 확인

```bash
ls -l /var/mnt/games/Shared/
```

파일 소유자가 현재 Bazzite 사용자로 보이면 정상

```txt
-rw-r--r--. 1 deck deck ... windows-write-test.txt
```

#### 10. Steam 라이브러리 분리

Windows Steam에서는 아래 폴더만 라이브러리로 추가

```txt
G:\SteamLibrary-Windows
```

Bazzite Steam에서는 아래 폴더만 라이브러리로 추가

```txt
/var/mnt/games/SteamLibrary-Bazzite
```
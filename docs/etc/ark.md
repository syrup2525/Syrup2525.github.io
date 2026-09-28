| 항목 | 이번 설치 예시 |
|---|---|
| 서버 PC | Windows 11 |
| 설치 경로 | `C:\ASA` |
| 맵 | **The Island** |
| 서버 이름 | `USER_SERVER_NAME-ARK` |
| 접속 제한 | **15명** — 성능 보장치가 아니라 초기 설정값 |
| 게임 방식 | **PvE / 모드 없음 / 접속 비밀번호 사용** |
| 관리 방식 | 서버 PC 내부에서 RCON으로 저장·종료 |

## 1. SteamCMD와 실행 라이브러리 준비
### ① 설치 폴더와 SteamCMD 준비
```bash
$ErrorActionPreference = 'Stop'

New-Item -ItemType Directory -Force -Path `
    'C:\ASA\SteamCMD', `
    'C:\ASA\Server', `
    'C:\ASA\Backups', `
    'C:\ASA\Tools' | Out-Null

Invoke-WebRequest `
    -Uri 'https://client-update.steamstatic.com/installer/steamcmd.zip' `
    -OutFile 'C:\ASA\steamcmd.zip'

Expand-Archive `
    -Path 'C:\ASA\steamcmd.zip' `
    -DestinationPath 'C:\ASA\SteamCMD' `
    -Force
```

### ② Visual C++ 런타임 설치
```bash
Invoke-WebRequest `
    -Uri 'https://aka.ms/vc14/vc_redist.x64.exe' `
    -OutFile 'C:\ASA\vc_redist.x64.exe'

Start-Process `
    -FilePath 'C:\ASA\vc_redist.x64.exe' `
    -Wait
```

## 2. 서버 설치·업데이트용 배치 파일 생성
```txt
C:\ASA\update-server.bat
```

::: code-group
```bat [update-server.bat]
@echo off
setlocal

set "ROOT=%~dp0"

if not exist "%ROOT%SteamCMD\steamcmd.exe" (
    echo [ERROR] steamcmd.exe was not found.
    pause
    exit /b 1
)

tasklist /FI "IMAGENAME eq ArkAscendedServer.exe" /NH | find /I "ArkAscendedServer.exe" >nul
if not errorlevel 1 (
    echo [ERROR] Stop the ASA server before updating.
    pause
    exit /b 1
)

"%ROOT%SteamCMD\steamcmd.exe" +force_install_dir "%ROOT%Server" +login anonymous +app_update 2430930 validate +quit

set "RESULT=%ERRORLEVEL%"

echo.
echo SteamCMD exit code: %RESULT%
echo Check the output above for successful installation or update.
pause

exit /b %RESULT%
```
:::

`update-server.bat` 실행

## 3. 접속 비밀번호와 기본 설정 생성
```bash
New-Item -ItemType Directory -Force `
    -Path 'C:\ASA\Server\ShooterGame\Saved\Config\WindowsServer' |
    Out-Null

notepad 'C:\ASA\Server\ShooterGame\Saved\Config\WindowsServer\GameUserSettings.ini'
```

```txt
[ServerSettings]
ServerPassword=CHANGE_ME_JOIN
ServerAdminPassword=CHANGE_ME_ADMIN
ServerPVE=True
RCONEnabled=True
RCONPort=27020
AutoSavePeriodMinutes=15.0
```

| 설정 | 의미 |
|---|---|
| `ServerPassword` | 플레이어가 서버에 접속할 때 사용하는 비밀번호 |
| `ServerAdminPassword` | 관리자 명령과 RCON 인증에 사용하는 비밀번호 **접속자에게 공유 X** |
| `ServerPVE=True` | 이번 예시에서는 PvE로 운영 |
| `RCONEnabled=True` | 게임을 실행하지 않고도 서버 관리 명령을 보낼 수 있도록 설정 |
| `RCONPort=27020` | 이번 구성에서 사용할 관리용 TCP 포트 |
| `AutoSavePeriodMinutes=15.0` | 자동 저장 간격을 15분으로 설정 |

## 4. 서버 실행용 배치 파일 생성
```txt
C:\ASA\start-server.bat
```

::: code-group
```bat [start-server.bat]
@echo off
setlocal

set "ROOT=%~dp0"
set "MAP=TheIsland_WP"
set "SERVER_NAME=USER_SERVER_NAME-AARK"
set "MAX_PLAYERS=15"
set "GAME_PORT=7777"

set "CONFIG=%ROOT%Server\ShooterGame\Saved\Config\WindowsServer\GameUserSettings.ini"

if not exist "%CONFIG%" (
    echo [ERROR] GameUserSettings.ini was not found.
    pause
    exit /b 1
)

findstr /L /C:"CHANGE_ME_" "%CONFIG%" >nul
if not errorlevel 1 (
    echo [ERROR] Replace the example passwords in GameUserSettings.ini first.
    pause
    exit /b 1
)

tasklist /FI "IMAGENAME eq ArkAscendedServer.exe" /NH | find /I "ArkAscendedServer.exe" >nul
if not errorlevel 1 (
    echo [ERROR] An ASA server is already running.
    pause
    exit /b 1
)

pushd "%ROOT%Server\ShooterGame\Binaries\Win64"
if errorlevel 1 (
    echo [ERROR] Server directory was not found.
    pause
    exit /b 1
)

ArkAscendedServer.exe "%MAP%?listen?SessionName=%SERVER_NAME%" -Port=%GAME_PORT% -WinLiveMaxPlayers=%MAX_PLAYERS% -log

popd
pause
```
:::

| 항목 | 이번 값 |
|---|---|
| 맵 | `TheIsland_WP` |
| 서버 이름 | `USER_SERVER_NAME-ASA` |
| 게임 포트 | `7777` |
| 최대 접속 인원 | `-WinLiveMaxPlayers=15` |

`server-start.bat` 실행

## 5. Windows 방화벽과 공유기 포트포워딩 설정
### ① 이번 구성에서 공개 포트

| 용도 | 프로토콜·포트 | 처리 방법 |
|---|---|---|
| **플레이어의 게임 접속** | **UDP 7777** | Windows 방화벽 허용 + 공유기 포트포워딩 |
| **RCON 관리** | **TCP 27020** | **외부에 공개하지 않고 서버 PC 내부에서 사용** |

### ② Windows 방화벽을 설정
```bash
New-NetFirewallRule `
  -DisplayName "ARK ASA Game Ports" `
  -Direction Inbound `
  -Protocol UDP `
  -LocalPort 7777-7778 `
  -Action Allow
```

```bash
Get-NetFirewallRule -DisplayName "ARK ASA Game Ports"
```

### ③ 공유기에서 서버 PC로 포트포워딩
| 공유기 설정 항목 | 입력 예시 |
|---|---|
| 규칙 이름 | `ASA` |
| 프로토콜 | **UDP** |
| 외부 포트 | **7777-7778** |
| 내부 IP | **192.168.0.50 — 실제 서버 PC 주소로 변경.** |
| 내부 포트 | **7777-7778** |

## 6. 접속 확인
### UDP 포트가 열려 있는지 확인
```bash
Get-NetUDPEndpoint -LocalPort 7777 -ErrorAction SilentlyContinue |
    Select-Object LocalAddress, LocalPort, OwningProcess
```

### 인게임에서 세션 접속

- 세션 필터: 비공식, USER_SERVER_NAME-ARK 검색
- 비밀번호: `ServerPassword`

※ 암호로 보호된 세션 표시 옵션 켜기   
※ Show Player Servers에 해당하는 플레이어 서버 표시 옵션을 켜고 목록을 새로고침
# Worker 구성
## 사전 작업
::: details 사용자 생성하기 (선택)
::: tip
root 권한 전환
``` bash
sudo -i
```

user 계정 추가
``` bash
useradd -m user
```

user 계정에 SSH 디렉토리 생성 및 권한 설정
``` bash
mkdir -p /home/user/.ssh
chmod 700 /home/user/.ssh
cp /home/rocky/.ssh/authorized_keys /home/user/.ssh/
chmod 600 /home/user/.ssh/authorized_keys
chown -R user:user /home/user/.ssh
```

sudo 권한 부여
``` bash
usermod -aG wheel user
```
:::

### 메모리 swap off
메모리 swap off
```bash
swapoff -a
```

메모리 swapp off (재부팅 후 설정 초기화 방지)
```bash
vi /etc/fstab
```

해당 라인을 주석(삭제)처리 해준다.
```txt
UUID=1234-ABCD          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
/dev/mapper/cs-home     /home                   xfs     defaults        0 0
/dev/mapper/cs-swap     none                    swap    defaults        0 0 // [!code --]
```

swap 영역 확인
```bash
free -h
```

실행 결과
```bash
              total        used        free      shared  buff/cache   available
Mem:           15Gi       255Mi        13Gi       773Mi       1.4Gi        13Gi
Swap:            0B          0B          0B
```

## RKE2 를 이용해 쿠버네티스 설치
### RKE2 agent 설치
명령어 실행 전 root 계정 전환 필요
```bash
su root
```

curl 명령어 실행
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

::: details 특정 버전 설치시
``` bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION="v1.31.4+rke2r1" sh -
```
:::

실행 결과
```bash
[INFO]  finding release for channel stable
[INFO]  using 1.28 series from channel stable
Rancher RKE2 Common (stable)                                                                                                                        1.6 kB/s | 2.9 kB     00:01    
Rancher RKE2 1.28 (stable)                                                                                                                          2.4 kB/s | 4.6 kB     00:01    
Dependencies resolved.

...

Installed:
  rke2-agent-1.28.9~rke2r1-0.el8.x86_64                        rke2-common-1.28.9~rke2r1-0.el8.x86_64                        rke2-selinux-0.18-1.el8.noarch                       

Complete!
```


::: details air-gap artifact 설치 방법
#### 필요한 경우 패키지 설치
``` bash
dnf install -y tar gzip zstd
```

#### 작업 공간 준비
``` bash
mkdir -p /root/rke2-artifacts
cd /root/rke2-artifacts
```

#### 필요 파일 다운로드
``` bash
curl -OLs "https://github.com/rancher/rke2/releases/download/v1.32.7%2Brke2r1/rke2-images.linux-amd64.tar.zst"
curl -OLs "https://github.com/rancher/rke2/releases/download/v1.32.7%2Brke2r1/rke2.linux-amd64.tar.gz"
curl -OLs "https://github.com/rancher/rke2/releases/download/v1.32.7%2Brke2r1/sha256sum-amd64.txt"
curl -sfL https://get.rke2.io -o install.sh
```

#### 파일 유효성 검증
``` bash
grep -E 'rke2-images.linux-amd64.tar.zst|rke2.linux-amd64.tar.gz' sha256sum-amd64.txt | sha256sum -c -
```

#### 설치 진행
``` bash
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts \
INSTALL_RKE2_TYPE="agent" \
sh install.sh
```
:::

config.yaml 설정
```bash
mkdir -p /etc/rancher/rke2/
vi /etc/rancher/rke2/config.yaml
```

```yaml title="config.yaml"
server: https://<server>:9345
token: <token from server node> # 마스터 노드 토큰
node-name: worker1 # agent 이름
```

::: tip
token 값은 [Worker 노드 등록에 필요한 token 확인](/kubernetes/01-install/02-k8s/install/step1-master.html#worker-노드-등록에-필요한-token-확인) 에서 확인
:::

### 서비스 시작 및 등록
```bash
systemctl enable rke2-agent
```
```bash
systemctl start rke2-agent
```
```bash
systemctl status rke2-agent
```

실행 결과
```bash
● rke2-agent.service - Rancher Kubernetes Engine v2 (agent)
   Loaded: loaded (/usr/lib/systemd/system/rke2-agent.service; disabled; vendor preset: disabled)
   Active: active (running)
   ...
```

::: tip
로그 확인 명령어
```bash
journalctl -u rke2-agent -f
```
:::

노드 확인 (Master 에서 확인)
```bash
kubectl get nodes
```

실행 결과
```bash
NAME      STATUS   ROLES                       AGE     VERSION
master1   Ready    control-plane,etcd,master   9m46s   v1.28.9+rke2r1
worker1    Ready    <none>                      46s     v1.28.9+rke2r1
```

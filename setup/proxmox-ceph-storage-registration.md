# Proxmox Ceph Storage 등록 작업 기록

## 1. 작업 목적

Proxmox VE 클러스터에서 외부 Ceph 클러스터의 팀별 RBD pool과 CephFS를 Datacenter Storage로 등록한다.

이 작업의 목표는 다음과 같다.

- VM 디스크와 VM 템플릿 디스크를 Ceph RBD에 저장한다.
- Proxmox 노드 간 clone, migration, HA 구성을 위한 shared storage 기반을 만든다.
- ISO, backup, container template, snippets 저장소로 CephFS를 사용할 수 있게 준비한다.
- 팀별 RBD pool과 팀별 Ceph 인증 계정을 사용해 다른 팀 pool 접근을 제한한다.

## 2. 전체 구조

현재 Ceph는 팀별로 별도 클러스터를 만드는 방식이 아니라, 하나의 Ceph 클러스터 안에 팀별 pool을 나누어 사용하는 구조다.

```text
Ceph Cluster
├─ ceph1: 10.10.10.11
├─ ceph2: 10.10.10.12
├─ ceph3: 10.10.10.13
└─ ceph4: 10.10.10.14

RBD Pools
├─ rbd-team1
├─ rbd-team2
├─ rbd-team3
└─ rbd-team4

CephFS
├─ cephfs_metadata
└─ cephfs_data
```

3조 Proxmox 클러스터에서는 다음 값을 사용한다.

| 항목                   | 값                                                         |
| ---------------------- | ---------------------------------------------------------- |
| Ceph MON 대역          | `10.10.10.0/24`                                            |
| Ceph MON 후보          | `10.10.10.11`, `10.10.10.12`, `10.10.10.13`, `10.10.10.14` |
| 3조 RBD pool           | `rbd-team3`                                                |
| 3조 Ceph auth 계정     | `client.TEAM3`                                             |
| Proxmox RBD Storage ID | `TEAM3` 또는 `ceph-rbd-team3`                              |
| CephFS 이름            | `cephfs`                                                   |

> `10.10.10.11~14`는 팀별 pool이 아니라 Ceph 클러스터를 구성하는 노드 IP다. 팀별 구분은 `rbd-team3` 같은 pool과 `client.TEAM3` 같은 auth 계정으로 한다.

## 3. Ceph pool 생성 여부 확인

Ceph CLI가 가능한 노드에서 pool 목록을 확인한다.

```bash
ceph osd pool ls
```

정상적으로 생성되어 있으면 다음 pool들이 보여야 한다.

```text
rbd-team1
rbd-team2
rbd-team3
rbd-team4
cephfs_metadata
cephfs_data
```

RBD pool의 상세 상태는 다음 명령으로 확인한다.

```bash
ceph osd pool ls detail
ceph osd pool get rbd-team3 all
ceph osd pool application get rbd-team3
```

`rbd-team3`에는 RBD application이 활성화되어 있어야 한다.

```bash
rbd pool init rbd-team3
ceph osd pool application enable rbd-team3 rbd
```

이미 활성화되어 있으면 위 명령을 다시 실행할 필요는 없다.

## 4. TEAM3 Ceph auth 계정 생성

팀별 pool 분리를 위해 Proxmox에서 `client.admin` 대신 `client.TEAM3` 계정을 사용한다.

Ceph CLI가 가능한 노드에서 다음 명령을 실행한다.

```bash
ceph auth get-or-create client.TEAM3 \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd-team3' \
  mgr 'profile rbd pool=rbd-team3' \
  -o /etc/ceph/ceph.client.TEAM3.keyring
```

계정 생성 여부를 확인한다.

```bash
ceph auth get client.TEAM3
ceph auth ls | grep client.TEAM
```

정상 권한 예시는 다음과 같다.

```text
client.TEAM3
    caps: [mon] profile rbd
    caps: [osd] profile rbd pool=rbd-team3
    caps: [mgr] profile rbd pool=rbd-team3
```

## 5. Proxmox용 RBD secret 파일 생성

Proxmox RBD storage는 일반 Ceph keyring 전체 형식이 아니라, secret 파일에 순수 key 값만 저장하는 방식을 사용한다.

Ceph CLI가 가능한 노드에서 TEAM3 key 값을 확인한다.

```bash
ceph auth get-key client.TEAM3
```

출력된 key 값을 Proxmox 노드의 다음 파일에 한 줄로 저장한다.

```bash
mkdir -p /etc/pve/priv/ceph
echo '<client.TEAM3 key value>' > /etc/pve/priv/ceph/TEAM3.keyring
chmod 600 /etc/pve/priv/ceph/TEAM3.keyring
```

파일 내용은 다음처럼 key 값 한 줄만 있어야 한다.

```text
AQD...
```

다음처럼 `[client.TEAM3]`, `caps`가 포함된 전체 keyring 형식이면 Proxmox에서 RBD 인증 파일로 인식하지 못한다.

```text
[client.TEAM3]
        key = AQD...
        caps mon = ...
        caps osd = ...
```

이 경우 `pvesm status`에서 다음 오류가 발생할 수 있다.

```text
Not a proper rbd authentication file: /etc/pve/priv/ceph/TEAM3.keyring
```

## 6. Proxmox Datacenter에 RBD Storage 등록

Proxmox GUI에서 다음 경로로 이동한다.

```text
Datacenter -> Storage -> Add -> RBD
```

입력값은 다음 기준으로 설정한다.

| 항목     | 값                            |
| -------- | ----------------------------- |
| ID       | `TEAM3` 또는 `ceph-rbd-team3` |
| Pool     | `rbd-team3`                   |
| Monitor  | `ceph mon dump`에 나온 MON IP |
| Username | `TEAM3`                       |
| Content  | `Disk image`                  |
| KRBD     | 기본값 또는 비활성            |
| Nodes    | 3조 Proxmox 노드 전체         |

CLI 설정 파일은 `/etc/pve/storage.cfg`에서 확인할 수 있다.

예시는 다음과 같다.

```text
rbd: TEAM3
        content images
        krbd 0
        monhost 10.10.10.11 10.10.10.12 10.10.10.13 10.10.10.14
        pool rbd-team3
        username TEAM3
```

`username`과 secret 파일의 key는 반드시 같은 계정 기준이어야 한다.

| username | 필요한 secret 파일 내용      |
| -------- | ---------------------------- |
| `TEAM3`  | `client.TEAM3`의 순수 key 값 |
| `admin`  | `client.admin`의 순수 key 값 |

3조 pool 분리 목적상 RBD는 `username TEAM3` 사용을 기본으로 한다.

## 7. RBD 등록 검증

Proxmox 노드에서 Storage 상태를 확인한다.

```bash
pvesm status
```

정상 예시는 다음과 같다.

```text
Name      Type  Status   Total (KiB)  Used (KiB)  Available (KiB)  %
TEAM3      rbd  active   ...
local      dir  active   ...
local-lvm  lvmthin active ...
```

GUI에서는 다음 위치에서 확인한다.

```text
Datacenter -> Storage
```

`TEAM3` 또는 `ceph-rbd-team3`의 상태가 `active`로 보여야 한다. `unknown` 또는 `inactive`이면 아직 정상 등록된 상태가 아니다.

RBD storage 접근 테스트는 다음 명령으로 확인한다.

```bash
pvesm list TEAM3
```

에러 없이 실행되면 Proxmox가 해당 RBD storage를 조회할 수 있는 상태다.

VM 생성 또는 clone 후 Ceph 쪽에서 이미지가 생겼는지도 확인한다.

```bash
rbd ls rbd-team3
```

정상적으로 VM 디스크가 생성되면 다음과 같은 이미지가 보인다.

```text
vm-9000-disk-0
vm-101-disk-0
```

## 8. CephFS 생성 여부 확인

CephFS 파일시스템이 생성되어 있는지 확인한다.

```bash
ceph fs ls
```

정상 예시는 다음과 같다.

```text
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]
```

pool 목록에서도 다음 항목이 보여야 한다.

```bash
ceph osd pool ls | grep cephfs
```

```text
cephfs_metadata
cephfs_data
```

## 9. Proxmox Datacenter에 CephFS Storage 등록

Proxmox GUI에서 다음 경로로 이동한다.

```text
Datacenter -> Storage -> Add -> CephFS
```

입력값은 다음 기준으로 설정한다.

| 항목       | 값                                                                  |
| ---------- | ------------------------------------------------------------------- |
| ID         | `cephfs` 또는 `cephfs-team3`                                        |
| Monitor    | `ceph mon dump`에 나온 MON IP                                       |
| Username   | `admin` 또는 CephFS 권한이 있는 별도 계정                           |
| Filesystem | `cephfs`                                                            |
| Content    | `ISO image`, `VZDump backup file`, `Container template`, `Snippets` |
| Nodes      | 필요한 Proxmox 노드 전체                                            |

CephFS는 RBD와 용도가 다르다.

| 저장소 | Proxmox Storage Type | 주 용도                                    |
| ------ | -------------------- | ------------------------------------------ |
| RBD    | `RBD`                | VM disk, VM template disk, CloudInit drive |
| CephFS | `CephFS`             | ISO, backup, container template, snippets  |

VM의 실제 디스크는 CephFS가 아니라 RBD에 둔다.

## 10. VM clone / migration 기준

RBD는 shared storage이므로 VM 디스크를 RBD에 두면 다른 Proxmox 노드에서 clone 또는 migration이 가능하다.

템플릿에서 다른 노드로 VM을 clone할 때는 다음 기준을 사용한다.

```text
Template VM -> More -> Clone
Target Node: 대상 Proxmox 노드
Mode: Full Clone 권장
Storage: TEAM3
```

CLI 예시는 다음과 같다.

```bash
qm clone 9000 110 \
  --name k8s-cp-1 \
  --target kosa-team3-02 \
  --full 1 \
  --storage TEAM3
```

이미 존재하는 VM을 다른 노드로 이동하는 경우는 clone이 아니라 migration을 사용한다.

```bash
qm migrate <VMID> <target-node>
```

VM 디스크가 RBD에 있고 대상 노드에서도 `TEAM3` storage가 active이면 디스크 복사 없이 이동할 수 있다.

## 11. 자주 발생한 오류와 확인 방법

### 11-1. Storage 상태가 `unknown` 또는 `inactive`

확인 명령:

```bash
pvesm status
cat /etc/pve/storage.cfg
cat /etc/pve/priv/ceph/TEAM3.keyring
```

주요 원인:

- `monhost`가 실제 MON IP가 아니다.
- Proxmox 노드에서 MON IP로 ping이 되지 않는다.
- `/etc/pve/priv/ceph/TEAM3.keyring` 파일 형식이 잘못되었다.
- `username`과 secret 파일의 key가 서로 다른 계정 기준이다.
- pool 이름이 `rbd-team3`가 아니라 다른 이름으로 입력되었다.

### 11-2. `Not a proper rbd authentication file`

원인:

```text
/etc/pve/priv/ceph/TEAM3.keyring 파일에 key 값만 들어 있지 않음
```

해결:

```bash
ceph auth get-key client.TEAM3
echo '<client.TEAM3 key value>' > /etc/pve/priv/ceph/TEAM3.keyring
chmod 600 /etc/pve/priv/ceph/TEAM3.keyring
```

### 11-3. `rbd: couldn't connect to the cluster`

CLI에서 다음과 같은 오류가 날 수 있다.

```text
can't open ceph.conf: No such file or directory
unable to get monitor info
rbd: couldn't connect to the cluster
```

이 오류는 RBD 권한보다 먼저 Ceph client 설정 파일을 찾지 못해 MON 정보를 모르는 경우다.

CLI 테스트를 위해서는 `/etc/ceph/ceph.conf`가 필요하다.

```bash
mkdir -p /etc/ceph
vi /etc/ceph/ceph.conf
```

예시:

```ini
[global]
fsid = <ceph-fsid>
mon_host = 10.10.10.11,10.10.10.12,10.10.10.13,10.10.10.14
```

`fsid`는 Ceph CLI가 가능한 노드에서 확인한다.

```bash
ceph fsid
ceph mon dump
```

단, Proxmox GUI Storage 등록 자체는 `/etc/pve/storage.cfg`와 `/etc/pve/priv/ceph/<storage-id>.keyring` 기준으로 동작하므로, CLI 테스트 실패가 곧 GUI 등록 실패를 의미하지는 않는다.

### 11-4. apt install 실패

다음 오류는 디스크 문제가 아니라 DNS 문제다.

```text
Temporary failure resolving 'deb.debian.org'
```

확인:

```bash
ip route
cat /etc/resolv.conf
ping -c 3 8.8.8.8
ping -c 3 deb.debian.org
```

`8.8.8.8` ping은 되지만 `deb.debian.org`가 실패하면 DNS 서버를 수정한다.

```bash
cat > /etc/resolv.conf <<EOF
search co.kr
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF
```

Proxmox에서는 GUI에서 영구 DNS도 확인한다.

```text
Node -> System -> DNS
```

## 12. 완료 기준

다음 항목이 모두 만족되면 Proxmox Ceph Storage 등록이 완료된 상태로 본다.

- `ceph osd pool ls`에서 `rbd-team3`, `cephfs_metadata`, `cephfs_data`가 확인된다.
- `ceph auth get client.TEAM3`에서 `rbd-team3` 전용 권한이 확인된다.
- `/etc/pve/priv/ceph/TEAM3.keyring`에 `client.TEAM3`의 순수 key 값만 저장되어 있다.
- `/etc/pve/storage.cfg`에 RBD storage가 `pool rbd-team3`, `username TEAM3` 기준으로 등록되어 있다.
- `pvesm status`에서 RBD storage가 `active`로 표시된다.
- Proxmox VM 생성 화면에서 RBD storage를 VM disk 저장소로 선택할 수 있다.
- CephFS storage가 ISO, backup, container template, snippets 용도로 등록되어 있다.
- RBD에 생성한 템플릿 또는 VM을 다른 Proxmox 노드로 clone 또는 migration할 수 있다.

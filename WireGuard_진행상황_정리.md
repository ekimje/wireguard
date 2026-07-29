# WireGuard + Azure VPN 구축 — 진행 상황 및 개념 정리

**작성일**: 2026-07-29
**원본 계획서**: `WireGuard_연결테스트_계획서.md`

---

## 1. 핵심 개념 정리

### WireGuard란
가볍고 빠른 VPN 프로토콜. 서로 다른 네트워크(집 와이파이, 모바일 데이터, 다른 건물망 등)에 있는 기기들을 **가상 사설망(10.8.0.0/24)** 하나로 묶어서, 실제 네트워크가 뭐든 상관없이 가상 IP로 서로 통신하게 해준다.

### 키(Key) 구조
- **개인키(PrivateKey)**: 각 기기가 자기 자신만 갖고 있는 비밀값. 절대 외부 노출 금지.
- **공개키(PublicKey)**: 개인키에서 유도되는 값. 상대방에게 "나는 이 키의 주인이다"를 증명하는 용도로 공유.
- 구조: `wg genkey`로 개인키 생성 → `wg pubkey`로 그 개인키에서 공개키 유도.
- 서버의 `wg0.conf`를 재생성하면 공개키가 바뀌므로, 이미 등록된 클라이언트들의 `[Peer] PublicKey` 값도 전부 다시 맞춰야 함 (이번에 실제로 겪은 이슈).

### Peer(피어)와 AllowedIPs
- 서버 입장에서 각 클라이언트는 "피어"이고, `allowed-ips`로 "이 피어가 어떤 가상 IP를 쓰는지"를 등록해야 트래픽을 인식하고 라우팅해준다.
- 클라이언트 conf의 `AllowedIPs = 10.8.0.0/24`는 "이 대역 전체는 터널로 보낸다"는 뜻 — 서버뿐 아니라 다른 피어(PC↔폰 등)와도 서버를 경유해 통신 가능하게 해줌(허브형 토폴로지).
- 이게 가능하려면 서버에서 `net.ipv4.ip_forward=1`(IP 포워딩)이 켜져 있어야 함.

### NSG(Network Security Group)
- Azure의 방화벽 개념. 인바운드/아웃바운드 규칙을 우선순위(숫자가 낮을수록 먼저 평가) 순으로 적용.
- 사용자 정의 규칙(100~4096) + 기본 규칙(65000번대, 삭제 불가, 항상 마지막에 평가).
- 이번 프로젝트에서 연 규칙: `nsg-wire` — UDP 51820 인바운드 허용, 우선순위 100.

### 고정 공인 IP
- Azure 공용 IP는 SKU가 "표준(Standard)"이어야 하고, 할당 방식을 반드시 **"정적(Static)"**으로 해야 VM을 재시작해도 IP가 안 바뀜. WireGuard 클라이언트들이 이 IP를 Endpoint로 고정 참조하기 때문에 필수.

---

## 2. 최종 네트워크 구성

```
                [Azure VM: vm-wireguard (Ubuntu 24.04)]
                  공인 IP: 20.249.40.254 (고정)
                  가상 IP: 10.8.0.1  (WireGuard 서버)
                              |
        ┌─────────────┬───────┴───────┬─────────────┐
     [집 PC]       [핸드폰]        [아이패드]      [A건물 - 예정]
    10.8.0.2       10.8.0.3        10.8.0.4         (핸드폰으로 대체 예정)
    Windows      Android/모바일    iPadOS
```

| 항목 | 값 |
|---|---|
| Azure VM 이름 | vm-wireguard |
| OS | Ubuntu Server 24.04 LTS (계획서 원안은 Rocky Linux였으나 포털에 없어 변경) |
| 공인 IP | 20.249.40.254 (Standard SKU, Static) |
| WireGuard 포트 | UDP 51820 |
| NSG 규칙 | nsg-wire (UDP 51820, 우선순위 100) / SSH 22 (우선순위 300) |
| 서버 공개키 | `SLbQdj7yG0Y1QwF7em7vpynwUNw+tKnqoX5AS9txGy4=` |
| PC 공개키 | `HwRtR0Q4s+H8EbFmVyl87SPvk5PtMfIjgY1p5ZqMhTw=` |

---

## 3. 진행 상황 (완료된 것)

### ✅ 1단계 — Azure VM 생성
- Ubuntu 24.04, 계정 인증은 암호 방식(원래 계획은 SSH 키였으나 실습 편의상 암호로 진행)
- 네트워킹: 신규 VNet/서브넷/공용 IP 자동 생성, 공용 IP는 표준+정적으로 별도 설정

### ✅ 2단계 — NSG 설정
- UDP 51820 인바운드 허용 규칙(`nsg-wire`) 추가

### ✅ 3단계 — 서버에 WireGuard 설치 및 구동
- `apt install wireguard`, 키 생성, `/etc/wireguard/wg0.conf` 작성
- `systemctl enable --now wg-quick@wg0`로 부팅 시 자동 시작되게 설정

### ✅ 4단계 — PC 클라이언트 연결
- Windows에 `winget install WireGuard.WireGuard`로 설치
- PC 전용 키 생성, 서버에 피어 등록, 터널을 Windows 서비스로 등록 (`/installtunnelservice`)
- **Handshake 성공, ping 성공**

### ✅ 5단계 — 핸드폰(Android) 클라이언트 연결
- 서버에서 폰 전용 키 생성 → QR코드(`qrencode`)로 변환 → 폰 WireGuard 앱에서 QR 스캔으로 한 번에 등록
- **Handshake 성공, ping 성공**

### ✅ Phase 7 — 핸드폰 → PC 원격 제어 (RDP)
- PC에서 원격 데스크톱 활성화, 방화벽/서비스 문제 다수 해결(아래 트러블슈팅 참고)
- 핸드폰에 Windows App(Microsoft Remote Desktop 리브랜딩) 설치, `10.8.0.2`로 접속 성공
- 게임 프로젝트 실행 및 조작, 오디오 전달까지 화면 녹화로 확인 완료

### ✅ Phase 8 — PC → 핸드폰 화면 제어 (scrcpy)
- PC에 scrcpy(Genymobile 공식 릴리스) 설치
- 핸드폰 무선 디버깅 활성화 → `adb pair` → `adb connect` → `scrcpy` 실행 성공

### 🔄 Phase 9 — 아이패드 클라이언트 (진행 중)
- 서버에서 아이패드용 키 생성 및 피어 등록(`10.8.0.4`), QR코드 준비까지 완료
- 아이패드 소프트웨어 업데이트로 일시 중단, 재개 예정
- 아이패드는 셀룰러 데이터가 없어 "모바일 데이터 전환" 대신 **핸드폰 개인 핫스팟**으로 대체 예정

### ⏳ Phase 10 — A건물(다른 네트워크) 검증 (내일 예정)
- 준비된 노트북이 없어 **이미 설정된 핸드폰**으로 대체 진행 예정
- 방법: 핸드폰을 A건물 자체 와이파이에 연결 → WireGuard 터널 ON → 집 PC와 ping/RDP 테스트
- 실패 시 모바일 데이터로 전환해서 재시도(그 자체가 "이 건물망이 UDP를 막는다"는 유의미한 결과)
- 전날 밤 Azure VM은 할당 취소(Deallocate)해도 무방 — Standard 고정 IP는 유지되므로 재시작해도 `20.249.40.254` 그대로

---

## 4. 트러블슈팅 기록 (겪었던 문제 → 원인 → 해결)

| 증상 | 원인 | 해결 |
|---|---|---|
| `wg genkey \| tee A \| wg pubkey \| tee B` 실행 후 A 파일이 0바이트 | 파이프 중간 명령이 stdin을 안 읽고 먼저 끝나면서 broken pipe로 앞 tee가 죽음 | `wg genkey > A`, `wg pubkey < A > B`처럼 리다이렉션으로 분리 |
| 공개키 자리에 엉뚱한 값 생성 | `wg genkey`를 두 번 친 오타 (`wg pubkey`여야 함) | 명령어 재확인 후 재실행 |
| PC에서 폰 접속 안 됨 (handshake는 되는데 계속 재시도) | 서버 wg0.conf를 재생성하면서 서버 공개키가 바뀌었는데 클라이언트 conf는 예전 값 그대로 | 클라이언트 `[Peer] PublicKey`를 새 서버 공개키로 갱신 |
| `C:\Program Files\WireGuard\`에 파일 생성 시 액세스 거부 | Program Files는 일반 쓰기 권한이 제한된 보호 폴더 | 별도 작업 폴더(`C:\WireGuard-client`) 생성해서 진행 |
| cmd에서 `&` 호출 연산자 에러 | PowerShell 문법을 cmd.exe에 사용 | cmd에서는 따옴표로 직접 실행 파일 경로 호출 |
| RDP "PC를 찾을 수 없음" / `Connection refused` | 원격 데스크톱 서비스(TermService)가 꺼져 있었음 + 레지스트리 `fDenyTSConnections=1`(차단 상태) | 서비스 시작, 레지스트리 값 0으로 변경, 방화벽 규칙 활성화 |
| 서비스 다 RUNNING인데도 3389 리스닝이 안 뜸 | 수동으로 서비스 stop/start를 반복하며 상태가 꼬임 | **PC 재부팅**으로 서비스 초기화 순서 정상화 → 해결 |
| `Win+G`(Game Bar)가 안 열림 | 포커스가 관리자 권한 창에 있으면 Game Bar가 보안상 오버레이 불가 | 일반 권한 창으로 포커스 이동 후 재시도 |
| ADB pair 시 "포트"에 6자리 숫자 입력 | 화면에 같이 뜨는 "페어링 코드"(6자리)와 "IP:포트"를 혼동 | `adb pair <IP>:<포트> <6자리코드>` 형식으로 분리 입력 |

---

## 5. 생성된 파일 목록

### Azure VM (`/etc/wireguard/`)

| 파일 | 용도 |
|---|---|
| `server_private.key` / `server_public.key` | 서버 자체 키 쌍 |
| `wg0.conf` | 서버 인터페이스 설정. `[Interface]`(Address 10.8.0.1/24, ListenPort 51820, PrivateKey, `SaveConfig=true`) + `wg set`으로 추가한 PC/폰/아이패드 `[Peer]` 항목들이 `wg-quick save wg0` 실행 시마다 여기 누적 저장됨 |
| `phone_private.key` / `phone_public.key` / `phone.conf` | 핸드폰용 키 쌍 + QR코드 생성 원본 conf |
| `ipad_private.key` / `ipad_public.key` / `ipad.conf` | 아이패드용 키 쌍 + QR코드 생성 원본 conf (진행 중) |
| `qrencode` (패키지) | conf 파일을 터미널에 QR코드로 렌더링하는 용도로 설치 |

> 참고: 위 개인키 파일들은 서버에만 있으면 되고(클라이언트에 그대로 심어준 뒤), 보안상 캡처/공유 시 노출되지 않도록 주의.

### PC — WireGuard 클라이언트 (`C:\WireGuard-client\`)

| 파일 | 용도 |
|---|---|
| `client_private.key` / `client_public.key` | PC용 키 쌍 |
| `wg0-pc.conf` | PC 클라이언트 설정. `/installtunnelservice`로 등록해서 Windows 서비스(`WireguardTunnel$wg0-pc`)로 상시 구동 중 |

### PC — scrcpy (`C:\Users\김지윤\Desktop\scrcpy-win64-v4.1\`)

- Genymobile 공식 릴리스를 압축 해제한 실행 파일 모음(`scrcpy.exe`, `adb.exe` 등) — 별도로 작성한 코드/설정 파일은 없고, 압축 해제한 그대로 사용 중
- adb 페어링/연결은 파일이 아니라 그때그때 `adb pair`, `adb connect` 명령으로만 처리 (저장되는 설정 파일 없음)

## 6. 다음에 할 일

1. 아이패드 업데이트 완료 후 QR 스캔 → 핫스팟 연결 테스트 → RDP 시연
2. 내일 A건물에서 핸드폰으로 이종 네트워크 검증 (VM 사전 기동 확인)
3. 전체 캡처/녹화 영상 정리 — 개인키·QR코드 노출 여부 검수 후 포트폴리오용으로 편집

# WireGuard 설정 파일 (공개용 — 마스킹본)

> 이 문서는 블로그/포트폴리오 등 외부 공개용입니다. 개인키, 공인 IP 등 민감 정보는
> 전부 `<...>` 형태의 placeholder로 대체했습니다. 실제 값이 들어간 원본 `.conf`/`.key` 파일은
> 절대 git이나 외부에 공개하지 마세요.

---

## 서버 (Azure VM) — `wg0.conf`

```ini
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.8.0.1/24
ListenPort = 51820
SaveConfig = true

# 아래는 wg-quick save로 자동 추가되는 Peer 목록 예시
[Peer]   # PC
PublicKey = <PC_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32

[Peer]   # 핸드폰
PublicKey = <PHONE_PUBLIC_KEY>
AllowedIPs = 10.8.0.3/32

[Peer]   # 아이패드
PublicKey = <IPAD_PUBLIC_KEY>
AllowedIPs = 10.8.0.4/32
```

## PC 클라이언트 — `wg0-pc.conf`

```ini
[Interface]
PrivateKey = <PC_PRIVATE_KEY>
Address = 10.8.0.2/24

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

## 핸드폰(Android) — `phone.conf` (QR코드로 등록)

```ini
[Interface]
PrivateKey = <PHONE_PRIVATE_KEY>
Address = 10.8.0.3/24

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

## 아이패드(iPadOS) — `ipad.conf` (QR코드로 등록)

```ini
[Interface]
PrivateKey = <IPAD_PRIVATE_KEY>
Address = 10.8.0.4/24

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

---

## 참고

- `PrivateKey`, `<SERVER_PUBLIC_IP>`는 절대 실제 값으로 채우지 말 것
- `PublicKey` 값 자체는 노출돼도 보안상 문제없지만, 이 문서에선 예시 통일을 위해 마스킹함
- 실제 값이 담긴 `.key`, `wg0.conf`, `wg0-pc.conf`, `phone.conf`, `ipad.conf`는 `.gitignore`로 반드시 제외

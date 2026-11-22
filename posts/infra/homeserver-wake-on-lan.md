---
title: "Wake-on-LAN 동작 원리: 꺼진 PC는 어떻게 패킷을 받아 켜질까"
date: "2026-11-22"
category: "infra"
description: "Wake-on-LAN(WOL)이 동작하는 원리를 홈서버 사례로 정리했습니다. 꺼진 PC가 매직 패킷을 받아 켜지는 과정, 같은 LAN에서만 전달되는 이유, Tailscale로는 깨울 수 없는 이유와 중계기 구성, Wi-Fi WOL이 잘 안 되는 이유를 다룹니다."
---

## 개요

노트북 WSL2 환경에서 메모리가 계속 부족해서 이번 광군제 때 Ryzen 5 7640HS(6코어 12스레드)·24GB 미니PC를 개발 머신으로 들였습니다.

![FIREBAT F1 미니PC 주문 내역: Ryzen 5 7640HS, LPDDR5 24GB, 512GB, 총 결제 금액 386,896원](/images/homeserver-wake-on-lan-purchase.png)

50만 원짜리를 할인받아 38만 원대에 샀는데, 통관하면서 세금이 3만 원 정도 붙어 결국 41만 원 남짓 들었습니다.

집에는 이미 24시간 켜 두는 N100 서버가 있지만 저전력 4코어라 개발용으로는 버겁습니다.

새 PC는 Claude Code Remote Control로 밖에서도 쓰려고 했는데, 전력을 많이 쓰는 만큼 계속 켜 두기보다는 필요할 때만 켜고 싶었습니다.
그래서 꺼진 PC를 네트워크로 켜는 Wake-on-LAN(WOL)을 구성했습니다.

이 글은 구축 과정보다 WOL이 어떻게 동작하는지에 초점을 둡니다.

- 꺼진 PC가 어떻게 패킷을 받아 켜지는지
- 매직 패킷의 구조와, 왜 같은 LAN 안에서만 전달되는지
- Tailscale 같은 VPN만으로는 왜 꺼진 PC를 깨울 수 없는지
- 유선은 되고 Wi-Fi는 잘 안 되는 이유

## 꺼진 PC는 어떻게 패킷을 받을까

PC 전원을 꺼도 전원선이 꽂혀 있으면 메인보드에는 대기 전력이 흐릅니다. WOL을 지원하는 랜카드(NIC)는 이 대기 전력으로 최소한의 회로만 살려 둔 채 들어오는 패킷을 계속 지켜봅니다. 운영체제는 꺼져 있으니 TCP/IP 스택도 없고 IP 주소도 없습니다. 랜카드 혼자서 정해진 모양의 패킷이 오는지만 봅니다.

약속된 패킷이 들어오면 랜카드는 PCIe의 PME(Power Management Event) 신호를 올립니다. 이 신호가 칩셋과 ACPI를 거쳐 전원 회로까지 가면 전원 버튼을 누른 것처럼 부팅이 시작됩니다.

![깨우기 신호 경로: 매직 패킷이 오면 대기 전력으로 감시하던 랜카드가 MAC을 확인하고 PCIe PME 신호를 올려 ACPI와 전원 회로를 거쳐 부팅됨](/images/homeserver-wake-on-lan-wake-path.png)

그래서 WOL이 되려면 이 경로의 모든 단계가 깨우기를 허용해야 합니다. 랜카드가 매직 패킷을 감시하도록 켜져 있어야 하고 운영체제가 장치의 wakeup을 막지 않아야 하며 BIOS도 PCIe 장치가 전원을 켜는 것을 허용해야 합니다. 하나라도 빠지면 패킷은 도착해도 아무 일도 일어나지 않습니다.

## 매직 패킷의 구조

랜카드가 기다리는 "정해진 모양"이 매직 패킷입니다. 구조는 단순합니다. `FF`를 6바이트 보낸 뒤 깨울 PC의 MAC 주소를 16번 반복하면 끝입니다. 모두 102바이트입니다.

```text
FF FF FF FF FF FF                 ← 동기화 6바이트
AA BB CC DD EE FF                 ← 대상 MAC 주소
AA BB CC DD EE FF
... (총 16번 반복)
```

보통 UDP 9번이나 7번 포트로 보내지만 포트 번호에는 의미가 없습니다. 랜카드는 IP 헤더나 포트를 해석하지 않고 프레임 안에 이 바이트 패턴이 있는지만 봅니다. 꺼진 PC가 이해할 수 있는 건 자기 MAC 주소뿐이기 때문입니다.

```python
import socket

MAC = "AA:BB:CC:DD:EE:FF"
packet = b"\xff" * 6 + bytes.fromhex(MAC.replace(":", "")) * 16

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
s.sendto(packet, ("192.168.0.255", 9))   # 같은 LAN의 브로드캐스트 주소
```

## 왜 같은 LAN 안에서만 전달될까

꺼진 PC에는 IP 주소가 없습니다. 정확히는 IP를 갖고 응답할 운영체제가 없습니다. 보내는 쪽이 IP를 알아도 그 IP에 해당하는 MAC 주소를 물어보는 ARP 요청에 답할 대상이 없으니, 일반적인 유니캐스트로는 패킷을 보낼 길이 막힙니다. ARP 캐시에 예전 기록이 남아 있으면 우연히 전달되기도 하지만 믿을 방법은 아닙니다.

그래서 매직 패킷은 보통 브로드캐스트로 보냅니다. 같은 네트워크의 모든 장비에 뿌리면 대상 랜카드도 받게 되고 랜카드는 그 안에서 자기 MAC 주소를 찾아냅니다. 문제는 브로드캐스트가 공유기(라우터)를 넘지 못한다는 점입니다. 브로드캐스트는 L2(같은 스위치·공유기 아래) 범위에서만 퍼지고 라우터는 기본적으로 이를 다른 네트워크로 넘기지 않습니다. 결국 매직 패킷은 깨울 PC와 같은 LAN 안에서 출발해야 합니다.

## Tailscale로는 왜 못 깨울까

밖에서 집 PC에 접속할 때 Tailscale을 쓰고 있었기 때문에, 처음에는 Tailscale로 매직 패킷을 보내면 되지 않을까 생각했습니다. 하지만 두 가지 이유로 안 됩니다.

첫째, Tailscale은 각 기기에서 돌아가는 데몬이 만드는 L3(IP) 터널입니다. PC가 꺼지면 데몬도 꺼지니 그 PC로 가는 터널 자체가 없습니다. 둘째, 터널이 있다 해도 Tailscale은 IP 패킷을 실어 나를 뿐 집 LAN에 브로드캐스트를 뿌려 주지 않습니다.

해결책은 같은 LAN에 항상 켜져 있는 기기를 하나 두는 것입니다. 밖에서는 Tailscale로 그 기기에 접속하고, 그 기기가 LAN 안에서 매직 패킷을 브로드캐스트합니다. 저는 원래 24시간 켜 두던 N100 서버가 이 중계 역할을 맡았습니다.

![집 네트워크 구성: 외부 노트북은 Tailscale로 N100 중계기에 접속하고, N100이 Wi-Fi 공유기를 거쳐 새 PC 유선 랜카드로 매직 패킷을 보냄](/images/homeserver-wake-on-lan-infra.png)

N100은 Wi-Fi로 공유기에 붙어 있지만 상관없습니다. 매직 패킷을 보내는 쪽은 켜져 있으니 무선이든 유선이든 LAN에 브로드캐스트만 하면 되고 중요한 건 받는 쪽인 새 PC가 어떤 랜카드로 기다리느냐입니다. 이 차이는 아래 Wi-Fi 이야기에서 다시 나옵니다.

실제로 꺼 둔 새 PC를 밖에서 이 경로로 깨워 보니, 매직 패킷을 보내고 약 8초 뒤 부팅이 시작됐고 10초 남짓이면 SSH로 접속할 수 있었습니다.

## 유선은 되고 Wi-Fi는 잘 안 되는 이유

새 PC에는 유선과 무선 랜카드가 모두 있었고 처음에는 집에 쓸 만한 랜선이 없어서 Wi-Fi로 WOL을 시도했습니다. Wi-Fi 버전의 WOL을 WoWLAN(Wake-on-Wireless-LAN)이라고 부르는데, 결론부터 말하면 서버를 원격으로 켜는 용도로는 믿기 어렵습니다.

이유는 PC의 전원 상태에 있습니다. ACPI는 전원 상태를 S0(동작)부터 S5(완전 종료)까지 나누는데, WOL이 어디서 되는지가 유선과 무선에서 다릅니다.

| 상태 | 의미 | 유선 WOL | Wi-Fi WOL |
|---|---|---|---|
| S0ix / s2idle | 얕은 절전 | 가능 | 이론상 가능 |
| S3 | RAM만 전원 유지하는 절전 | 가능 | 이론상 가능 |
| S4 | 최대 절전(디스크에 저장 후 꺼짐) | BIOS 지원 시 가능 | 불가 |
| S5 | 완전 종료 | BIOS 지원 시 가능 | 불가 |

유선 랜카드는 케이블로 공유기와 물리적으로 연결돼 있으니 대기 전력만 있으면 패킷을 받을 수 있습니다. Wi-Fi는 다릅니다. 무선은 AP(공유기)와 연결을 맺고 유지해야 패킷을 받을 수 있는데, 완전히 꺼지면 이 연결이 끊깁니다. 그래서 WoWLAN은 절전 상태에서 칩 펌웨어가 AP 연결을 유지할 때만 동작합니다.

절전 상태에서도 안정적이지 않습니다. 칩 펌웨어와 드라이버, 플랫폼 조합에 따라 실제 동작 여부가 크게 갈리고 AP가 절전 중인 기기에 브로드캐스트를 늦게 주거나 버리기도 합니다. 제 PC도 운영체제 쪽 설정을 모두 켜고 같은 LAN에서 패킷을 반복해서 보냈지만 Wi-Fi 칩에서 깨우기 신호가 한 번도 올라오지 않았습니다. 결국 Wi-Fi는 포기하고 다이소에서 3,000원짜리 랜선을 사 와 연결했는데, 유선으로는 완전 종료(S5) 상태에서도 바로 켜졌습니다.

## 실제로 설정한 것

원리를 알고 나면 설정은 앞에서 말한 깨우기 경로를 하나씩 여는 일입니다. 새 PC는 Ubuntu Server이고 유선 랜카드 이름은 `enp2s0`입니다.

### 랜카드가 매직 패킷을 감시하게 하기

먼저 랜카드가 WOL을 지원하는지, 지금 켜져 있는지 `ethtool`로 확인합니다.

```bash
$ sudo ethtool enp2s0 | grep Wake-on
	Supports Wake-on: pumbg
	Wake-on: d
```

`Supports Wake-on`은 하드웨어가 지원하는 방식이고, `Wake-on`은 현재 설정입니다. `d`는 꺼짐(disabled), `g`는 매직 패킷으로 깨우기입니다. 나머지 `p`·`u`·`m`·`b`는 링크 변화나 유니캐스트·멀티캐스트·브로드캐스트 패킷 아무거나로 깨우는 방식인데, 의도치 않게 켜질 수 있어서 보통은 `g`만 씁니다.

```bash
sudo ethtool -s enp2s0 wol g
```

이 설정은 재부팅하면 다시 `d`로 돌아가는 경우가 많습니다. 그래서 부팅할 때마다 설정하도록 systemd 서비스를 만들었습니다. 운영체제가 이 장치의 깨우기를 막지 않도록 sysfs의 `power/wakeup`도 같이 켭니다.

<details>
<summary><code>wol.service</code> 전체 보기</summary>

```ini title="/etc/systemd/system/wol.service"
[Unit]
Description=Enable Wake-on-LAN (magic packet) on enp2s0
After=sys-subsystem-net-devices-enp2s0.device
BindsTo=sys-subsystem-net-devices-enp2s0.device

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s enp2s0 wol g
ExecStart=/bin/sh -c "echo enabled > /sys/class/net/enp2s0/device/power/wakeup"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

</details>

```bash
sudo systemctl enable --now wol.service
```

### BIOS에서 허용하기

완전 종료(S5) 상태에서는 운영체제가 없으니 BIOS 설정이 전부입니다. 항목 이름은 제조사마다 다르지만 대개 `Wake on LAN`, `Power On By PCI-E`, `PME Event Wake Up` 같은 이름입니다. 이걸 켜고 `ErP`나 `Deep Sleep` 같은 절전 규격 항목은 꺼야 합니다. 이 항목이 켜져 있으면 종료 상태에서 랜카드로 가는 대기 전력까지 끊어 버립니다.

### 유선 랜카드에 IP 주기 (netplan)

Wi-Fi만 쓰던 PC에 랜선을 꽂고 나서 엉뚱한 문제가 하나 생겼습니다. 부팅이 2분씩 걸리고 그동안 SSH로 접속하면 이런 메시지와 함께 거부됐습니다.

```text
System is booting up. Unprivileged users are not permitted to log in yet.
```

원인은 netplan 설정이었습니다. 유선 랜카드 항목에 IP 설정이 없으니 `systemd-networkd-wait-online`이 이 인터페이스가 준비되기를 기다리다 2분 뒤 타임아웃됐고 부팅이 끝나지 않은 동안에는 `/run/nologin` 파일이 있어 일반 사용자 로그인이 막힌 것입니다. `dhcp4: true`로 IP를 받게 하고 `optional: true`로 이 인터페이스 때문에 부팅을 기다리지 않게 하면 해결됩니다.

기존 netplan 파일의 `ethernets` 아래 유선 랜카드 항목에 `dhcp4`와 `optional` 두 줄만 추가하면 됩니다. Wi-Fi 설정처럼 이미 있던 항목은 그대로 둡니다.

```yaml title="/etc/netplan/00-installer-config.yaml" {5-6}
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: true
      optional: true
```

SSH로 접속한 상태에서 네트워크 설정을 바꾸는 것이니, 적용은 `sudo netplan try`로 하는 편이 안전합니다. 설정이 잘못돼 접속이 끊겨도 확인 입력이 없으면 일정 시간 뒤 이전 설정으로 되돌아갑니다.

### 켜졌는지 확인하기

깨운 뒤 켜졌는지 보려고 `ping`을 쓰면 헷갈릴 수 있습니다. 제 환경에서는 PC가 켜져 있어도 ping에 응답하지 않았습니다. 원인까지 확인하지는 않았지만, 방화벽에서 ICMP를 막아 두면 이렇게 됩니다. 이럴 때는 SSH 포트가 열렸는지 보는 편이 확실합니다. 아래 `192.168.0.10`은 새 PC의 LAN 주소 자리에 넣은 예시값입니다.

```bash
nc -z -w3 192.168.0.10 22 && echo "켜짐"
```

## 정리

WOL은 꺼진 PC의 랜카드가 대기 전력으로 매직 패킷을 기다리다가, 자기 MAC 주소를 발견하면 PME 신호로 전원을 켜는 구조입니다. 랜카드, 운영체제, BIOS 중 한 곳이라도 깨우기를 막으면 패킷이 도착해도 켜지지 않으니, 안 될 때는 이 경로를 차례로 확인하면 됩니다.

꺼진 PC가 알아듣는 건 MAC 주소뿐이라 패킷은 같은 LAN 안에서 브로드캐스트로 출발해야 합니다. 밖에서 켜고 싶다면 VPN만으로는 부족하고 집 안에 항상 켜져 있는 기기가 하나 있어야 합니다. 저는 원래 있던 N100 서버가 그 자리를 맡아 준 덕분에 새 PC는 필요할 때만 켜서 쓰고 있습니다.

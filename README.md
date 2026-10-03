# AWS 하이브리드 멀티리전 아키텍처

서울·싱가포르 두 리전과 **온프레미스를 가정한 별도 VPC(IDC)** 를 Site-to-Site VPN으로 잇고,
환경 전체를 CloudFormation으로 코드화한 개인 프로젝트입니다.

클라우드 교육 과정에서 학습을 목적으로 시작해 혼자 설계하고 구축했습니다.
네트워크·DNS·VPN·데이터베이스를 하나씩 따로 다루는 것과, 그것들이 한 환경에서 맞물려 돌아가게 만드는 것은
다른 일이라고 봤고, 배운 것을 한곳에 모아 실제로 동작하는 상태까지 가보는 것이 목적이었습니다.

- 기간 2026.03.19 ~ 03.24 · 1인
- 📄 [상세 자료 22p](https://jinuuukim.github.io/portfolio/pdf/AWS-Personal.pdf) · 🔗 [프로젝트 페이지](https://jinuuukim.github.io/portfolio/personal/)

---

## 요구사항 → 설계 대응

| 요구사항 | 설계 대응 |
| --- | --- |
| **고가용성** — 각 리전에 웹서버를 둔다 | ALB + 웹서버 이중화 |
| **보안 통신망** — 리전·IDC 간 프라이빗 통신, 사설 도메인 운영 | Site-to-Site VPN · Transit Gateway · Route 53 |
| **트래픽 제어** — 점검 시 웹서버 제어, IDC 장애 시 다른 리전 전환 | Global Accelerator |

## 구성

| 영역 | 대역 | 구성 |
| --- | --- | --- |
| 서울 VPC | `10.1.0.0/16` | ALB · 웹서버 2대 · NAT Instance 2대(AZ 분리) |
| 싱가포르 VPC | `10.3.0.0/16` | 웹서버 1대 · NAT Instance 1대 |
| 서울 IDC | `10.2.0.0/16` | MySQL MASTER · bind9 · Customer Gateway |
| 싱가포르 IDC | `10.4.0.0/16` | MySQL SLAVE · bind9 · Customer Gateway |

리전 간은 Transit Gateway 피어링, 각 리전과 IDC는 Site-to-Site VPN으로 연결됩니다.
IDC는 실제 데이터센터 대신 **별도 VPC를 온프레미스로 간주**해 구성했고, AWS 쪽과는 VPN 터널로만 통신합니다.

---

## 설계에서 판단한 것

### 1. 환경 전체를 코드로

콘솔로 만들면 과제는 끝나지만 다시 세울 수 없습니다.
AWS VPC와 IDC VPC, 서브넷·라우팅·보안 그룹, ALB, Route 53 호스팅 영역과 Resolver 규칙,
Transit Gateway, Customer Gateway와 VPN 연결까지 리전별 템플릿 하나에 담았습니다.

두 리전이 완전히 같지는 않아(서울은 ALB와 웹서버 2대, 싱가포르는 웹서버 1대) 템플릿도 리전별로 따로 작성했습니다.
같은 파일을 두 번 돌리는 구조가 아니라, **각 리전을 코드로 다시 세울 수 있게 하는 것**이 목표였습니다.

> Transit Gateway 피어링과 Global Accelerator는 두 리전·두 스택에 걸쳐 있어 콘솔에서 연결했습니다.

### 2. 하이브리드 DNS — 나누되 서로 찾게 한다

내부 도메인이 네 개입니다. 환경 성격이 다르니 DNS도 다르게 가져갔습니다.

| 영역 | 도메인 | DNS |
| --- | --- | --- |
| AWS 리전 | `awsseoul.internal` · `awssingapore.internal` | Route 53 프라이빗 호스팅 영역 |
| IDC | `idcseoul.internal` · `idcsingapore.internal` | bind9 |

문제는 **나누기만 하면 서로를 못 찾는다**는 점이었습니다. 그래서 양쪽에 길을 냈습니다.

| 방향 | 방법 |
| --- | --- |
| AWS → IDC · 상대 리전 | **Route 53 Resolver 아웃바운드 엔드포인트** + 도메인별 포워딩 규칙 3개 |
| IDC → AWS · 상대 IDC | **bind9 `forward only` zone** 3개 → 상대 Resolver 인바운드 엔드포인트 / 상대 bind9 |
| IDC 내부 질의 | **DHCP Options Set** 으로 해당 VPC의 DNS 서버를 bind9로 지정 |

### 3. NAT Gateway 대신 NAT Instance

관리형 게이트웨이는 시간 요금에 데이터 처리 요금이 따로 붙는 두 갈래 구조이고, 리전이 둘이라 NAT도 둘 이상 필요했습니다.
비용을 고려해 인스턴스로 올렸고, 출발지·대상 확인을 끄고 IP 포워딩과 `iptables MASQUERADE` 규칙이 부팅 시 적용되도록 구성했습니다.

> 다만 **운영 환경이었다면 게이트웨이를 썼을 것**입니다. 아끼는 비용보다 장애 위험과 관리 부담이 큽니다.

### 4. 리전 전환은 DB가 아니라 트래픽 계층에서

IDC DB에 장애가 나면 그 리전의 웹서버는 데이터를 못 읽습니다. 그런데 웹서버 자체는 멀쩡하므로 헬스체크를 통과해 버립니다.
**정상으로 보이지만 쓸 수 없는 상태**가 가장 곤란합니다.

그래서 웹서버에서 1분 주기 크론으로 IDC DB에 ping을 보내고, 닿지 않으면 `systemctl`로 웹 데몬을 내립니다.
ALB 대상그룹이 Unhealthy가 되면 Global Accelerator가 반대 리전으로 트래픽을 보냅니다.
DB가 돌아오면 크론이 웹 데몬을 다시 올려 원래대로 복귀합니다.

> ⚠️ **이건 DB 페일오버가 아닙니다.** SLAVE를 MASTER로 승격시키는 로직은 넣지 않았습니다.
> 층위가 데이터베이스가 아니라 **트래픽 라우팅** 입니다 —
> "DB에 닿지 않는 리전으로는 사용자를 보내지 않는다"까지가 이 구성이 하는 일입니다.

---

## 검증

| 확인한 것 | 방법 · 결과 |
| --- | --- |
| VPN 터널 | 양 리전 터널 4개 수립 확인 |
| 내부 통신 | `nslookup` · `ping` 으로 리전 ↔ IDC 간 내부 도메인 조회 및 도달 확인 |
| DB 복제 | `SHOW REPLICA STATUS` IO/SQL 모두 Running, 한쪽 수정이 양쪽에 반영 |
| GA 라우팅 | 접속 위치를 바꿔가며 지연 기반 분배 결과 확인 |
| **장애 전환** | IDC DB 중지 → 웹서버 Unhealthy → 반대 리전 전환 → DB 복구 → 원복까지 **양방향** |

### 남은 한계

- `ping`은 **호스트 도달만** 봅니다. 서버는 살아 있는데 MySQL 프로세스만 죽은 경우는 감지하지 못합니다. 포트 확인이나 쿼리 한 번으로 바꾸는 편이 맞습니다.
- 크론 주기가 1분이라 **감지까지 시간이 걸립니다.** 실제 운영이라면 이 지연이 그대로 장애 시간이 됩니다.

---

## 레포 구성

```
stacks/
  AWS_Project_Seoul.yml       서울 리전 + 서울 IDC
  AWS_Project_Singapore.yml   싱가포르 리전 + 싱가포르 IDC
scripts/
  pingcheck_cron.txt          DB 연결 확인 → 웹 데몬 제어 (1분 크론)
```

### 스택 파라미터

| 파라미터 | 설명 |
| --- | --- |
| `RootPassword` | EC2 UserData에 주입되는 root 비밀번호 (`NoEcho`) |
| `VpnPreSharedKey` | Site-to-Site VPN 터널 사전공유키 (`NoEcho`) |
| `KeyName` | 기존 EC2 키 페어 이름 |

> 🔑 **시크릿은 템플릿에 두지 않았습니다.** 스택 생성 시 입력받습니다.

### ⚠️ 포함하지 않은 것

- **VPN 터널 설정** — 템플릿에 `AWS::EC2::VPNConnection` 리소스는 있지만, Customer Gateway 인스턴스의 IPsec 설정은 포함하지 않았습니다. **템플릿만으로 전체가 재현되지는 않습니다.**
- 웹 애플리케이션·DB 스키마

---

## 기술 스택

| 영역 | 기술 |
| --- | --- |
| IaC | CloudFormation |
| 네트워크 | VPC · 서브넷 · 라우팅 · 보안 그룹 · ALB · Transit Gateway · Site-to-Site VPN · Customer Gateway · Global Accelerator |
| DNS | Route 53 프라이빗 호스팅 영역 · Route 53 Resolver(인바운드·아웃바운드 · 포워딩 규칙) · bind9 · DHCP Options |
| 컴퓨트 | EC2 · NAT Instance (IP 포워딩 · iptables MASQUERADE) |
| 데이터 | MySQL Master-Slave 복제 (IDC 간) |
| 운영 | Linux · systemd · crontab · Apache httpd |

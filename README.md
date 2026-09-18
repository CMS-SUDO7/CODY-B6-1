# 내가 만든 웹사이트를 인터넷에 올려 누구나 쓰게 하기  

## 1. 목표와 결과  

직접 구성한 VPC의 퍼블릭 서브넷에 Ubuntu EC2를 배치하고 Nginx를 설치했다. HTTP는 모든 IPv4에서 접근하도록, SSH는 실습자 공인 IP 하나에서만 접근하도록 설정했다. Windows PowerShell에서 SSH 접속 후 서버 내부 및 외부 통신을 확인했다.  

외부 접속 검증은 **A 방식: 브라우저 접속**을 선택했다.  

- 검증 URL: **http://3.38.108.17**  
- 검증 결과: **Welcome to nginx!** 페이지 표시  
- 검증 시점의 주소이며, 인스턴스 중지·시작 시 바뀔 수 있고 삭제 후에는 접속할 수 없다.  

![외부 브라우저 접속 증거](docs/screenshots/10-external-http.png)  

## 2. 제출 파일  

| 파일 | 내용 |  
|---|---|
| README.md | 환경, 구축 절차, 검증 결과 및 증거 |  
| docs/architecture.png | 네트워크 구성 및 요청 흐름 |  
| docs/troubleshooting.md | SSH 개인 키 권한 오류 분석·해결 |  
| docs/cleanup-checklist.md | 실습 자원 목록, 삭제 순서, 확인란 |  
| docs/screenshots/ | 실습자가 제공한 원본 증거 화면 26개 |  

## 3. 구성도  

![AWS 실습 구성도](docs/architecture.png)  

인터넷 게이트웨이는 VPC에 연결되고, 서브넷은 라우팅 테이블을 참조한다. 보안 그룹은 EC2의 네트워크 인터페이스에 적용된다. 그림의 라우팅 표는 통신 중간에 놓이는 서버가 아니라 경로 선택 규칙이다.  

## 4. 실습 환경과 자원  

| 항목 | 실제 구성 |  
|---|---|
| 작업 PC | Windows 노트북, PowerShell/OpenSSH |  
| AWS 계정 플랜 | Free plan 가입 완료(실습자 진술) |  
| 실습 사용자 | codyssey-student |  
| 리전 / 가용 영역 | ap-northeast-2 / ap-northeast-2a |  
| VPC | codyssey-vpc / vpc-0e808b35dd5e2de5f / 10.0.0.0/16 |  
| 서브넷 | codyssey-public-subnet / subnet-0e66198c96f9834df / 10.0.1.0/24 |  
| 인터넷 게이트웨이 | codyssey-igw / igw-0713ed7b6c2dadf71 / Attached |  
| 라우팅 테이블 | codyssey-public-rt / rtb-055eafc503f43cf59 |  
| 보안 그룹 | codyssey-web-sg / sg-09e767b31257706d3 |  
| EC2 | codyssey-web-server / i-0969f9283bfd25094 / t3.micro |  
| 서버 OS | Ubuntu 24.04.4 LTS / x86_64(SSH 출력 확인) |  
| 퍼블릭 IPv4 | 3.38.108.17 |  
| 프라이빗 IPv4 | 10.0.1.165 |  
| 키페어 이름 | codyssey-key(개인 키 파일은 제출물에서 제외) |  
| Nginx | 1.24.0 Ubuntu(HTTP 응답 헤더 확인) |  
| 스토리지 | 8GiB gp3 / vol-04ff34996b7e2d954 / 연결됨·사용 중 확인 |  

### 라우팅과 보안 규칙  

| 구분 | 목적지 또는 소스 | 대상 또는 포트 |  
|---|---|---|
| VPC 내부 경로 | 10.0.0.0/16 | local |  
| 인터넷 경로 | 0.0.0.0/0 | igw-0713ed7b6c2dadf71 |  
| 인바운드 HTTP | 0.0.0.0/0 | TCP 80 |  
| 인바운드 SSH | 121.135.181.35/32(검증 당시) | TCP 22 |  
| 아웃바운드 | 0.0.0.0/0 / 모든 트래픽을 유지하도록 안내 | 전체 트래픽 허용 화면 및 외부 HTTPS 응답 확인 |  

퍼블릭 서브넷에는 인터넷 게이트웨이로 향하는 기본 경로가 필요하며, 이번 EC2의 IPv4 인터넷 통신에는 퍼블릭 IPv4도 필요하다. 보안 그룹의 HTTP 허용만으로 라우팅과 IP 설정을 대신할 수 없다.  

### IAM 권한  

화면에서 연결을 확인한 정책은 `CodysseyCloudLabSeoul`과 `IAMUserChangePassword`다. 루트는 초기 사용자·권한 준비에 사용하고, 이후 자원 생성은 codyssey-student로 진행했다.  

안내한 인라인 정책은 `aws:RequestedRegion = ap-northeast-2` 조건 아래 EC2 조회와 이번 VPC·서브넷·게이트웨이·라우팅·보안 그룹·키페어·인스턴스 생성/삭제 및 태그 작업을 허용한다. S3·RDS 관리나 AdministratorAccess는 부여하지 않았다. `IAMUserChangePassword`는 본인 비밀번호 변경에 사용한다.  

## 5. 구축 순서  

1. AWS 계정 가입 및 루트 MFA 설정 후 실습용 IAM 사용자와 정책을 준비했다.  
2. 서울 리전에서 VPC(10.0.0.0/16)와 서브넷(10.0.1.0/24)을 생성했다.  
3. 인터넷 게이트웨이를 VPC에 연결했다.  
4. 라우팅 테이블에 `0.0.0.0/0 → 인터넷 게이트웨이`를 추가하고 서브넷을 명시적으로 연결했다.  
5. HTTP 80 전체 허용, SSH 22 실습자 IP만 허용하는 보안 그룹을 생성했다.  
6. Ubuntu EC2 한 대를 생성하고 해당 서브넷·보안 그룹·키페어를 지정했다. 퍼블릭 IPv4 자동 할당을 활성화했다.  
7. Windows에서 SSH로 접속하고 Nginx를 설치·실행했다.  
8. 실행 상태, 내부 HTTP, 외부 HTTPS, 브라우저 HTTP 접속을 검증했다.  

### Windows에서 SSH 접속  

```powershell
cd "$env:USERPROFILE\Downloads"  
ssh -i ".\codyssey-key.pem" ubuntu@3.38.108.17  
```

개인 키 권한 오류가 발생하여 [트러블슈팅 보고서](docs/troubleshooting.md)의 절차로 해결했다.

### Ubuntu에서 Nginx 설치 및 검사  

```bash
sudo apt update  
sudo apt install -y nginx  
sudo systemctl enable --now nginx  
systemctl is-active nginx  
curl -I http://localhost  
curl -I https://example.com  
```

`sudo`는 패키지 설치와 서비스 설정 등 관리자 권한이 필요한 명령에 사용했다. `curl -I`는 본문 대신 응답 헤더를 확인하는 HEAD 요청이다. 외부 접속 제출 방식은 GET /health가 아닌 브라우저 A 방식이다.  

## 6. 검증 결과  

| 검증 | 실제 관찰 결과 | 판정 |  
|---|---|---|
| SSH 사용자 | whoami → ubuntu | 통과 |  
| Nginx 서비스 | active | 통과 |  
| 서버 내부 HTTP | HTTP/1.1 200 OK | 통과 |  
| 서버에서 외부 HTTPS | HTTP/2 200 | 통과 |  
| 외부 브라우저 HTTP | Welcome to nginx! | 통과 |  
| 생성 직후 EC2 상태 검사 | 제공 화면은 초기화 / 2/3 통과 시점 | 3/3 결과 미수집 |  
| 필수 5종 자원 정리 | EC2 종료, EBS 없음, EIP 미할당, IGW·VPC 삭제 | 확인 완료 |  

![Nginx 실행 상태](docs/screenshots/11-nginx-active.png)  

![서버 내부 HTTP 응답](docs/screenshots/12-local-http.png)  

![외부 HTTPS 응답](docs/screenshots/13-outbound-https.png)  

### 구성 증거  

- [VPC](docs/screenshots/01-vpc.png), [서브넷](docs/screenshots/02-subnet.png), [게이트웨이 연결](docs/screenshots/03-internet-gateway.png)  
- [라우팅](docs/screenshots/04-routes.png), [서브넷 연결](docs/screenshots/05-subnet-association.png)  
- [인바운드 보안 규칙](docs/screenshots/06-security-group.png), [IAM 정책 목록](docs/screenshots/07-iam-policies.png)  
- [EC2 정보](docs/screenshots/08-instance.png), [SSH 연결 설정](docs/screenshots/09-ssh-settings.png)  


### 추가 확인: 스토리지와 아웃바운드  

루트 EBS는 `/dev/sda1`, `vol-04ff34996b7e2d954`, 8GiB gp3이며 인스턴스에 연결되어 사용 중이다. 볼륨 상세 화면의 상태 검사는 정상이며, 이것은 EC2 전체 상태 검사 3/3 통과 여부와 별개다. IOPS는 3000이다. 아웃바운드는 IPv4 모든 트래픽을 `0.0.0.0/0`으로 허용한다. 이후 `종료 시 삭제=true` 필터에 해당 볼륨이 조회되는 화면으로 자동 삭제 설정을 확인했으며, 종료 후 전체 볼륨 목록이 비어 있는 것도 확인했다.  

![EC2 루트 볼륨](docs/screenshots/14-instance-storage.png)  

![EBS 상세](docs/screenshots/15-ebs-details.png)  

![아웃바운드 규칙](docs/screenshots/16-outbound-rules.png)  

## 7. 정리 완료 증거  

실습 EC2와 네트워크는 삭제되어 기존 URL은 더 이상 이 실습 웹 서버의 접속 주소로 사용할 수 없다. 같은 IP가 이후 다른 대상에 재할당될 수 있으므로 결과 재검증을 위해 접속하지 않는다.  

| 자원 | 확인 결과 | 증거 |  
|---|---|---|
| EC2 | 대상 인스턴스 종료됨 | [종료 화면](docs/screenshots/20-ec2-terminated.png) |  
| EBS | 필터 없는 목록에 볼륨 없음 | [볼륨 목록](docs/screenshots/21-ebs-empty.png) |  
| EIP | 필터 없는 목록에 할당 없음 | [EIP 목록](docs/screenshots/19-eip-empty.png) |  
| IGW | 실습 ID 삭제 성공 | [삭제 알림](docs/screenshots/23-igw-deleted.png) |  
| VPC | 실습 ID와 이름 삭제 성공 | [삭제 알림](docs/screenshots/24-vpc-deleted.png) |  

[SSH 접속·서비스 통합 증거](docs/screenshots/18-ssh-verification.png), [종료 시 삭제 설정](docs/screenshots/17-delete-on-termination.png)도 포함했다.  

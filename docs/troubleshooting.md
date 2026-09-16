# 트러블슈팅 보고서: Windows SSH 개인 키 권한 오류

- 발생일: 2026-09-15
- 환경: Windows PowerShell → Ubuntu 24.04.4 LTS EC2
- 대상: ubuntu@3.38.108.17
- 근거: 실습자가 제공한 실제 PowerShell 오류·명령 실행 결과·SSH 로그인 출력. 원본 오류 스크린샷은 없으며 아래는 대화에 남긴 텍스트 기록이다.

## 1. 증상

```powershell
ssh -i ".\codyssey-key.pem" ubuntu@3.38.108.17
```

최초 서버 호스트 키 확인 후 접속을 시도했지만 다음 오류로 실패했다.

```text
Bad permissions. Try removing permissions for user:
DESKTOP-K462GM3\CodexSandboxUsers
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions for '.\codyssey-key.pem' are too open.
This private key will be ignored.
Load key ".\codyssey-key.pem": bad permissions
ubuntu@3.38.108.17: Permission denied (publickey).
```

이때 `whoami`는 `desktop-k462gm3\chae`였다. SSH 접속이 실패했으므로 Windows 사용자 이름이 표시된 것이다.

## 2. 원인 가설

OpenSSH가 키를 읽기 전에 Windows 파일 ACL을 확인하고, 다른 사용자 그룹의 접근 권한 때문에 개인 키 사용을 거부했다고 가설을 세웠다. `Permission denied (publickey)`는 이 경우 개인 키가 무시된 뒤 발생한 최종 증상이다.

## 3. 검증

오류에 특정 Windows 그룹과 `private key will be ignored`가 명시되어 있다. 또한 서버의 ED25519 호스트 키 확인 단계까지 도달했으므로 해당 시도의 SSH 네트워크 연결 자체는 성립했다. 따라서 보안 그룹의 SSH 포트를 더 넓게 개방하지 않고 로컬 개인 키 파일의 권한을 점검·수정했다.

수정 전 ACL 전체 출력은 보관하지 않았다. 상속 권한인지 명시 권한인지의 정확한 구성은 확정하지 않으며, OpenSSH가 해당 그룹의 접근을 문제로 판정했다는 사실을 근거로 한다.

## 4. 조치

Windows PowerShell의 개인 키 파일이 있는 폴더에서 아래 명령을 순서대로 실행하도록 안내하고 권한을 수정했다.

```powershell
icacls ".\codyssey-key.pem" /grant:r "$(whoami):(R)"
icacls ".\codyssey-key.pem" /inheritance:r
icacls ".\codyssey-key.pem" /remove "DESKTOP-K462GM3\CodexSandboxUsers"
```

| 명령 옵션 | 목적 |
|---|---|
| /grant:r | 현재 사용자에게 명시적인 읽기 권한 부여 |
| /inheritance:r | 상위 폴더에서 상속받은 권한 제거 |
| /remove | 오류에 표시된 그룹의 권한 제거 |

현재 사용자의 권한을 먼저 확보한 뒤 상속을 제거했다. 수정 대상은 개인 키 파일 하나이며 다운로드 폴더 전체가 아니다. 다른 PC에서는 사용자·그룹 이름과 실제 ACL을 확인하여 적용해야 한다.

재접속 명령:

```powershell
ssh -i ".\codyssey-key.pem" ubuntu@3.38.108.17
```

## 5. 결과

실습자가 제공한 출력에서 파일 처리 성공과 SSH 로그인을 확인했다.

```text
1 파일을 처리했으며 0 파일은 처리하지 못했습니다.
Welcome to Ubuntu 24.04.4 LTS
ubuntu@ip-10-0-1-165:~$ whoami
ubuntu
```

ACL 수정 이후 동일한 키와 IP로 접속에 성공했다. 따라서 이번 실패 원인이 로컬 개인 키 접근 권한과 관련되어 있었다는 가설을 뒷받침한다. AWS의 보안 그룹·라우팅·키페어를 재생성할 필요가 없었다.

## 6. 재발 방지

- 개인 키는 본인만 접근 가능한 위치에 보관하고 공유 폴더에 두지 않는다.
- 파일을 복사·이동하거나 계정을 바꾼 뒤에는 `icacls ".\codyssey-key.pem"`으로 ACL을 확인한다.
- 키 내용을 채팅·Git·README에 포함하지 않는다.
- 접속 실패 시 timeout, bad permissions, publickey 오류를 구분한다.
- SSH 허용 범위는 내 공인 IP/32를 유지한다. 키 권한 오류를 해결하려고 SSH를 전체 IP에 개방하지 않는다.
- 접속 성공은 프롬프트와 `whoami` 결과로 확인한다.

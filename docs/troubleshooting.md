# 트러블슈팅 보고서 - codyssey-aws-web-deploy (B3-1)

## 이슈 1: t2.micro가 프리 티어 대상이 아니라는 오류

| 항목 | 내용 |
| --- | --- |
| 증상(문제 상황) | `run-instances`로 `t2.micro` 인스턴스를 생성하려 하자 `InvalidParameterCombination: The specified instance type is not eligible for Free Tier` 오류가 발생하며 인스턴스 생성이 즉시 실패했다. |
| 원인 가설 | 이 AWS 계정/리전 조합에서는 `t2.micro`가 아니라 `t3.micro`(또는 `t4g.micro`)가 프리 티어 대상으로 지정되어 있을 것이다. |
| 검증 방법 | `aws ec2 describe-instance-types --filters "Name=free-tier-eligible,Values=true"`로 이 계정에서 실제 프리 티어 대상 인스턴스 타입 목록을 조회했다. |
| 조치 내용 | 조회 결과 `t3.micro`, `t3.small`, `t4g.micro`, `t4g.small` 등이 프리 티어 대상으로 확인되어, 인스턴스 타입을 `t2.micro` → `t3.micro`로 변경해 재실행했다(미션 요구사항도 t2.micro/t3.micro 중 택1을 허용). |
| 결과 | `run-instances`가 정상적으로 성공했고, 이후 인스턴스가 정상 기동해 `instance-status-ok`까지 통과했다. |
| 재발 방지 | 새 계정/리전에서 인스턴스를 생성하기 전에는 `describe-instance-types --filters Name=free-tier-eligible,Values=true`로 실제 프리 티어 대상 타입을 먼저 확인하는 것을 체크리스트에 추가한다. 프리 티어 대상 여부는 계정 가입 시점/리전에 따라 달라질 수 있어, 문서상의 "t2.micro가 항상 프리 티어"라는 가정은 위험하다. |

## 이슈 2: 루트 계정으로 자격증명이 등록되어 있었음

| 항목 | 내용 |
| --- | --- |
| 증상(문제 상황) | 자격증명 구성 직후 `aws sts get-caller-identity`를 실행하자 `Arn`이 `arn:aws:iam::<계정>:root`로 나와, 사용자가 등록한 자격증명이 IAM 사용자가 아니라 루트 계정의 액세스 키였다. |
| 원인 가설 | `aws configure`로 루트 계정의 액세스 키를 그대로 등록한 것으로 판단했다. |
| 검증 방법 | `sts get-caller-identity`의 `Arn` 필드가 `:root`로 끝나는지 확인 — IAM 사용자라면 `:user/<이름>` 형태여야 한다. |
| 조치 내용 | 루트 자격증명은 EC2/VPC 권한만 가진 IAM 사용자(`codyssey-mission2`, `AmazonEC2FullAccess`만 부착, `AdministratorAccess` 미부여)를 1회 생성하는 부트스트랩 용도로만 사용하고, 즉시 해당 IAM 사용자의 액세스 키를 발급해 별도 CLI 프로파일(`codyssey-mission2`)로 전환했다. 이후 모든 VPC/EC2/보안그룹 작업은 이 프로파일로만 수행했다. |
| 결과 | `sts get-caller-identity --profile codyssey-mission2` 결과 `Arn`이 `:user/codyssey-mission2`로 확인되어, 이후 작업이 루트가 아닌 IAM 사용자로 수행됨을 검증했다. |
| 재발 방지 | 앞으로 새 AWS 실습 환경에서는 `aws configure` 직후 항상 `sts get-caller-identity`로 `Arn`을 먼저 확인하고, `:root`이면 즉시 최소권한 IAM 사용자를 만들어 전환하는 것을 표준 절차로 삼는다. |

> 2차 실습에서 이 사용자의 정책을 `AmazonEC2FullAccess`에서 커스텀 정책 `codyssey-ec2-lab`
> (실제 호출한 EC2 API만 허용, 서울 리전 한정)으로 교체했다.
> 정책 원문: [`iam/codyssey-ec2-lab-policy.json`](../iam/codyssey-ec2-lab-policy.json),
> 거부 확인 로그: [`screenshots/iam_least_privilege_check.txt`](../screenshots/iam_least_privilege_check.txt)
> (S3·IAM·RDS 호출과 us-east-1 리전 EC2 호출이 모두 거부됨)

## 이슈 3: 서버는 정상인데 외부에서 접속이 안 됨 (timeout)

2차 실습(2026-11-15)에서 미션 소개의 상황("서버에 올렸는데 외부에서 안 들어와요")을 의도적으로
재현해 분석했다. 원문 로그: [`screenshots/troubleshooting_sg80_transcript.txt`](../screenshots/troubleshooting_sg80_transcript.txt)

| 항목 | 내용 |
| --- | --- |
| 증상(문제 상황) | 직전까지 200을 반환하던 `curl http://13.125.53.24/health`가 응답 없이 `curl: (28) Connection timed out after 5001 milliseconds`로 실패했다. |
| 원인 가설 | ① Nginx 프로세스 중단, ② 라우팅/IGW 문제, ③ Security Group이 80 포트를 차단. 증상이 `refused`가 아니라 **timeout**이므로 패킷이 서버에 닿기 전에 버려지는 ③ 또는 ②일 가능성이 높다. 프로세스만 죽었다면 OS가 RST를 보내 즉시 `Connection refused`가 났을 것이다. |
| 검증 방법 | **A. 서버 내부**: 22 포트로 SSH 접속해 `systemctl is-active nginx` → `active`, `curl localhost/health` → `OK`. Nginx `access.log`에는 마지막 외부 요청이 장애 직전(08:27:26)까지만 있고, timeout된 요청은 **기록되지 않음** = 요청이 Nginx까지 도달하지 못함. SSH가 된다는 것은 라우팅·IGW·퍼블릭 IP도 정상이라는 뜻이므로 ①②는 배제.<br>**B. SG 규칙**: `describe-security-groups` 결과 인바운드가 `tcp 22 <내 IP>/32` 하나뿐이고 80 규칙이 없음.<br>**C. 대조 실험**: 프로세스가 없는 8080 포트를 내 IP에만 잠깐 열고 요청하니 즉시 `Connection refused` → "SG 차단 = timeout, 프로세스 없음 = refused"를 직접 확인한 뒤 8080 규칙은 바로 제거. |
| 조치 내용 | `aws ec2 authorize-security-group-ingress --group-id <SG> --protocol tcp --port 80 --cidr 0.0.0.0/0`로 HTTP 인바운드 규칙 복구. |
| 결과 | `curl http://13.125.53.24/health` → `OK`, `HTTP_CODE:200`. SG 인바운드가 다시 `80 ← 0.0.0.0/0`, `22 ← 내 IP/32` 두 개로 확인됨. |
| 재발 방지 | ① 배포 체크리스트에 "SG 인바운드 80/22 규칙 확인(`describe-security-groups`)"을 추가한다. ② 외부 접속 장애는 **timeout이면 SG·라우팅, refused면 서버 프로세스**부터 본다. ③ 서버 내부 `curl localhost`와 외부 `curl <퍼블릭IP>`를 함께 확인해 문제 구간(서버 안/밖)을 먼저 나눈다. |

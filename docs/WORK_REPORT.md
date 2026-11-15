# 작업 설명문 - codyssey-aws-web-deploy (B3-1)

## 무엇을 했는가

AWS 서울 리전(ap-northeast-2)에 VPC(10.0.0.0/16), Public Subnet, Internet Gateway,
Route Table을 직접 만들었습니다. 그 안에 EC2(t3.micro, Amazon Linux 2023)를 띄우고 Nginx를
설치·기동했습니다. 최소권한 보안 그룹을 적용한 상태에서 외부(내 노트북)에서 HTTP 접속이
되는지 curl과 브라우저로 확인했습니다. 검증 후에는 모든 리소스를 삭제하고, 삭제됐는지
CLI로 다시 조회해 확인했습니다.

## 왜 IAM 사용자를 먼저 만들었는가

`aws configure`로 등록한 자격증명을 `aws sts get-caller-identity`로 확인해 보니 Arn이
`...:root`였습니다. 미션은 "루트 계정은 사용하지 않는다"를 요구합니다. 그래서 루트
자격증명은 EC2 권한만 가진 IAM 사용자(`codyssey-mission2`)를 만드는 데 한 번만 쓰고,
바로 이 사용자의 액세스 키로 바꿔서 이후 모든 VPC/EC2 작업을 진행했습니다.
AdministratorAccess를 주지 않았기 때문에, 이 사용자로 `aws s3 ls`나
`iam:ListAttachedUserPolicies`를 호출하면 거부되는 것도 확인했습니다.

## 겪은 문제와 해결

[트러블슈팅 보고서](troubleshooting.md)에 증상 → 가설 → 검증 → 조치 → 결과 → 재발방지 형식으로 기록했습니다.

1. `t2.micro`가 이 계정에서는 프리 티어 대상이 아니라서 `t3.micro`로 변경
2. 등록된 자격증명이 루트 계정이라는 것을 발견하고 IAM 사용자로 전환

## 외부 접속 증빙에 대해

브라우저로 `http://<퍼블릭IP>`에 접속해 "Welcome to nginx!" 페이지가 뜨는 것을 확인했습니다.
그런데 그때 화면 캡처를 저장하지 않아서 1차 실습의 증빙은 curl 요청/응답 원문
(`screenshots/*.txt`, `index_response.html`)뿐입니다. 이 한계를 보완하는 재실습 결과는
아래 "보완 작업"에 정리했습니다.

## 요구사항 대비 체크 (1차 실습)

- [x] VPC 1개, Public Subnet 1개, IGW 연결, Route Table(0.0.0.0/0 -> IGW)
- [x] Public Subnet 아웃바운드 인터넷 통신 (인스턴스 내부 `curl https://example.com` -> 200)
- [x] EC2 1대, SSH 접속(ec2-user), Nginx 설치/기동, 내부 `curl http://localhost` -> 200
- [x] 보안 그룹: 80은 0.0.0.0/0, 22는 내 IP만, 0.0.0.0/0 전체 포트 허용 규칙 없음
- [x] IAM: EC2 전용 정책만 부착, AdministratorAccess 없음, 루트 계정 미사용(부트스트랩 1회 제외)
- [x] 외부 접속 검증 (B: `/health` 200 "OK", A: 브라우저 접속도 확인)
- [x] 실습 종료 후 EC2/EBS/EIP/IGW/VPC 삭제 및 CLI 재조회로 확인
- [x] 아키텍처 다이어그램, 트러블슈팅 보고서, 리소스 정리 체크리스트

## 보완 작업 (2차 실습, 2026-11-15)

1차 실습 후 검토에서 아래 항목을 보완 대상으로 정했고, [실습 시나리오](scenario.md)대로 처음부터 다시 구축해 보완했습니다.

| 항목 | 1차 실습 상태 | 2차 실습 결과 |
| --- | --- | --- |
| 접속 스크린샷 | 텍스트 증빙만 있음 | `screenshots/browser_health.png`, `browser_index.png` |
| IAM 정책 범위 | `AmazonEC2FullAccess` (ELB·ASG·CloudWatch 포함) | 커스텀 정책 `codyssey-ec2-lab`으로 교체. S3·IAM·RDS·타 리전 거부를 `iam_least_privilege_check.txt`에 기록 |
| 트러블슈팅 | 설정·계정 이슈만 있음 | SG 80 차단 → timeout 재현·분석 (이슈 3) |
| 내부 검증 원문 | 설명만 있음 | `instance_checks.txt`, `network_config.txt` |
| 다이어그램 | 오른쪽이 잘림 | SVG로 다시 그림 (`architecture.svg` → `architecture.png`) |

2차 실습 리소스도 검증 직후 모두 삭제했고, 실습용 액세스 키도 삭제했습니다.

마지막으로 콘솔에서 청구서가 **USD 0.00**인 것을 확인했고(`screenshots/billing.png`), 부트스트랩에 쓴 루트 계정 액세스 키도 삭제했습니다.

## 검증

이 저장소의 수치, 응답, 오류 메시지는 모두 AWS CLI·curl·브라우저를 실제로 실행해 얻은 원문입니다.
삭제 후 재조회 결과(`InvalidVpcID.NotFound` 등)는 [cleanup-checklist.md](cleanup-checklist.md)에 남겼습니다.

# 작업 설명문 - codyssey-aws-web-deploy (B3-1)

## 무엇을 했는가

AWS ap-northeast-2(서울) 리전에 VPC(10.0.0.0/16) + Public Subnet +
Internet Gateway + Route Table을 직접 만들고, 그 안에 EC2(t3.micro,
Amazon Linux 2023)를 띄워 Nginx를 설치·기동한 뒤, 최소권한 보안그룹
설정 하에서 실제로 외부(이 세션이 실행 중인 맥)에서 HTTP 접속이
가능함을 curl과 브라우저 둘 다로 확인했다. 검증이 끝난 뒤 모든
리소스를 실제로 삭제하고, 삭제 완료를 CLI 명령 결과로 재확인했다.

## 왜 IAM 사용자를 먼저 만들었는가

aws configure로 등록된 자격증명을 확인해보니 루트 계정
(arn:...:root)이었다. 미션이 "루트 계정은 사용하지 않는다"를
명시적으로 요구하기 때문에, 루트 자격증명은 AmazonEC2FullAccess
정책 하나만 가진 IAM 사용자(codyssey-mission2)를 만드는 딱 한
번의 부트스트랩에만 쓰고, 그 즉시 이 IAM 사용자의 액세스 키로
전환해서 이후 모든 VPC/EC2 작업을 진행했다. AdministratorAccess를
주지 않았기 때문에 같은 계정으로 aws s3 ls나
iam:ListAttachedUserPolicies를 호출하면 거부되는 것까지 실제로
확인했다 - README/troubleshooting.md에 그 증거를 그대로 남겼다.

이 부분은 IAM 사용자 생성이라는 "계정/보안 설정 변경"에 해당하는
작업이라, 실행 전 사용자에게 직접 확인을 받고 진행했다.

## 겪은 문제와 해결

트러블슈팅 문서(docs/troubleshooting.md)에 정식 포맷으로 2건을
기록했다: (1) t2.micro가 이 계정에서는 프리 티어 대상이 아니어서
t3.micro로 바꾼 것, (2) 자격증명이 루트였던 것을 발견하고 IAM
사용자로 전환한 것.

## 외부 접속 스크린샷에 대한 한계

브라우저 도구로 http://<퍼블릭IP>에 실제로 접속해 "Welcome to
nginx!" 페이지가 정상 렌더링되는 것을 이 세션에서 직접 확인했다.
다만 이 실행 환경에는 화면 캡처(macOS screencapture) 권한이 없어
PNG 스크린샷 파일을 디스크에 저장할 수 없었다. 대신 같은 응답을
받는 curl -v 전체 트랜잭션과 응답 바디를 screenshots/에 텍스트
파일로 남겨, 브라우저가 받은 것과 동일한 서버 응답을 파일 증빙으로
대체했다. 이 사실을 숨기지 않고 README에 명시했다.

## 요구사항 대비 체크

- [x] VPC 1개, Public Subnet 1개, IGW 연결, Route Table
  (0.0.0.0/0 -> IGW)
- [x] Public Subnet 아웃바운드 인터넷 통신 확인 (인스턴스 내부에서
  curl https://example.com -> 200)
- [x] EC2 1대, SSH 접속 가능(비루트 계정 ec2-user), Nginx 설치/기동,
  인스턴스 내부 curl http://localhost -> 200
- [x] 보안그룹: 80은 0.0.0.0/0, 22는 실습자 IP만, 0.0.0.0/0 전체 포트
  허용 규칙 없음
- [x] IAM 최소권한: EC2 전용 정책 1개만 부착, AdministratorAccess 없음,
  루트 계정 미사용(부트스트랩 1회 제외)
- [x] 외부 접속 검증(B: /health 200 "OK", A: 브라우저 렌더링도 확인)
- [x] 실습 종료 후 EC2/EBS/EIP/IGW/VPC 전부 삭제 및 CLI로 재확인
- [x] 아키텍처 다이어그램(docs/architecture.png), 트러블슈팅 보고서,
  리소스 정리 체크리스트 전부 제출

## 검증

이 저장소의 모든 수치/응답/오류 메시지는 실제로 AWS CLI/curl/브라우저를
실행해 얻은 원문이다(가상의 로그를 지어내지 않았다). 리소스는 검증 직후
전부 삭제했고, 삭제 후 재조회 결과(InvalidVpcID.NotFound 등)까지
docs/cleanup-checklist.md에 그대로 남겼다.

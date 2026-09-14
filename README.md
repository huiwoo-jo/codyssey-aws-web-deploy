# codyssey-aws-web-deploy

AWS에 VPC로 격리된 네트워크를 직접 구성하고, 그 안에 Nginx 웹 서버를
배포해 외부에서 접속 가능하게 만든 실습 결과물입니다(B3-1).

## 외부 접속 검증

- **선택한 방식**: (B) `GET http://<퍼블릭IP>/health` — 200과 고정 문구 `OK` 확인.
  (A) 브라우저 접속도 세션 중 실제로 확인했습니다: `http://13.124.219.14` 접속 시
  "Welcome to nginx!" 페이지가 정상 렌더링됨을 이 세션의 Browser 도구로
  직접 확인했습니다. 다만 이 실행 환경에서는 화면 캡처(screencapture)
  권한이 없어 PNG 스크린샷 파일을 디스크에 저장할 수 없었고, 대신
  `screenshots/`에 실제 curl 요청/응답 원문(HTTP 200, 응답 바디, 헤더)을
  파일로 남겼습니다 — 브라우저가 받은 것과 동일한 서버 응답의 텍스트
  증빙입니다.
- **접속 정보**: `http://13.124.219.14` (또는 `http://13.124.219.14/health`)

```
$ curl http://13.124.219.14/health
OK
$ curl -o /dev/null -s -w "%{http_code}\n" http://13.124.219.14/health
200
```

증빙 파일: `screenshots/curl_transcript.txt`(전체 curl -v 로그),
`screenshots/index_response.html`(Nginx 기본 페이지 응답 원문),
`screenshots/health_response.txt`(/health 응답).

## 아키텍처

![architecture](docs/architecture.png)

- VPC `codyssey-vpc` (10.0.0.0/16)
- Public Subnet `codyssey-public-subnet` (10.0.1.0/24, ap-northeast-2a)
- Internet Gateway `codyssey-igw`, Route Table `codyssey-public-rtb`
  (`0.0.0.0/0 -> IGW`)
- Security Group `codyssey-web-sg`: 80/tcp는 0.0.0.0/0, 22/tcp는 실습자
  IP(`121.135.181.35/32`)만 허용. 0.0.0.0/0 전체 포트 허용 규칙 없음.
- EC2 `codyssey-web` (t3.micro, Amazon Linux 2023, Nginx)

## IAM 최소권한

콘솔/CLI 접근은 루트 계정을 쓰지 않고, EC2 실습에만 쓰는 IAM 사용자
`codyssey-mission2`로 진행했습니다.

- 부착 정책: `AmazonEC2FullAccess` **하나만** (AdministratorAccess 없음)
- 실제로 S3 등 무관한 서비스 접근은 거부됨을 확인:
  ```
  $ aws s3 ls --profile codyssey-mission2
  AccessDenied: ... is not authorized to perform: s3:ListAllMyBuckets
  ```
  (같은 계정으로 `iam:ListAttachedUserPolicies`조차 거부되어, IAM 관련
  권한도 전혀 없음을 확인했습니다.)
- 루트 계정 액세스 키는 이 IAM 사용자를 만드는 1회성 부트스트랩에만
  사용했고, 이후 모든 리소스 생성/조회는 `--profile codyssey-mission2`로만
  수행했습니다. 자세한 경위는 `docs/troubleshooting.md` 이슈 2 참고.

## 산출물 구성

```
docs/architecture.png         아키텍처 다이어그램
docs/troubleshooting.md        트러블슈팅 보고서 (2건)
docs/cleanup-checklist.md       리소스 정리 체크리스트
docs/WORK_REPORT.md             작업 설명문
screenshots/                     외부 접속 검증 증빙(curl 원문)
```

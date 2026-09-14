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

# 리소스 정리 체크리스트 - codyssey-aws-web-deploy (B3-1)

실습은 두 번 진행했고, 두 번 모두 검증이 끝난 직후 아래 순서로 정리했습니다.
각 항목은 AWS CLI 실행 결과로 확인했습니다.

- 1차 실습 (2026-09-14): `vpc-083fc4c4f639da820`, `i-0273bcfcfe5dce918` → 전부 삭제 (아래 1차 기록)
- 2차 실습 (2026-11-15, 보완용 재실습): 아래 체크리스트. 전체 실행 로그는 [`screenshots/cleanup_log.txt`](../screenshots/cleanup_log.txt)

## 정리 순서 (의존 관계의 역순)

```
EC2 종료(EBS 자동 삭제) → SG → Route Table(연결 해제 → 삭제) → IGW(분리 → 삭제)
  → Subnet → VPC → Key Pair(AWS + 로컬) → 실습용 액세스 키
```

인스턴스가 남아 있으면 SG 삭제가, IGW가 붙어 있으면 VPC 삭제가 `DependencyViolation`으로 실패합니다.

## 체크리스트 (2차 실습)

| # | 항목 | 리소스 ID | 정리 명령 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 1 | [x] EC2 인스턴스 | `i-01cdb1886c3edc83d` | `terminate-instances` → `wait instance-terminated` | `State.Name = terminated` |
| 2 | [x] EBS 볼륨 (미사용 포함) | `vol-0a2163b11b6c2fa28` | `DeleteOnTermination=true`로 인스턴스와 함께 삭제 | `InvalidVolume.NotFound`, 리전 전체 볼륨 0개 |
| 3 | [x] Elastic IP | (할당 안 함) | 자동 할당 퍼블릭 IP만 사용 | `describe-addresses` 0건 |
| 4 | [x] Security Group | `sg-06cc1b31e3ecde061` | `delete-security-group` | `"Return": true` |
| 5 | [x] Route Table | `rtb-0fc9d6519ab658b16` | `disassociate-route-table` → `delete-route-table` | exit 0 |
| 6 | [x] Internet Gateway | `igw-02ff94b98e8f44d17` | `detach-internet-gateway` → `delete-internet-gateway` | `InvalidInternetGatewayID.NotFound` |
| 7 | [x] Subnet | `subnet-0b18bc655e35a2f38` | `delete-subnet` | exit 0 |
| 8 | [x] VPC | `vpc-0b70fc1f60b0af234` | `delete-vpc` | `InvalidVpcID.NotFound` |
| 9 | [x] 키 페어 | `codyssey-mission2-key` | `delete-key-pair` + 로컬 `.pem` 삭제 | `InvalidKeyPair.NotFound` |
| 10 | [x] NAT Gateway | (생성 안 함) | - | `describe-nat-gateways` 0건 |
| 11 | [x] ELB/ALB, RDS | (생성 안 함) | - | 해당 없음 (실습 사용자에게 권한 자체가 없음) |
| 12 | [x] 실습용 액세스 키 | `codyssey-mission2` 키 | `iam delete-access-key` + 로컬 프로파일 삭제 | `list-access-keys` 0개 |

서울 리전에 남은 VPC는 AWS 기본 VPC(`IsDefault=True`) 하나뿐입니다.

## 최종 확인 명령 결과 (발췌)

```
$ aws ec2 describe-instances --instance-ids i-01cdb1886c3edc83d --query 'Reservations[0].Instances[0].State.Name' --output text
terminated

$ aws ec2 describe-volumes --volume-ids vol-0a2163b11b6c2fa28
An error occurred (InvalidVolume.NotFound) ... The volume 'vol-0a2163b11b6c2fa28' does not exist.

$ aws ec2 describe-addresses --query 'Addresses[].PublicIp' --output text
(빈 출력)

$ aws ec2 describe-internet-gateways --internet-gateway-ids igw-02ff94b98e8f44d17
An error occurred (InvalidInternetGatewayID.NotFound) ...

$ aws ec2 describe-vpcs --vpc-ids vpc-0b70fc1f60b0af234
An error occurred (InvalidVpcID.NotFound) ... The vpc ID 'vpc-0b70fc1f60b0af234' does not exist

$ aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,IsDefault]' --output text
vpc-0d5228ec567a13ddd	True
```

## 1차 실습 정리 기록 (2026-09-14)

| 항목 | 결과 |
| --- | --- |
| EC2 `i-0273bcfcfe5dce918` | `terminated` |
| EBS | 리전 전체 볼륨 0개 |
| Elastic IP | 할당 안 함, 0건 |
| SG `sg-09652d5c8a602fe72`, Route Table, IGW, Subnet `subnet-09423428157cff78a` | 삭제 성공 |
| VPC `vpc-083fc4c4f639da820` | 재조회 시 `InvalidVpcID.NotFound` |
| 키 페어, 실습용 액세스 키 | 삭제 |

## 콘솔 확인 (2026-11-15)

- [x] **Billing 확인**: 결제 및 비용 관리 → 청구서(2026년 11월) **예상 총합계 USD 0.00**. 캡처: [`screenshots/billing.png`](../screenshots/billing.png) (계정 ID 가림)
  - 비용 데이터는 최대 24시간 늦게 반영되므로 당일 EC2 사용분은 아직 포함되지 않았을 수 있다. t3.micro 1시간 미만 사용, EBS 8GiB는 프리 티어 범위다.
  - 청구서의 Glue(26 요청)·S3(1 요청) 항목은 시드니 리전 사용분으로 이 실습과 무관하며 $0이다.
- [x] **루트 액세스 키 삭제**: 콘솔 → 보안 자격 증명에서 삭제. 이후 해당 키로 `aws sts get-caller-identity` 호출 시 `InvalidClientTokenId` 반환 확인.

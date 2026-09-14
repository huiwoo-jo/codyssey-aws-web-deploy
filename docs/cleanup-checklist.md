# 리소스 정리 체크리스트 - codyssey-aws-web-deploy (B3-1)

모든 검증(외부 접속, SSH, 아웃바운드 통신) 완료 후 아래 순서로 정리했으며,
각 단계는 실제 AWS CLI 명령 실행 결과로 확인했습니다.

- [x] **EC2 인스턴스 종료 확인**
  `aws ec2 terminate-instances --instance-ids i-0273bcfcfe5dce918` 후
  `aws ec2 wait instance-terminated` 대기 완료.
  최종 확인: `describe-instances` → `State.Name = terminated`
- [x] **EBS 볼륨(미사용 포함) 삭제 확인**
  루트 볼륨은 DeleteOnTermination(기본값)으로 인스턴스 종료 시 자동 삭제됨.
  `describe-volumes`(리전 전체) 결과 볼륨 0개로 확인.
- [x] **Elastic IP Release 확인**
  이번 실습에서는 EIP를 별도 할당하지 않고 인스턴스의 자동 할당 퍼블릭
  IP만 사용했음. `describe-addresses` 결과 0건으로, 정리할 EIP 자체가
  없음을 확인.
- [x] **Security Group 삭제 확인**
  `aws ec2 delete-security-group --group-id sg-09652d5c8a602fe72` 성공.
- [x] **Route Table 삭제 확인**
  서브넷 연결 해제(`disassociate-route-table`) 후
  `delete-route-table` 성공.
- [x] **Internet Gateway Detach 및 삭제 확인**
  `detach-internet-gateway` → `delete-internet-gateway` 순서로 성공.
- [x] **Subnet 삭제 확인**
  `aws ec2 delete-subnet --subnet-id subnet-09423428157cff78a` 성공.
- [x] **VPC 삭제 확인**
  `aws ec2 delete-vpc --vpc-id vpc-083fc4c4f639da820` 성공. 이후
  `describe-vpcs`로 재조회 시 `InvalidVpcID.NotFound` 반환 확인(=완전히
  삭제됨).
- [x] **(해당 없음) NAT Gateway** — 생성하지 않음.
- [x] **(해당 없음) ELB/ALB** — 생성하지 않음.
- [x] **(해당 없음) RDS** — 생성하지 않음.
- [x] **키 페어 정리**
  AWS 측 `aws ec2 delete-key-pair --key-name codyssey-mission2-key` 완료,
  로컬 개인키 파일(`~/.ssh/codyssey-mission2-key.pem`)도 삭제.
- [x] **IAM 자격증명 정리**
  실습용 IAM 사용자(`codyssey-mission2`)의 액세스 키를 삭제해 더 이상
  사용 불가능한 상태로 만들었다(사용자/정책 자체는 최소권한 구성의
  증빙으로 남겨둠 — AmazonEC2FullAccess 정책 하나만 부착되어 있고
  AdministratorAccess는 없음).

## 최종 검증 명령 결과 (그대로 인용)

```
$ aws ec2 describe-vpcs --vpc-ids vpc-083fc4c4f639da820
An error occurred (InvalidVpcID.NotFound) when calling the DescribeVpcs operation:
The vpc ID 'vpc-083fc4c4f639da820' does not exist

$ aws ec2 describe-instances --instance-ids i-0273bcfcfe5dce918 --query 'Reservations[0].Instances[0].State.Name'
terminated

$ aws ec2 describe-addresses --query 'Addresses'
(empty)

$ aws ec2 describe-volumes --query 'Volumes[].[VolumeId,State]'
(empty)
```

모든 리소스 삭제 후 Billing Dashboard 확인은 콘솔 접근이 필요해 이
CLI 세션에서는 직접 수행하지 못했습니다 — 실습자가 AWS 콘솔에서
한 번 더 확인하는 것을 권장합니다.

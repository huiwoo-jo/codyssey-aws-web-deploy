# 실습·시연 시나리오 — B3-1 웹 서비스 배포

이 문서는 **처음부터 끝까지 다시 따라 할 수 있는 실습 절차**이자 **발표 시연 대본**입니다.
모든 단계는 AWS CLI 기준이고, 단계마다 **"확인"** 명령으로 요구사항 충족을 바로 검증합니다.
콘솔로 진행할 때 들어갈 메뉴는 각 단계 제목 아래에 적었습니다.

| 단계 | 내용 | 대응 요구사항 | 예상 소요 |
| --- | --- | --- | --- |
| 0 | 사전 준비, 루트 계정 여부 확인 | 제약: 루트 미사용, 서울 리전 | 3분 |
| 1 | IAM 최소권한 사용자 준비 | IAM 최소권한 | 5분 |
| 2 | VPC · Subnet · IGW · Route Table | 네트워크 구성 | 5분 |
| 3 | Security Group | 접근 제어 | 3분 |
| 4 | 키 페어 + EC2 + Nginx | 컴퓨트 및 웹 서버 | 5분 (+부팅 2분) |
| 5 | 인스턴스 내부 검증 (SSH) | SSH, `curl localhost`, 아웃바운드 | 3분 |
| 6 | 외부 접속 검증 + 스크린샷 | 외부 접속 (A/B) | 3분 |
| 7 | 장애 재현 데모 (SG 80 차단) | 트러블슈팅 | 5분 |
| 8 | 리소스 정리 + 삭제 확인 | 운영 안정성(과금 방지) | 5분 |

> ⚠️ 2단계부터는 리소스가 생성됩니다. 시연이 끝나면 **반드시 8단계까지** 진행하세요.

---

## 0단계. 사전 준비

준비물: AWS CLI v2, 터미널(macOS/Linux), 실습용 IAM 사용자의 액세스 키

```bash
export AWS_PROFILE=codyssey-mission2
export AWS_REGION=ap-northeast-2
export AWS_PAGER=""
```

**확인 — 루트 계정이 아닌지**

```bash
aws sts get-caller-identity --query Arn --output text
# 기대값: arn:aws:iam::<계정ID>:user/codyssey-mission2
# ':root'로 끝나면 즉시 중단 → 1단계로 IAM 사용자부터 만든다 (트러블슈팅 이슈 2)
```

**확인 — 이 계정의 프리 티어 인스턴스 타입** (트러블슈팅 이슈 1 재발 방지)

```bash
aws ec2 describe-instance-types \
  --filters Name=free-tier-eligible,Values=true \
  --query 'InstanceTypes[].InstanceType' --output text
# t3.micro가 목록에 있는지 확인
```

> 💬 **발표 멘트**: "가장 먼저 지금 누구 권한으로 일하는지 확인합니다. 저는 처음에 이 명령에서 `:root`가 나와서 작업 방식을 바꿨습니다."

---

## 1단계. IAM 최소권한 사용자 준비

> 콘솔: IAM → 정책 → 정책 생성(JSON) / IAM → 사용자 → 사용자 생성

이 단계는 **IAM 관리 권한이 있는 계정으로 딱 한 번** 실행합니다. 이후 단계는 전부 실습용 사용자로 진행합니다.

`AmazonEC2FullAccess` 대신 **이 실습에서 실제로 호출하는 EC2 API만** 허용하고, **서울 리전으로 묶은** 정책을 씁니다.
S3·RDS·IAM·ELB 권한은 없습니다.

```bash
cat > /tmp/codyssey-ec2-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyDescribe",
      "Effect": "Allow",
      "Action": ["ec2:Describe*", "ec2:GetConsoleOutput"],
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": "ap-northeast-2" } }
    },
    {
      "Sid": "LabNetworkComputeLifecycle",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc", "ec2:DeleteVpc", "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet", "ec2:DeleteSubnet", "ec2:ModifySubnetAttribute",
        "ec2:CreateInternetGateway", "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway", "ec2:DeleteInternetGateway",
        "ec2:CreateRouteTable", "ec2:DeleteRouteTable", "ec2:CreateRoute", "ec2:DeleteRoute",
        "ec2:AssociateRouteTable", "ec2:DisassociateRouteTable",
        "ec2:CreateSecurityGroup", "ec2:DeleteSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress", "ec2:RevokeSecurityGroupIngress",
        "ec2:CreateKeyPair", "ec2:ImportKeyPair", "ec2:DeleteKeyPair",
        "ec2:RunInstances", "ec2:StartInstances", "ec2:StopInstances",
        "ec2:RebootInstances", "ec2:TerminateInstances",
        "ec2:DeleteVolume",
        "ec2:AllocateAddress", "ec2:AssociateAddress",
        "ec2:DisassociateAddress", "ec2:ReleaseAddress",
        "ec2:CreateTags", "ec2:DeleteTags"
      ],
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": "ap-northeast-2" } }
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --profile <관리자-프로파일>)
aws iam create-policy --policy-name codyssey-ec2-lab \
  --policy-document file:///tmp/codyssey-ec2-policy.json --profile <관리자-프로파일>
aws iam create-user --user-name codyssey-mission2 --profile <관리자-프로파일>
aws iam attach-user-policy --user-name codyssey-mission2 \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/codyssey-ec2-lab --profile <관리자-프로파일>
aws iam create-access-key --user-name codyssey-mission2 --profile <관리자-프로파일>
# 출력된 키를 `aws configure --profile codyssey-mission2`로 등록 (키는 저장소에 커밋 금지)
```

> 콘솔 로그인이 필요하면 `aws iam create-login-profile`로 비밀번호를 설정합니다(초기 비밀번호는 콘솔에서 직접 입력).
> 콘솔 EC2 화면 일부(AMI 별칭, IAM 역할 목록)는 SSM/IAM 조회 권한이 없어 경고가 뜰 수 있지만, 실습 진행에는 지장이 없습니다.

**확인 — 무관한 서비스와 다른 리전은 거부되는지**

```bash
aws s3 ls                                    # AccessDenied (S3 권한 없음)
aws iam list-users                           # AccessDenied (IAM 권한 없음)
aws ec2 describe-vpcs --region us-east-1     # UnauthorizedOperation (리전 제한)
aws ec2 describe-vpcs                        # 성공 (서울 리전 조회는 허용)
```

> 💬 **발표 멘트**: "관리자 권한 대신 이 실습에서 실제로 호출한 API만 열었습니다. S3는 물론이고 같은 EC2라도 다른 리전에서는 거부됩니다."

> 🔐 루트 계정에 액세스 키를 만들었다면, IAM 사용자 전환 후 **루트 액세스 키는 삭제**합니다(콘솔: 계정 → 보안 자격 증명).

---

## 2단계. 네트워크 구성 (VPC · Subnet · IGW · Route Table)

> 콘솔: VPC → VPC 생성("VPC만") → 서브넷 → 인터넷 게이트웨이 → 라우팅 테이블

```bash
# helper: 리소스 생성 시 Name 태그 부여
tag() { echo "ResourceType=$1,Tags=[{Key=Name,Value=$2},{Key=Project,Value=codyssey}]"; }

# VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications "$(tag vpc codyssey-vpc)" \
  --query Vpc.VpcId --output text)

# Public Subnet (+ 퍼블릭 IP 자동 할당)
SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone ap-northeast-2a \
  --tag-specifications "$(tag subnet codyssey-public-subnet)" \
  --query Subnet.SubnetId --output text)
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID --map-public-ip-on-launch

# Internet Gateway 생성 + VPC 연결
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications "$(tag internet-gateway codyssey-igw)" \
  --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Route Table: 0.0.0.0/0 → IGW, 서브넷에 연결
RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications "$(tag route-table codyssey-public-rtb)" \
  --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RTB_ID \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
RTB_ASSOC_ID=$(aws ec2 associate-route-table --route-table-id $RTB_ID \
  --subnet-id $SUBNET_ID --query AssociationId --output text)

echo "VPC=$VPC_ID SUBNET=$SUBNET_ID IGW=$IGW_ID RTB=$RTB_ID ASSOC=$RTB_ASSOC_ID"
```

> 터미널을 닫으면 변수가 사라집니다. 위 `echo` 출력을 메모해 두세요(8단계 정리에 필요).

**확인 — 라우팅 테이블**

```bash
aws ec2 describe-route-tables --route-table-ids $RTB_ID \
  --query 'RouteTables[0].Routes[].[DestinationCidrBlock,GatewayId,State]' --output table
# 10.0.0.0/16  local   active
# 0.0.0.0/0    igw-... active   ← 이 줄이 있어야 "퍼블릭" 서브넷
```

> 💬 **발표 멘트**: "서브넷 이름에 public을 붙인다고 퍼블릭이 되지 않습니다. 이 `0.0.0.0/0 → igw` 한 줄이 있어야 퍼블릭 서브넷입니다."

---

## 3단계. Security Group

> 콘솔: EC2 → 보안 그룹 → 보안 그룹 생성 (SSH 소스는 "내 IP" 선택)

```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)/32

SG_ID=$(aws ec2 create-security-group --group-name codyssey-web-sg \
  --description "HTTP from anywhere, SSH from my IP only" --vpc-id $VPC_ID \
  --tag-specifications "$(tag security-group codyssey-web-sg)" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr $MY_IP
```

**확인 — 인바운드 규칙이 정확히 2개인지**

```bash
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions[].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]' \
  --output table
# tcp  80  80  0.0.0.0/0
# tcp  22  22  <내 IP>/32
# → 0.0.0.0/0 + 0-65535(또는 protocol -1) 규칙이 없어야 함
```

---

## 4단계. 키 페어 + EC2 + Nginx

> 콘솔: EC2 → 인스턴스 시작 (AMI: Amazon Linux 2023, t3.micro, 네트워크 설정에서 위 VPC/서브넷/SG 선택, 고급 세부 정보 → 사용자 데이터)

```bash
# 키 페어 (개인키는 재발급 불가 → 안전하게 보관, 저장소에 커밋 금지)
# ~/.ssh가 없는 새 계정에서도 저장되도록 먼저 만들고, 실패하면 여기서 멈춘다
mkdir -p ~/.ssh && chmod 700 ~/.ssh
aws ec2 create-key-pair --key-name codyssey-mission2-key \
  --query KeyMaterial --output text > ~/.ssh/codyssey-mission2-key.pem \
  && chmod 400 ~/.ssh/codyssey-mission2-key.pem \
  || { echo "키 페어 생성/저장 실패 — 다음 단계로 진행하지 마세요"; return 1 2>/dev/null || exit 1; }

# 최신 Amazon Linux 2023 AMI (SSM 권한 없이 EC2 API로 조회)
AMI_ID=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

# 부팅 시 Nginx 설치 + /health 엔드포인트 생성
cat > /tmp/user-data.sh <<'EOF'
#!/bin/bash
set -euxo pipefail
dnf install -y nginx
cat > /etc/nginx/default.d/health.conf <<'NGINX'
location = /health {
    default_type text/plain;
    return 200 "OK\n";
}
NGINX
systemctl enable --now nginx
EOF

INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t3.micro \
  --key-name codyssey-mission2-key --subnet-id $SUBNET_ID --security-group-ids $SG_ID \
  --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=8,VolumeType=gp3,DeleteOnTermination=true}' \
  --user-data file:///tmp/user-data.sh \
  --tag-specifications "$(tag instance codyssey-web)" \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-status-ok --instance-ids $INSTANCE_ID   # 2~3분
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "INSTANCE=$INSTANCE_ID PUBLIC_IP=$PUBLIC_IP"
```

> `t2.micro`로 `not eligible for Free Tier` 오류가 나면 0단계 목록에 있는 타입(`t3.micro`)을 씁니다. (트러블슈팅 이슈 1)

---

## 5단계. 인스턴스 내부 검증 (SSH)

```bash
ssh -i ~/.ssh/codyssey-mission2-key.pem ec2-user@$PUBLIC_IP
```

인스턴스 안에서 실행합니다. `instance-status-ok`는 인스턴스 상태만 보는 것이라, user-data(Nginx 설치)가 아직 진행 중일 수 있습니다. 먼저 cloud-init이 끝나기를 기다립니다.

```bash
cloud-init status --wait        # status: done 이 나와야 함
# status: error 라면 아래 로그로 원인을 확인하고, 해결 전에는 검증 단계로 넘어가지 않는다
sudo tail -n 50 /var/log/cloud-init-output.log
```

그다음 서비스를 확인합니다.

```bash
systemctl is-active nginx                                       # active
curl -s -o /dev/null -w "%{http_code}\n" http://localhost       # 200
curl -s http://localhost/health                                 # OK
curl -s -o /dev/null -w "%{http_code}\n" https://example.com    # 200  ← 아웃바운드 인터넷 통신
exit
```

**증빙 저장** (로컬에서 실행, 결과를 저장소에 남김)

```bash
ssh -i ~/.ssh/codyssey-mission2-key.pem ec2-user@$PUBLIC_IP '
  echo "## systemctl"; systemctl is-active nginx
  echo "## curl localhost"; curl -s -o /dev/null -w "%{http_code}\n" http://localhost
  echo "## curl example.com"; curl -s -o /dev/null -w "%{http_code}\n" https://example.com
' | tee screenshots/instance_checks.txt
```

---

## 6단계. 외부 접속 검증 + 스크린샷

**(B) 헬스체크** — README에 기재한 방식

```bash
curl -i http://$PUBLIC_IP/health
# HTTP/1.1 200 OK
# Content-Type: text/plain
# OK
curl -s http://$PUBLIC_IP/health > screenshots/health_response.txt
curl -sv http://$PUBLIC_IP/ -o screenshots/index_response.html 2> screenshots/curl_transcript.txt
```

**(A) 브라우저** — `http://<PUBLIC_IP>` 접속 → "Welcome to nginx!" 확인

**스크린샷 저장** (과제 최소 규격: 접속 결과 스크린샷 1장 이상)

```bash
# macOS: 실행 후 캡처할 창을 클릭 (권한 오류가 나면 Cmd+Shift+4 → Space → 창 클릭)
screencapture -w screenshots/browser_index.png   # 브라우저 창
screencapture -w screenshots/health_curl.png     # curl 결과가 보이는 터미널 창
```

> 💬 **발표 멘트**: "이제 제 노트북, 즉 VPC 바깥에서 퍼블릭 IP로 요청을 보냅니다. IGW → 라우트 → SG → Nginx를 모두 통과해야 이 `OK`가 돌아옵니다."

---

## 7단계. 장애 재현 데모 — "서버는 떴는데 외부에서 안 들어와요"

미션 소개의 상황을 직접 만들고, **증상 → 가설 → 검증 → 조치 → 결과** 순서로 해결합니다.
이 결과를 `docs/troubleshooting.md` 이슈 3으로 추가할 수 있습니다.

**① 장애 주입** — SG에서 80 포트 규칙 제거

```bash
aws ec2 revoke-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**② 증상** — 외부에서 응답 없음 (timeout)

```bash
curl -m 5 -s -o /dev/null -w "%{http_code}\n" http://$PUBLIC_IP/health
# 000  (curl: (28) Connection timed out)
```

**③ 가설 세우기** — 계층별로 후보를 좁힌다

| 후보 | 이 증상과 맞나? |
| --- | --- |
| Nginx가 죽었다 | 프로세스가 없으면 OS가 RST를 돌려줘 **즉시 `Connection refused`**가 남 → timeout과 안 맞음 |
| 라우팅/IGW 문제 | 가능. 하지만 방금 전까지 됐고 네트워크는 건드리지 않음 |
| **SG가 80을 막는다** | SG는 허용되지 않은 패킷을 **응답 없이 버림** → timeout과 일치 ✅ |

**④ 검증** — 서버 내부와 SG 규칙을 각각 확인

```bash
# 서버 내부는 정상인가? (SSH 22는 아직 열려 있음)
ssh -i ~/.ssh/codyssey-mission2-key.pem ec2-user@$PUBLIC_IP \
  'systemctl is-active nginx; curl -s http://localhost/health'
# active / OK  → 서버는 정상, 문제는 바깥

# SG 인바운드 규칙 확인
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions[].[FromPort,IpRanges[0].CidrIp]' --output text
# 22  <내 IP>/32   ← 80 규칙이 없음
```

**⑤ 조치 → ⑥ 결과**

```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
curl -s -o /dev/null -w "%{http_code}\n" http://$PUBLIC_IP/health   # 200
```

**⑦ 재발 방지** — 배포 체크리스트에 "SG 인바운드 80/22 규칙 확인"을 추가합니다. 증상이 timeout이면 SG·라우팅부터, refused면 서버 프로세스부터 확인합니다.

> 💡 **추가 데모(선택)**: `aws ec2 delete-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0`로 라우트를 지워도 똑같이 timeout이 납니다. SG는 그대로인데 왜 안 되는지 설명하면 "응답이 돌아갈 길"의 의미가 잘 전달됩니다. 복구: `aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID`.

---

## 8단계. 리소스 정리 + 삭제 확인

**생성의 역순**으로 지웁니다. 순서가 틀리면 `DependencyViolation` 오류가 납니다.

```bash
# 1) EC2 종료 (루트 EBS는 DeleteOnTermination=true로 함께 삭제)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID

# 2) Security Group (인스턴스가 완전히 종료된 뒤에만 삭제 가능)
aws ec2 delete-security-group --group-id $SG_ID

# 3) Route Table (서브넷 연결 해제 후 삭제)
aws ec2 disassociate-route-table --association-id $RTB_ASSOC_ID
aws ec2 delete-route-table --route-table-id $RTB_ID

# 4) Internet Gateway (분리 → 삭제)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 5) Subnet → VPC
aws ec2 delete-subnet --subnet-id $SUBNET_ID
aws ec2 delete-vpc --vpc-id $VPC_ID

# 6) 키 페어 (AWS 측 + 로컬 개인키)
aws ec2 delete-key-pair --key-name codyssey-mission2-key
rm -f ~/.ssh/codyssey-mission2-key.pem
```

**확인 — 과금 위험 항목이 하나도 남지 않았는지**

```bash
aws ec2 describe-instances --filters Name=tag:Project,Values=codyssey \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]' --output text   # terminated 만
aws ec2 describe-volumes   --query 'Volumes[].VolumeId'    --output text         # (빈 출력)
aws ec2 describe-addresses --query 'Addresses[].PublicIp'  --output text         # (빈 출력)
aws ec2 describe-nat-gateways --filter Name=state,Values=available \
  --query 'NatGateways[].NatGatewayId' --output text                             # (빈 출력)
aws ec2 describe-vpcs --vpc-ids $VPC_ID 2>&1 | grep -o InvalidVpcID.NotFound     # InvalidVpcID.NotFound
```

마지막으로 **콘솔 → Billing and Cost Management** 화면에서 과금 항목이 없는지 보고, 캡처해서 `screenshots/billing.png`로 저장합니다.
실습이 완전히 끝났으면 실습용 IAM 사용자의 액세스 키도 삭제합니다.

```bash
aws iam delete-access-key --user-name codyssey-mission2 \
  --access-key-id <키ID> --profile <관리자-프로파일>
```

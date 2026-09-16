- step0 환경변수 세팅 및 로그인
```sh
export STUDENT_ID="studentXX" # 이것 꼭 수정해주세요
# export STUDENT_ID="student00"
export AWS_PROFILE="$STUDENT_ID"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""
```

```sh
export MY_KEY_NAME="${STUDENT_ID}-key"
export MY_SG_NAME="${STUDENT_ID}-web-sg"
export MY_INSTANCE_NAME="${STUDENT_ID}-managed-ec2"
export MY_REUSE_INSTANCE_NAME="${STUDENT_ID}-compose-ec2"
export MY_DATA_SG_NAME="${STUDENT_ID}-data-sg"
export MY_DB_ID="${STUDENT_ID}-mysql-db"
export MY_DB_SUBNET_GROUP="${STUDENT_ID}-db-subnet-group"
export MY_CACHE_ID="${STUDENT_ID}-redis"
export MY_CACHE_SUBNET_GROUP="${STUDENT_ID}-cache-subnet-group"
export MY_AMI_NAME="${STUDENT_ID}-app-image"
export MY_INSTANCE_NAME_2="${MY_INSTANCE_NAME}-2"
export MY_TG_NAME="${STUDENT_ID}-app-tg"
export MY_ALB_SG_NAME="${STUDENT_ID}-alb-sg"
export MY_ALB_NAME="${STUDENT_ID}-app-alb"
# 로그인이 필요
# export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export MY_BUCKET="${STUDENT_ID}-app-assets-${ACCOUNT_ID}"
export MY_APP_IMAGE="ghcr.io/a1l1ke/simple-back-ghcr:latest"
export ACTIVE_INSTANCE_NAME="$MY_INSTANCE_NAME"
```

```sh
aws sts get-caller-identity
aws configure sso --profile "${STUDENT_ID}"
aws sso login --profile "${STUDENT_ID}"
# SSO session name: infra-training
aws sts get-caller-identity
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $ACCOUNT_ID
```

- step1 aws 기본 세팅
```sh
# 기존 키 삭제 및 신규 발급
aws ec2 delete-key-pair --key-name "$MY_KEY_NAME"
aws ec2 create-key-pair --key-name "$MY_KEY_NAME" \
    --tag-specifications "ResourceType=key-pair,Tags=[{Key=Name,Value=$MY_KEY_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
    --query "KeyMaterial" --output text > ./"$MY_KEY_NAME".pem
chmod 400 ./"$MY_KEY_NAME".pem 
```

```sh
# 보안 그룹 ID 확인
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)
export MY_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=$MY_SG_NAME" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)
echo $MY_SG_ID
```

```sh
export MY_IP=$(curl -fsS https://checkip.amazonaws.com)
echo $MY_IP

# ssh 연결을 위한 보안그룹 규칙 넣기
aws ec2 authorize-security-group-ingress \
    --group-id "$MY_SG_ID" --protocol tcp --port 22 --cidr "$MY_IP/32"
# 80 접속을 위한 것
aws ec2 authorize-security-group-ingress \
    --group-id "$MY_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

```sh
# 가장 최신의 리눅스 이미지(ubuntu 26.04) 버전을 확인
export BASE_AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/26.04/stable/current/arm64/hvm/ebs-gp3/ami-id \
  --query "Parameter.Value" --output text)
# 해당 이미지로 docker compose 정도의 실행이 가능한 인스턴스를 생성
export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$BASE_AMI_ID" --instance-type t4g.small \
  --key-name "$MY_KEY_NAME" --security-group-ids "$MY_SG_ID" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$MY_INSTANCE_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
  --query "Instances[0].InstanceId" --output text)
# 실행 여부를 감지
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"
# 실행되고 나서는 해당 생성된 인스턴스의 공인 IP(우리가 ssh, http 접속을 시도할 주소)를 환경변수화
export PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

echo $PUBLIC_IP
```
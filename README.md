### step0 환경변수 세팅 및 로그인
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

### step1 aws 기본 세팅
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

```sh
ssh -i ./"$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"
```

### step2 도커 구동

```sh
# 인스턴스 내부에서

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo systemctl is-active docker
sudo docker --version
sudo docker compose version
```

```sh
# 도커 이미지 확보
sudo docker pull nginx:alpine
export MY_APP_IMAGE="ghcr.io/a1l1ke/simple-back-ghcr:latest"
sudo docker pull "$MY_APP_IMAGE"
sudo docker images
```

### step3 RDS / ElastiCache 보안 규칙 및 서브넷 그룹 생성
```sh
export MY_DATA_SG_ID=$(aws ec2 create-security-group \
    --group-name "$MY_DATA_SG_NAME" \
    --description "RDS and ElastiCache access from EC2 SG" --vpc-id "$VPC_ID" \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=$MY_DATA_SG_NAME},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]" \
    --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress \
    --group-id "$MY_DATA_SG_ID" --protocol tcp --port 3306 --source-group "$MY_SG_ID"
aws ec2 authorize-security-group-ingress \
    --group-id "$MY_DATA_SG_ID" --protocol tcp --port 6379 --source-group "$MY_SG_ID"
```

```sh
# 서브넷
export SUBNET_IDS=($(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].SubnetId" --output text))
echo $SUBNET_IDS

aws rds create-db-subnet-group \
  --db-subnet-group-name "$MY_DB_SUBNET_GROUP" \
  --db-subnet-group-description "Default VPC subnets for RDS" \
  --subnet-ids "${SUBNET_IDS[@]}"

aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name "$MY_CACHE_SUBNET_GROUP" \
  --cache-subnet-group-description "Default VPC subnets for ElastiCache" \
  --subnet-ids "${SUBNET_IDS[@]}"
```

### step4 RDS & ElastiCache 인스턴스 생성
```sh
export MY_DB_PASSWORD=qwer1234!
echo $MY_DB_PASSWORD
```

```sh
aws rds create-db-instance \
  --db-instance-identifier "$MY_DB_ID" \
  --db-instance-class db.t4g.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password "$MY_DB_PASSWORD" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --vpc-security-group-ids "$MY_DATA_SG_ID" \
  --db-subnet-group-name "$MY_DB_SUBNET_GROUP" \
  --no-multi-az \
  --no-publicly-accessible \
  --backup-retention-period 0 \
  --tags Key=Name,Value="$MY_DB_ID" Key=Course,Value=infra-training Key=Owner,Value="$STUDENT_ID"
```
- https://948806325749-ticmxh4e.ap-northeast-2.console.aws.amazon.com/rds/home?region=ap-northeast-2#databases:
- https://aws.amazon.com/ko/ec2/instance-types/t4/
```sh
aws elasticache create-cache-cluster \
  --cache-cluster-id "$MY_CACHE_ID" \
  --cache-node-type cache.t4g.micro \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name "$MY_CACHE_SUBNET_GROUP" \
  --security-group-ids "$MY_DATA_SG_ID" \
  --tags Key=Name,Value="$MY_CACHE_ID" Key=Course,Value=infra-training Key=Owner,Value="$STUDENT_ID"
```

### step5 S3

```sh
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export MY_BUCKET="${STUDENT_ID}-app-assets-${ACCOUNT_ID}"
echo "S3 버킷 이름:$MY_BUCKET"
```

```sh
aws s3 mb "s3://$MY_BUCKET" --region "$AWS_REGION"
aws s3api put-bucket-tagging \
  --bucket "$MY_BUCKET" \
  --tagging "TagSet=[{Key=Name,Value=$MY_BUCKET},{Key=Course,Value=infra-training},{Key=Owner,Value=$STUDENT_ID}]"
```

- https://948806325749-ticmxh4e.ap-northeast-2.console.aws.amazon.com/s3/home?region=ap-northeast-2#

```sh
touch hello.txt
echo "Hello World" >> hello.txt
aws s3 cp hello.txt "s3://$MY_BUCKET/hello.txt"
aws s3 ls "s3://$MY_BUCKET/"
```

```sh
# 1. 300초(5분) 만료 Presigned URL 발급
export PRESIGNED_URL=$(aws s3 presign "s3://$MY_BUCKET/hello.txt" --expires-in 300)
echo "발급된 Presigned URL:$PRESIGNED_URL"

# 2. curl로 임시 서명 URL 다운로드 확인
curl -i -s "$PRESIGNED_URL"

# 3. 대조: 서명 없이 같은 객체를 직접 호출하면 차단됩니다
curl -i -s "https://${MY_BUCKET}.s3.${AWS_REGION}.amazonaws.com/hello.txt" | head -1
```

### step6 RDS / ElastiCache 생성 확인

```sh
# AWS RDS (RDBMS) MySQL
aws rds wait db-instance-available --db-instance-identifier "$MY_DB_ID"
```

```sh
export RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "$MY_DB_ID" \
  --query "DBInstances[0].Endpoint.Address" --output text)

export RDS_PORT=$(aws rds describe-db-instances \
  --db-instance-identifier "$MY_DB_ID" \
  --query "DBInstances[0].Endpoint.Port" --output text)

echo "확정된 RDS 엔드포인트:$RDS_ENDPOINT:$RDS_PORT"
```

```sh
echo "ElastiCache Redis 상태 폴링 시작..."
while [ "$(aws elasticache describe-cache-clusters \
  --cache-cluster-id "$MY_CACHE_ID" \
  --query "CacheClusters[0].CacheClusterStatus" --output text)" != "available" ]; do
  echo "현재 상태 대기 중... (15초 대기)"
  sleep 15
done
echo "ElastiCache Redis 프로비저닝 완료"
```

```sh
export REDIS_ENDPOINT=$(aws elasticache describe-cache-clusters \
  --cache-cluster-id "$MY_CACHE_ID" --show-cache-node-info \
  --query "CacheClusters[0].CacheNodes[0].Endpoint.Address" --output text)

export REDIS_PORT=$(aws elasticache describe-cache-clusters \
  --cache-cluster-id "$MY_CACHE_ID" --show-cache-node-info \
  --query "CacheClusters[0].CacheNodes[0].Endpoint.Port" --output text)

echo "확정된 ElastiCache 엔드포인트:$REDIS_ENDPOINT:$REDIS_PORT"
```

### step7 compose + RDS

```sh
ssh -i ./"$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"
```

```dotenv
# .env.rds
DB_NAME=appdb
DB_USER=admin
DB_PASSWORD=$MY_DB_PASSWORD
RDS_ENDPOINT=$RDS_ENDPOINT
APP_MESSAGE=Live on AWS EC2 Node 1 via Compose + RDS!
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_DATASOURCE_HIKARI_CONNECTIONTIMEOUT=30000
```

- https://948806325749-ticmxh4e.ap-northeast-2.console.aws.amazon.com/rds/home?region=ap-northeast-2#databases:
- mysql -> 3306 ... 주소를 받아와서 $RDS_ENPOINT
- $MY_DB_PASSWORD -> 위에 입력했던 것

```yaml
# compose.yml
name: aws-managed

services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    deploy:
      resources:
        limits:
          memory: 64M
    networks:
      - frontend-net

  app:
    image: ghcr.io/a1l1ke/simple-back-ghcr:latest
    container_name: spring-app
    restart: on-failure
    env_file:
      - .env.rds
    environment:
      PORT: 8080
      JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75.0"
      SPRING_DATASOURCE_URL: "jdbc:mysql://${RDS_ENDPOINT}:3306/${DB_NAME}?createDatabaseIfNotExist=true"
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    deploy:
      resources:
        limits:
          memory: 1024M
    networks:
      - frontend-net

networks:
  frontend-net:
```

```sh
# 리눅스 내부에서
# rm compose.yml
vi compose.yml
```

# step8 ElasticCache

- https://948806325749-ticmxh4e.ap-northeast-2.console.aws.amazon.com/elasticache/home?region=ap-northeast-2#/redis
```sh
export REDIS_ENDPOINT=student**-redis.******.0001.apn2.cache.amazonaws.com
export STUDENT_ID=student**
export REDIS_PORT=6379
echo ${REDIS_ENDPOINT} ${REDIS_PORT} ${STUDENT_ID}
```

```sh
set -e
sudo apt-get update -y
sudo apt-get install -y redis-tools
```

```sh
redis-cli -h "$REDIS_ENDPOINT" -p "$REDIS_PORT" ping
redis-cli -h "$REDIS_ENDPOINT" -p "$REDIS_PORT" \
  set "${STUDENT_ID}:ec2" "connected-from-ec2"
redis-cli -h "$REDIS_ENDPOINT" -p "$REDIS_PORT" \
  get "${STUDENT_ID}:ec2"
redis-cli -h "$REDIS_ENDPOINT" -p "$REDIS_PORT" \
  del "${STUDENT_ID}:ec2"
```
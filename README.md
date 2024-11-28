## VPC 생성

- 이름 : ash-test
- IPv4 CIDR 블록 : 192.168.0.0/16
- 가용 영역 수 : 2
- 퍼블릭 서브넷 수 : 2
- 프라이빗 서브넷 수 : 2
- NAT 게이트웨이 : 없음
- VPC 엔드포인트 : S3 게이트웨이

![VPC 생성 이미지](https://github.com/user-attachments/assets/41674902-8357-4dfb-a1e0-8c4dffd6f5a0)

## Bastion Host EC2 생성

- 이름 : bastion-host-ec2
- OS 이미지 : Amazon Linux 2023 AMI
- 인스턴스 유형 : t3.micro
- 키 페어 : 신규 키 페어 생성
- VPC : ash-test-vpc
- 서브넷 : ash-test-subnet-public1-ap-northeast-2a
- 보안 그룹 : bastion-host-sg
   
   - 인바운드 : `SSH`  `22`  `0.0.0.0/0`
 
   - 아웃바운드 : `모든 트래픽`  `전체`  `0.0.0.0/0`

![bastion-host-ec2-sg 생성 이미지](https://github.com/user-attachments/assets/847c4c1a-678b-4a60-abc2-deb6f82af722)

## EIP 연결

![bastion-host-ec2-eip 연결 이미지](https://github.com/user-attachments/assets/17d9f956-f9e6-454a-ad30-377c7978254b)

## EC2 Role 생성/연결

![bastion-host-ec2-role 생성 이미지](https://github.com/user-attachments/assets/b5fef33f-9d66-44ee-bc26-73140ae27da4)

![bastion-host-ec2-role 연결 이미지](https://github.com/user-attachments/assets/ec4094e2-57a9-4ce4-8880-06ce95feadb3)

## IAM Policy 생성

- 정책 이름 : ash-iam-user-policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ec2:ap-northeast-2:<ACCOUNT_ID>:instance/<INSTANCE_ID>"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession",
                "ssm:ResumeSession",
                "ssm:TerminateSession"
            ],
            "Resource": [
                "arn:aws:ssm:ap-northeast-2:<ACCOUNT_ID>:document/SSM-SessionManagerRunShell"
            ]
        }
    ]
}
```

## IAM User 생성

- 이름 : ash-iam-user
- 직접 정책 연결 : ash-iam-user-policy

![iam-user-access-key 생성 이미지](https://github.com/user-attachments/assets/080b5b42-39ec-411f-a574-bb426b73b1ff)

## SSM 연결 테스트

```bash
$ aws ssm start-session --target <INSTANCE_ID> --profile <AWS_PROFILE_NAME>
```

![스크린샷 2024-11-18 오후 4 35 50](https://github.com/user-attachments/assets/bd370a2b-7889-414e-bb6a-664956113f87)

## Aurora Mysql 파라미터 그룹 생성

- 파라미터 그룹 이름 : aurora-mysql-parameter-group
- 설명 : aurora-mysql-parameter-group
- 엔진 유형 : Aurora MySQL
- 파라미터 그룹 패밀리 : aurora-mysql8.0
- 유형 : DB Cluster Parameter Group

RDS > 파리미터 그룹 > 파라미터 그룹 수정:aurora-mysql-parameter-group

![스크린샷 2024-11-18 오후 7 30 43](https://github.com/user-attachments/assets/bcb43c38-e06d-4e38-ab14-09c51ccf1a3c)

- server_audit_events : CONNECT,QUERY,QUERY_DCL,QUERY_DDL,QUERY_DML,TABLE
- server_audit_excl_users : rdsadmin
- server_audit_logging : 1
- server_audit_log_upload : 1

## Aurora MySQL 생성

- 데이터베이스 생성 방식 - 표준 생성
- 엔진 유형 : Aurora (MySQL Compatible)
- 엔진 버전 : Aurora MySQL 3.05.2 (compatible with MySQL 8.0.32)
- DB 인스턴스 크기 : 개발/테스트
- DB 클러스터 식별자 : aurora-mysql-test
- 마스터 사용자 이름 : <사용자 입력>
- 자격 증명 관리 : 자체 관리
- 마스터 암호 : <사용자 입력>
- 클러스터 스토리지 구성 : Aurora Standard
- 인스턴스 구성 : db.r7g.large
- 가용성 및 내구성 : 다른 AZ에 Aurora 복제본/리더 노드 생성
- EC2 연결 설정 : bastion-host-ec2
- DB 서브넷 그룹 : 자동 설정
- VPC 보안 그룹 : 새로 생성
- 데이터베이스 인증 : 데이터베이스 인증
- 초기 데이터베이스 이름 : <사용자 입력>
- DB 클러스터 파라미터 그룹 : aurora-mysql-parameter-group
- 로그 내보내기
    - 감사 로그 : ✅
    - 에러 로그 : ✅
    - 일반 로그 : ✅
    - 느린 쿼리 로그 : ✅

**자격 증명 세부 정보 보기 > 마스터 사용자 이름, 마스터 암호, 엔드 포인트 확인**

![스크린샷 2024-11-18 오후 4 26 21](https://github.com/user-attachments/assets/ed2b81ba-bcc8-4652-b8af-0000dfd3f692)

**Aurora MySQL 생성 확인**

![스크린샷 2024-11-18 오후 4 36 59](https://github.com/user-attachments/assets/c85be01b-ede8-44d5-b694-0a9e0fec3687)

## RDS 접근 시 IAM 인증 토큰 사용하는 계정 생성

```bash
$ aws ssm start-session --target <INSTANCE_ID> --profile <AWS_PROFILE_NAME>
$ sudo dnf update -y
$ sudo dnf install mariadb105
$ mysql -h <AURORA_MSYQL_CLUSTER_ENDPOINT> -P 3306 -u <DB_ROOT_USER> -p
$ CREATE USER <USER_ID> IDENTIFIED WITH AWSAuthenticationPlugin as 'RDS';
$ ALTER USER <USER_ID> REQUIRE SSL;
```

## RDS 접근 정책 생성

CLUSTER_RESOURCE_ID = RDS > 데이터베이스 > aurora-mysql-database-test > 구성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds-db:connect"
      ],
      "Resource": [
        "arn:aws:rds-db:ap-northeast-2:<ACCOUNT_ID>:dbuser:<CLUSTER_RESOURCE_ID>/<DB_USER>"
      ]
    }
  ]
}
```

## IAM 인증 토큰 생성 / MySQL 연결 테스트

```bash
$ aws configure
aws_access_key_id = <ACCESS_TOKEN>
aws_secret_access_key = <ACCESS_TOKEN_SECRET>
region = ap-northeast-2
format = json

$ wget https://truststore.pki.rds.amazonaws.com/ap-northeast-2/ap-northeast-2-bundle.pem
$ TOKEN=$(aws rds generate-db-auth-token --hostname <AURORA_MSYQL_CLUSTER_ENDPOINT> --port 3306 --username <DB_USER>)
$ echo $TOKEN
$ mysql --host=<AURORA_MSYQL_CLUSTER_ENDPOINT> --ssl-ca=./ap-northeast-2-bundle.pem --port=3306 --user=<DB_USER> --password=$TOKEN
```

## MySQL 접속 시 CONNECT 로그 확인

RDS > 데이터베이스 > aurora-mysql-test > aurora-mysql-test-instance-1 (Writer) > 로그 및 이벤트

![스크린샷 2024-11-18 오후 7 33 03](https://github.com/user-attachments/assets/dbd50d42-b9be-44af-b1e2-5719aaec6e08)

## SSM Remote Port Forwarding 사용한 Localhost에서 RDS 연결

```bash
aws ssm start-session --target i-<INSTANCE_ID> --profile <AWS_PROFILE> \
                       --document-name AWS-StartPortForwardingSessionToRemoteHost \
                       --parameters '{"portNumber":["3306"],"localPortNumber":["3306"],"host":["<RDS_END_POINT>"]}'
```

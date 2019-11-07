# StorageGateway検証環境
StorageGateway(ファイルゲートウェイ)の検証環境を作成するCloudFormationと手順

# 作成環境
![CFn_configuration](./Documents/Arch.png)

# 作成手順
## (1)事前設定
### (a) 作業環境の準備
下記を準備します。
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定
### (b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
export REGION=ap-northeast-1
```
## (2)VPCの作成(CloudFormation利用)
IGWでインターネットアクセス可能で、パブリックアクセス可能なサブネットx3、プライベートなサブネットx3の合計6つのサブネットを所有するVPCを作成します。
### (a)テンプレートのダウンロード
私が作成し利用しているVPC作成用のCloudFormationテンプレートを利用します。まず、githubからテンプレートをダウンロードします。
```shell
curl -o vpc-2subnets.yaml https://raw.githubusercontent.com/Noppy/CfnCreatingVPC/master/vpc-2subnets.yaml
```
### (b)CloudFormationによるOn-Prem VPC作成
ダウンロードしたテンプレートを利用し、VPCをデプロイします。
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "OnPremVPC"
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "on-prem."
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.2.0.0/16"
  },
  {
    "ParameterKey": "Subnet1Cidr",
    "ParameterValue": "10.2.64.0/24"
  },
  {
    "ParameterKey": "Subnet2Cidr",
    "ParameterValue": "10.2.128.0/24"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name SGWPoC-OnPrem-VPC \
    --template-body "file://./vpc-2subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```
### (c)CloudFormationによるSGW-VPC作成
ダウンロードしたテンプレートを利用し、VPCをデプロイします。
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "SgwVPC"
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "sgw."
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "on-prem."
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.1.0.0/16"
  },
  {
    "ParameterKey": "Subnet1Cidr",
    "ParameterValue": "10.1.64.0/24"
  },
  {
    "ParameterKey": "Subnet2Cidr",
    "ParameterValue": "10.1.128.0/24"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name SGWPoC-Sgw-VPC \
    --template-body "file://./vpc-2subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```
## (3) Peering & VPCEndpoint設定
### (a) 構成情報取得
```shell
#構成情報取得
ONPRE_VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-OnPrem-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

ONPRE_VPC_CIDR=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-OnPrem-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

ONPRE_ROUTETABLEID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-OnPrem-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`RouteTableId`].[OutputValue]')

SGW_VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-Sgw-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

SGW_VPC_CIDR=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-Sgw-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

SGW_ROUTETABLEID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-Sgw-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`RouteTableId`].[OutputValue]')

SGW_SUBNET1=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-Sgw-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`Subnet1Id`].[OutputValue]')

SGW_SUBNET2=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-Sgw-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`Subnet2Id`].[OutputValue]')

echo $ONPRE_VPCID $ONPRE_VPC_CIDR $ONPRE_ROUTETABLEID $SGW_VPCID $SGW_VPC_CIDR $SGW_ROUTETABLEID $SGW_SUBNET1 $SGW_SUBNET2
```
### (b) VPCのPeering接続とルーティング設定
```shell
#Peering作成
PeeringID=$(aws --profile ${PROFILE} --output text \
    ec2 create-vpc-peering-connection \
        --peer-vpc-id ${ONPRE_VPCID} \
        --vpc-id  ${SGW_VPCID} \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' )

#Peeringの承諾
aws --profile ${PROFILE} \
    ec2 accept-vpc-peering-connection \
        --vpc-peering-connection-id ${PeeringID};

#ルートテーブルへルーティング追加(OnPremVPC)
aws --profile ${PROFILE} \
    ec2 create-route \
        --route-table-id ${ONPRE_ROUTETABLEID} \
        --destination-cidr-block ${SGW_VPC_CIDR} \
        --vpc-peering-connection-id ${PeeringID};

#ルートテーブルへルーティング追加(SgwVPC)
aws --profile ${PROFILE} \
    ec2 create-route \
        --route-table-id ${SGW_ROUTETABLEID} \
        --destination-cidr-block ${ONPRE_VPC_CIDR} \
        --vpc-peering-connection-id ${PeeringID};
```
### (c) VPCEndpointの作成
```shell
#VPC Endpoint用SecurityGroup作成
VPCENDPOINT_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name VpcEndpointSG \
        --description "Allow https" \
        --vpc-id ${SGW_VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${VPCENDPOINT_SG_ID} \
        --tags "Key=Name,Value=VpcEndpointSG" ;

aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_SG_ID} \
        --protocol tcp \
        --port 443 \
        --cidr 0.0.0.0/0 ;

#S3用VPCEndpoint作成
aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${SGW_VPCID} \
        --service-name com.amazonaws.${REGION}.s3 \
        --route-table-ids ${SGW_ROUTETABLEID}

#StorageGateway用VPCEndpoint作成
aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${SGW_VPCID} \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.${REGION}.storagegateway \
        --subnet-id $SGW_SUBNET1 $SGW_SUBNET2 \
        --security-group-id ${VPCENDPOINT_SG_ID} ;
```

## (4) Peering & VPCEndpoint設定













## (1)ベースの構築
### (1)-(a) デプロイ用シェルのプロファイル設定
デプロイ用のシェル(run_cfn.sh)に実行環境のプロファイルを設定します。
- run_cfn.shをエディタで開き下記を編集します。
```shell

# list of Environment
Envs[0]=PoC;  ProfileList[0]=XXXXXXX #実行する端末に設定しているプロファイル名に修正する(デフォルトを利用したい場合は、defaultと指定)
EnvsLast=0
```

### (1)-(c) EC2キーペア設定
"run_cfn.sh"があるディレクトリで下記コマンドを実行して、stackのパラメータ用JSONのキーペア設定を環境に合わせた内容に更新します。
```shell
KeyPairName="KEY_PAIR_NAME" #環境に合わせてキーペア名称を設定"

for i in ExterResources/InputParameter-SgPoC-Proxy.json ExterResources/InputParameter-SgPoC-Proxy.json ExterResources/InputParameter-SgPoC-Bastion.json FuncResources/InputParameter-SgPoC-Client.json FuncResources/InputParameter-SgPoC-Manager.json StorageGateway/InputParameter-SgPoC-Gateway1.json
do
    sed -i".bak" -e "s/KEY_PAIR_NAME/${KeyPairName}/g" ${i}
done
```
### (1)-(d) CloudFormationデプロイ 
```shell
./run_cfn.sh SgPoC Iam create
./run_cfn.sh SgPoC VpcFunc create
./run_cfn.sh SgPoC VpcExter create
./run_cfn.sh SgPoC VpcPeer create
./run_cfn.sh SgPoC ExterSg create
./run_cfn.sh SgPoC Bastion create
./run_cfn.sh SgPoC Proxy create
./run_cfn.sh SgPoC Vpce create
./run_cfn.sh SgPoC Sg create
./run_cfn.sh SgPoC S3 create

./run_cfn.sh SgPoC Client create
./run_cfn.sh SgPoC Manager create
```

## (2)StorageGatewayのVPC Endpointとファイルゲートウェイのインスタンスデプロイ
### (2)-(a) StorageGateway用 VPCE&Logsのデプロイ
```shell
./run_cfn.sh SgPoC StrageVpce create
./run_cfn.sh SgPoC Logs create
```
### (2)-(b) StorageGateway用 Gateway用EC2インスタンスのデプロイ

デプロイ前に、"/StorageGateway/InputParameter-SgPoC-Gateway1.json"で、AMI-IDとインスタンスタイプを確認し、必要に応じて修正します。
- AMI-ID: [StorageGatewayのユーザーガイドの](https://docs.aws.amazon.com/storagegateway/latest/userguide/ec2-gateway-file.html)「Amazon EC2 ホストで ファイルゲートウェイ をデプロイする」を参考に、最新のAMI-IDを設定します
- インスタンスタイプ: c5.4xlargeを設定しているので、必要に応じて変更してください。

```shell
./run_cfn.sh SgPoC Gateway1 create

```
## (3) ファイルゲートウェイのアクティベーション
ここからは、Managerインスタンス上でAWS CLIで実行します。
### (3)-(a) Managerインスタンスへのログイン
- マネージメントコンソールを開き、「Systems Manager」のサービスに移動する
- 左のバーから「セッションマネージャー」を選び、「セッションの開始」を押す
- インスタンス名「SgPoC-Manager」を選び、「セッションの開始」を押す
- ログインしたら下記を実行し、ec2-userになり、aws cliの初期設定を行う
```shell
sudo -u ec2-user -i

# AWS cli初期設定
Region=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
aws configure set region ${Region}
aws configure set output json

#FowardProxyIP=<FowardProxy IP Address>
FowardProxyIP=$(aws --output text cloudformation describe-stacks --stack-name SgPoC-Proxy --query 'Stacks[].Outputs[?OutputKey==`ProxyInstance2PrivateIp`].[OutputValue]')
FowardProxyPort=3128

#AWS Cliアップデート
curl -x http://${FowardProxyIP}:${FowardProxyPort} \
     -o "get-pip.py" \
     "https://bootstrap.pypa.io/get-pip.py" 
sudo python get-pip.py --proxy="http://${FowardProxyIP}:${FowardProxyPort}"

pip install --upgrade --user awscli --proxy="http://${FowardProxyIP}:${FowardProxyPort}"
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
. ~/.bashrc

```
### (3)-（b) アクティベーションキーの取得
ファイルゲートウェイから、 アクティベーションキーを取得します。
取得したアクティベーションキーは
```shell
Gateway1DNS=$(aws --output text ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --filters "Name=tag:Name,Values=SgPoC-Gateway-1" "Name=instance-state-name,Values=running")
Region=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
VPCEndpointDNSname=$(aws --output text ec2 describe-vpc-endpoints  --query 'VpcEndpoints[*].DnsEntries[0].DnsName' --filters "Name=service-name,Values=com.amazonaws.ap-northeast-1.storagegateway")

ACTIVATION_KEY=$(curl -s "http://${Gateway1DNS}/?gatewayType=FILE_S3&activationRegion=${Region}&vpcEndpoint=${VPCEndpointDNSname}&no_redirect")

# 確認のため、取得したアクティベーションキーを確認する
echo $ACTIVATION_KEY
```
参考
https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/gateway-private-link.html#GettingStartedActivateGateway-file-vpc

### (3)-（c) アクティベーション
ファイルゲートウェイをアクティベーションします。
```shell
aws storagegateway activate-gateway \
    --activation-key ${ACTIVATION_KEY} \
    --gateway-name SgPoC-Gateway-1 \
    --gateway-timezone "GMT+9:00" \
    --gateway-region ${Region} \
    --gateway-type FILE_S3

#作成したGatewayのARN取得
# atewayState"が "RUNNING"になるまで待つ
aws storagegateway list-gateways

#アクティベートしたゲートウェイの構成情報確認
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ match($0, /arn:aws:storagegateway:\S*/); print substr($0, RSTART, RLENGTH) }')
aws storagegateway describe-gateway-information --gateway-arn ${GATEWAY_ARN}

```
＜参考 gateway-typeの説明>
- "STORED" : VolumeGateway(Store type)
- "CACHED" : VolumeGateway(Cache tyep)
- "VTL"    : VirtualTapeLibrary
- "FILE_S3": File Gateway

### (3)-（d) ローカルディスク設定
ゲートウェイのローカルディスクが割り当てられます。ここでは、これらのディスクを使用するようにゲートウェイを設定します。
```shell

#ローカルストレージの確認
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ match($0, /arn:aws:storagegateway:\S*/); print substr($0, RSTART, RLENGTH) }')
DiskIds=$(aws --output text storagegateway list-local-disks --gateway-arn ${GATEWAY_ARN} --query 'Disks[*].DiskId'| sed -e 's/\n/ /')
echo ${DiskIds}

#ローカルストレージの割り当て
aws storagegateway add-cache \
    --gateway-arn ${GATEWAY_ARN} \
    --disk-ids ${DiskIds}

#ローカルストレージの確認
# "DiskAllocationType"が"CACHE STORAGE"で、"DiskStatus"が"present"であることを確認
aws --output text storagegateway list-local-disks --gateway-arn ${GATEWAY_ARN}
```
参照：https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/create-gateway-file.html

### (3)-(e) CloudWatchログ設定
```shell
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ match($0, /arn:aws:storagegateway:\S*/); print substr($0, RSTART, RLENGTH) }')
Region=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
AccuntId=$(curl -s http://169.254.169.254/latest/meta-data/identity-credentials/ec2/info|awk '/AccountId/{ i=gsub( /"/, "", $3); print $3}')
echo "GATEWAY_ARN=$GATEWAY_ARN  Region=${Region} AccuntId=${AccuntId}"

aws storagegateway update-gateway-information \
    --gateway-arn ${GATEWAY_ARN} \
    --cloud-watch-log-group-arn "arn:aws:logs:${Region}:${AccuntId}:log-group:StorageGW-Gateway-1"
```

### (3)-(e) TimeServer問題の回避策(Private Hosted-zoneを利用した対策)
別途作成

### (3)-(f) CloudWatch AlarmによるS3へのFileBackd完了通知設定
別途作成

## (４) ファイル共有の作成
とりあえず、「ゲストアクセスによる SMB ファイル共有」で簡易確認

参照:
- https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/GettingStartedCreateFileShare.html
- https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/CreatingAnSMBFileShare.html

### (4)-(a) SMB設定(SMBSecurityStrategy)
```shell
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{print $4}')

aws storagegateway update-smb-security-strategy \
    --gateway-arn ${GATEWAY_ARN} \
    --smb-security-strategy MandatoryEncryption
```
### (4)-(b) ゲストアクセス用の SMB ファイル共有を設定
```shell
PASSWORD="HogeHoge@"
aws storagegateway set-smb-guest-password \
    --gateway-arn ${GATEWAY_ARN} \
    --password ${PASSWORD}
```
### (4)-(c) SMBファイル共有
```shell
#情報取得
BUCKETARN=$(aws --output text cloudformation describe-stacks --stack-name SgPoC-S3 --query 'Stacks[*].Outputs[*].[OutputKey,OutputValue]' | awk -e '/StorageGatewayStoredS3BucketArn/{print $2}')
ROLE=$(aws --output text cloudformation describe-stacks --stack-name SgPoC-Iam --query 'Stacks[*].Outputs[*].[OutputKey,OutputValue]' |awk -e '/StorageGatewayBucketAccessRoleArn/{print $2}')
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{print $4}')
CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)
echo -e "BUCKET=${BUCKETARN}\nROLE=${ROLE}\nGATEWAY_ARN=${GATEWAY_ARN}\nCLIENT_TOKEN=${CLIENT_TOKEN}"

#実行
aws storagegateway create-smb-file-share \
    --client-token ${CLIENT_TOKEN} \
    --gateway-arn "${GATEWAY_ARN}" \
    --location-arn "${BUCKETARN}" \
    --role "${ROLE}" \
    --object-acl bucket-owner-full-control \
    --default-storage-class S3_STANDARD \
    --guess-mime-type-enabled \
    --authentication GuestAccess
```
## (5) クライアントからのマウントテスト
```shell
rem 共有接続と切断(ドライブレター割り当て)
net use E: \\＜GatewayのIP>\＜S３バケット名と同一> /user:<Gateway名称>\smbguest
net use
net use E: /delete

rem 共有接続と切断(UNC、IPアドレスでの接続)
net use \\＜GatewayのIP>\＜S３バケット名と同一> /user:<Gateway名称>\smbguest
net use
net use \\＜GatewayのIP>\＜S３バケット名と同一> /delete


rem 共有接続と切断(UNC、FQDNでの接続)
net use \\＜GatewayのFQDN＞\＜S３バケット名と同一> /user:<Gateway名称>\smbguest
net use
net use \\＜GatewayのFQDN＞\＜S３バケット名と同一> /delete
```
## (6)Managerインスタンスからのsshメンテナンスログテインテスト
ManagerインスタンスにSystem Managerでログインする
```shell
sudo -u ec2-user -i
# キーペアの秘密鍵設定
cat > ~/.ssh/id_rsa
<キーペアの秘密鍵を入力し、"ctr＋d"で終了>
chmod 600 ~/.ssh/id_rsa

#ゲートウェイのIP取得
Gateway1DNS=$(aws --output text ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --filters "Name=tag:Name,Values=SgPoC-Gateway-1" "Name=instance-state-name,Values=running")

#sshログイン
ssh admin@${Gateway1DNS}
```
Storage Gatewayのローカルコンソール
```
        AWS Storage Gateway - Configuration

        #######################################################################
        ##  Currently connected network adapters:
        ##
        ##  eth0: 10.1.167.130
        #######################################################################

        1: Configure HTTP Proxy
        2: Network Configuration
        3: Test Network Connectivity
        4: View System Resource Check (0 Errors)
        5: Command Prompt

        Press "x" to exit session

        Enter command:
```
参考
https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/ec2-local-console-fwg.html#EC2_MaintenanceGatewayConsole-fgw



## (7) ファイルゲートウェイのリフレッシュ

```shell

#FileShareARNの確認
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{print $4}')
aws storagegateway list-file-shares --gateway-arn ${GATEWAY_ARN}

FILE_SHARE_ARN="上記結果より対象となるファイル共有のARNを設定"
FOLDER_LIST="/ "    #設定例"/ /xxxx/ /yyyy/zzz/"。デフォルトは、”/"

# リフレッシュ実行
aws storagegateway refresh-cache \
    --file-share-arn ${FILE_SHARE_ARN} \
    --folder-list ${FOLDER_LIST}
```

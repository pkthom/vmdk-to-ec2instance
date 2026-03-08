# vmdk-to-ec2instance

vmdkファイルから、EC2インスタンスを作るやり方

https://community.cisco.com/kxiwq67737/attachments/kxiwq67737/5041-docs-security/1960/1/SA_Resource%20Connector%20deployment%20in%20AWS%20rev1.0.pdf

# 目次

- [②-1 Resource Connectorイメージのダウンロード](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-1-resource-connector%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%81%AE%E3%83%80%E3%82%A6%E3%83%B3%E3%83%AD%E3%83%BC%E3%83%89)
- [②-2 イメージをS3に格納](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-2-%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92s3%E3%81%AB%E6%A0%BC%E7%B4%8D)
- [②-3 VMインポートするためのIAMロール作成](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-3-vm%E3%82%A4%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AEiam%E3%83%AD%E3%83%BC%E3%83%AB%E4%BD%9C%E6%88%90)
- [②-4 VMインポート/AMI作成](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-4-vm%E3%82%A4%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88ami%E4%BD%9C%E6%88%90)
- [②-5 AMIからインスタンス起動](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-5-ami%E3%81%8B%E3%82%89%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E8%B5%B7%E5%8B%95)


# ②-1 Resource Connectorイメージのダウンロード

Cisco Secure Access がないので、適当なテスト用vmdkファイルをダウンロードする　→　https://cloud-images.ubuntu.com/jammy/current/

<img width="1294" height="300" alt="image" src="https://github.com/user-attachments/assets/588987b8-8c19-4a7e-8c76-c7a97e426fbb" />


# ②-2 イメージをS3に格納

まずAWSで、S3バケットを作る

<img width="1433" height="520" alt="image" src="https://github.com/user-attachments/assets/336ee214-9a76-462f-9b40-7cec8262e148" />

適当な名前をつける

<img width="1266" height="538" alt="image" src="https://github.com/user-attachments/assets/c5eb89a3-6ec5-462e-ac4b-e8526f521965" />

できた

<img width="1191" height="576" alt="image" src="https://github.com/user-attachments/assets/c92447bf-354e-4d66-a49b-a006c5cb3f6b" />

バケットができたら、vmdkをアップロードする

<img width="1057" height="536" alt="image" src="https://github.com/user-attachments/assets/c73a4fcd-086b-4c62-8f24-da5c06a56de4" />


# ②-3 VMインポートするためのIAMロール作成

awsコマンドを使えるようにしないといけない　→ AWSCLIをインストール：https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

※作業場所はどこでも良い（今回は家の適当なVMでやる）
```
ubuntu@test1:~$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
...
ubuntu@test1:~$ sudo apt install -y unzip
...
ubuntu@test1:~$ unzip awscliv2.zip
ubuntu@test1:~$ sudo ./aws/install
You can now run: /usr/local/bin/aws --version
ubuntu@test1:~$ aws --version
aws-cli/2.34.4 Python/3.13.11 Linux/6.8.0-101-generic exe/x86_64.ubuntu.24
```

自分のAWSアカウントと、作業場所（今回は　家のVM）を繋げたい　

→ AWSで、自分のIAMユーザーのアクセスキーを作成する

<img width="1502" height="461" alt="image" src="https://github.com/user-attachments/assets/a7ab236f-bb29-4b32-b424-eed16cb53bdd" />

<img width="1509" height="707" alt="image" src="https://github.com/user-attachments/assets/32fe50ef-5fb2-47f5-8dde-259779d33459" />

アクセスキー　と　シークレットアクセスキー　を控え、

家のVMに入力　→ aws configure 
```
ubuntu@test1:~$ aws configure
AWS Access Key ID [None]: ABC
AWS Secret Access Key [None]: ABC
Default region name [None]: ap-northeast-3
Default output format [None]: json
ubuntu@test1:~$ 
```

自分のAWSアカウント情報が以下で返ってくれば、AWSアカウントとの連携完了！
```
ubuntu@test1:~$ aws sts get-caller-identity
{
    "UserId": "ABC",
    "Account": "123456789",
    "Arn": "arn:aws:iam::123456789:user/pkthom"
}
ubuntu@test1:~$ 
```

### IAMロールを作る

S3に格納したVMDKをAMIにインポートするVM Importで使用するIAMロールを作成します
```
cat << 'EOF' > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vmie.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "vmimport"
        }
      }
    }
  ]
}
EOF
```
```
aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
```

できてる！

<img width="1339" height="341" alt="image" src="https://github.com/user-attachments/assets/fb5c8eac-6569-4b4f-a594-55292b42b6b4" />

### IAMポリシーを作る

```
cat << 'EOF' > role-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::pkthom-test",
                "arn:aws:s3:::pkthom-test/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:ModifySnapshotAttribute",
                "ec2:CopySnapshot",
                "ec2:RegisterImage",
                "ec2:Describe*"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```
```
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"
```

できてる！

<img width="1065" height="696" alt="image" src="https://github.com/user-attachments/assets/799bb1ec-66d8-44a7-af1e-ea56af30b0eb" />

# ②-4 VMインポート/AMI作成

```
cat << 'EOF' > containers.json
[
  {
    "Description": "test",
    "Format": "vmdk",
    "UserBucket": {
        "S3Bucket": "pkthom-test",
        "S3Key": "jammy-server-cloudimg-amd64.vmdk"
    }
  }
]
EOF
```
```
aws ec2 import-image --description "test" --disk-containers "file://containers.json"
```
※ ⚠️ description の文字列は合わせる

うまくいったのか　ステータス確認　→ 変換中、、、
```
ubuntu@test1:~$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-54be2e9c40116f35t
{
    "ImportImageTasks": [
        {
            "Description": "test",
            "ImageId": "",
            "ImportTaskId": "import-ami-54be2e9c40116f35t",
            "Progress": "9",
            "SnapshotDetails": [
                {
                    "DiskImageSize": 671000576.0,
                    "Format": "VMDK",
                    "Status": "active",
                    "UserBucket": {
                        "S3Bucket": "pkthom-test",
                        "S3Key": "jammy-server-cloudimg-amd64.vmdk"
                    }
                }
            ],
            "Status": "active",
            "StatusMessage": "converting",
            "Tags": []
        }
    ]
}
ubuntu@test1:~$ 
```

completed
```
ubuntu@test1:~$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-54be2e9c40116f35t
{
    "ImportImageTasks": [
        {
            "Architecture": "x86_64",
            "Description": "test",
            "ImageId": "ami-0a8c283c9a238f8a7",
            "ImportTaskId": "import-ami-54be2e9c40116f35t",
            "LicenseType": "BYOL",
            "Platform": "Linux",
            "SnapshotDetails": [
                {
                    "DeviceName": "/dev/sda1",
                    "DiskImageSize": 671000576.0,
                    "Format": "VMDK",
                    "SnapshotId": "snap-0e181020e22608625",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "pkthom-test",
                        "S3Key": "jammy-server-cloudimg-amd64.vmdk"
                    }
                }
            ],
            "Status": "completed",
            "Tags": []
        }
    ]
}
ubuntu@test1:~$ 

```

AMI ができた

<img width="1811" height="605" alt="image" src="https://github.com/user-attachments/assets/a9d38b89-b975-4689-972e-32aebe45639f" />

# ②-5 AMIからインスタンス起動

「AMIからインスタンス起動」をクリック

<img width="1809" height="566" alt="image" src="https://github.com/user-attachments/assets/a9bf6af1-eea7-4051-9e56-855f22bd2638" />

諸設定は　以下の通り
```
名前 -> 適当につける

キーペア　-> 無ければ「新しいキーペアの作成」

パブリック IP の自動割り当て -> 有効化

セキュリティグループ -> 「「自分のIP」からのSSHトラフィックを許可」 

あとは全部デフォルト
```
→ 「インスタンスを起動」

インスタンスできた

<img width="1618" height="289" alt="image" src="https://github.com/user-attachments/assets/fe606d20-7215-4080-b19d-d8b9fafc0280" />

できたインスタンスに、SSHしてみる

```
chmod 400 Downloads/aws.pem
ssh ubuntu@<<PUBLIC_IP>> -i Downloads/aws.pem
```
SSHできた
```
ubuntu@ip-172-31-0-137:~$ 
```

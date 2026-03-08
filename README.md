# vmdk-to-ec2instance

これの練習　↓

https://community.cisco.com/kxiwq67737/attachments/kxiwq67737/5041-docs-security/1960/1/SA_Resource%20Connector%20deployment%20in%20AWS%20rev1.0.pdf

# 目次

- [テスト用vmdkを用意する]()
- [vmdkをS3にアップロードする]()
- [AMIを作る]()
- [IAMユーザーを作る]()
- [IAMロール・ポリシーを作る]()
- []()
- []()
- []()
- []()

# ②-1 Resource Connectorイメージのダウンロード

Cisco Secure Access がないので、一旦テスト用にvmdkファイルを作る

まずは適当なrawファイルをダウンロードする　→　https://download.cirros-cloud.net/0.6.3/

<img width="787" height="719" alt="image" src="https://github.com/user-attachments/assets/eda8ead0-df1d-4ce3-9fa4-72f19a569b80" />

それをローカルでvmdkに変換する
```
root@pve:~# qemu-img convert -f raw cirros-0.6.3-x86_64-disk.img -O vmdk test.vmdk
```

# ②-2 イメージをS3に格納

まずAWSで、S3バケットを作る

<img width="1433" height="520" alt="image" src="https://github.com/user-attachments/assets/336ee214-9a76-462f-9b40-7cec8262e148" />

<img width="1266" height="538" alt="image" src="https://github.com/user-attachments/assets/c5eb89a3-6ec5-462e-ac4b-e8526f521965" />

<img width="1191" height="576" alt="image" src="https://github.com/user-attachments/assets/c92447bf-354e-4d66-a49b-a006c5cb3f6b" />

バケットができたら、vmdkをアップロードする

<img width="1742" height="860" alt="image" src="https://github.com/user-attachments/assets/8dd6cbb7-8c87-47d6-815b-d3cb0e958013" />

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

自分のAWSアカウントと、家のVMを繋げたい　→ AWSで、自分のIAMユーザーのアクセスキーを作成する

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


```
cat << 'EOF' > containers.json
[
  {
    "Description": "test",
    "Format": "vmdk",
    "UserBucket": {
        "S3Bucket": "pkthom-test",
        "S3Key": "test.vmdk"
    }
  }
]
EOF
```
```
aws ec2 import-image --description "test" --disk-containers "file://containers.json"
```
※ ⚠️ description の文字列は合わせる

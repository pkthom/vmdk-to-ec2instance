# vmdk-to-ec2instance

vmdkファイルから、EC2インスタンスを作るやり方

https://community.cisco.com/kxiwq67737/attachments/kxiwq67737/5041-docs-security/1960/1/SA_Resource%20Connector%20deployment%20in%20AWS%20rev1.0.pdf

# 目次

- [②-1 Resource Connectorイメージのダウンロード](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-1-resource-connector%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%81%AE%E3%83%80%E3%82%A6%E3%83%B3%E3%83%AD%E3%83%BC%E3%83%89)
- [②-2 イメージをS3に格納](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-2-%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92s3%E3%81%AB%E6%A0%BC%E7%B4%8D)
- [②-3 VMインポートするためのIAMロール作成](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-3-vm%E3%82%A4%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AEiam%E3%83%AD%E3%83%BC%E3%83%AB%E4%BD%9C%E6%88%90)
- [②-4 VMインポート/AMI作成](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-4-vm%E3%82%A4%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88ami%E4%BD%9C%E6%88%90)
- [②-5 AMIからインスタンス起動](https://github.com/pkthom/vmdk-to-ec2instance/blob/main/README.md#-5-ami%E3%81%8B%E3%82%89%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E8%B5%B7%E5%8B%95)

### おまけ（EC2インスタンスをDNATサーバーにする）

- [DNAT先サーバーを、ファイルサーバーにする]()
- [EC2を、DNATサーバーにする]()
- [ローカルから、EC2経由で、目的地のファイルを覗く]()


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

# DNAT先サーバーを、ファイルサーバーにする

以下のように、EC2経由で、目的地のファイルを覗きたい
```
ローカル -> DNAT(EC2) -> 目的地(ファイルサーバ)
```

VMがあれば、サンバを入れる
```
sudo apt update
sudo apt install -y samba
mkdir ~/test_share
chmod 777 ~/test_share ※ゲストユーザーに好き勝手させるため、777
chmod 755 /home/ubuntu ※デフォだと750なので、ゲストユーザーが通過できない
sudo bash -c 'cat << EOF >> /etc/samba/smb.conf
[test]
   path = /home/ubuntu/test_share
   read only = no
   guest ok = yes
EOF'
sudo systemctl restart smbd

touch aaa
```



# EC2を、DNATサーバーにする

iptables-persistent をインストールすると、netfilter-persistentも入る
```
ubuntu@test1:~$ sudo apt update
ubuntu@test1:~$ sudo apt install -y iptables-persistent 
ubuntu@test1:~$ dpkg -l | grep persistent
ii  iptables-persistent             1.0.20                                  all          boot-time loader for netfilter rules, iptables plugin
ii  netfilter-persistent            1.0.20                                  all          boot-time loader for netfilter configuration
```

<img width="1162" height="515" alt="image" src="https://github.com/user-attachments/assets/296f1e9b-24b9-44e3-b3b6-46e2b4bd6c29" />

上記で、自動でファイル作るが、今ルールないので、空である

```
ubuntu@test1:/etc/iptables$ cat rules.v4
ubuntu@test1:/etc/iptables$ cat rules.v6
ubuntu@test1:/etc/iptables$ 
```

DNATの設定を入れる
```
ubuntu@test1:/etc/iptables$ cat rules.v4
# =============================================================
# ネットワークアドレス変換 (NAT) 設定
# 目的: EC2への通信を、目的地へ「転送」する
# =============================================================
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# 1. [外からの通信] Mac等が「EC2」の445番(SMB)に送ったパケットを、中継先の「目的地」へ書き換える
-A PREROUTING -d <<EC2のグローバルIP>> -p tcp --dport 445 -j DNAT --to-destination <<目的地グローバルIP>>:445

# 2. [自分からの通信] EC2自身が「EC2」にアクセスした場合も、中継先の「目的地」へ飛ばす
# -A OUTPUT -d <<EC2のグローバルIP>> -p tcp --dport 445 -j DNAT --to-destination <<目的地グローバルIP>>:445

# 3. [帰り道の確保] 目的地へ送る際、差出人をEC2に化かす (MASQUERADE)
# これにより、目的地からの返信が必ずEC2を経由してMacに戻るようになる
-A POSTROUTING -d <<目的地グローバルIP>> -p tcp --dport 445 -j MASQUERADE

COMMIT

# =============================================================
# パケットフィルタリング設定
# 目的: 転送(FORWARD)の「通行許可」を出す
# =============================================================
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# 4. [通行許可] 目的地の445番に向かう「通りすがり」のパケットを許可する
-A FORWARD -d <<目的地グローバルIP>> -p tcp --dport 445 -j ACCEPT

# 5. [接続維持] すでに許可された通信の「続きのパケット」は無条件で通す (顔パス設定)
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

COMMIT
ubuntu@test1:/etc/iptables$ 
```


```
ubuntu@test1:/etc/iptables$ sudo iptables-restore --test /etc/iptables/rules.v4
ubuntu@test1:/etc/iptables$ sudo service netfilter-persistent reload 
 * Loading netfilter rules...                                                                                                                                      run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables start
run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables start
                                                                                                                                                            [ OK ]
ubuntu@test1:/etc/iptables$ sudo iptables -nvL
Chain INPUT (policy ACCEPT 901 packets, 62980 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     6    --  *      *       0.0.0.0/0            <<目的地のグローバルIP>>        tcp dpt:445
    0     0 ACCEPT     0    --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 453 packets, 40408 bytes)
 pkts bytes target     prot opt in     out     source               destination         
ubuntu@test1:/etc/iptables$
```

DNATを有効化する
```
ubuntu@test1:/etc/iptables$ sudo vi /etc/sysctl.conf
net.ipv4.ip_forward=1    -> この行をコメントアウトする
```
設定反映
```
ubuntu@test1:/etc/iptables$ sudo sysctl -p
net.ipv4.ip_forward = 1
```
ローカルから、EC2に445が通るようになったか確認
```
nc -vz EC2のグローバルIP 445
```

# ローカルから、EC2経由で、目的地のファイルを覗く

EC2のパブリックIP宛に、目的地のファイルを要求してみる

# vmdk-to-ec2instance

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

# テスト用vmdkを用意する

vmdkファイルがないので作りたい

まずは適当なrawファイルをダウンロードする　→　https://download.cirros-cloud.net/0.6.3/

<img width="787" height="719" alt="image" src="https://github.com/user-attachments/assets/eda8ead0-df1d-4ce3-9fa4-72f19a569b80" />

それをローカルでvmdkに変換する
```
root@pve:~# qemu-img convert -f raw cirros-0.6.3-x86_64-disk.img -O vmdk test.vmdk
```

# vmdkをS3にアップロードする

まず、S3バケットを作る

<img width="1433" height="520" alt="image" src="https://github.com/user-attachments/assets/336ee214-9a76-462f-9b40-7cec8262e148" />

<img width="1266" height="538" alt="image" src="https://github.com/user-attachments/assets/c5eb89a3-6ec5-462e-ac4b-e8526f521965" />

<img width="1191" height="576" alt="image" src="https://github.com/user-attachments/assets/c92447bf-354e-4d66-a49b-a006c5cb3f6b" />

vmdkをアップロードする

<img width="1742" height="860" alt="image" src="https://github.com/user-attachments/assets/8dd6cbb7-8c87-47d6-815b-d3cb0e958013" />

# 作業マシンを用意する

作業場所は、ローカルでもEC2インスタンスでも、AWSCLIが使えればどこでも良い

今回は家のVMでやる

まずAWSCLIをインストール　→ https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

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

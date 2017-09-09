# 今日やること
1. ローカルでRailsアプリを動かしてみる。
2. AWS上でサンプルを動かすための準備
3. EC2上でサンプルアプリを動かしてみる
4. Dockerでサンプルを動かしてみる
5. （時間があったら）ECSでサンプルを動かしてみる

---

# セッション1 ローカルでRailアプリを動かしてみる。

以下の1章をやっていきます
https://railstutorial.jp/

---

# セッション2　事前準備

## 作成するもの
- クーポン登録
- IAMの権限準備の準備
- 作業用EC2 （ローカルで実施する人は不要）

---

## IAMの権限準備の準備
### EC2ロールの準備
作業用EC2に割り当てるロールを作成します。
以下の権限を付与して下さい。

管理ポリシー
AmazonEC2ContainerRegistryFullAccess
AmazonEC2ContainerServiceFullAccess
AmazonEC2FullAccess
CloudWatchFullAccess

信頼関係
ec2.amazonaws.com

-----

## 作業用EC2の準備

作業用のEC2から今回の作業は進めます。
作る上で以下の設定を入れて下さい。
・動作確認のためにパブリックIPを付与して下さい。
・SGは開けすぎないようにしてください。
・使用するAMIは以下のAmazonLinuxの最新のAMIを使います。

---

# セッション3 RoR のサンプルを動かしてみましょう
RoR環境を作って実際に動かしてみます。

作成したEC2上で以下のコマンドを叩いていくとrailsの環境が出来ます（はず）

---

```
# 必須パッケージのインストール
sudo yum install -y gcc gcc-6 bzip2 openssl-devel libyaml-devel libffi-devel readline-devel zlib-devel gdbm-devel ncurses-devel sqlite-devel git w3m 
# rbenvのセットアップ
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-buil
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
source ~/.bash_profile
# RoR環境のセットアップ
rbenv install 2.4.1
rbenv local 2.4.1
gem install rails
# サンプルを動かす
rails new test_app
cd test_app 

rails server -b 0.0.0.0

### JSのエラーがでたら以下を実行
sudo yum install --enablerepo=epel nodejs
```

--- 

## ALBをAWSコンソールから作成する
・ターゲットは3000ポート
・EC2と通信できるようにする

--- 

##  RoRのサンプルを動かす
```
cd ~/
git clone  https://github.com/yasslab/sample_apps.git
cd sample_apps/5_1_2/ch14
bundle install --without production
rails db:migrate
rails test
rails server -b 0.0.0.0
```

---

# セッション4 Docker でサンプルRoRアプリを動かしてみる

```
sudo yum install -y docker
sudo service docker start
```

```
FROM ruby:2.3.3
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs
RUN mkdir /myapp
WORKDIR /myapp
ADD ./ch14/Gemfile /myapp/Gemfile
ADD ./ch14/Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
ADD ./ch14 /myapp
RUN rails db:migrate
CMD ["rails","server","-b","0.0.0.0"]
```

```
docker build -t demo_app .
```

ここまででRoRのサンプルが動くDocker イメージができました。


---
# セッション5 ECS で動かしてみよう
以下の作業は作業用のEC2から行います。
## 変数の設定
```
export Cluster_Name=kumamoto_jawsug_`date +%y%m%d` 

# 個別設定いれる場合は以下も設定
export Aws_Region=ap-northeast-1
echo Cluster_Name=${Cluster_Name}
```
-----
## ECSクラスタの作成
```
sudo aws --region ${Aws_Region} \
ecs create-cluster --cluster-name ${Cluster_Name}
```
-----
## クラスタ用のEC2準備
http://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ecs-optimized_AMI_launch_latest.html

使うのは最新のAMIです。
構築はGUIで行います
その時にUser dataに以下を追加します。
```
#!/bin/bash
echo ECS_CLUSTER= ${Cluster_Name} >> /etc/ecs/ecs.config
```
※ ${Cluster_Name} は一個前の手順で作ったものをコピペして下さい。

-----
## ec2の起動

GUIで実施

-----
## ECRの作成

```
export ECR_Rep_Name=kumamoto-jawsug-`date +%y%m%d` 

sudo aws --region ${Aws_Region} \
ecr create-repository  --repository-name ${ECR_Rep_Name}
```

-----
## ECRのログイン情報の取得

```
sudo aws --region ${Aws_Region} \
ecr get-login --no-include-email > ./docker_login.sh
cat ./docker_login.sh
```

-----
## CWL Group Create
```
export CWL_Group=kumamoto_jawsug_`date +%y%m%d` 

sudo aws --region ${Aws_Region} \
logs create-log-group --log-group-name ${CWL_Group}
```

-----
### ECRプッシュするための準備

```
export ECR_Image_Tag_Name=sample_app
export ECR_Image_Tag_Num=0.1 
export ECR_Url=`aws --region ${Aws_Region} ecr get-login \
| cut -d " " -f 9 \
| sed 's/https:\/\///'`


cat << EXT
ECR_Url=${ECR_Url}
ECR_Rep_Name=${ECR_Rep_Name}
ECR_Image_Tag_Name=${ECR_Image_Tag_Name}
ECR_Image_Tag_Num=${ECR_Image_Tag_Num}
EXT
```

---
## docker push 
```
sudo sh ./docker_login.sh
sudo docker tag demo_app:latest ${ECR_Url}/${ECR_Rep_Name}:${ECR_Image_Tag_Name}-${ECR_Image_Tag_Num} 
sudo docker push ${ECR_Url}/${ECR_Rep_Name}:${ECR_Image_Tag_Name}-${ECR_Image_Tag_Num} 

```

---
# ECS ECRの準備  
## Register ECS Task

```
export Task_Name=sample_app
export ECS_Task_Memory=1000
export Logs_Stream_Prefix=sampleapp-0.1
export Aws_Region=ap-northeast-1

cat << EXT
Task_Name=${Task_Name}
ECS_Task_Memory=${ECS_Task_Memory}
Logs_Stream_Prefix=${Logs_Stream_Prefix}
Aws_Region=${Aws_Region}
ECR_Url=${ECR_Url}
ECR_Rep_Name=${ECR_Rep_Name}
EXT
```
---
## Register ECS Task
```
cat <<EOT>sample.json
{
    "family": "${Task_Name}",
    "volumes": [
        {
            "name": "my-vol",
            "host": {}
        }
    ],
    "containerDefinitions": [
        {
            "name": "test-ror-app",
            "image": "${ECR_Url}/${ECR_Rep_Name}:${ECR_Image_Tag_Name}-${ECR_Image_Tag_Num}",
            "memory": ${ECS_Task_Memory},
            "portMappings": [
              {
                "containerPort": 3000,
                "hostPort": 3000
              }
            ],
            "logConfiguration": {
                    "logDriver": "awslogs",
                    "options": {
                      "awslogs-group": "${CWL_Group}",
                      "awslogs-region": "${Aws_Region}",
                      "awslogs-stream-prefix": "${Logs_Stream_Prefix}"
                    }
            }
        }
    ]
}
EOT

cat sample.json
```
---
## Register ECS Task
```
sudo aws --region ${Aws_Region} \
ecs register-task-definition --cli-input-json file://sample.json
```

---

## ALB Setting
GUIで設定します。

以上でECS上でRoRのサンプルが動くかと思います。

ちなみに、ECRのログインファイルを作成しましたが、現状ではログイン時に使用したファイルを使えば
どこからでもイメージがPush/Pullできます。

```
./docker_login.sh
sudo docker pull ${ECR_Url}/${ECR_Rep_Name}:${ECR_Image_Tag_Name}-${ECR_Image_Tag_Num} 
```
そのため、ハンズオンではやりませんでしたが、ECRにはアクセス制限をしましょう。

以上です。

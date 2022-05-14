---
title: "docker composeを使って負荷試験環境をECS Fargate上に最速で立ち上げる"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "ecs", "fargate", "locust", "dockercompose"]
published: false
---
# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。AWSに構築したシステムの負荷試験を行うために、AWS上に負荷試験環境を構築しました。AWS上で立ち上げることで十分に負荷をかけることができ、低コストで実現できますが、設定ファイルを準備するのは少し手間です。そこで、docker composeを利用し、短期間でECS Fargateにクラスタを構築する方法を紹介します。

負荷試験については無知なため、以下の本を参考にさせていただき、Locustを導入してみました。

[Amazon Web Services負荷試験入門―クラウドの性能の引き出し方がわかる](https://www.amazon.co.jp/Amazon-Services%E8%B2%A0%E8%8D%B7%E8%A9%A6%E9%A8%93%E5%85%A5%E9%96%80-%E2%80%95%E2%80%95%E3%82%AF%E3%83%A9%E3%82%A6%E3%83%89%E3%81%AE%E6%80%A7%E8%83%BD%E3%81%AE%E5%BC%95%E3%81%8D%E5%87%BA%E3%81%97%E6%96%B9%E3%81%8C%E3%82%8F%E3%81%8B%E3%82%8B-Software-Design-ebook/dp/B075SV3VN3)

# ご参考
以下の２つの記事にdocker composeでの立ち上げ方が詳しく記載されています。これらの記事を参考にdocker composeでの立ち上げを試しています。

https://aws.amazon.com/jp/blogs/news/automated-software-delivery-using-docker-compose-and-amazon-ecs/

https://aws.amazon.com/jp/blogs/news/migrating-from-docker-swarm-to-amazon-ecs-with-docker-compose/

# 設定ファイルの準備

まずは、Dockerfileを準備します。

```Dockerfile:Dockerfile
FROM locustio/locust:latest

COPY ./ /mnt/locust
```

Locustで実行するPythonファイルを準備します。

```python:locustfile.py
from locust import HttpUser, task

class HelloWorldUser(HttpUser):
    @task
    def hello_world(self):
        self.client.get("/hello")
        self.client.get("/world")
```

dockerコマンドを利用してビルドし、AWS CLIを用いてECRにイメージをプッシュします。

```bash
#!/bin/sh

docker context use default

aws ecr get-login-password --region ${AWS_REGION}| docker login --username AWS --password-stdin ${ECR_URI}

for SERVICE in locust filebeat;
do
  docker image build -t ${ECR_URI}:${SERVICE} ${SERVICE}/
  docker image push ${ECR_URI}:${SERVICE}
done
```

`docker-compose.yaml`ファイルにLocustを設定します。`x-aws-*`を設定することで、VPCやClusterを指定でき、さまざまな環境に合わせて柔軟に対応できます。設定できる内容はDockerのドキュメントに記載されています。

https://docs.docker.com/cloud/ecs-integration/#using-existing-aws-network-resources

```yaml:docker-compose.yaml
version: '3'

x-aws-vpc: ${AWS_VPC}

services:
  master:
    image: ${ECR_URI}:locust
    ports:
      - "8089:8089"
    command: -f /mnt/locust/locustfile.py --master -H http://master:8089
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '2'
          memory: 4096M
  worker:
    image: ${ECR_URI}:locust
    command: -f /mnt/locust/locustfile.py --worker --master-host master
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 4096M
```

# ECS Fargateへのデプロイ
ECS用のコンテキストを作成します。

```bash
# demo_ecs_contextを作成
$ docker context create ecs demo_ecs_context
```

作成したコンテキストに切り替えます。
```bash
# demo_ecs_contextを使用
$ docker context use demo_ecs_context
# contextを元に戻す
$ docker context use default
# context一覧を確認する
$ docker context ls
# 作成したcontextを削除する
$ docker context rm demo_ecs_context
```

切り替えたコンテキストで起動すると、CloudFormationの実行が開始します。CloudFormationのコンソールから起動状態が確認できます。ECSだけで無くCloudMap、ELB、SecurityGroupなども自動で生成されるため、細かい設定が不要です。

```bash
$ docker compose up
```

# うまく立ち上がらない時
この方法は、パブリックサブネットが２つ以上必要であることや、同じアベイラビリティゾーンに複数のサブネットがあることなどの制約があり、うまく立ち上げることができない場合があります。

docker composeにはconvertコマンドが用意されており、エラーがなければ生成されるCloudFormationファイルをファイルに出力することができます。別のVPCなどで立ち上げてみて正常に立ち上がれば、convertコマンドでCloudFormationファイルを生成し、生成されたCloudFormationファイルを編集してCloudFormationで直接立ち上げてみることが簡単に調整できる方法です。

```bash
$ docker compose convert
```

# 最後に
docker composeを利用して、最速でECS Fargate上にLocustを作成することができました。ぜひお試しください。

![](/images/article-0006/locust-monitoring.png)
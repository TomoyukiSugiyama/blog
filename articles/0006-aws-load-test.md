---
title: "docker composeを使ってlocustをECS Fargate上に最速で立ち上げる"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "ecs", "fargate", "locust", "dockercompose"]
published: false
---
# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。AWSに構築したシステムの負荷試験を行うために、AWS上で負荷試験環境を構築しました。
docker composeを利用することで、短期間でECS Fargateにクラスタを構築することができます。構築の仕方を紹介します。

負荷試験については無知なため、以下の本を参考にさせていただき、Locustを導入してみました。

[Amazon Web Services負荷試験入門―クラウドの性能の引き出し方がわかる](https://www.amazon.co.jp/Amazon-Services%E8%B2%A0%E8%8D%B7%E8%A9%A6%E9%A8%93%E5%85%A5%E9%96%80-%E2%80%95%E2%80%95%E3%82%AF%E3%83%A9%E3%82%A6%E3%83%89%E3%81%AE%E6%80%A7%E8%83%BD%E3%81%AE%E5%BC%95%E3%81%8D%E5%87%BA%E3%81%97%E6%96%B9%E3%81%8C%E3%82%8F%E3%81%8B%E3%82%8B-Software-Design-ebook/dp/B075SV3VN3)


```yaml
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
      replicas: 4
      resources:
        limits:
          cpus: '2'
          memory: 4096M
```
# 最後に
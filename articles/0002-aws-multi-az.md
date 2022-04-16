---
title: "AWSにおける信頼性を考慮した構成の実践（社内システム）"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "CloudFormation"]
published: false
---
# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。工場のデータを見える化するため、AWS上にシステムを構築しています。

設計の原則にベストプラクティスとして記載されているとおり、単一の障害がワークロード全体に影響しないように、水平方向にスケールし可用性を高めていきます。

https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/reliability-pillar/design-principles.html

# 信頼性を考慮した構成
DirectConnectを利用し、オンプレ環境とAWS間を接続しています。社内限定のシステムのため、Internet Gatewayは利用せずPrivate Subnetに閉じています。インターネットへの接続が必要なサービスは全てPrivate Linkで接続しています。

考慮した以下のポイントについて紹介します。
* ELB(ALB, NLB)を配置し、


![](/images/article-0002/elastic-stack-on-aws-architecture.png)

# ELBを用いた
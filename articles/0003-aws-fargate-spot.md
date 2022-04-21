---
title: "Lambda(golang)でFargate Spotの終了通知を受けたECSのタスクをNLBから切り離す"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, lambda, fargate, ecs, cloudformation]
published: true
---
# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。今回は、Fargate spot上のECSに中断通知が来た時のELBに対するDeregisterTargetの実行が保証されない課題に対して取り組みました。

Fargate Spotを利用する目的は、以下の資料にあるように、コスト削減が可能になるからです。
また、Fargateで立ち上げたECSのCapacity ProviderにFargateとFargate Spotを併用することで、システムの安定性とコスト削減を両立した仕組みを実現します。

https://d1.awsstatic.com/webinars/jp/pdf/services/202109_AWS_Black_Belt_Containers303-ECS-Spot-Fargate.pdf

# Capacity Providerの導入
参考実装：

https://github.com/TomoyukiSugiyama/ElasticStack/pull/35/files#diff-e294b70d2a4a1e225aa9f6a19dc7fff8942e8fa32698e9b7ca742e1703862c89R39

Clusterのデフォルト値、Serviceに実際に使用するCapacity Providerを設定します。Baseは最小実行タスク数を示し、以下の例では２つのタスクがFargateで実行されます。Weightは実行するタスクの総数に対する相対的な割合を示し、スケールアウトした場合でも、1対1になるようにFargateとFargate Spot上にタスクが配置されます。AWSコンソールからは、各タスクの設定を確認するとどちらを利用しているか確認できます。

注意するポイントは、ServiceにCapacityProviderStrategyとLaunchTypeを両方設定することは出来ないことです。

```yaml
  Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: ecs
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Base: 2
          Weight: 1
        - CapacityProvider: FARGATE_SPOT
          Base: 0
          Weight: 1
  Service:
    Type: AWS::ECS::Service
    Properties:
      CapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Base: 2
          Weight: 1
        - CapacityProvider: FARGATE_SPOT
          Base: 0
          Weight: 1
```

デフォルトで SIGTERM の 30 秒後に SIGKILL が発⾏されますが、 StopTimeout を設定することで120秒などの設定に変更できます。

```yaml
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Name: logstash
          StopTimeout: 120
```

# Fargate Spotの終了通知をLambdaで処理する
参考実装：

https://github.com/TomoyukiSugiyama/ElasticStack/pull/35/files#diff-5151c8e56ed376b508fdfd4b764031a5b206765554fa7715598262df048e2f57R1

Event Bridgeから終了通知を受け取り、対象のタスクをNLBのターゲットから解除します。

1. Event Bridgeからイベントを受け取り予め用意した構造体に必要なデータを格納
2. StopCodeが終了通知でなければ、Lambdaを終了
3. 対象タスクのIPを取得
4. 環境変数に設定したAWS::ElasticLoadBalancingV2::LoadBalancerのAmazon Resource Name (以下、ARN)、AWS::ElasticLoadBalancingV2::TargetGroupのARNを取得
5. aws-lambda-go、aws-sdk-go-v2のライブラリを使用するための初期設定
6. LoadBalancerのARNからLoadBalancerを取得
7. TargetGroupのARNからTargetGroupを取得
8. TargetGroupに対象タスクのIPがあれば登録を解除

```go
type NetworkInterface struct {
	PrivateIpv4Address string `json:"privateIpv4Address"`
}
type Container struct {
	NetworkInterfaces []NetworkInterface
	Name              string `json:"name"`
}
type EcsEvent struct {
	StopCode   string `json:"stopCode"`
	Containers []Container
}

func HandleLambdaEvent(_ context.Context, event events.CloudWatchEvent) {
	// 1. Event Bridgeからイベントを受け取る
	var ecsEvent EcsEvent
	if err := json.Unmarshal(event.Detail, &ecsEvent); err != nil {
		os.Exit(1)
	}
	// 2. 終了通知かチェック
	fmt.Printf("stopCode = %s\n", ecsEvent.StopCode)
	if ecsEvent.StopCode != "TerminationNotice" {
		return
	}
	// 3. 対象タスクのIP取得
	var ecsIp string
	for _, contaier := range ecsEvent.Containers {
		if contaier.Name == "logstash" {
			for _, ni := range contaier.NetworkInterfaces {
				ecsIp = ni.PrivateIpv4Address
			}
		}
	}
	fmt.Printf("ip v4 = %s\n", ecsIp)
	// 4. 環境変数取得
	nlbId := os.Getenv("NlbId")
	nlbTargetGroupId := os.Getenv("NlbTargetGroupId")
	fmt.Printf("GET ENV AlbId: %s AlbTargetGroupId: %s\n", nlbId, nlbTargetGroupId)
	// 5. 初期設定
	svc := Init()
	// 6. 指定したLoadbalancerを取得
	lb := GetSpecifiedLoadbalancer(svc, nlbId)
	fmt.Printf("GET LoadbalancerName: %s LoadbalancerArn: %s\n", *lb.LoadBalancerName, *lb.LoadBalancerArn)
	// 7. 指定したLoadbalancerのTargetGroupを取得
	tg := GetSpecifiedTargetGroup(svc, lb, nlbTargetGroupId)
	fmt.Printf("GET TargetGroupName: %s TargetGroupArn: %s\n", *tg.TargetGroupName, *tg.TargetGroupArn)
	// 8. TargetGroupからTargetの登録を解除
	if HasTarget(svc, tg, ecsIp) {
		const tcpPort = 5044
		DeregisterSpecifiedTarget(svc, tg, ecsIp, tcpPort)
		fmt.Println("DEREGISTER")
	}
}
```

# CloudformationにLambda、Event Bridgeのルールを追加
参考実装：

https://github.com/TomoyukiSugiyama/ElasticStack/pull/35/files#diff-54c53a21ae7349d2d6c3e6031e07f720b140ebdba7d993c88c9a159d245f7de9R21


ECSのドキュメントに記載されている様にルールを設定しました。

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/fargate-capacity-providers.html

```yaml
  EventRule:
    Type: AWS::Events::Rule
    Properties:
      Description: detach ecs task that received terminate notification from nlb
      Name: detach-task-to-be-terminated-from-nlb
      EventPattern:
        source:
          - aws.ecs
        detail-type:
          - ECS Task State Change
        detail:
          clusterArn:
            - !Ref ClusterId
      State: ENABLED
      Targets:
        - Arn: !GetAtt Function.Arn
          Id: lambda
```

# 最後に
Lambdaのテストを使って対象のタスクがNLBから削除されることを確認しました。ECS上で動いているアプリのGraceful Shutdownについては、利用するアプリのドキュメントを参照して対処するのが良いかと思います。今回利用するLogstashについては以下に記載されています。

https://www.elastic.co/guide/en/logstash/current/shutdown.html
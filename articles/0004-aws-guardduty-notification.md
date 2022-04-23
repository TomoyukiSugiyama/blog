---
title: "GuardDutyの脅威検出結果をSlack/Teamsに通知する"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "guardduty", "lambda", "cloudformation"]
published: false
---
# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。AWSのセキュリティ対応のため、GuardDutyを利用しています。GuardDutyを利用することで、悪意のあるアクティビティや異常な動作を継続的にモニタリングすることができます。しかし、検知した結果に気が付かなければ意味がありません。チャットツールに結果を転送することで、早急な対応ができる体制を目指します。

# Slack/Teamsへの通知方法
脅威を検知した結果は、自動的にEventBridge(旧Amazon CloudWatch Events)に送信されるため、EventBridgeでイベントをトリガし、SNSに転送します。

* Slackを利用されている場合は、Chatbotとの連携が可能なため、ChatbotをSNSのターゲットにします。
* Teamsを利用されている場合は、LambdaをSNSのターゲットとし、Lambdaで結果を加工してIncoming WebhookでTeamsに転送します。

![](/images/article-0004/guardduty-notification.png)

# Slackを利用する場合のCloudformation
GuardDutyはCloudTrail、Kubernetes、VPCフローログ、DNSログ等をもとに脅威検知を行います。ただし、これらの設定は不要であり、GuardDutyは有効にするだけで、脅威検知が自動的に開始されます。EventBridgeは、以下に記載されているイベントのルールを設定し、TargetにSNSのトピックを指定します。

https://docs.aws.amazon.com/ja_jp/guardduty/latest/ug/guardduty_findings_cloudwatch.html

SNSのエンドポイントにはChatbotのAPI `https://global.sns-api.chatbot.amazonaws.com` を指定します。ChatBotは予め、AWSコンソールから、Slackと連携しておきます。CloudformationのChatBotには、SnsTopicArnsにSNSトピックのARNを指定し、SlackChannelIdとSlackWorkspaceIdを指定することで、通知することができます。SlackChannelIdは、対象のSlackチャンネルを右クリックし、「リンクをコピー」を選択します。コピーしたURLの最後のスラッシュ以降の文字列が対象のIdになります。

`https://sample.slack.com/archives/XXXXXXXXXXX`

SlackWorkspaceIdは、ChatBotのAWSコンソールから「結果サンプルの生成」をクリックすることで確認できます。これらのIdはパラメータストアに保存し、読み込んで使用しています。

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Guard Duty
# ------------------------------------------------------------------------------
# Resources
# ------------------------------------------------------------------------------
Resources:
  GuardDuty:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: True
      FindingPublishingFrequency: FIFTEEN_MINUTES
  EventRule:
    Type: AWS::Events::Rule
    Properties:
      Description: guardduty notification
      Name: guardduty-notification
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
      State: ENABLED
      Targets:
        - Arn: !Ref Topic
          Id: sns-topic
  Topic:
    Type: AWS::SNS::Topic
  Subscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: https
      TopicArn: !Ref Topic
      Endpoint: "https://global.sns-api.chatbot.amazonaws.com"
  ChatBot:
    Type: AWS::Chatbot::SlackChannelConfiguration
    Properties:
      ConfigurationName: GuardDutyNotification
      IamRoleArn: !GetAtt ChatBotRole.Arn
      LoggingLevel: NONE
      SlackChannelId: "{{resolve:ssm:GuardDutySlackChannelId:1}}"
      SlackWorkspaceId: "{{resolve:ssm:GuardDutySlackWorkspaceId:1}}"
      SnsTopicArns:
        - !Ref Topic
      UserRoleRequired: false
  ChatBotRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: chatbot.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/ReadOnlyAccess
        - arn:aws:iam::aws:policy/AWSSupportAccess
        - arn:aws:iam::aws:policy/AWSIncidentManagerResolverAccess
```

# Slackへの通知の確認
ChatbotとSlack間の疎通は、ChatbotのAWSコンソールから、「テストメッセージを送信」をクリックすることで確認できます。Slackには以下のようなメッセージが送信されます。

![](/images/article-0004/chatbot-test.png)

GuardDutyからSlackまでの疎通はGuardDutyのAWSコンソールから「結果サンプルを生成」をクリックすることで確認できます。Slackには以下のようなメッセージが送信されます。

![](/images/article-0004/guardduty-to-slack.png)

# Teamsを利用する場合のCloudformation
Slackへ通知する場合との違いは、EventBridgeのターゲットを、Lambdaに設定するところです。

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Guard Duty
# ------------------------------------------------------------------------------
# Resources
# ------------------------------------------------------------------------------
Resources:
  GuardDuty:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: True
      FindingPublishingFrequency: FIFTEEN_MINUTES
  EventRule:
    Type: AWS::Events::Rule
    Properties:
      Description: guardduty notification
      Name: guardduty-notification
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
      State: ENABLED
      Targets:
        - Arn: !GetAtt Function.Arn
          Id: lambda
  EventPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref Function
      Principal: events.amazonaws.com
      SourceAccount: !Ref AWS::AccountId
      SourceArn: !GetAtt EventRule.Arn
  Function:
    Type: AWS::Lambda::Function
    Properties:
      Handler: guardduty-notification
      Role: !GetAtt LambdaRole.Arn
      Code:
        S3Bucket: lambda-m090d3wnw8nug47yfdcw3-artifact
        S3Key: guardduty-notification.zip
      Runtime: go1.x
      ReservedConcurrentExecutions: 1
      Timeout: 5
      TracingConfig:
        Mode: Active
      Environment:
        Variables:
          TeamsUrl: "{{resolve:ssm:TeamsUrl:1}}"
  LambdaRole:
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Action:
              - sts:AssumeRole
            Condition: {}
            Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
        Version: 2012-10-17
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Tags:
        - Key: f-iot.service.name
          Value: lambda
    Type: AWS::IAM::Role
```

# Teamsを利用する場合のLambda
EventBridgeから受け取ったイベントを加工し、必要な情報を環境変数から読み込んだTeamsUrlに対して、送信します。

```go
// main GuardDutyが検知した異常を通知します。
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"strconv"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

type GuardDutyEvent struct {
	Severity    string `json:"severity"`
	AccountId   string `json:"accountId"`
	Id          string `json:"id"`
	Type        string `json:"type"`
	Region      string `json:"region"`
	Description string `json:"description"`
}

type Fact struct {
	Name  string `json:"name"`
	Value string `json:"value"`
}

type Section struct {
	Facts []Fact `json:"facts"`
}

type Target struct {
	Os  string `json:"os"`
	Uri string `json:"uri"`
}

type Link struct {
	Type    string   `json:"@type"`
	Name    string   `json:"name"`
	Targets []Target `json:"targets"`
}

func HandleLambdaEvent(_ context.Context, event events.CloudWatchEvent) {
	var guardDutyEvent GuardDutyEvent
	if err := json.Unmarshal(event.Detail, &guardDutyEvent); err != nil {
		os.Exit(1)
	}
	TeamsUrl := os.Getenv("TeamsUrl")

	facts := []Fact{
		{Name: "Severity", Value: guardDutyEvent.Severity},
		{Name: "Type", Value: guardDutyEvent.Type},
		{Name: "Description", Value: guardDutyEvent.Description},
	}
	sections := []Section{{Facts: facts}}

	severity, err := strconv.ParseFloat(guardDutyEvent.Severity, 64)
	if err != nil {
		fmt.Printf("%s\n", err.Error())
		return
	}
	var color string
	if severity >= 7.0 {
		color = "#ff0000"
	} else {
		color = "#ffff00"
	}
	pipelineURL := "https://console.aws.amazon.com/guardduty/home?region=" + guardDutyEvent.Region + "#/findings?search=id=" + guardDutyEvent.Id
	targets := []Target{{Os: "default", Uri: pipelineURL}}
	links := []Link{{Type: "OpenUri", Name: "Jump To GuardDuty", Targets: targets}}

	payload, err := json.Marshal(struct {
		Summary         string    `json:"summary"`
		Type            string    `json:"@type"`
		ThemeColor      string    `json:"themeColor"`
		Title           string    `json:"title"`
		Sections        []Section `json:"sections"`
		PotentialAction []Link    `json:"potentialAction"`
	}{
		Summary:         "Summary",
		Type:            "MessageCard",
		ThemeColor:      color,
		Title:           "GuardDuty",
		Sections:        sections,
		PotentialAction: links,
	})
	if err != nil {
		fmt.Println(err)
		return
	}

	resp, err := http.Post(TeamsUrl, "application/json; charset=UTF-8", bytes.NewReader(payload))
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		fmt.Printf("HTTP: %v\n", resp.StatusCode)
	}
}

func main() {
	lambda.Start(HandleLambdaEvent)
}
```

# Teamsへの通知の確認
LambdaとTeams間の疎通は、LambdaのAWSコンソールから、テストイベントを作成し「テスト」をクリックすることで確認できます。Teamsには以下のようなメッセージが送信されます。

![](/images/article-0004/lambda-test.png)

# 最後に

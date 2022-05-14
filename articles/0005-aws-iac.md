---
title: "CodePipelineの構築からデプロイまでの導線をIaC化"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cloudformation", "codepipeline", "codebuild"]
published: false
---

# 初めに
生産技術部で製品の検査工程を担当しているエンジニアです。AWS Well-Architected フレームワークの中から運用の優秀性に焦点を当てて、CodePipeline構築からデプロイまでのコード化を紹介します。

https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/operational-excellence-pillar/design-principles.html

# CodePipeline構築のIaC化
AWS CLIをシェルスクリプトから実行し、環境構築を行います。

![](/images/article-0005/build-environment.png)


```bash
#!/bin/sh

# Common settings
DEPLOY_ENV="dev" # set dev or prod

# Environment settings
SCRIPT_DIR=$(cd $(dirname $0); pwd)
WORK_DIR="${SCRIPT_DIR}/../.."
alias cfn-stack-ops="${WORK_DIR}/provisioning/helper-scripts/cfn-stack-ops.sh $1"

# KMS
# Create kms to encrypt/decrypt logs.
cfn-stack-ops deploy kms ${SCRIPT_DIR}/cfn/kms.yaml FargateLogKeyAliasName=alias/${DEPLOY_ENV}/fargate LambdaLogKeyAliasName=alias/${DEPLOY_ENV}/lambda

# SSM
# Create parameters.
# Need to set github connection id and slack workspace/channel ids in secrets/*.yaml, before create parameters.
${WORK_DIR}/provisioning/params/create-params-${DEPLOY_ENV}.sh

# S3
# Get s3 bucket name from ssm.
S3CfnBucketName=$(aws ssm get-parameter --name /${DEPLOY_ENV}/s3/cfn/BucketName --query "Parameter.Value" --output text)
S3LambdaBucketName=$(aws ssm get-parameter --name /${DEPLOY_ENV}/s3/lambda/BucketName --query "Parameter.Value" --output text)
# Create s3 buckets
cfn-stack-ops deploy s3 ${SCRIPT_DIR}/cfn/s3.yaml S3CfnBucketName=${S3CfnBucketName} S3LambdaBucketName=${S3LambdaBucketName}

# ECR
# Create repogitory in elastic container registory.
cfn-stack-ops deploy ecr ${SCRIPT_DIR}/cfn/ecr.yaml EcrRepogitoryName=${DEPLOY_ENV}-repogitory

# CodeBuild, CodePipeline
# Create codebuild and codepipeline.
cfn-stack-ops package ${WORK_DIR}/provisioning/pre-build/cfn/pre-build.yaml ${S3CfnBucketName} ${WORK_DIR}/provisioning/artifacts/pre-build-artifact.yaml
cfn-stack-ops deploy dev ${WORK_DIR}/provisioning/artifacts/pre-build-artifact.yaml
```


```bash
#!/bin/sh
# ref: https://www.slideshare.net/yktko/cloudformation-getting-started-with-yaml

mode=$1; shift
arg1=$1; shift
arg2=$1; shift
arg3=$1; shift
arg4=$1; shift
if [ "$mode" != "create" -a "$mode" != "update" -a "$mode" != "delete" -a "$mode" != "list" -a "$mode" != "describe" -a "$mode" != "validate" -a "$mode" != "package"  -a "$mode" != "deploy" ]; then
    echo ""
    echo "Usage: $0 MODE ARGS"
    echo ""
    echo "Mode:     Args:"
    echo "create    stack-name s3-bucket [param1=val1 param2=val2]"
    echo "update    stack-name s3-bucket [param1=val1 param2=val2]"
    echo "package   path-to-cfn-template-file s3-bucket output-template-file"
    echo "deploy    stack-name path-to-cfn-template-filee"
    echo "list      ";
    echo "describe  stack-name";
    echo "validate  s3-bucket"
    echo "delete    stack-name"; exit 1
fi

if [ "$mode" == "create" -o "$mode" == "update" ]; then
    param1=$(echo ${arg3} | perl -pe "s/([^= ]+)=([^ ]+)/--parameter-overrides \1=\2/g")
    param2=$(echo ${arg4} | perl -pe "s/([^= ]+)=([^ ]+)/\1=\2/g")
    args="--template-url https://s3.amazonaws.com/${arg2}/artifact.yaml --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM ${param1} ${param2}"
    stack_name_option="--stack-name ${arg1}"
    mode_option="${mode}-stack"
fi

if [ "$mode" == "package" ]; then
    args="--template-file ${arg1} --s3-bucket ${arg2} --output-template-file ${arg3}"
    stack_name_option=""
    mode_option="${mode}"
fi

if [ "$mode" == "deploy" ]; then
    param1=$(echo ${arg3} | perl -pe "s/([^= ]+)=([^ ]+)/--parameter-overrides \1=\2/g")
    param2=$(echo ${arg4} | perl -pe "s/([^= ]+)=([^ ]+)/\1=\2/g")
    args="--template-file ${arg2} --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM ${param1} ${param2}"
    stack_name_option="--stack-name ${arg1}"
    mode_option="${mode}"
fi

if [ "$mode" == "list" ]; then
    args="--stack-status-filter CREATE_COMPLETE"
    stack_name_option=""
    mode_option="${mode}-stacks"
fi

if [ "$mode" == "describe" ]; then
    args=""
    stack_name_option="--stack-name ${arg1}"
    mode_option="${mode}-stacks"
fi

if [ "$mode" = "validate" ]; then
    args="--template-url https://s3.amazonaws.com/${arg1}/artifact.yaml"
    stack_name_option=""
    mode_option="${mode}-template"
fi

if [ "$mode" = "delete" ]; then
    args=""
    stack_name_option="--stack-name ${arg1}"
    mode_option="${mode}-stack"
fi
cmd="aws cloudformation ${mode_option} ${stack_name_option} ${args}"
echo ${cmd}
eval ${cmd}
```
# デプロイのIaC化

# 最後に
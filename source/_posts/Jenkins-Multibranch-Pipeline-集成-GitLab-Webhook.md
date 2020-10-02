---
title: Jenkins Multibranch Pipeline 集成 GitLab Webhook
tags:
  - Jenkins
  - GitLab
categories:
  - Linux
date: 2020-09-13 11:08:00
---

# 所需插件

在 Jenkins 中安装 [Generic Webhook Trigger](https://plugins.jenkins.io/generic-webhook-trigger/) 插件

# Jenkinsfile 配置

在 GitLab 中创建一个用于集成的演示项目

并在该项目根路径创建一个 Jenkinsfile 文件, 文件内容如下

```groovy
pipeline {
    agent any
    // 触发器, 默认存在三种, 但是不满足 Webhook 需求, 所以使用插件提供的 GenericTrigger 触发器
    triggers {
        // 由 Generic Webhook Trigger 提供的触发器
        GenericTrigger(
                // 参数, 在这里配置的参数会被配置为环境变量
                // 支持使用 JSONPath 和 XPath 进行提取
                // 提取来源为 webhook 发送的 request body 中的内容
                // GitLab 发送的 request body 内容格式可通过文档查看, 文档地址为你的 GitLab 访问地址 + /help/user/project/integrations/webhooks
                genericVariables: [
                        // 提取分支名称, 格式为 refs/heads/{branch}
                        [key: 'WEBHOOK_REF', value: '$.ref'],
                        // 提取用户显示名称
                        [key: 'WEBHOOK_USER_NAME', value: '$.user_name'],
                        // 提取最近提交 id
                        [key: 'WEBHOOK_RECENT_COMMIT_ID', value: '$.commits[-1].id'],
                        // 提取最近提交 message
                        [key: 'WEBHOOK_RECENT_COMMIT_MESSAGE', value: '$.commits[-1].message'],
                        // 如需更多参数可通过查看 request body 参数文档进行提取
                ],

                // 项目运行消息, 会显示在 Jenkins Blue 中的项目活动列表中
                causeString: '$WEBHOOK_USER_NAME 推送 commit 到 $WEBHOOK_REF 分支',

                // token, 因为对外的 webhook url 是一致的, 当希望触发某个特定的 Job 时可以为每个 Job 配置不同的 token, 然后在 webhook 中配置该参数
                token: 'webhook-test-token',

                // 打印通过 genericVariables 配置的变量
                printContributedVariables: true,
                // 打印 request body 内容
                printPostContent: true,

                // 避免使用已触发工作的信息作出响应
                silentResponse: false,

                // 可选的正则表达式过滤, 比如希望仅在 master 分支上触发, 你可以进行如下配置
                regexpFilterText: '$WEBHOOK_REF',
                regexpFilterExpression: 'refs/heads/master'
        )
    }
    stages {
        stage('查看环境变量') {
            steps {
                sh "env"
                sh "echo \\\$WEBHOOK_REF = $WEBHOOK_REF"
                echo "env.WEBHOOK_REF = ${env.WEBHOOK_REF}"
            }
        }
    }
}
```

# Jenkins 配置

在 Jenkins 创建一个 Multibranch Pipeline 项目

配置分支源为 Git, 项目仓库为 GitLab 中的项目地址

同时配置不通过SCM自动化触发, 该配置是指保存后不要直接运行 Job

![不通过SCM自动化触发配置](/images/Jenkins-Multibranch-Pipeline-集成-GitLab-Webhook/不通过SCM自动化触发配置.png)

然后进行保存

# Webhook 配置

进入项目 Webhook 配置页面

![webhook配置](/images/Jenkins-Multibranch-Pipeline-集成-GitLab-Webhook/webhook配置.png)

URL 地址格式为 `{jenkins 地址}/generic-webhook-trigger/invoke?token={token}`

> 如果 jenkins 地址为内网, 需要在 GitLab 管理配置页面的 Outbound requests 下勾选 Allow requests to the local network from web hooks and services 属性, 否则会保存失败
>
> 参考链接: https://docs.gitlab.com/ee/security/webhooks.html

> Generic Webhook Trigger 插件可以通过配置 IP 白名单的方式仅接受允许的 IP
>
> 参考链接: https://github.com/jenkinsci/generic-webhook-trigger-plugin#whitelist-hosts

然后点击最下方的保存按钮即可

保存后在 webhooks 页面最下面可以看到该记录, 点击 **Test** 按钮, 然后选择 Push events, 此时会手动触发一次 webhook 请求, 如果响应成功, 会显示一条 **Hook executed successfully: HTTP 200** 的消息提示

同时在 Jenkins 的 Blue 页面查看项目活动记录, 可以看到项目运行成功的记录, 其中的消息部分为即为 causeString 属性

# 手动运行

除了通过 Webhook 运行的方式外, 我们可能存在手动运行的需求

如果我们在 Jenkinsfile 中没有依赖 genericVariables 配置的环境变量的话, 我们可以直接手动运行, 否则的话使用该变量会出现问题

这个时候需要定义一个与 genericVariables 中 key 名称相同的 parameter

配置方式如下

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'WEBHOOK_REF', defaultValue: 'refs/heads/master', description: '分支名称?')
    }
    triggers {
        // 由 Generic Webhook Trigger 提供的触发器
        GenericTrigger(
            genericVariables: [
                // 提取分支名称, 格式为 refs/heads/{branch}
                [key: 'WEBHOOK_REF', value: '$.ref'],
            ],
            
            token: 'webhook-test-token',
        )
    }
    stages {
        stage('查看环境变量') {
            steps {
                sh "env"
                sh "echo \\\$WEBHOOK_REF = $WEBHOOK_REF"
                echo "env.WEBHOOK_REF = ${env.WEBHOOK_REF}"
                echo "params.WEBHOOK_REF = ${params.WEBHOOK_REF}"
            }
        }
    }
}
```

如果是通过 webhook 运行, Generic Webhook Trigger 会使用 genericVariables 中的 WEBHOOK_REF 填充 parameter 中的 WEBHOOK_REF

如果是手动运行, Generic Webhook Trigger 会使用 parameter 中的 WEBHOOK_REF 填充 genericVariables 中的 WEBHOOK_REF

# GenericTrigger 自定义配置

GenericTrigger 的属性说明可以查看 [插件官方文档](https://github.com/jenkinsci/generic-webhook-trigger-plugin)

同时可以在项目配置页面点击**流水线语法**, 然后使用可视化语法生成器生成 trigger 配置

![webhook配置](/images/Jenkins-Multibranch-Pipeline-集成-GitLab-Webhook/流水线语法1.png)

![webhook配置](/images/Jenkins-Multibranch-Pipeline-集成-GitLab-Webhook/流水线语法2.png)

填写配置属性后点击 Generate Declarative Directive 按钮即可生成
bedrock

---

> 调用 AWS Bedrock 服务，为微信公众号消息提供自动回复功能。

## 体验

关注“哈德韦”公众号，发送任意消息即可体验。

![](https://ml.jiwai.win/mp-hardway.png)

效果如下图所示：

![](WX20240116-181835@2x.png)

## 为什么有这个库

微信公众号提供了自动回复功能，但是不够智能，只能根据关键字进行回复。这个库可以让你的公众号更加智能，可以根据用户的消息内容，调用 AWS Bedrock 服务，得到更加智能的回复。当然，关键字回复也是同时支持的，即使 AWS Bedrock 服务没有响应，也会回复提前设置好的关键字回复。

即该库在微信公众号自动回复的基础上，**增加**了智能回复的功能。是一个 AI 增强的微信公众号自动回复库。

## 难点

调用 AWS Bedrock 服务本身并不难，难点在于微信服务器的限制。微信服务器只等待 5 秒，如果超过 5 秒没有响应，就会重试。而最多重试 3 次，如果 3 次都没有响应，用户就会收不到回复。而微信服务器等待 5 秒的时间，是包括网络传输的时间的，所以，实际上，从接收到微信服务器发来的消息，一次只有不到 5 秒的时间来响应。而 AWS Bedrock 服务的响应时间很可能超过 5 秒，所以，这里使用了缓存，如果是同一消息，就不再调用 AWS Bedrock 服务，而是等待上次的调用结果。如果到第 3 次还没收到 AWS Bedrock 服务的响应，那就只能回复提前设置好的关键字回复了（当然，在这时，可以带上一个链接，允许用户稍后查看回复结果）。

## 时序图

有时候，AWS Bedrock 服务的响应很快，时序图如下：

```mermaid
sequenceDiagram
    participant 微信用户
    participant 微信服务器
    participant 本项目
    participant AWS Bedrock 服务
    微信用户 ->> 微信服务器: 发送消息
    微信服务器 ->> 本项目: 发送消息
    本项目 ->> AWS Bedrock 服务: 发送消息
    AWS Bedrock 服务 ->> 本项目: 回复消息
    本项目 ->> 微信服务器: 回复消息
    微信服务器 ->> 微信用户: 回复消息
```

有时候，AWS Bedrock 服务的响应很慢，引起了微信服务器的一次重试，时序图如下：

```mermaid
sequenceDiagram
    participant 微信用户
    participant 微信服务器
    participant 本项目
    participant AWS Bedrock 服务
    微信用户 ->> 微信服务器: 发送消息
    微信服务器 ->> 本项目: 发送消息
    本项目 ->> 缓存: 将消息编号缓存起来
    本项目 ->> AWS Bedrock 服务: 发送消息
    note over AWS Bedrock 服务: 思考中
    note over 微信服务器: 等待 5 秒后超时
    微信服务器 ->> 本项目: 重试消息
    AWS Bedrock 服务 ->> 本项目: 回复消息
    本项目 ->> 缓存: 更新缓存，将回答写回对应的消息编号
    本项目 ->> 微信服务器: 回复消息
    微信服务器 ->> 微信用户: 回复消息
```

如果再慢一点，就会引起两次重试，时序图如下：

```mermaid
sequenceDiagram
    participant 微信用户
    participant 微信服务器
    participant 本项目
    participant AWS Bedrock 服务
    微信用户 ->> 微信服务器: 发送消息
    微信服务器 ->> 本项目: 发送消息
    本项目 ->> 缓存: 将消息编号缓存起来
    本项目 ->> AWS Bedrock 服务: 发送消息
    note over AWS Bedrock 服务: 思考中
    note over 微信服务器: 等待 5 秒后超时
    微信服务器 ->> 本项目: 重试消息
    本项目 ->> 缓存: 检查缓存，仍未得到回答
    note over 微信服务器: 等待 5 秒后超时
    微信服务器 ->> 本项目: 重试消息
    AWS Bedrock 服务 ->> 本项目: 回复消息
    本项目 ->> 缓存: 更新缓存，将回答写回对应的消息编号
    本项目 ->> 微信服务器: 回复消息
    微信服务器 ->> 微信用户: 回复消息
```

以上是终于在微信服务器的重试次数内得到了回复的情况。如果 AWS Bedrock 服务的响应时间超过了微信服务器 3 次重试等待的总时间，那么，就只能回复提前设置好的关键字回复了。时序图如下：

```mermaid
sequenceDiagram
    participant 微信用户
    participant 微信服务器
    participant 本项目
    participant AWS Bedrock 服务
    微信用户 ->> 微信服务器: 发送消息
    微信服务器 ->> 本项目: 发送消息
    本项目 ->> 缓存: 将消息编号缓存起来
    本项目 ->> AWS Bedrock 服务: 发送消息
    note over AWS Bedrock 服务: 思考中
    note over 微信服务器: 等待 5 秒后超时
    微信服务器 ->> 本项目: 重试消息
    本项目 ->> 缓存: 检查缓存，仍未得到回答
    note over 微信服务器: 等待 5 秒后超时
    微信服务器 ->> 本项目: 重试消息
    本项目 ->> 缓存: 检查缓存，仍未得到回答
    本项目 ->> 微信服务器: 回复消息兜底消息，带上稍后查看链接
    微信服务器 ->> 微信用户: 回复消息
    AWS Bedrock 服务 ->> 本项目: 回复消息
    本项目 ->> 缓存: 更新缓存，将回答写回对应的消息编号
    微信用户 ->> 本项目: 点击稍后查看链接
    本项目 ->> 缓存: 检查缓存，得到回答
    本项目 ->> 微信用户: 展示消息和回答页面
```

## 测试

```bash
yarn test
```

## 部署

可以部署到任意 Kubernetes 集群。注意如果采用了内存缓存，replica 就需要设置成 1，如果设置成多个 Pod，会导致缓存失效。

如果采用了 Redis 缓存，可以设置成多个 Pod，但是，需要注意，如果 Pod 重启，缓存会丢失，所以，需要设置成多个 Pod 的时候，需要使用 Redis 缓存。

本来本项目部署在 Okteto 提供的免费 Kubernetes 集群上，域名是 https://bedrock-backstage-jeff-tian.cloud.okteto.net/message ，但是在 2024 年 1 月 15 日，Okteto 关闭免费服务。于是对本项目进行了重新部署，选择 [cyclic](https://app.cyclic.sh/#/join/Jeff-Tian)，它是一个无服务器平台，不能再使用内存缓存，因为微信服务的消息重试会打在不同的实例上。好在，[cyclic](https://app.cyclic.sh/#/join/Jeff-Tian) 提供了方便的 DynamoDb 存储，于是，将缓存从内存缓存改成了 DynamoDb 缓存。

## 缓存

由于微信公众号消息只等待 5 秒，而调用 AWS Bedrock 服务很可能在超过 5 秒后才能得到响应。超时后微信会再重试请求，为了避免重复调用 AWS Bedrock 服务，这里使用了缓存，如果是同一消息，就不再调用 AWS Bedrock 服务，而是等待上次的调用结果。

## 公众号后台配置

如何让你的微信公众号支持自动回复功能？只需要将该项目部署，然后在公众号后台配置即可。

![](./WX20240112-115802@2x.png)

## 依赖

### 代码依赖

本项目基于 Koa js 和 AWS 的 Bedrock SDK，并依赖 [Cyclic](https://app.cyclic.sh/#/join/Jeff-Tian) 提供的对 AWS SDK 的上层封装。

### 基础设施依赖

本项目依赖 AWS Bedrock 服务，并且需要 [Cyclic 平台](https://app.cyclic.sh/#/join/Jeff-Tian) 或者 AWS DynamoDb 和任意的 Kubernetes 平台。

## 部署

可以部署到多个环境，包括本地、Vercel、AWS Lambda、Cyclic 和 Kubernetes 等等。推荐使用 GitHub Actions 进行自动部署。

### 部署到 Cyclic

1. 在 [Cyclic](https://app.cyclic.sh/#/join/Jeff-Tian) 上创建一个项目
2. 链接到你的 GitHub 仓库
3. 自动部署

### 部署到 Kubernetes

参考 GitHub Actions 的配置文件，将其中的环境变量替换成你自己的即可。

### 部署到 AWS Lambda

参考 GitHub Actions 的配置文件，将其中的环境变量替换成你自己的即可。主要是利用了 AWS SAM，详见《[将免费架构进行到底，使用 AWS SAM 将 Koa 服务从 Cyclic 迁移到 Lambda - Jeff Tian的文章 - 知乎](https://zhuanlan.zhihu.com/p/678946260) 》。

## 相关文章

- https://zhuanlan.zhihu.com/p/674125115

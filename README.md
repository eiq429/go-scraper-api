# Scraper API Go 实战体验：用 Go 语言对接 ScraperAPI 的完整踩坑记录与套餐选择指南

我之前用 Python 写爬虫写了两年多，后来项目需要高并发抓取，切到 Go 之后发现生态里能直接用的代理 API 服务并不多。折腾了一圈，最后稳定用下来的是 ScraperAPI。这篇就聊聊我用 Go 对接 ScraperAPI 的真实过程，包括怎么调、哪些坑要避、以及不同套餐到底该怎么选。

## 为什么用 Go 对接 ScraperAPI 而不是直接裸写代理池

说实话，自建代理池在 Go 里并不难实现。但问题出在维护成本上。

IP 被封、验证码弹出、目标站反爬升级——这些事每周都在发生。我之前自己维护了一个住宅代理池，每个月光 IP 成本就要好几百美元，还得写轮换逻辑、失败重试、指纹伪装。后来算了笔账，时间成本加上 IP 费用，还不如直接用 ScraperAPI 这类托管服务。

ScraperAPI 的核心卖点就是：你只管发请求，代理轮换、浏览器渲染、验证码处理它全包了。对 Go 开发者来说，它本质上就是一个 HTTP API，对接起来非常干净。

## Go 语言对接 ScraperAPI 的基本方式

ScraperAPI 提供两种对接方式，都很适合 Go：

**方式一：Query 参数模式**

直接把你的 API Key 和目标 URL 拼到请求里：

```go
package main

import (
    "fmt"
    "io
    "net/http"
    "net/url"
)

func main() {
    apiKey := "YOUR_API_KEY"
    targetURL := "https://example.com/products"

    endpoint := fmt.Sprintf("http://api.scraperapi.com?api_key=%s&url=%s",
        apiKey, url.QueryEscape(targetURL))

    resp, err := http.Get(endpoint)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

**方式二：代理模式**

把 ScraperAPI 当成 HTTP 代理来用，这种方式改动最小，适合已有爬虫框架的项目：

```go
proxyURL, _ := url.Parse("http://scraperapi:YOUR_API_KEY@proxy-server.scraperapi.com:8001")
client := &http.Client{
    Transport: &http.Transport{
        Proxy: http.ProxyURL(proxyURL),
    },
}
```

两种方式我都用过。如果你的项目是从零开始写，推荐方式一，控制粒度更细。如果是老项目迁移，方式二改动最少。

## 实际使用中踩过的几个坑

### 并发控制是关键

Go 天生适合高并发，goroutine 开起来不要钱似的。但 ScraperAPI 的并发数是按套餐限制的。我刚开始用 Hobby 套餐，10 个并发线程，结果一口气开了 50 个 goroutine 去抓，大量请求直接返回 429。

解决方案很简单，用 semaphore 控制：

```go
sem := make(chan struct{}, 10) // 按套餐并发数设置
for _, u := range urls {
    sem <- struct{}{}
    go func(target string) {
        defer func() { <-sem }()
        // 发请求
    }(u)
}
```

这个数字一定要跟你的套餐并发上限对齐，别浪费请求额度。

### 超时设置要放宽

ScraperAPI 后端会做代理轮换和重试，响应时间比直连慢不少。我一开始 `http.Client` 的 Timeout 设了 10 秒，结果大量请求超时。后来调到 60 秒才稳定下来。官方也建议至少给 60 秒。

### 渲染 JavaScript 页面要加参数

抓 SPA 页面时，加上 `render=true` 参数就行。但要注意，开启渲染后每个请求消耗的 API credit 会翻倍甚至更多，规划额度时要算进去。

## 不同套餐到底差在哪——全套餐对比

这是我觉得很多人选套餐时容易踩坑的地方。ScraperAPI 的套餐差异不只是请求数，并发数和高级功能的区别才是关键。

| 套餐名称 | API 请求数/月 | 并发线程数 | 地理定位 | JS 渲染 | 价格（月付） | 适合谁 | 链接 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Free | 5,000 | 1 | ✓ | ✓ | $0 | 开发调试、验证可行性 | [ 免费注册开始测试](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | 10 | ✓ | ✓ | $49 | 个人项目、小规模数据采集 | [ 开通 Hobby 套餐查看完整配置](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 500,000 | 25 | ✓ | ✓ | $149 | 中型项目、需要稳定并发的团队 | [ 直接开通 Startup 套餐](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 | 50 | ✓ | ✓ | $299 | 大规模采集、电商监控、SEO 数据 | [ 开通 Business 套餐锁定当前价格](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | ✓ | ✓ | 联系销售 | 超大规模需求、需要定制 SLA | [ 联系销售获取 Enterprise 方案](https://www.scraperapi.com/?fp_ref=coupons) |

年付的话每个套餐都有折扣，大概能省下几个月的费用。我自己用的是 Startup，25 个并发对Go 的 goroutine 调度来说刚好够用，不用太纠结限流逻辑。

说个不完美的地方：免费套餐只有 1 个并发，基本只能用来测试 API 通不通，真跑业务完全不够。而且所有套餐的请求如果触发了高级反爬处理（比如某些电商站），实际消耗的 credit 会是标称的 5-25 倍，这个在官网的计费说明里有写，但很容易被忽略。

## 用 Go 写一个生产级的 ScraperAPI 客户端

分享一下我自己封装的简化版客户端结构，实际项目里可以直接拿去改：

```go
type ScraperClient struct {
    apiKey     string
    httpClient *http.Client
    semaphore  chan struct{}
}

func NewScraperClient(apiKey string, concurrency int) *ScraperClient {
    return &ScraperClient{
        apiKey: apiKey,
        httpClient: &http.Client{
            Timeout: 60 * time.Second,
        },
        semaphore: make(chan struct{}, concurrency),
    }
}

func (c *ScraperClient) Fetch(targetURL string, render bool) ([]byte, error) {
    c.semaphore <- struct{}{}
    defer func() { <-c.semaphore }()

    params := url.Values{}
    params.Set("api_key", c.apiKey)
    params.Set("url", targetURL)
    if render {
        params.Set("render", "true")
    }

    resp, err := c.httpClient.Get("http://api.scraperapi.com?" + params.Encode())
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != 200 {
        return nil, fmt.Errorf("status: %d", resp.StatusCode)
    }

    return io.ReadAll(resp.Body)
}
```

这个结构把并发控制、超时、参数拼接都封好了。实际用的时候再加上重试逻辑和错误分类就差不多了。

[👉 注册 ScraperAPI 获取 API Key 开始集成](https://www.scraperapi.com/?fp_ref=coupons)

## 跟其他方案比起来怎么样

我之前用过 Bright Data 和自建的 Squid 代理。Bright Data 功能确实强，但配置复杂，Go SDK 也不太好用，价格门槛高。自建方案前面说了，维护成本是隐性的大头。

ScraperAPI 的优势在于简单。对 Go 开发者来说，它就是一个 HTTP endpoint，不需要装 SDK，不需要学新的 DSL，标准库 `net/http` 就能搞定。缺点是遇到特别难搞的站（比如 Cloudflare 高防），成功率不是 100%，偶尔还是会失败。但大部分常规站点，成功率在 95% 以上我是能感受到的。

## FAQ

#### ScraperAPI 的 Go SDK 在哪里下载？

ScraperAPI 目前没有官方的 Go SDK，但这反而是好事。它的 API 就是标准 HTTP 接口，用 Go 标准库 `net/http` 直接调就行，不需要额外依赖。我上面贴的代码基本就是生产可用的封装了。

#### 免费套餐够不够日常开发测试用？

够测试，不够跑业务。5000 次请求、1 个并发，验证一下 API 能不能正常返回数据、目标站能不能抓到是没问题的。但如果你要跑哪怕是小规模的定时任务，至少得上 Hobby。

[👉 免费注册先测试 API 连通性](https://www.scraperapi.com/?fp_ref=coupons)

#### 开启 JavaScript 渲染后请求速度会慢多少？

体感上慢 3-5 倍。不开渲染的请求一般 2-5 秒返回，开了渲染可能要 10-20 秒。而且 credit 消耗也会增加。我的建议是：能不开就不开，先试不渲染能不能拿到数据。

#### 请求失败了会扣 credit 吗？

不会。ScraperAPI 只对成功返回 200 的请求计费。如果返回 4xx 或 5xx，不扣额度。这点比较良心，不用担心因为目标站抽风而浪费钱。

#### 怎么处理 ScraperAPI 返回 429 Too Many Requests？

429 说明你超过了套餐的并发上限。在 Go 里用 channel 做semaphore 限制并发数就行，把 channel 容量设成你套餐的并发线程数。另外也可以加一个简单的指数退避重试。

#### 抓取电商网站（Amazon、Shopee 等）成功率怎么样？

Amazon 是 ScraperAPI 重点优化的目标，他们甚至有专门的 Amazon 结构化数据 API。Shopee 这类东南亚电商我试过，大部分页面能抓到，但偶尔会遇到验证码拦截导致失败。整体来说主流电商站的成功率还行，但别指望 100%。

## 如果让我重新选一次

我还是会选 ScraperAPI 配合 Go 来做数据采集。原因很实际：Go 的并发模型天然适合批量抓取，而 ScraperAPI 把最烦人的代理管理和反爬对抗都接管了。两者配合起来，我只需要关心业务逻辑和数据解析。

对于刚开始的项目，我建议先用免费套餐跑通流程，确认目标站能正常抓取，然后根据实际并发需求选套餐。大部分中型项目 Startup 就够了，除非你要同时监控几万个 SKU，那再考虑 Business。

[👉 注册 ScraperAPI 免费套餐开始你的 Go 爬虫项目](https://www.scraperapi.com/?fp_ref=coupons)

---
title: Observability 三本柱之 Distributed Tracing 介紹
date: 2020-07-26 19:42:46
tags: [kubernetes, distributed-tracing, observability,opentelemetry]
---

最近在工作中使用到了 **Distributed Tracing**（分散式追蹤），發現 Distributed Tracing 對於 Microservices 的維護與開發是非常好用且重要的技術。

而雖然 **Logging**、**Metrics** 及 **Distributed Tracing** 被並稱為 [Observability 三本柱](https://www.scalyr.com/blog/three-pillars-of-observability/)，Distributed Tracing 得到的關注度卻不如前兩者。原因可能是 Logging 與 Metric 在 Monolith System 中即被大量使用，Distributed Tracing 卻是建立在 Microservice 前提之下，普及度自然比不上前兩者。

而這段時間在學習與實作 Distributed Tracing 時，也發現繁體中文的相關資料有點少，因此決定寫個系列文介紹 Distributed Tracing，也當作是自己的工作筆記。

## 何謂 Distributed Tracing？

**`Distributed Tracing`**，簡單的來說就是在監控 request 在 distributed system 或是 microservices 中的行為的技術。一個 **`Trace`**（追蹤）是用來描述網路服務在回應一個 request 時觸發的一系列事件，這些事件可能跨越了不同 process 、network boundary 或是 subsystem boundary。

**"Distributed"** Tracing 為何重要？以往在 Monolith System 架構時，只要在 log 中記錄好相關的 request-id，就能從 logging system 中監控與分析每個 request 的狀況。而在 microservice 的架構中，一個服務往往背後會用到多個不同的子服務，這時若沒有對記錄下來的 log 做額外處理，logging system 會失去同個 request 在不同服務之間的連結。

Distributed Tracing 就是為了解決此問題而誕生的。

## 觀念介紹

在 Distributed Tracing 中，**一個 `Trace` 描述了一個 request 在多個子服務中的不同階段分別觸發的一至多個 event**。為了記錄事件在跨服務間的從屬關係以及發生順序， Distributed Tracing 規範了一系列的 headers（以 HTTP 為主，但也有 queue 與 RPC 的實作），每個子服務必須要乖乖收集這些 headers，並在對子服務送 request 時一路 propagate （傳遞）給所有子服務。

**`Trace`**（追蹤）與 **`Span`**（跨度，中文好像很少用到這個詞）是 Distributed Tracing 裡最重要的兩個元件。每個 Span 還會帶有一個 **`SpanContext`** 描述 Span 之間共用的狀態。

### Trace
**`Trace`** 用來描述一個 request，由一到多個  **`Span`** 組成。

### Span
**`Span`** 為在各服務中發生的多個不同階段的資訊，一個 Span 包含了以下資料：
- 事件名稱
- 開始與結束時間
- Attributes（屬性）：一系列的 key-value
- 零到多個事件資料，每個事件也會包含自己的 Name, Timestamp 及 Attributes
- Parent Span ID
- 連結到零或多個 Span
- SpanContext ID

### SpanContext
**`SpanContext`** 中包含了識別 Span 的資訊，也負責傳遞 parent span 設定的 options。
- **Trace ID**：Trace 的 ID，為隨機產生的 16 bytes 值，用來 group 跨越服務邊界的所有屬於同個 Trace 的 Spans。
- **Span ID**：Span 的 ID，為隨機產生的 8 bytes 值，用來識別 Span。當 SpanID 被傳遞至子 Span 時，SpanID 會成為子 Span 中的 Parent Span ID。
- **TraceFlags**：Trace 的 options。，為 1 byte 值（含 8 bit），目前只定義了 Sampling Bit (0x1)，代表此 Trace 是否有被 sample（取樣）到。
- **TraceState**：TraceState 讓 vendor 能夾帶自行定義的 key-value pairs，能用來彈性的傳遞額外的資訊或是處理 legacy field。


## Trace 範例

假設有一個服務由 5 個子服務組成，服務間使用 RPC （Remote Procedure Call）溝通，架構如下圖：

<div style="text-align:center;padding: 20px 0;">
<figure class="image">
<img style="width:50%;margin-bottom:22px;" src="/blog/2020/07/20/distributed-tracing-1/services.png">
  <figcaption>Figure 1. Example distributed system</figcaption>
</figure>

</div>

要為這個服務設計 Distributed Tracing，最簡單的方式就是對每個子服務產生一個 Span，每次 Request 會得到 1 個 Trace，Trace 中包含了 5 個。

Distributed Tracing 中有主要有兩種視覺化的方式：**Directed Acyclic Graph (DAG)** 或是 **Time-based Gragh**。

### DAG View of Trace

Trace 也可以被理解為由多個 Span 組成的有向無環圖（Directed acyclic graph，簡稱 DAG），如下圖：


<div style="text-align:center;width:100%;">
<figure class="image">
<img style="width:50%;margin-bottom:0px;" src="/blog/2020/07/20/distributed-tracing-1/span-DAG.png">
  <figcaption>Figure 2. DAG View of a Trace </figcaption>
</figure>
</div>



主服務 A 收到 request 時，request header 會帶有一個在系統中唯一的 TraceID，這個 TraceID 可以在服務的 Infrastructure 層產生（例如在 K8S 中可以使用 [Istio 來注入](https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/)）。若 TraceID 不存於 request 中，則 A 需自行產生一個 ID。TraceID 及其他的資訊（Metadata）會以 key-value 的形式存在 SpanContext 中。系統中所有子服務在溝通時必須傳遞同一個 SpanContext 至下一個服務。

一個服務可以根據需要自行產生多個 Span 以標註程式中不同的階段。但根據目前最主流的分散式追蹤的工具 OpenTelemetry 的實作，當 Trace 跨越服務邊界時，必須產生新的 Span，也就是同個 Span 是不能跨越服務邊界的（服務邊界包含了 process boundary, network 等等）

### Time-based View of Trace

除了 DAG 圖以外，Span 也經常被視覺化在時間軸上，如下圖：
<div style="text-align:center;width:100%;">
<figure class="image">
<img style="width:70%;margin-bottom:0px;" src="/blog/2020/07/20/distributed-tracing-1/sync-services.png">
  <figcaption>Figure 4. Time-based View of Trace - 同步取用 D, E </figcaption>
</figure>
</div>

以時間軸顯示 trace 時，可以很清楚地看出 request 在每個子系統中被處理的先後順序與花費時間，進而對潛在的優化方式提供 insight。

以 B 服務為例，若改以非同步（asynchronous）取用 D, E 服務，能夠有效縮短 b 的 duration，進而縮短整體 a 的 duration，如下圖：

<div style="text-align:center;width:100%;">
<figure class="image">
<img style="width:70%;margin-bottom:0px;" src="/blog/2020/07/20/distributed-tracing-1/async-services.png">
  <figcaption>Figure 5. Time-based View of Trace - 非同步取用 D, E </figcaption>
</figure>
</div>

一般而言，Time-based 的視覺化會比 DAG 更常用到，也是 [Jaeger](https://www.jaegertracing.io/docs/1.18/) 這個 Trace 收集系統預設的視覺化方式。

## Sample（取樣）

雖然分散式追蹤產生一個 Span 的延遲極低（nano second 等級），但是對於跨多個子服務的高併發低延遲服務，開啟 Tracing 能造成的 Performance Impact 依然是很可觀的。因此實務上來說不一定會記錄所有 Request 的 Trace，而是使用 Sample（取樣）的機制選擇性記錄。

下面表格是 Google 在 2010 提出的 paper ["Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"](https://research.google/pubs/pub36356/) （強烈推薦閱讀原文，此 paper 定義了現代 Distributed Tracing 架構）中，對於當時內部使用的分散式追蹤工具 Dapper 所做的效能測試：

<div style="text-align:center;width:100%;">
<figure class="image">
<img style="width:50%;margin-bottom:0px;" src="/blog/2020/07/20/distributed-tracing-1/tracing-perf.png">
  <figcaption>Table 1. Performance Benchmark of Dapper </figcaption>
</figure>
</div>

在這個測試中，Google 對自家的搜尋引擎 API 加上 Distributed Tracing，並設定不同的 Sampling frequency（取樣頻率）來觀察效能影響。在 Sampling rate 為 1 時（意即記錄所有 Requests 的 trace ），平均 latency 增加了 **16.3%** 之多，之後 Latency change % 隨著 Sampling Rate 降低而降低。而根據 paper 的說明，把 Sampling rate 降至 **1/16** 以下後，Tracing 對效能造成的影響就沒有那麼顯著了。（另外，Paper 中也提到了 Latency 與 Throughput 的實驗誤差個別是 2.5% 及 0.15%，這也是為什麼表格中的 Latency change % 在 Sampling rate = 1/1024 時會是負的。）

### Sampling Rate 設定多少好？

這個效能測試的結果是否代表我們不該使用 Sampling Rate = 1 呢？我認為不是。前面也提到了產生一個 Span 只需要幾 nano second 而已，而 Google 的實驗結果對 latency 影響如此顯著，我想背後可能原因為：
1. Google 搜尋 API 原本效能就很好，延遲很低，所以算出來的 change  % 就會比較高。
2. 此 API 的背後可能通過了很多個子服務，因此會產生很多 span，增加了較多 latency。

Sampling Rate 應該設為多少？這個問題必須依據你的系統架構、效能需求以及使用情境來決定。但不變的真理是：**Benchmark 是好物**，在決定 Sampling Rate 前最好搭配適當的效能測試、取得數據來幫助決策。

### 如何實作 Distributed Tracing？

要對所有跨服務的 Request 增加額外的 Header 其實是滿辛苦的工作，首先不同子服務之間不一定是用一樣的 protocol 溝通，目前比較流行的方式就有 HTTP、gRPC 或是 Message Queue。且若是一個子服務實作 Tracing 時不小心寫了 bug 而沒有正確傳遞 Header，整個 Trace - Span 的結構就會中斷。

由於 Distributed Tracing 的條件是如此嚴格，早在 2010 年以前分散式追蹤還是 Google 自己的內部專案時，就習慣把 Tracing（Header 收集與傳遞）實作在 Web Framework 或 Library level（稱為 "instrument"），使用者只要在跟子服務互動時使用 instrumented library 就能完成 Tracing。而後來 Open Source 社群的實作，不管是 OpenTracing、OpenConsensus 到最近的 OpenTelemetry，都會直接對較流行的程式語言以及該語言主流的 Web framework、Request library 實作相關的 instrumented library。

簡單一句總結就是：先找現有的 Tracing library 來用（並且有很大的機會你可以找到！）

## 小結

Distributed Tracing 是很好用的技術，筆者曾經使用它來找到不少 API 的效能瓶頸，直觀的視覺化方式也能幫助團隊成員快速理解自家服務的運作。

雖然 Distributed Tracing 以往得到的關注度不如 Logging 與 Metrics，但現在由於 Docker、Kubernetes 技術的普及，microservices 也幾乎是網路服務的預設架構了，相信之後 Distributed Tracing 也會漸漸的成為 microservice cluster 中的必備定番。





## 參考資料
- OpenTelemetry Specification:
https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md
- "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"(Google Paper):
https://research.google/pubs/pub36356/
- Istio Distributed Tracing Overview:
https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/
- Jaeger - A Distributed Tracing System:
https://www.jaegertracing.io/docs/1.18/
- Three Pillars of Observability:
https://www.scalyr.com/blog/three-pillars-of-observability/

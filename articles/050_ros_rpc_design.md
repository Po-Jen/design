---
layout: default
title: ROS 上 RPC API 的設計細節
permalink: articles/ros_rpc.html
abstract:
  這篇文章是用來探索下個世代的 ROS 的 Remote Procedure Call 的介面(interface)適合用什麼 design patterns，我們著重於特定的使用者 API 並且不實作(implement)細節。我們期待一個 RPC 可以選擇不同的實作，就像 Apache Thrift, ROS RPC 或 MsgPack。

author: '[Tully Foote](https://github.com/tfoote)'
published: true
translator: yenWu
---

* This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

ROS 有兩種 RPC 的原型(primitive)，ROS Services 是最基本的 request-response 類型的 RPC，當 requests 送出時，ROS Actions 是具有可搶占性(preemptive)並會產生 feedback。

## Ideal System
思考如何完成理想系統，我們必須明瞭他和現在系統的關西，和如何使一個新的系統能夠運作，一個理想的 RPC 系統能必須確保每個元件的品質。
It is useful to consider the ideal system to understand how it relates to the current system and how a new system could work.
An ideal RPC system would have the qualities laid out in the following paragraphs.

### Asynchronous API

Asynchronous API 提供一個變相的 threading models，並且比 Synchronous API 有更好的彈性，
Synchronous API 將可以建置在 Asynchronous API 之上，而相反的，將 Asynchronous API 建置在 Synchronous API 是相對困難且比較沒效率的。

### Timeouts

當一個 service 提供者 hang 住或者是無法正確的 return，那麼 calling thread 也將會無期限的 hang 住。
Timeout 提供了一個錯誤回復的機制(ex: 連線失敗)，並讓使用者可以選擇繼續等待或是 abort。

If a service provider hangs or otherwise does not return correctly, then a calling thread may be hung indefinitely.
Having a timeout allows for recovery behavior in the case of failure conditions besides a dropped connection, allowing the user to choose to continue waiting for another timeout cycle or abort the request.

### Preemptibility

Preemption 是個非常需要的特色，特別是在長時間運行或是不確定執行時間的 remote procedues。
我們特別希望使用 timeout 或是 asynchronous request 來 preempt 一個長時間運行的 procedue。
Preemptibility 是個用來實現 ROS Actions 的需求之一(這是 ROS Action 為什麼要使用 asynchronous ROS Message 而不是 synchronous ROS Services)。

### Feedback

為了增進 preeption without timeout 的效率，我用需要週期性或是 procedural 的 feedback。
Feedback 的實現可以透過額外的機制(ex: 隱性的 publish-subscribe channel)。
Feedback 是 ROS Actions 的核心觀念。

### Reliable Transport

不讓系統陷入不確定狀態(ex:封包遺失)是很重要的。
當一個 request 或是 response 已經無法遺失了，我們必須要能夠察覺到這一點，並且使用一些方法來回復這個資訊或是通知 user 遺失了。ROS 1.x 缺乏這個機制，導致 ROS Actions 會出問題(e.g, 當使用 wireless 連結)。

It is important that the system cannot get into an undetermined state if there is packet loss.
If a request or response is never received by either side the system must be able to notice this loss, then recover and/or inform the user in some way.
In ROS 1.x, this lack of reliability has been a problem for ROS Actions, e.g., when they are used over lossy wireless links.

### Logging and Introspection

When logging a ROS 1.x system (e.g., using `rosbag`), recording data transmitted on topics is insufficient to capture any information about service calls.
Because service calls are conceptually point to point, rather than broadcast, logging them is difficult.
Still, it should be possible to efficiently record some level of detail regarding RPC interactions, such that they could be later played back in some manner (though it it not clear exactly how playback would work).

## Proposed Approach

The features outlined above are desirable but if provided as a monolithic implementation will be much more complicated than necessary for most use cases.  E.g., feedback is not always required, but in a monolithic system it would always be an exposed part of the API.
We propose four levels of abstraction into which the above features can be sorted, wich each higher level providing more functionality to the user.

![ROS RPC Higherarchy](/img/ros_rpc_design/rpc_diagram.png)

### Plain RPC

The Plain RPC API is expected to be able to be leveraged from one or more externally developed RPC libraries.
We expect several libraries to meet the minimum requirements and aim to make them interchangeable.

### ROS Asynchronous RPC API

The ROS Asynchronous RPC API will provide the lowest level of abstraction.
It will provide a callback-based API with a timeout.
It will utilize the Plain RPC API to do the communication as well as provide reliable communications either by leveraging the Plain RPC's capabilities or providing a layer on top of them with message acknowledgments.

For logging/introspection purposes the RPC Server instance might publish all incoming requests and outgoing responses on topics inside the namespace of the service.

### ROS Preemptible RPC API

The ROS preemptible RPC API will extend the Asynchronous API to enable preemption of an RPC in progress using a UID.
This UID will be provided by the initial request method.

### ROS Action RPC API (Not effecting RPC Protocol)

The feedback topic can be isolated to a separate topic, which avoids integrating the feedback into the core RPC implementation.
The ROS Action RPC API will extend the preemptible RPC API to provide a feedback channel via published ROS topic.
This can be built on top of the preemptible RPC API with the PubSub API thus isolating it from the RPC design.

### ROS Synchronous RPC API (Not Effecting RPC Protocol)

For each of the above Asynchronous APIs a thin wrapper can be built on top to provide a single function-based interface for ease of use.
It will block until a response is returned or the timeout is reached.
If wrapping a preemptible RPC, it will both timeout on the user side as well as preempt the remote side.
This will just be a thin layer on top of the Asynchronous API requiring no additional features of the core RPC protocol.

## Technical Issues

There are some issues with the above proposed approach, which are outlined below.

### Visibility of UID's

UID's are generally necessary for asynchronous communications to pair the requests and the responses.
There are possible ways to build this without a UID embedded in the data type, however it will require some level of heuristics to do data association.

There are two options: (i) require the user to embed the UID into the message, or (ii) add those fields automatically at message-generation time.
It is unclear how this choice affects the user's ability to introspect the messages.
This also introduces issues when trying to record and potentially play back Service or Actions.

### Action Files?

Should there be a separate `.action` file type?
Or should it be more like a `.srv` + `.msg` pair?
This is highly influenced by the way UID's are handled.

### Logging

Recorded service calls can not be played back generically like topics because the service client will not handle it because it will not be associated with a request.
There could be some test service servers which can respond with similar queries, however it is not clearly defined.

Logging of RPCs is still valuable for debugging and introspection.
It would be valuable to have a view of events that happened in sequence as well as their content.
It should be possible to associate recorded request and response pairs,
which raises the question of how to embed this association without significantly affecting the user's API.

This is a generic issue with logging and affects potentially all logging and should be captured in a separate article.
It might be possible to pad communications with debugging data.

The above UID's may be only locally unique (client-specific for instance).
For logging, UID's need to be unique within the entire ROS instance to support debugging.

### Collapse Preemptible and Asynchronous?

These two types are very similar and limiting the variants on the API might be easier for the user.
If the UID must be generated/embedded into the protocol,
then it should be embedded into all calls, which will help with logging.
For the non-preemptible case the implementation can simply not instantiate the state machine and preemption mechanisms.

### caller_id availability?

There are use cases when the concept of `caller_id` is valuable.
Users of ROS Actions would sometimes use it to provide connection-based information.
By default the anonymous publish-subscribe mechanism is to not provide the `caller_id`.
In the current implementation the `caller_id` is actually the node name, which is ambiguous in cases like nodelets.
Providing a mechanism for declaring the `caller_id` might be helpful, but would require a lot of tools to parameterize and remap in order to gain the full benefit.

One possible solution is to write a spec where a field that matches `[CallerID caller_id]` would be automatically substituted by a publisher if embedded into a message to provide this specifically where valuable.

### Do not bundle feedback at core level?

If namespace remapping works, feedback could simply be a topic which is in the same namespace as a peer.
This would enable multiple feedback topics of differing types and frequencies.
Action API classes could be provided on top of the core infrastructure to bundle an RPC with a feedback.

---
layout: default
title: 使用ZeroMQ跟相關的函式庫來開發ROS
html_title: 使用ZeroMQ跟相關的函式庫來開發ROS
permalink: articles/ros_with_zeromq.html
abstract:
  這篇文章討論如何使用[ZeroMQ](http://blog.ez2learn.com/2011/12/31/transport-lib-of-new-era-zeromq/)跟其他的函式庫來開發ROS 2.0。此外，我們在OSRF(Open Source Robotic Foundation)使用ZeroMQ開發出的prototype也會在這篇文章中被討論。
published: true
author: '[William Woodall](https://github.com/wjwwood)'
translator: 賴柏任
---

{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

> **這篇文章是在決定使用DDS當作ROS 2.0的資料傳輸工具之前所撰寫的**

Original Author: {{ page.author }}

Translator: {{ page.translator }}

雖然這篇文章主要聚焦在使用ZeroMQ來建一個新的中介軟體（middleware），但其實也討論了如何利用數個函式庫來構建出一個新的中介軟體。值得注意的是，使用數個函式庫來打造來ROS的策略，跟使用已存在且提供許多功能的中介軟體（例如DDS）來打造ROS的策略是大相逕庭的。

## 從頭打造一個中介軟體的Prototype

為了滿足ROS的需求，我們新開發出的中介軟體需要提供幾個重要的服務。

首先，這個中介軟體需要讓建立在其之上的各個子系統可以發現彼此(discovery)，並且可以在執行期間動態地建立連結以利互相溝通。第二，建立連結之後，要有一個以上的傳輸機制讓子系統之間可以傳遞訊息，以ROS而言，至少要提供訂閱-發布的溝通機制。如此一來，額外的溝通機制（例如送出要求跟回應的service機制）都可以使用訂閱-發布的機制來實作。最後，這個中介軟體應該提供方法讓使用者定義訊息格式並提供傳輸機制（也就是將訊息序列化）。不過即使中介軟體本身不實作這個機制，也能透過使用其他函式庫來解決這個問題。

### 讓各個子系統發現彼此

在考慮discovery這個問題時，我們最先想到的解決方案是[Zeroconf](http://blog.csdn.net/ccskyer/article/details/7616673)這個協議，搭配Avahi跟Bonjour這兩套實作zeroconf協議的函式庫。
我們使用[pybonjour](https://code.google.com/p/pybonjour/)做了一些簡單的實驗，來測試以zeroconf協議為基礎的系統中discovery的效果。
其中的核心技術是`mDNSresponder`，它是蘋果公司提供的自由軟體，同時被Bonjour(OS X跟Windows)以及Avahi(Linux)所使用。

但是經過我們的實驗發現，這些實作zeroconf協議的函式庫並不能非常穩定地維持多機器之間的網路狀態圖。如果使用子程序一次加入或移除超過20台機器，在整個網路中至少會有一台以上的電腦無法跟其他電腦保持相同的網路狀態圖。舉例來說，假設A電腦已經發現了幾個nodes(一個node表示網路中的一個連線單位，在此例中是電腦)，B跟C電腦也發現這幾個nodes並已經把這些nodes顯示在自己的連線狀態圖中。當這些nodes所表示的電腦在幾秒之後被關掉電源，理論上，A~C這三台電腦上的網路狀態圖都應該同時把這些nodes移除掉，但問題就是，通常只有部份的nodes會被移除（被留下來的nodes就變成了zombie nodes），更麻煩的是，三台電腦移除的nodes還不一致。這個問題只有在多機器網路中使用Avahi才會出現，所以我們更進一步地深入了解Avahi的核心，來確認這個問題是否可以解決。然而，更進一步的了解讓我們更擔心Avahi的穩定性和品質，尤其是在[Multicast DNS](http://en.wikipedia.org/wiki/Multicast_DNS)和[DNS Service Discovery](http://en.wikipedia.org/wiki/Zero_configuration_networking#Service_discovery)這兩項技術的實作上。

除此之外，DNS-SD似乎傾向於讓網路的負載變小以維持穩定性。
這個特性在查詢網路中有哪些服務可用時運作得相當良好，但是在我們實驗用的ROS graph測試中（畫出網路中所有ROS node的圖），它沒有辦法快速且穩定地發現所有存在於網路中的node。
因為這個缺點，我們發展了自製的discovery系統，以下的連結中有我們用數種程式語言實作的prototype:

[https://bitbucket.org/osrf/disc_zmq/src](https://bitbucket.org/osrf/disc_zmq/src)

這個自製的discovery系統使用了群播的UDP封包來傳送通知，例如"Node啟動"、"產生一個發布者"、"新增一個訂閱"等等，其中也包含一些運作時需要用到的後設資料（meta data），例如發布者會需要跟ZeroMQ取得資料該傳輸到哪個地址。其他跟這個自製discovery系統相關的細節可以從上面的網址中找到。

這個系統雖然簡單，但卻很有效率，而且可以證明即使用多種語言實作這樣的自製discovery系統，還是一個可以處理的問題。

### 資料傳輸

要在不同程序間傳遞資料，有一個很常被使用的函式庫就是[ZeroMQ](http://zeromq.org/)，當然我們還有一些其他的選項例如[nanomsg](http://nanomsg.org/)跟[RabbitMQ](http://www.rabbitmq.com/)。不管是哪一個函式庫，它們的主要功能都是讓使用者可以明確地建立連結，利用某些通訊的樣式來傳遞一系列的字串或位元組。

近年來，ZeroMQ變得相當熱門，它是一個採取LGPL授權條款規範的函式庫，主要是用C++撰寫，提供C的API，而且有提供許多程式語言的binding。
nanomsg是一個受MIT授權條款規範的函式庫，它的作者也是ZeroMQ的原作者群其中之一，實作跟API都是使用C，但它的成熟度遠比ZeroMQ來得低。
RabbitMQ是一個受Mozilla Public License授權條款規範的函式庫，它主要的角色是一個中間人(Broker)，其中實作了幾種資料傳輸的協議，主要是[AMQP](http://lab.howie.tw/2012/07/whats-different-between-amqp-and-jms.html)，不過它也提供了跟ZeroMQ、STOMP和MQTT溝通的閘道器（gateway）。作為一個Broker，它也滿足了一些discovery跟資料傳輸的需求。雖然ZeroMQ常常被用在brokerless的環境中，但它也常跟RabbitMQ搭配使用，讓訊息的傳遞更加穩定。
上述的這些函式庫都可能可以取代ROSTCP的傳輸機制，但因為這篇文章主要聚焦在ZeroMQ上，所以我們討論使用ZeroMQ的情況，且我們不使用broker（http://zeromq.org/whitepapers:brokerless）。

在我們製作的這個prototype中：

[https://bitbucket.org/osrf/disc_zmq/src](https://bitbucket.org/osrf/disc_zmq/src)

ZeroMQ被用來處理資料傳輸（它提供了C、C++跟Python的binding，相當方便）。在使用前一個小section提到的函式庫建立的簡易discovery系統來找到系統中其他節點之後，ZeroMQ被用來建立連結（使用`ZMQ_PUB`跟`ZMQ_SUB`的socket）。在我們的實驗中，這個prototype運作順暢，使得程序之間可以使用簡單又有效率的方式互相溝通。不過，如果要讓系統具備更進階的功能，例如instance latching，那我們就需要再自己額外進行實作。總結來說，使用ZeroMQ來實作雖然可以維持基本功能的精簡性，但也意味著我們需要實作且維護更多的程式碼來
提供進階的功能。

除此之外，ZeroMQ特別依賴不會出錯的傳輸機制，例如TCP或[PGM (Pragmatic General Multicast)](http://en.wikipedia.org/wiki/Pragmatic_General_Multicast)，所以它並不適合用在soft real-time的情境之下。

### 訊息序列化

在ROS 1.x裡面，訊息被定義在`.msg`檔裡面，這個訊息格式對應到的程式碼會在編譯時被自動產生。ROS 1.x所產生的程式碼可以初始化訊息，並將其序列化以利資料的傳遞。自從ROS出現之後，有幾個處理訊息序列化的著名函式庫也開始出現，其中包含其中Google的[Protocol Buffers (Protobuf)](https://code.google.com/p/protobuf/), [MessagePack](http://msgpack.org/)、[BSON](http://bsonspec.org/)以及[Cap'n Proto](http://kentonv.github.io/capnproto/)。

如果要討論不同訊息格式、序列化方式之間的優缺點，我們可以另外撰寫一整篇文章，但因為我們只是要測試prototype是否可行，所以僅僅使用一般的字串或是Protobuf就足夠了。

## 結論

在實作了一個簡易的prototype之後，我們認為有幾個關鍵點值得提出來討論。首先，目前不存在一個discovery系統可以支援跨中介軟體之間的discovery，雖然自己實作可以滿足上述功能的discovery系統是可行的，但非常花時間。

第二個值得注意的點是，如果想整合discovery、資料傳輸跟資料序列化的功能，我們還需要更多的函式庫被撰寫。舉例來說，若我們考慮連線建立的方式，不論是用點對點傳輸或是廣播，都需要額外的程式碼來處理(因為這件事並不在資料傳輸或是discovery系統這些函式庫的實作範圍內)。另外一個例子是跨程序間的溝通，ZeroMQ提供INPROC socket來處理這件事，但這種socket接收的資料格式是位元組，所以你必須先將資料進行序列化再將指向這些資料的指標傳給INPROC socket。當你使用傳指標的方式來實作，程序之間的傳輸和程序之內的傳輸的行為就會相當相似，但這兩者在ROS API的抽象層會被抽象化掉。另一個需要為其撰寫程式的功能就是在資料序列化系統和資料傳輸系統之間的型別檢查系統。在看了這麼多例子之後，我們清楚地知道，就算已經存在幾個重要的函式庫，在這些函式庫之間的銜接部分仍需要被實作，以完成像ROS這樣的中介軟體。

雖然使用ZeroMQ和Protobuf這些函式庫來實作ROS這樣的中介軟體需要花費許多資源，至少完成的結果很有可能可以被調校到相當穩定，而且各個子系統分工清楚。這種實作方式還可以將ROS的控制權交給ROS社群(因為構成子系統的函式庫是可以被抽換的)。

但是，可以隨自己所想來更動ROS這個中介軟體的代價就是，對於不同實作版本的說明文件會更重要，才能讓使用者可以依照說明文件上的指示驗證手上的ROS版本具備有其應該有的特性。這件事情一點都不簡單，至少在ROS 1.x的階段就做得不是很好，這正是因為ROS有幾種不同的實作版本。有許多使用者在嘗試把ROS應用到需要精密控制的任務上或是商業化的產品上時，都在抱怨ROS缺少了一份詳盡的、說明文件，讓他們無法驗證或是完整審視整個系統。在一開始就把這個中介軟體實作得相當完善也是一個選項，但這並不容易，而且對於新版本的實作來說，這絕對是相當耗費成本的一種作法。

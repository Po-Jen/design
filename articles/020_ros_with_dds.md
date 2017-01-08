---
layout: default
title: 使用DDS來開發ROS
permalink: articles/ros_on_dds.html
abstract:
  這篇文章討論如何使用DDS來開發ROS 2.0，列出這種作法的優缺點，並討論使用者經驗和API帶來的影響。我們也會針對我們實作的prototype "ros_dds"給一些總結意見，並探討其中潛藏的問題。
author: '[William Woodall](https://github.com/wjwwood)'
published: true
translator: 賴柏任
categories: Middleware
---

{:toc}

# {{ page.title }}

<div class="alert alert-warning" markdown="1">
This article was written at a time before decisions were made to use DDS and RTPS as the underlying communication standards for ROS 2.
For details on how ROS 2 has been implemented, see the [Core Documentation](http://docs.ros2.org/)
</div>

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

Translator: {{ page.translator }}

專業術語:

- [Data Distribution Service (DDS)](http://en.wikipedia.org/wiki/Data_Distribution_Service)
- [Real-Time Publish Subscribe (RTPS)](https://en.wikipedia.org/wiki/Real-Time_Publish-Subscribe_(RTPS)_Protocol)
- The [Object Management Group (OMG)](http://www.omg.org/)
- OMG [Interface Description Language (IDL)](http://www.omg.org/gettingstarted/omg_idl.htm) | [Formal description](http://www.omg.org/cgi-bin/doc?formal/2014-03-01)

## 為什麼考慮DDS?

當我們在探索可以用於開發下一代ROS的通訊系統的選項時，最先考慮的是改進ROS 1.x的傳輸機制或是使用現有的函式庫來建構(例如使用ZeroMQ、Protocol Buffer和zeroconf)。但是，若採取上述這些選項，我們要不是得從頭打造整個中介軟體、不然就是要從各個部分開始組合，所以我們也考慮直接使用功能完整的其他中介軟體來打造ROS 2.0。經過我們的調查之後，其中一個脫穎而出的選項就是DDS。

### 一個完整的中介軟體

使用一個像DDS這種完整的中介軟體來開發的好處是，我們需要維護的程式碼少了很多，而且這種中介軟體的行為和詳細規格都已經被詳細地撰寫成文件。除了系統層級的說明文件之外，DDS也有推薦的使用個案和軟體API。有了這些具體的說明文件和規格，第三方的開發者就可以檢驗、審視，甚至可以根據自己的需求來實作具備不同程度的互動性的版本。這項特色是ROS從未有過的(ROS只有少量的基礎說明文件和一些可供參考的實作版本)，而且，就算使用現有的一些函式庫來建構起ROS，對於規格和軟體行為的詳細說明文件還是不可或缺的。

當然，這種開發方式也有缺點，就是ROS 2.0也會被DDS的設計所限制。如果DDS的設計不夠用於解決我們想要的使用情境或是不夠彈性，我們就必須去更動DDS的設計。某種程度上來說，採用完整的中介軟體來開發也包含了要採用這個中介軟體的開發哲學和社群文化，這件事也不應被輕忽。

## DDS是什麼?

DDS提供跟ROS 1.x很相似的一種訂閱-發佈傳輸機制。它使用[Object Management Group (OMG)](http://www.omg.org/)定義的“Interface Description Language (IDL)”來實作訊息的定義和序列化。雖然DDS還沒有提供request-response的傳輸機制(這可以被當作ROS的service系統)，但這種傳輸方式的實作已經有初步規劃，第一個Beta版也已經在2015年春天發佈(稱作 [DDS-RPC](http://www.omg.org/spec/DDS-RPC/))。

DDS提供的預設discovery系統是一種分散式的discovery系統(這有別於ROS 1.x必須由master來掌管所有node的方式)，其中也使用DDS的發佈-訂閱傳輸機制。這使得任意兩個DDS程式可以不透過ROS master這種機制就能相互溝通。這使整個系統的容錯程度更高、也更彈性。而且，我們不一定要使用動態的discovery機制，有幾種DDS的實作版本都提供了靜態discovery機制的選項。

### DDS是怎麼出現的?

DDS起源自一群擁有提供相似中介軟體的公司，因為這些公司的客戶希望這些公司提供的中介軟體可以彼此溝通，所以這些中介軟體漸漸地被整合起來形成一套標準。DDS的標準是由Object Management Group建立起來的，跟建立UML、CORBA和SysML和其他軟體標準的是同一群人。這可能是好消息也可能是壞消息，看你怎麼想。一方面，你擁有一個標準制定委員會，他們會不斷討論而且在軟體工程社群擁有極大的影響力；但另一方面，他們前進地相對緩慢、對於改變的適應力也較差，所以也很可能無法跟上軟體工程界的最新潮流。

DDS一開始是幾個相似的中介軟體，後來因為彼此變得太相近，制定一個統一的標準來整合這些中介軟體才變得合理。在這樣的脈絡之下，雖然DDS規格的制定是由委員會撰寫出來的，但他的演進原本就相當貼近使用者的需求。由於在標準制定之前就已經是藉由跟使用者的互動來演進，這多少可以消弭大家對於DDS的擔心，擔心它只是一個憑空想像出來的標準、但不符合實際應用的需求。歷史上也曾有一些標準委員會所制訂出的標準，雖然立意良善且制訂得很完整，但沒人想要使用或是不符合使用者需求。不過DDS看起來沒有這種問題。

除了上述的擔憂，也有人擔心DDS將會是一個不更新的標準，這種刻板印象來自於UML和CORBA給人的不良回憶，因為這兩者也都是OMG所制定的標準。不過DDS看起來不會有這種問題，除了最近仍持續在更新之外，現在也還在新增更多的規格，包含websockets、SSL安全協議、可擴充的型別、request-response傳輸機制以及C++11風格的API(用來取代現在的C++介面)。對我們來說，DDS的標準仍持續地演進是一件很正面的事，雖然跟現今軟體工程的潮流比起來仍相對緩慢，但它依然在往滿足其使用者需求的方向在前進。

### DDS技術上的可信度

有許多要求高精密度的系統都已經在使用DDS，它已經被用在:

- 軍艦
- 大型公共建設(例如水壩)
- 金融系統
- 航太系統
- 飛行系統
- 火車控制系統

除了這些使用情境之外，還有許多同等重要的不同系統都已經在使用DDS，這些成功的案例使得DDS設計的穩定度和彈性更加被信任。

DDS不僅滿足了這些使用情境下的需求，我們進一步訪問了政府及NASA的使用者(其中有些人也是ROS的使用者)，他們對於DDS的穩定性和彈性都讚譽有加。不過他們也提醒，DDS具備高彈性的代價就是高複雜度，它在API以及設定上的複雜度是我們應用DDS來開發ROS時需要注意的問題。

DDS的連接協議(DDSI-RTPS)具有極高的彈性，使得DDS可以被使用在需要高穩定性、高層次的的系統整合，也可以被real-time地執行在嵌入式裝置上。有幾個DDS函式庫的提供者都各自實作了用於嵌入式裝置的DDS，也都各自誇耀自己的實作在函式庫佔用的空間和記憶體使用量只需幾百KB。而由於DDS使用UDP傳輸協儀來實作，它不必依賴穩固的硬體或網路來傳輸。不過這也表示DDS需要自己重新建立確保資料傳輸不會遺漏的機制(基本上就是TCP機制，可能多幾個或少幾個功能而已)，好處是DDS較容易被移植到不同裝置上(因為較不穩定的硬體也能使用)、且對軟體行為的控制權增加。在DDS的實作中，可以藉由Quality of Service(QoS)來控制決定穩定性的參數，讓使用者有最高的彈性來控制傳輸時的行為。舉例來說，如果你希望程式執行滿足soft real-time的要求，那網路延遲(latency)就是一個需要考慮的問題，你可以將DDS設定成只使用UDP來傳輸。在另一種情形中，你可能希望DDS像TCP一樣可以可靠地傳送正確的資料，那就可以透過DDS的QoS參數來調整這些行為。

雖然DDS的實作預設是使用UDP傳輸協議，而且只要求這種等級的傳輸功能，OMG仍然在1.2版的DDS標準中新增了DDS使用TCP的部分。稍微survey一下，就會發現有兩個DDS實作版本的提供者(RTI跟PrismTech)都提供用TCP實作的DDS。

RTI的網站上有一段話([http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport](http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport)):

> RTI Connext DDS預設使用UDPv4和共享記憶體的方式來跟其他的DDS應用溝通。不過在某些情況下，discovery和資料傳輸時需要用到TCP的傳輸方式。如果你想知道更多關於RTI的TCP傳輸方式，可以去看RTI Core Libraries and Utilities User Manual裡面的 “RTI TCP Transport” 章節。

PrismTech的規格說明書上也寫了他們支援DDSI-RTPS1.2版裡的TCP部分:

[http://www.prismtech.com/opensplice/products/opensplice-enterprise/opensplice-dds-core](http://www.prismtech.com/opensplice/products/opensplice-enterprise/opensplice-dds-core) (search for TCP)
=======
From RTI's website ([http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport](http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport)):

> By default, RTI Connext DDS uses the UDPv4 and Shared Memory transport to communicate with other DDS applications.
> In some circumstances, the TCP protocol might be needed for discovery and data exchange.
> For more information on the RTI TCP Transport, please refer to the section in the RTI Core Libraries and Utilities User Manual titled "RTI TCP Transport".

From PrismTech's website, they support TCP as of OpenSplice v6.4:
>>>>>>> 42d8775846c51ae520960dd874923d294d5e96b4

[http://www.prismtech.com/vortex/vortex-opensplice/communication/ddsi2](http://www.prismtech.com/vortex/vortex-opensplice/communication/ddsi2)

### 發佈者跟授權條款

OMG跟幾間公司一起制定了DDS的標準規格，這幾間公司也同時是DDS實作版本的發佈者，幾間比較知名的有:

- RTI
- PrismTech
- Twin Oaks Software

這些公司提供的實作版本在實作策略和授權條款上都有些差異，OMG列出了一份DDS發佈者的[清單](http://dds-directory.omg.org/vendor/list.htm)。

RTI實作的Connext DDS是使用他們自訂的[Community Infrastructure License](https://www.rti.com/downloads/IC-license.html)來授權，這個授權條款符合ROS社群的需求，不過若要採用RTI的Connext DDS當作ROS主要使用的DDS實作版本，仍需要進一步的討論。我們這裡所說的"符合ROS社群的需求"的意思是，雖然Community Infrastructure License並不是[OSI](http://opensource.org/licenses)認可的授權條款，但是有研究指出這個授權條款還是可以允許ROS以[BSD](http://www.openfoundry.org/tw/legal-column-list/524--bsd)授權條款的形式被散佈，而且ROS的使用者可以任意地以原始碼或執行檔的形式散布自己重製的版本。RTI看起來也願意對授權條款進行溝通協調以滿足ROS社群的需求，不過這當然需要RTI和ROS社群間的來回討論來確保這件事情可行。跟其他發佈者相似的是，RTI的授權條款授權的部分只有函式庫的核心功能，也就是基本的DDS API，但其他像是開發工具或是introspection工具都是RTI私有的。此外，RTI看起來擁有最多的使用者。
=======
In addition to vendors providing implementations of the DDS specification's API, there are software vendors which provide an implementation with more direct access to the DDS wire protocol, RTPS.
For example:

- eProsima

These RTPS-centric implementations are also of interest because they can be smaller in scope and still provide the needed functionality for implementing the necessary ROS capabilities on top.

RTI's Connext DDS is available under a custom "Community Infrastructure" License, which is compatible with the ROS community's needs but requires further discussion with the community in order to determine its viability as the default DDS vendor for ROS.
By "compatible with the ROS community's needs," we mean that, though it is not an [OSI-approved license](https://opensource.org/licenses), research has shown it to be adequately permissive to allow ROS to keep a BSD style license and for anyone in the ROS community to redistribute it in source or binary form.
RTI also appears to be willing to negotiate on the license to meet the ROS community's needs, but it will take some iteration between the ROS community and RTI to make sure this would work.
Like the other vendors this license is available for the core set of functionality, basically the basic DDS API, whereas other parts of their product like development and introspection tools are proprietary.
RTI seems to have the largest on-line presence and installation base.
>>>>>>> 42d8775846c51ae520960dd874923d294d5e96b4

PrismTech的DDS實作版本OpenSplice是用LGPL授權條款來發佈，跟其他著名的開源函式庫是相同的，例如glibc、ZeroMQ跟Qt。可以直接在Github上找到:

[https://github.com/PrismTech/opensplice](https://github.com/PrismTech/opensplice)

PrismTech的實作版本還包含基本的[編譯功能](http://www.opensplice.org/dds-community/building)，而且相對容易被包成package。OpenSplice的使用者人數看起來是第二多的，不過很難判斷到底是不是真的如此。

TwinOaks實作的CoreDX DDS完全是私有的，而且他們專住在最精簡的實作以利讓CoreDX DDS可以在嵌入式裝置，甚至一塊開發板上就能運行。

<<<<<<< HEAD
既然擁有LGPL授權的OpenSplice以及RTI自訂的授權條款(可以進一步溝通協調)這些選項，使用現成的DDS實作版本或把它當作相依的函式庫重新發佈看起來都是相當可行的。我們在設計ROS 2.0的其中一個目標就是讓DDS的實作版本可以被替換，舉例來說，假設預設的DDS實作版本是RTI的Connext DDS，如果有人想要使用OpenSplice，那只要更改一個選項、重新編譯ROS原始碼，就可以使用OpenSplice。

![DDS and ROS API Layout](../img/ros_on_dds/api_levels.png "DDS and ROS API Layout")

這件事之所以可行是因為DDS的標準規格中已經制定了它的API，所以不同實作版本的API是一致的。有人做過較深入的研究確定這件事是可行的，雖然不同實作版本之間還是有一些細微差距，例如回傳的形別不同(pointer或是shared_ptr這種差別)跟標頭檔的規劃方式。
=======
eProsima's FastRTPS implementation is available on GitHub and is LGPL licensed:

[https://github.com/eProsima/Fast-RTPS](https://github.com/eProsima/Fast-RTPS)

eProsima Fast RTPS is a relatively new, lightweight, and open source implementation of RTPS.
It allows direct access to the RTPS protocol settings and features, which is not always possible with other DDS implementations.
eProsima's implementation also includes a minimum DDS API, IDL support, and automatic code generation and they are open to working with the ROS community to meet their needs.

Given the relatively strong LGPL option and the encouraging but custom license from RTI, it seems that depending on and even distributing DDS as a dependency should be straightforward.
One of the goals of this proposal would be to make ROS 2.0 DDS vendor agnostic.
So, just as an example, if the default implementation is Connext, but someone wants to use one of the LGPL options like OpenSplice or FastRTPS, they simply need to recompile the ROS source code with some options flipped and they can use the implementation of their choice.
>>>>>>> 42d8775846c51ae520960dd874923d294d5e96b4

This is made possible because of the fact that DDS defines an API in its specification.
Research has shown that making code which is vendor agnostic is possible if not a little painful since the APIs of the different vendors is almost identical, but there are minor differences like return types (pointer versus shared_ptr like thing) and header file organization.

### 社群及其文化

<<<<<<< HEAD
DDS從一群老公司的產品中產生出來，由OMG這個老派的軟體工程標準制定組織規劃，並大量被使用在政府跟軍事單位，所以DDS的使用者社群看起來跟ROS或是ZeroMQ這些較新的軟體社群相當不同，也就不令人感到意外。雖然RTI看起來有很多使用者，但社群裡的使用者所問的問題幾乎都是由RTI的員工回答的(而非其他使用者熱心參與回答)。而且雖然在技術上是開源的，但RTI的Connext或PrismTech的OpenSplice都沒有提供Ubuntu的apt或是Homebrew這些較流行的套件管理工具的package，也沒有由大量使用者撰寫的wiki或活躍的github repository。

DDS和ROS在社群文化上的巨大差異，對於採用DDS來開發ROS 2.0是最需要擔心的問題之一，跟繼續使用rostcp或使用ZeroMQ這兩種選項的重要差異在於，看起來並沒有很多使用者依賴DDS。但是DDS函式庫的發佈者在我們研究的過程中，一直很積極地回應我們的問題。不過如果是由廣大的ROS社群來問問題，這種積極回應的態度能否持續就很難說了。

雖然社群文化這個議題在決定是否採用DDS的過程中應該要被考慮，但也不能把這個議題的重要程度看得比DDS在技術層面的優缺點還要高。


## 建立在DDS之上的ROS

我們的目標是希望DDS變成ROS 2.0的實作細節，換句話說，DDS相關的API跟訊息格式的定義都需要被隱藏起來(如果你是ROS 2.0的使用者，完全不需要了解DDS的API)。DDS提供了Discovery、訊息格式的定義、訊息序列化和發佈-訂閱的傳輸機制，因此，我們至少會利用DDS的discovery、發佈-訂閱的傳輸機制和訊息序列化來開發ROS。ROS 2.0會提供跟ROS 1.x類似的介面，這個介面建立在DDS之上，所以對於大部分的ROS使用者來說，並不需要接觸到DDS複雜的部分。但同時，我們也會另外提供方法來更換底層的DDS函式庫，讓有特殊需求或需要使用其他DDS實作版本的使用者擁有高彈性的發揮空間。要更換底層的DDS函式庫，需要額外的package來處理dependency的問題，如此一來，使用者只需要看這個額外的package的dependency，就可以知道現在是用哪個DDS的實作版本。在設計建構於DDS函式庫之上的ROS API時，應該要將目標放在滿足ROS使用者社群的需求，因為當使用者開始觸碰到某個特定的DDS實作版本之後，就喪失了使用不同DDS實作版本的可移植性(如果某個package需要更動到底層的DDS函式庫，那要安裝到其他使用者的ROS環境中，就會引起麻煩)。我們之所以讓使用者可以在多種不同的DDS實作版本中切換，並不是為了鼓勵使用者常常更換實作版本，而是要讓使用者可以在有特殊需求的時候，利用不同實作版本的切換彈性來滿足自己的需求，同時也避免掉ROS 2.0在未來會抗拒除了預設的DDS實作版本之外的可能性。所以，ROS 2.0還是會有一個推薦使用、支援最完整的預設DDS實作版本。
=======
DDS comes out of a set of companies which are decades old, was laid out by the OMG which is an old-school software engineering organization, and is used largely by government and military users.
So it comes as no surprise that the community for DDS looks very different from the ROS community and that of similar modern software projects like ZeroMQ.
Though RTI has a respectable on-line presence, the questions asked by community members are almost always answered by an employee of RTI and though technically open source, neither RTI nor OpenSplice have spent time to provide packages for Ubuntu or Homebrew or any other modern package manager.
They do not have extensive user-contributed wikis or an active Github repository.

This staunch difference in ethos between the communities is one of the most concerning issues with depending on DDS.
Unlike options like keeping TCPROS or using ZeroMQ, there isn't the feeling that there is a large community to fall back on with DDS.
However, the DDS vendors have been very responsive to our inquiries during our research and it is hard to say if that will continue when it is the ROS community which brings the questions.

Even though this is something which should be taken into consideration when making a decision about using DDS, it should not disproportionately outweigh the technical pros and cons of the DDS proposal.

## ROS Built on DDS

The goal is to make DDS an implementation detail of ROS 2.0.
This means that all DDS specific APIs and message definitions would need to be hidden.
DDS provides discovery, message definition, message serialization, and publish-subscribe transport.
Therefore, DDS would provide discovery, publish-subscribe transport, and at least the underlying message serialization for ROS.
ROS 2.0 would provide a ROS 1.x like interface on top of DDS which hides much of the complexity of DDS for the majority of ROS users, but then separately provides access to the underlying DDS implementation for users that have extreme use cases or need to integrate with other, existing DDS systems.

![DDS and ROS API Layout](/img/ros_on_dds/api_levels.png)
*DDS and ROS API Layout*

Accessing the DDS implementation would require depending on an additional package which is not normally used.
In this way you can tell if a package has tied itself to a particular DDS vendor by just looking at the package dependencies.
The goal of the ROS API, which is on top of DDS, should be to meet all the common needs for the ROS community, because once a user taps into the underlying DDS system, they will lose portability between DDS vendors.
Portability among DDS vendors is not intended to encourage people to frequently choose different vendors, but rather to enable power users to select the DDS implementation that meets their specific requirements, as well as to future-proof ROS against changes in the DDS vendor options.
There will be one recommended and best-supported default DDS implementation for ROS.
>>>>>>> 42d8775846c51ae520960dd874923d294d5e96b4

### Discovery

DDS可以完全取代以往由master為基礎的discovery系統。取代之後，ROS 2.0可以透過DDS API來取得node的列表、topic的列表，以及他們之間的連接關係。換句話說，使用者不需直接呼叫DDS的API，而是可以呼叫把這些細節都隱藏起來的ROS 2.0 API。

<<<<<<< HEAD
使用DDS實作discovery系統的好處在於，他原生就是分散式的，所以不會有中心的master發生錯誤、使得系統中各部份難以溝通的現象發生。另外，DDS允許使用者定義更多的meta data，這讓ROS 2.0可以在發佈-訂閱之上建立更高階的概念。

=======
>>>>>>> 42d8775846c51ae520960dd874923d294d5e96b4
### Publish-Subscribe Transport

The DDSI-RTPS (DDS-Interoperability Real Time Publish Subscribe) protocol would replace ROS's TCPROS and UDPROS wire protocols for publish/subscribe.
The DDS API provides a few more actors to the typical publish-subscribe pattern of ROS 1.x.
In ROS the concept of a node is most clearly paralleled to a graph participant in DDS.
A graph participant can have zero to many topics, which are very similar to the concept of topics in ROS, but are represented as separate code objects in DDS, and is neither a subscriber nor a publisher.
Then, from a DDS topic, DDS subscribers and publishers can be created, but again these are used to represent the subscriber and publisher concepts in DDS, and not to directly read data from or write data to the topic.
DDS has, in addition to the topics, subscribers, and publishers, the concept of DataReaders and DataWriters which are created with a subscriber or publisher and then specialized to a particular message type before being used to read and write data for a topic.
These additional layers of abstraction allow DDS to have a high level of configuration, because you can set QoS settings at each level of the publish-subscribe stack, providing the highest granularity of configuration possible.
Most of these levels of abstractions are not necessary to meet the current needs of ROS.
Therefore, packaging common workflows under the simpler ROS-like interface (Node, Publisher, and Subscriber) will be one way ROS 2.0 can hide the complexity of DDS, while exposing some of its features.

### Efficient Transport Alternatives

In ROS 1.x there was never a standard shared-memory transport because it is negligibly faster than localhost TCP loop-back connections.
It is possible to get non-trivial performance improvements from carefully doing zero-copy style shared-memory between processes, but anytime a task required faster than localhost TCP in ROS 1.x, nodelets were used.
Nodelets allow publishers and subscribers to share data by passing around `boost::shared_ptr`s to messages.
This intraprocess communication is almost certainly faster than any interprocess communication options and is orthogonal to the discussion of the network publish-subscribe implementation.

In the context of DDS, most vendors will optimize message traffic (even between processes) using shared-memory in a transparent way, only using the wire protocol and UDP sockets when leaving the localhost.
This provides a considerable performance increase for DDS, whereas it did not for ROS 1.x, because the localhost networking optimization happens at the call to `send`.
For ROS 1.x the process was: serialize the message into one large buffer, call TCP's `send` on the buffer once.
For DDS the process would be more like: serialize the message, break the message into potentially many UDP packets, call UDP's `send` many times.
In this way sending many UDP datagrams does not benefit from the same speed up as one large TCP `send`.
Therefore, many DDS vendors will short circuit this process for localhost messages and use a blackboard style shared-memory mechanism to communicate efficiently between processes.

However, not all DDS vendors are the same in this respect, so ROS would not rely on this "intelligent" behavior for efficient **intra**process communication.
Additionally, if the ROS message format is kept, which is discussed in the next section, it would not be possible to prevent a conversion to the DDS message type for intraprocess topics.
Therefore a custom intraprocess communication system would need to be developed for ROS which would never serialize nor convert messages, but instead would pass pointers (to shared in-process memory) between publishers and subscribers using DDS topics.
This same intraprocess communication mechanism would be needed for a custom middleware built on ZeroMQ, for example.

The point to take away here is that efficient **intra**process communication will be addressed regardless of the network/interprocess implementation of the middleware.

### Messages

There is a great deal of value in the current ROS message definitions.
The format is simple, and the messages themselves have evolved over years of use by the robotics community.
Much of the semantic contents of current ROS code is driven by the structure and contents of these messages, so preserving the format and in-memory representation of the messages has a great deal of value.
In order to meet this goal, and in order to make DDS an implementation detail, ROS 2.0 should preserve the ROS 1.x like message definitions and in-memory representation.

Therefore, the ROS 1.x `.msg` files would continue to be used and the `.msg` files would be converted into `.idl` files so that they could be used with the DDS transport.
Language specific files would be generated for both the `.msg` files and the `.idl` files as well as conversion functions for converting between ROS and DDS in-memory instances.
The ROS 2.0 API would work exclusively with the `.msg` style message objects in memory and would convert them to `.idl` objects before publishing.

![Message Generation Diagram](/img/ros_on_dds/message_generation.png "Message Generation Diagram")

At first, the idea of converting a message field-by-field into another object type for each call to publish seems like a huge performance problem, but experimentation has shown that the cost of this copy is insignificant when compared to the cost of serialization.
This ratio between the cost of converting types and the cost of serialization, which was found to be at least one order of magnitude, holds true with every serialization library that we tried, except [Cap'n Proto](http://kentonv.github.io/capnproto/) which doesn't have a serialization step.
Therefore, if a field-by-field copy will not work for your use case, neither will serializing and transporting over the network, at which point you will have to utilize an intraprocess or zero-copy interprocess communication.
The intraprocess communication in ROS would not use the DDS in-memory representation so this field-by-field copy would not be used unless the data is going to the wire.
Because this conversion is only invoked in conjunction with a more expensive serialization step, the field-by-field copy seems to be a reasonable trade-off for the portability and abstraction provided by preserving the ROS `.msg` files and in-memory representation.

This does not preclude the option to improve the `.msg` file format with things like default values and optional fields.
But this is a different trade-off which can be decided later.

### Services and Actions

DDS currently does not have a ratified or implemented standard for request-response style RPC which could be used to implement the concept of services in ROS.
There is currently an RPC specification being considered for ratification in the OMG DDS working group, and several of the DDS vendors have a draft implementation of the RPC API.
It is not clear, however, whether this standard will work for actions, but it could at least support non-preemptable version of ROS services.
ROS 2.0 could either implement services and actions on top of publish-subscribe (this is more feasible in DDS because of their reliable publish-subscribe QoS setting) or it could use the DDS RPC specification once it is finished for services and then build actions on top, again like it is in ROS 1.x.
Either way actions will be a first class citizen in the ROS 2.0 API and it may be the case that services just become a degenerate case of actions.

### Language Support

DDS vendors typically provide at least C, C++, and Java implementations since APIs for those languages are explicitly defined by the DDS specification.
There are not any well established versions of DDS for Python that research has uncovered.
Therefore, one goal of the ROS 2.0 system will be to provide a first-class, feature complete C API.
This will allow bindings for other languages to be made more easily and to enable more consistent behavior between client libraries, since they will use the same implementation.
Languages like Python, Ruby, and Lisp can wrap the C API in a thin, language idiomatic implementation.

The actual implementation of ROS can either be in C, using the C DDS API, or in C++ using the DDS C++ API and then wrapping the C++ implementation in a C API for other languages.
Implementing in C++ and wrapping in C is a common pattern, for example [ZeroMQ](http://zeromq.org/) does exactly this.
The author of [ZeroMQ](http://zeromq.org/), however, did not do this in his new library, [nanomsg](http://nanomsg.org/), citing increased complexity and the bloat of the C++ stdlib as a dependency.
Since the C implementation of DDS is typically pure C, it would be possible to have a pure C implementation for the ROS C API all the way down through the DDS implementation.
However, writing the entire system in C might not be the first goal, and in the interest of getting a minimal viable product working, the implementation might be in C++ and wrapped in C to begin with and later the C++ can be replaced with C if it seems necessary.

### DDS as a Dependency

One of the goals of ROS 2.0 is to reuse as much code as possible ("do not reinvent the wheel") but also minimize the number of dependencies to improve portability and to keep the build dependency list lean.
These two goals are sometimes at odds, since it is often the choice between implementing something internally or relying on an outside source (dependency) for the implementation.

This is a point where the DDS implementations shine, because two of the three DDS vendors under evaluation build on Linux, OS X, Windows, and other more exotic systems with no external dependencies.
The C implementation relies only on the system libraries, the C++ implementations only rely on a C++03 compiler, and the Java implementation only needs a JVM and the Java standard library.
Bundled as a binary (during prototyping) on both Ubuntu and OS X, the C, C++, Java, and C# implementations of OpenSplice (LGPL) is less than three megabytes in size and has no other dependencies.
As far as dependencies go, this makes DDS very attractive because it significantly simplifies the build and run dependencies for ROS.
Additionally, since the goal is to make DDS an implementation detail, it can probably be removed as a transitive run dependency, meaning that it will not even need to be installed on a deployed system.

## The ROS on DDS Prototype

Following the research into the feasibility of ROS on DDS, several questions were left, including but not limited to:

- Can the ROS 1.x API and behavior be implemented on top of DDS?
- Is it practical to generate IDL messages from ROS MSG messages and use them with DDS?
- How hard is it to package (as a dependency) DDS implementations?
- Does the DDS API specification actually make DDS vendor portability a reality?
- How difficult is it to configure DDS?

In order to answer some of these questions a prototype and several experiments were created in this repository:

[https://github.com/osrf/ros_dds](https://github.com/osrf/ros_dds)

More questions and some of the results were captured as issues:

[https://github.com/osrf/ros_dds/issues?labels=task&page=1&state=closed](https://github.com/osrf/ros_dds/issues?labels=task&page=1&state=closed)

The major piece of work in this repository is in the `prototype` folder and is a ROS 1.x like implementation of the Node, Publisher, and Subscriber API using DDS:

[https://github.com/osrf/ros_dds/tree/master/prototype](https://github.com/osrf/ros_dds/tree/master/prototype)

Specifically this prototype includes these packages:

- Generation of DDS IDLs from `.msg` files: [https://github.com/osrf/ros_dds/tree/master/prototype/src/genidl](https://github.com/osrf/ros_dds/tree/master/prototype/src/genidl)

- Generation of DDS specific C++ code for each generated IDL file: [https://github.com/osrf/ros_dds/tree/master/prototype/src/genidlcpp](https://github.com/osrf/ros_dds/tree/master/prototype/src/genidlcpp)

- Minimal ROS Client Library for C++ (rclcpp): [https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp](https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp)

- Talker and listener for pub-sub and service calls: [https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp_examples](https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp_examples)

- A branch of `ros_tutorials` in which `turtlesim` has been modified to build against the `rclcpp` library: [https://github.com/ros/ros_tutorials/tree/ros_dds/turtlesim](https://github.com/ros/ros_tutorials/tree/ros_dds/turtlesim).
  This branch of `turtlesim` is not feature-complete (e.g., services and parameters are not supported), but the basics work, and it demonstrates that the changes required to transition from ROS 1.x `roscpp` to the prototype of ROS 2.0 `rclcpp` are not dramatic.

This is a rapid prototype which was used to answer questions, so it is not representative of the final product or polished at all.
Work on certain features was stopped cold once key questions had been answered.

The examples in the `rclcpp_example` package showed that it was possible to implement the basic ROS like API on top of DDS and get familiar behavior.
This is by no means a complete implementation and doesn't cover all of the features, but instead it was for educational purposes and addressed most of the doubts which were held with respect to using DDS.

Generation of IDL files proved to have some sticking points, but could ultimately be addressed, and implementing basic things like services proved to be tractable problems.

In addition to the above basic pieces, a pull request was drafted which managed to completely hide the DDS symbols from any publicly installed headers for `rclcpp` and `std_msgs`:

[https://github.com/osrf/ros_dds/pull/17](https://github.com/osrf/ros_dds/pull/17)

This pull request was ultimately not merged because it was a major refactoring of the structure of the code and other progress had been made in the meantime.
However, it served its purpose in that it showed that the DDS implementation could be hidden, though there is room for discussion on how to actually achieve that goal.

## Conclusion

After working with DDS and having a healthy amount of skepticism about the ethos, community, and licensing, it is hard to come up with any real technical criticisms.
While it is true that the community surrounding DDS is very different from the ROS community or the ZeroMQ community, it appears that DDS is just solid technology on which ROS could safely depend.
There are still many questions about exactly how ROS would utilize DDS, but they all seem like engineering exercises at this point and not potential deal breakers for ROS.

# Hyperledger-Fabric（Consortium Blockchain）

所有的**Ordering**节点和**Peers**必须在channel所支持最底端capability之上，才可以在channel中运行。

**Orderer**和**channel**的capability**继承**自区块链网络系统的系统**channel**，

**Peer**和**Application**在加入channel之前应该看一看==genesis区块中定义的channel capability==，联盟成员可以共同决定是否更新channel capability。

在修改Application的capability之前应该小心，**因为ordering service并不会校验capability，而peer只会拒绝该Application发起的所有交易**。

## Hyperledger Fabric

> 1. Hyperleder Fabric中的节点通过一个信任的会员服务提供者机构授权进入（Membership Service Provider，MSP），账本数据可以以**多种格式存储**，**共识机制可以换来换去**，**并支持不同的MSP**
> 2. Hyperleder Fabric可以创建**channel**，在一个**channel**中一**伙参与者可以创建一个分割的交易账本**，==可以极大地保护隐私==，不适用channel即类似于公有链
> 3. 账本由两个部分组成：==世界状态==和==交易日志==，世界状态通过键值性数据库**LevelDB**来存储（**可替换**，如CouchDB），**交易日志不可替换**
> 4. chaincode一般只和**世界状态（world state）进行交互**而不是交易日志
> 5. 不同的关系需求，可以**自行选择共识机制**（从==偏向高度结构化的关系==到==点对点==的共识），并且交易在Hyperledger Fabric中**必须排序**

## Hyperledger Fabric Model

> - Assets：区块链中的资产，从可触摸的**房地产和硬件**到不可触摸的**合同和知识产权**，可通过chaincode进行修改
> - Chaincode：智能合同
> - Ledger Features：不可篡改的分布式账本，存储每个channel的交易历史信息并具有SQL形式的查询能力以监督和解决纠纷
> - Privacy：通过Channel实现交易的隐私，Collection可实现数据隐私保护，将数据和channel ledger分割
> - Security&Membership Services：可通过可信第三方机构追踪到所有的交易
> - Consensus，区块中所有交易的全流程的正确性验证，包括**proposal**、**endorsement**、**ordering**、**validation**以及**commitment**.

## Blockchian Network

> ![network.structure](https://hyperledger-fabric.readthedocs.io/en/release-2.2/_images/network.diagram.1.png)
>
> 1. Ordering Service是通过System Channel来实现的，System channel不同于Consortium中的私有channel，这是一个所有成员都可访问的公共channel
> 2. Application即用户必须通过智能合同（chaincode）来读写Peer中的Ledger，peer属于组织1，用于物理存储账本，而智能合同是由Organization中的开发人员编写的
> 3. 智能合同通过channel进行定义，即由组织的管理人员进行定义，存储于Peer节点上
> 4. 智能合约定义交易逻辑，然后打包生成chaincode部署于区块链网络（chaincode包含智能合约）ps：chaincode由metadata.json和code.tar.gz两部分组成，智能合同仅仅只是逻辑代码
> 5. 智能合约开发完成后，由Organization创建chaincode并将其存储到Peer Node上
> 6. Client调用智能合约的过程是输入内容，智能合约返回响应，交易响应和交易输入一起打包并由节点背书后分发网络
> 7. 并不是每个Peer都需要安装智能合约，但是安装后可以知道合约编写具体逻辑，不安装智能合约可以通过加入channel访问智能合约的接口，只是不知道合约编写逻辑
> 8. Channel中的每个Peer都是Committing Peer，但是只有安装了chaincode的Peer且为Client生成交易响应的Peer才能成为Endorsing Peer。
>     1. 两类Peer分别是**Committing Peer**和**Endorsing Peer**
>     2. 两类Peer角色分别是**Leader Peer**和**Anchor Peer**，
>         1. Leader Peer负责从orderer获取交易并分发至其余Peer，Leader Peer可以静态（0或多个Peer设置为Leader）也可动态选举（选举一个Leader）
>         2. Anchor Peer负责组织之间的通信
> 9. 网络和信道配置，==网络发起者==**网络配置**（**联盟**和**信道**的创建）网络发起组织管理，而==信道配置==由同一个信道内的组织进行管理
> 10. **Ordering Service**在网络层中具有依据**网络管理策略NC4**管理网络资源的权力，在Channel层中具有依照**Channel策略 CC1**收集交易并打包成块的作用

## Identity

> Hyperledger Fabric中的Identity由**PKI**（Public Key Infrastructure）和**MSP**（Membership Service Provider）组成
>
> 1. ==PKI主要包含四个元素==
>     1. **数字证书** Digital Certificates，基于`X.509标准`，HTTPS、POP等协议也基于该标准
>         1. 包含==SUBJECT==、==Public Key==以及==CA的签名==三个关键信息，SUBJECT中包含**`国家`**、**`省份`**、**``城市``**、**`组织`**、**``部门``**和**``唯一标识``**等信息
>     2. **公私钥** Public and Private Keys
>     3. **证书颁发机构** Certificates Authorities
>         1. 有多少个CA就有多少个组织在区块链网络中
>         2. 组织可以用不同的**根CA**或者根CA下不同的**中级CA**
>         3. Fabric使用自定义的Fabric CA，只适用于当Root CA
>     4. **证书撤销列表** Certificate Revocation List
>         1. 验证节点会先检测CRL进行身份核定后再进行交易验证
> 2. ==MSP主要负责将CA颁发的Identities（Certificates）和该CA组织进行关联==，MSP中包含一系列被允许的Identities，将Identity转变为Role
>     1. 每个组织必须有一个MSP，而MSP中记录了该组织颁发的所有Identity，MSP的默认名称定义为：`ORG1-MSP`,OU表示组织中的部门，可以在Policy文件中定义同一个组织下不同OU的权力，OU也可以不用
>     2. ==Node OU==
>         1. Node OU可以给Identity授予角色权力，需要将`Node OUs`设置为`Enable:true`状态，Node OU定义于`$FABRIC_CFG_PATH/msp/config.yaml`，包含了一系列的OU
>     3. 并不是组织中每个部门的admin都有添加新组织的权力，该权力只有在Channel Configuration中定义的部门admin具有
>     4. 两类MSP：Local MSP部署于actor Node上
>         1. local MSP服务于clients和==nodes(peers and orderers)==，可以**定义哪个节点作为peer admin来作者peer上的ledger和合约**，也可定义**clients==是否==允许在channel中发起交易**（即判断clients是否具有联盟中某个组织颁发的数字证书）
>         2. 每个node必须有local MSP，以验证谁具有Peer的写权力和哪个clients具有发起交易权力
>     5. 部署于信道配置中的Channel MSP
>         1. Channel MSP必须包含该Application Channel中的组织，以确定哪个Identity是可以在该channel中交易的
>         2. Channel MSP逻辑层面上是在channel configuration中，但实际上每个节点都有该配置的实例化文件并且通过共识来保持一致，local MSP以文件夹的形式存储

## Policies & Peers

> - Policies分为Signature和ImplictMeta两种策略，并且Policies贯穿整个区块链的全过程，定义了区块链所有步骤的规则。
>     - Signature策略是可以定制化的策略，当有新组织加入区块链网络时不能自适应变更。
>     - ImplicatMeta策略是较为泛华的策略可以随着组织的加入而自适应的变更。
> - Peers
>     - 区块链网络中拥有Identity的Peer，Application，Orderer等都被成为Principal
>     - Peer中存储Chaincode和Ledger，
>         - 当Application Clients和Peer**发生Query交互时**，不需和其它组织Peer交互，且==响应是实时的。==
>         - 但是当**Application和Peer发生Update交互时**，还需要**和其它组织的Peer进行交互**，Application收集其它Peer的响应和签名聚合后再发送给Orderer生成区块，**再由Orderer发送给Peer来更新本地Ledger**。（Application发送Proposal给哪个Peer进行背书验证是通过**Chaincode上定义的Endorsement Policy来确定的**，交易的本质是通过Chaincode产生的）

## Ledger

> Ledger由==世界状态==和==区块链==组成，其中区块链决定了世界状态如何变化，世界状态可以变化，但是区块链只能添加不可篡改
>
> - ==世界状态==以**键值对数据库的方式**进行存储
>     - 世界状态可以通过ledger的API进行**get**、**put**和**delete**
>     - 每次世界状态发生改变都会导致**该键值对的==version==**自增
>     - 世界状态有LevelDB和CouchDB两种数据库存储方式，LevelDB和Peer是同一个进程，而CouchDB则是单独的另一个进程，
>         - LevelDB适用于简单的键值对关系
>         - CouchDB支持复杂查询，以JSON文档为基础
> - ==区块链==以**文件系统的方式**进行存储
>     - genesis区块中包含当前channel的配置信息交易

## Chaincode & Smart Contract

> ![smart.diagram2](https://hyperledger-fabric.readthedocs.io/en/release-2.2/_images/smartcontract.diagram.02.png)
>
> 1. 在一个chaincode中包含多个smart contract，而且只有chaincode可以被部署于Peer节点上，一旦chaincode部署成功，其所包含的所有contract都将成功被激活
> 2. Contarct一般可以**`put`**、**`get`**、**`delete`**世界状态中的状态
>     1. get：类似于查询功能
>     2. put：修改功能
>     3. delete：删除功能，但是该状态的历史信息保留于区块链中无法删除
> 3. Contract的endorsement policy十分重要，它可以决定该合同是否有效。每个Contract都有一个Endorsement policy绑定。
> 4. **无论==交易==是否有效都会被添加到分布式账本中**，但是只有有效地交易才会更新世界状态。
> 5. Chaincode中所有的合同具有相同的endorsement策略，chaincode在安装到channel之前会被验证name、version和endorsement policy

## Private Data

> Private Data Collection可以在一个channel中实现数据的隐私保护，而隐私数据将不会通过ordering service nodes。关键点为通过Hash来实现数据的隐私保护和验证，Hash（data）是作为区块链中不可篡改的保证
>
> ![private-data.private-data](https://hyperledger-fabric.readthedocs.io/en/release-2.2/_images/PrivateDataConcept-2.png)
>
> - 对于非常敏感的数据，**甚至分享私人数据的各方**可能希望--或者政府法规可能要求--定**期 "清除 "其同行的数据**，在区块链上留下数据的哈希值，作为私人数据的不可改变的证据。
>
> - 在某些情况下，**私人数据只需要存在于对等体的私人数据库中，直到它可以被复制到对等体区块链外部的数据库中**。这些数据也可能只需要存在于对等体上，直到对其进行链码业务流程（**交易结算、合同履行**后可以将数据复制到Peer的区块链中）。

## Chaincode

> 1. Chaincode可以通过==Go，Node.js，JAVA==进行编，并部==署于Docker==容器之中，Chaincode是面向交易的区块链，即通过区块链来改变账本状态（类似于以太坊，不同于比特币的面向账户）。
>
> 2. 一个**Chaincode创建的账本更新只针对该Chaincode**，不能被其他Chaincode直接访问。然而，在**同一个网络中，如果有适当的权限**，一个Chaincode可以调用另一个Chaincode来访问其状态。
>
> 3. Chaincode需要组织们同意其参数（名字，版本和endorsement策略），并不是每个chanel的组织都需要完成每一步
>     1. **打包chaincode**：一个组织或者每个组织
>         1. chaincode必须为**tar文件**才可被安装在本地， 可通过**Fabric peer binaries**, **the Node Fabric SDK**, or **GNU tar**打包，chaincode需要包含一个简短可读的描述性package label，
>         2. 使用第三方GNU tar打包需要满足一定要求：==必须为.tar.gz的文件==，==必须包含两个文件（metadata.json和code.tar.gz）没有文件夹==，==metadata.json需要说明**chaincode语言**，**代码路径**和**package label**==
>         3. 区分同一个packaged chaincode通过**name**和**version**
>     2. **安装chaincode**：需要使用该chaincode来==endorse transaction==或==查询账本状态==的组织
>         1. **chaincode的identifier格式为**：package的label:package的hash值
>     3. **同意chaincode的定义**：每个使用这个chaincode的组织（一定数量同意或默认的多数同意endorsement策略）
>         1. chaincode定义中包含的参数：
>             1. **Name**：通过该名字调用chaincode
>             2. **Verison**：每一次更新chaincode后需要变更，**只有改变chaincode的binaries**后才需要更改，只修改Endorsement策略并==不需要修改version，但需修改Sequence==
>             3. **Sequence**：每更新一次chaincode后便需要自增1
>             4. **Endorsement Policy**：执行或验证交易时的共识策略,默认为多数节点同意的共识方式，通过该字符串发送到CLI实现：**`Channel/Application/Endorsement`**
>             5. **Collection Configuration**：隐私数据收集的定义方式文件的路径和chaincode关联
>             6. **ESCC / VSCC Plugins**：默认endorsement或validation的插件名称
>             7. **Initialization**：Init构造函数，Init默认包含在由低级Fabric API开发的chaincode中，其余chaincode则自行配置
>             8. Package ID：就是Identifier由**package的label:package的hash值**组成
>     4. **提交chaincode**：由==一个==**经过channel中一定数量组织同意**的==组织==提交chaincode
> 4. ==majority endorsement的策略==，即Fabric的默认策略，可以**自动更新**来包含新加入或退出的组织
> 5. 使用**一个chaincode package**创建多个chaincode具有不同的endorsement策略，不同chaincode的Name必须不同



## Fabric v2.2x LTS

### v1和v2的区别

>1. v2中多个组织**必须都赞成chaincode的参数**之后才可以成功激活和部署chaincode，**同时支持v1中的中心化信任模式**，而在==v1==中一**个组织可以直接发布chaincode而不需要其它组织同意**，其它组织可以拒绝安装chaincode并不参与交易。
>2. v2中chaincode更新只有多数节点同意后才可进行，v1不需要
>3. v2中节点可在**不更改或升级chaincode的前提下**，==更改endorsement策略==或==隐私数据收集配置==，当然v2中节点也可以支持多数节点endorse的**默认策略**
>4. chaincode**转变为已阅读的tar文件**，所以可以使得跨组织监管chaincode更轻松
>5. ==v1==中的chaincode当chaincode包建立时便需要指定**名字**和**版本**，==v2==可以使用**同一个chaincode包**以不同的**名字**部署在**同一个**chanel或**别的**chanel多次
>6. v2中**Chaincode packages**在信道成员中可以**不同**，即组织可以==定制化自己的chaincode包==，只需要指定数量的endorsement即可将交易添加到帐本中，这也**允许组织推出小的修复版本而不需要整个网络暂停**
>7. V2.2.10慢慢舍弃kafaka共识而转用Raft共识协议，同时不用gossip传播块而是使用ordering service来传播块


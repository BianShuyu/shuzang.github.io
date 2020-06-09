# 区块链实验8-建立私链网络及部署和测试合约


上一篇中已经在 Remix 中验证了合约的正确性，本篇记录 Quorum 私链网络的搭建及智能合约的部署和测试。

<!--more-->

## 1. 方案确定

上一次实验我们搭建 Quorum 私链网络是采用从零开始的方式，从 genesis.json 文件开始，手动编辑配置文件、创建节点、最后组建网络，不仅耗费时间，而且一旦错误就要重新开始，浪费了大量无意义的精力。另外，我们还需要一个区块链浏览器可视化网络、区块、交易和合约的状态，与智能合约的交互也需要优化，虽然 Truffle 集成了合约的部署和测试工作，但依然存在一些不足，国内也无法使用 `truffle init` 命令。

针对以上问题，结合 Quorum 社区的最新进展，我们本次调整了实验方案所使用的工具和手段：

1. 使用 [Quorum Wizard](https://github.com/jpmorganchase/quorum-wizard) 命令行工具快速建立 Quorum 网络；
2. 使用 [Cakeshop](https://github.com/jpmorganchase/cakeshop) 可视化区块链和智能合约状态；
3. 使用 [Remix](https://remix.ethereum.org/)  + [Quorum Plugin for Remix](https://github.com/jpmorganchase/quorum-remix) 的组合部署合约及与合约交互；

最后，我们对一些注意事项进行说明：

1. Quourm Wizard 建立网络有 Bash 、 Docker-compse 和 kubernete 三种可选方式，我们使用第一种，但会尝试一下第二种；
2. 之前在树莓派中建立区块链账户表示物联网网络和设备，但树莓派放在了实验室，由于疫情原因无法拿到手，因此本次建立一个 7 节点的私链网络，挑选一个节点代表物联网网关，然后建立一个新的账户表示设备，从而进行实验。

最后，确定本次方案时还有一些备选方案，比如 [Epirus-free](https://github.com/blk-io/epirus-free) 也是一个可用的 Quorum 区块链浏览器，但结构比较简单，展示的参数也比较少，而且我们测试的时候迟迟无法加载出来数据，因此不选用。

[quorum-maker](https://github.com/synechron-finlabs/quorum-maker) 是一个一体化方案，可以快速建立基于 Docker 的 Quorum 网络，并提供一个区块链浏览器查看区块链和合约状态，各方面的功能都足够晚上，唯一的问题是对 IBFT 共识的支持还处在开发阶段，暂时不可用，因此我们只能选择上述多个工具组合的方法完成本次实验。

### 1.1 环境准备

Win10 Home Edition 不支持 Docker，且实验中涉及的组件比较多，我们决定使用虚拟机来启动一个 Linux 环境。另外，方案中的几个工具对依赖的要求如下：

1. Quorum Wizard：
   - 基于 Bash 建立网络：如果需要隐私管理器，需要 Java 环境
   - 基于 Docker Compse：需要 Docker 和 docker-compose
   - 基于 Kubernetes：需要 Docker、kubectl 和 minikube
2. Cakeshop：需要 Java 8+ 及 Node.js
3. Geth 提供了接口供 Golang 使用来进行账户管理和合约监听，因为实验测试有可能用到，我们安装 Golang

根据说明，我们开始准备实验环境

1. 安装 VMware Workstation 15 Pro，输入批量许可激活，建立 Ubuntu20.04 系统的虚拟机，分配内存 4G（有条件应为8G，这里是因为电脑配置比较低，一共只有8G，再多发生内存交换的概率比较大）、硬盘100GB。

2. 进入 Ubuntu 20.04，更新系统，设置语言

3. 安装git、golang、Java

   ```bash
   # 安装git
   $ sudo apt install -y git
   $ git version
   git version 2.25.1
   
   # 安装golang
   $ sudo apt install -y golang
   $ go version
   go version go1.14.4 linux/amd64
   
   # 安装 JRE
   $ sudo apt install default-jre
   $ java -version
   openjdk version "11.0.7" 2020-04-14
   OpenJDK Runtime Environment (build 11.0.7+10-post-Ubuntu-3ubuntu1)
   OpenJDK 64-Bit Server VM (build 11.0.7+10-post-Ubuntu-3ubuntu1, mixed mode, sharing)
   
   # 安装JDK
   $ sudo apt install default-jdk
   $ javac -version
   javac 11.0.7
   ```

4. 安装 Node 和 npm，由于直接安装后在使用 npm 全局安装包时会出现权限错误，因此使用 Node.js 版本管理工具 [n](https://github.com/tj/n)

    ```bash
    $ sudo apt install curl
    $ curl -L https://git.io/n-install | bash
    # 重启终端
    $ n lts
    
      installing : node-v12.18.0
           mkdir : /home/shuzang/n/n/versions/node/12.18.0
           fetch : https://nodejs.org/dist/v12.18.0/node-v12.18.0-linux-x64.tar.xz
       installed : v12.18.0 (with npm 6.14.4)
    
    # 更新 npm 到最新，顺便测试全局安装
    $ npm install -g npm@latest
    /home/shuzang/n/bin/npm -> /home/shuzang/n/lib/node_modules/npm/bin/npm-cli.js
    /home/shuzang/n/bin/npx -> /home/shuzang/n/lib/node_modules/npm/bin/npx-cli.js
    + npm@6.14.5
    updated 5 packages in 16.718s
    ```

5. （可选）docker 和 docker-compose

   ```bash
   # 安装 docker CE
   $ curl -fsSL get.docker.com -o get-docker.sh
   $ sudo sh get-docker.sh --mirror Aliyun
   # 启动 docker CE
   $ sudo systemctl enable docker
   $ sudo systemctl start docker
   # 建立 docker 用户组并将当前用户加入 docker 组，这样就不需要 root 权限了
   $ sudo groupadd docker
   $ sudo usermod -aG docker $USER
   # 测试安装
   $ docker run hello-world
   # 安装 docker-compose
   $ sudo curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
   $ sudo chmod +x /usr/local/bin/docker-compose
   ```

### 1.2 建立测试网络

全局安装 quorum-wizard

```bash
$ npm install -g quorum-wizard
```

运行向导，建立测试网络，`-v` 参数用于输出日志记录。

```bash
$ quorum-wizard -v
...
Welcome to Quorum Wizard!

This tool allows you to easily create bash, docker, and kubernetes files to star
t up a quorum network.
You can control consensus, privacy, network details and more for a customized se
tup.
Additionally you can choose to deploy our chain explorer, Cakeshop, to easily vi
ew and monitor your network.
? 
Welcome to Quorum Wizard!

This tool allows you to easily create bash, docker, and kubernetes files to star
t up a quorum network.
You can control consensus, privacy, network details and more for a customized se
tup.
Additionally you can choose to deploy our chain explorer, Cakeshop, to easily vi
ew and monitor your network.
? 
Welcome to Quorum Wizard!

This tool allows you to easily create bash, docker, and kubernetes files to star
t up a quorum network.
You can control consensus, privacy, network details and more for a customized se
tup.
Additionally you can choose to deploy our chain explorer, Cakeshop, to easily vi
ew and monitor your network.

We have 3 options to help you start exploring Quorum:

  1.  Quickstart - our 1 click option to create a 3 node raft network with tesse
ra and cakeshop

  2.  Simple Network - using pregenerated keys from quorum 7nodes example,
      this option allows you to choose the number of nodes (7 max), consensus me
chanism, transaction manager, and the option to deploy cakeshop

  3.  Custom Network - In addition to the options available in #2, this selectio
n allows for further customization of your network.
      Choose to generate keys, customize ports for both bash and docker, or chan
ge the network id

Quorum Wizard will generate your startup files and everything required to bring 
up your network.
All you need to do is go to the specified location and run ./start.sh

 Simple Network
? Would you like to generate bash scripts, a docker-compose file, or a kubernete
s config to bring up your network? bash
? Select your consensus mode - istanbul is a pbft inspired algorithm with transa
ction finality while raft provides faster blocktimes, transaction finality and o
n-demand block creation istanbul
? Input the number of nodes (2-7) you would like in your network - a minimum of 
4 is recommended 7
? Which version of Quorum would you like to use? Quorum 2.6.0
? Choose a version of tessera if you would like to use private transactions in y
our network, otherwise choose "none" Tessera 0.10.5
? Do you want to run Cakeshop (our chain explorer) with your network? Yes
? What would you like to call this network? 7-nodes-istanbul-tessera-bash
...
Building network directory...
Generating network resources locally...
Building qdata directory...
Writing start script...
Initializing quorum...
Done
--------------------------------------------------------------------------------

Tessera Node 1 public key:
BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=

Tessera Node 2 public key:
QfeDAys9MPDs2XHExtc84jKGHxZg/aj52DTh0vtA3Xc=

Tessera Node 3 public key:
1iTZde/ndBHvzhcl7V68x44Vx7pl8nwx9LqnM/AfJUg=

Tessera Node 4 public key:
oNspPPgszVUFw0qmGFfWwh1uxVUXgvBxleXORHj07g8=

Tessera Node 5 public key:
R56gy4dn24YOjwyesTczYa8m5xhP6hF2uTMCju/1xkY=

Tessera Node 6 public key:
UfNSeSGySeKg11DVNEnqrUtxYRVor4+CvluI8tVv62Y=

Tessera Node 7 public key:
ROAZBWtSacxXQrOe3FGAqJDyJjFePR5ce4TSIzmJ0Bc=

--------------------------------------------------------------------------------
Quorum network created

Run the following commands to start your network:

cd network/7-nodes-istanbul-bash
./start.sh

A sample simpleStorage contract is provided to deploy to your network
To use run ./runscript.sh public-contract.js from the network folder

A private simpleStorage contract was created with privateFor set to use Node 2's public key: QfeDAys9MPDs2XHExtc84jKGHxZg/aj52DTh0vtA3Xc=
To use run ./runscript private-contract.js from the network folder
```

在向导执行页面选择了运行 Cakeshop 的情况下，不需要自己再去安装 Cakeshop，可以直接启动。

```bash
$ cd network/7-nodes-istanbul-bash
$ ./start.sh

Starting Quorum network...

Waiting until all Tessera nodes are running...
...
All Tessera nodes started
Starting Quorum nodes
Starting Cakeshop
Waiting until Cakeshop is running...
...
Cakeshop started at http://localhost:8999
Successfully started Quorum network.
```

此时浏览器打开 http://localhost:8999 页面，可以看到网络情况

![2020-06-09 09-40-08屏幕截图](/images/区块链实验8-建立私链网络及部署和测试合约/2020-06-09-09-40-08屏幕截图.png)

### 1.3 Remix 部署和交互说明

浏览器打开  [Remix IDE](https://remix.ethereum.org/) （保证是 http 页面），点击左侧 Plugins（插件）标签页，搜索 `Quorum Network`，点击 `Activate` 激活插件。

![](https://docs.goquorum.com/en/latest/RemixPlugin/images/quorum_network.png)

在左侧标签栏寻找激活的插件，图标为 ![](https://docs.goquorum.com/en/latest/RemixPlugin/images/tab_icon.png)

我们上面运行的网络各节点的 url 分别为

| 节点                        | url                                                          |
| --------------------------- | ------------------------------------------------------------ |
| Node1                       | Quorum RPC：http://localhost:22000<br>Tessera：http://localhost:9081 |
| Node2                       | Quorum RPC：http://localhost:22001<br>Tessera：http://localhost:9082 |
| Node3                       | Quorum RPC：http://localhost:22002<br/>Tessera：http://localhost:9083 |
| Node4                       | Quorum RPC：http://localhost:22003<br/>Tessera：http://localhost:9084 |
| Node5                       | Quorum RPC：http://localhost:22004<br/>Tessera：http://localhost:9085 |
| Node6                       | Quorum RPC：http://localhost:22005<br/>Tessera：http://localhost:9086 |
| Node7（程序崩溃，已kill掉） | Quorum RPC：http://localhost:22006<br/>Tessera：http://localhost:9087 |

输入 Node1 的 Quroum RPC 和 Tessera 的 url，点击确认，得到如下的侧面板

![2020-06-09-11-46-56屏幕截图](/images/区块链实验8-建立私链网络及部署和测试合约/2020-06-09-11-46-56屏幕截图.png)

从 Github 导入我们的合约

![2020-06-09-11-51-48屏幕截图](/images/区块链实验8-建立私链网络及部署和测试合约/2020-06-09-11-51-48屏幕截图.png)

Quorum-Remix 插件使用 Remix 的 Solidity 编译器的结果，所以在 Remix 编译后的合约可以在Quorum插件的 `Compiled Contracts` 选项下找到，到时候输入参数点击部署即可，操作与 Remix 原本的 Deploy 选项卡完全一致。

最后，运行 `.stop.sh` 脚本可以停止所有的 quorum/geth 和 cakeshop 实例。

如果我们编写了交互用的 js 脚本，假设脚本名为 test.js，可以使用如下命令执行

```bash
$ ./runscript.sh test.js
```

有输入参数的情况下，可以使用 Bash 、Python 或 Go 有选择的批量执行脚本。

值得注意到是，我们上述没有使用隐私管理器，但这是 Quorum 的一个最重要的特性。

### 1.4 错误排查记录

**2020.06.08**

Remix 无法显示所有插件，因此无法使用 Quorum Network 插件连接 Quorum 网络，经排查，为网络原因，连接手机开的热点后即可看到所有插件，深层原因未知。

**2020.06.09**

合约编译返回错误 `Uncaught JavaScript exception: RangeError: Maximum call stack size exceeded.`

调用栈溢出，猜测可能是虚拟机内存分配不足，在宿主机中使用 Remix 通过局域网 IP 地址连接

宿主机浏览器无法访问 http 连接，换用 Firefox 或者使用 Remix-IDE 桌面版本都无法访问

考虑到此时 Remix 与 后台 Quorum 网络拆分，尝试使用 WSL 子系统，并使用 npm 安装 remix-ide

WSL 对 npm 支持不友好，普通用户和 root 用户权限全部被拒绝，所有包都无法安装

尝试在 win10 本地使用 npm 安装 remix，依赖过多，安装无法完成

重新尝试解决 win10 系统下无法访问 http 网页的错误，关闭防火墙不起作用，恢复 hosts 文件起作用，经确认，无法访问 http 网页是因为 hosts 文件被修改

重新尝试虚拟机的 Ubuntu 系统编译智能合约，Chrome 浏览器失败，Firefox 浏览器成功，确认不是因为内存分配不足。

## 2. 实验设计

这部分我们需要明确两件事，一是实验测试的场景，二是需要测量的值。

### 2.1 场景

我们计划提取已有的该方向论文所使用的场景，并作一定的分析

#### Smart home

```
A. Dorri, S. S. Kanhere, and R. Jurdak, 
“Blockchain in internet of things: Challenges and Solutions,” arXiv:1608.05187 [cs], 
Aug. 2016, Accessed: Mar. 17, 2020. [Online]. Available: http://arxiv.org/abs/1608.05187.
```

Dorri 的论文是该领域起始的几篇论文之一，也是引用最多的论文之一，他的应用场景是 Smart home，其中涉及的访问控制交易如下

首先是**存储数据**。每个设备都有数据存储的需求，以一个智能恒温器为例，假设 Alice 在云中有一个账户并设置了上传数据的权限，当恒温器有存储数据的请求时，需要首先将数据发送到本地矿工，矿工根据已定义的策略对恒温器的权限进行验证，验证通过后将数据、数据哈希一起发送到云存储，云存储检查是否有剩余空间并匹配数据哈希，在存储完成后，将数据 Hash 和区块号返回，相关的交易收集到区块链中。

![存储数据](/images/区块链实验8-平台搭建及合约部署优化/存储数据.png)

然后是**访问数据**。服务提供者可能需要周期性的访问存储的数据或某个设备的全部数据，请求交易经过 Smart home 的矿工验证权限后，矿工从存储中获取数据，最后用请求者的公钥加密数据后将数据发给请求者。

![访问数据](/images/区块链实验8-平台搭建及合约部署优化/访问数据.png)

最后是**监控数据**。某些时候，智能家庭的所有者可能需要实时地访问家里某些设备的信息，比如恒温器当前配置。矿工收到用户请求后，验证用户权限然后返回设备的实时数据。

![监控数据](/images/区块链实验8-平台搭建及合约部署优化/监控数据.png)

以上三个关于数据操作的示例是访问控制的核心场景之一，具有较大的实验价值，但实际的测试还需要添加存储系统比如 IPFS，然后编写一个存储合约与现有系统关联，由于涉及的东西比较多，可以作为之后的工作。

```
P. Wang, Y. Yue, W. Sun, and J. Liu, 
“An Attribute-Based Distributed Access Control for Blockchain-enabled IoT,” 
in 2019 International Conference on WiMob, Barcelona, Spain, Oct. 2019, pp. 1–6, 
doi: 10.1109/WiMOB.2019.8923232.
```

这篇论文使用的场景也是 Smart home，运用了 ABAC模型，场景结构图如下

![](https://ieeexplore.ieee.org/mediastore_new/IEEE/content/media/8913409/8923119/8923232/wang1-p6-wang-small.gif)



该场景中有三种实体：IoT设备、网关和服务器。诸如智能手机、电脑、智能电视等设备通过 Wi-Fi 或有线连接直接访问互联网，摄像头、温湿度传感器等其它类型的设备则通过蓝牙、ZigBee等技术先连到网关，由网关访问互联网。服务器负责存储 IoT 设备的数据以及提供相应的服务，比如从传感器收集环境信息或发送操作命令给执行器。这些实体间有大量的数据交换和访问请求。

作者最终实验选用的实例是关于电视开关权限的，假设父母拥有对客厅电视的完全控制权，他们可以随时发起访问请求打开或关闭电视，而孩子只有在特定时间（环境属性）内才有权限打开电视。

Smart home 场景在基于区块链的物联网访问控制中具有适用性的原因是，随着物联网时代的到来，Smart home 不再是一个单一的信任域，大部分的智能家居设备数据都会传回生产商的服务器，然后由生产商提供各种各样的服务。我们以摄像头为典型用例，如今，很多家庭都选择在家里安装摄像头以提供防盗或其它功能，但是由于家用设备存储能力的不足，或者自身缺乏足够专业的能力进行远程访问，一般都是用设备厂商或第三方服务商提供的软件服务来存储摄录的内容及进行远程访问，监控的内容不可避免地需要上传到他们的服务器，近年来，第三方导致的监控内容泄露的情况频繁发生，用户缺乏对这些数据的控制能力的根本原因，另外，摄像头权限被非法获取也会造成隐私泄露及其它严重的安全问题。基于区块链对摄像头进行访问控制，可以令用户拥有对监控内容的完全控制能力，从而保证用户的隐私与安全。

我们进行测试时，MC 和 RC都可以由生产商/服务商提供，ACC则由 Smart home 属主自身部署，然后定义合理的数据存储、读取、摄像头控制等权限。由于存储系统暂未添加，我们可以暂时以摄像头控制为例，为家庭成员设置摄像头的控制权限，而阻止服务商和未知用户的控制，仅为服务商提供必要的权限用来为我们提供服务。

Smart home 是生活类、单信任域场景中的典型用例之一，可以有效证明方案的适用性。

#### Healthcare

```
Figueroa, Añorga, and Arrizabalaga, 
An Attribute-Based Access Control Model in RFID Systems Based on Blockchain Decentralized Applications for Healthcare Environments,
Computers, vol. 8, no. 3, p. 57, Jul. 2019, doi: 10.3390/computers8030057.
```

这篇论文提到的是一个典型的医疗领域的资产流动场景，考虑这样一件事：医院拥有大量的资产，比如外科手术器械，这些器械会在消毒部门、手术室、实验室等区域周期性的流动，器械位于错误的位置可能危及患者生命，缺乏详细的资产记录也可能造成资产损失。

下图用于详细说明整个流程。源房间（如灭菌室）将一些资产（如外科手术器械）运送到目的房间（如0号手术室、1号手术室）。由于 $Asset_1$ 已被分配到目的房间1（例如，1号手术室），假如由于人为错误试图访问目的房间0（例如，0号手术室），其访问将被拒绝。简而言之，这一系统的目的是建立医疗资产访问控制系统，防止由于人为错误或外部安全威胁导致资产进入错误区域（如房间）。

![图2 Healthcare system](/images/区块链实验8-平台搭建及合约部署优化/医疗系统.png)

医疗也是区块链和访问控制的重点场景之一，拥有大量的研究和实例。这篇论文提到的基于区块链医疗资产访问控制系统，实用价值在于可以保持一个不可篡改的资产记录，阻止未经授权的访问及命令，

#### Smart Factory

```
J. Wan, J. Li, M. Imran, D. Li, and Fazal-e-Amin, 
“A Blockchain-Based Solution for Enhancing Security and Privacy in Smart Factory,” 
IEEE Trans. Ind. Inf., vol. 15, no. 6, pp. 3652–3660, Jun. 2019, doi: 10.1109/TII.2019.2894573.
```

这篇论文的大背景是 IIoT 中的 Smart Factory，但具体的实例是温度采集。论文描述了温度采集过程中涉及的数据交互过程。一个设备节点申请存储权限的流程如下图所示，温度计发出的数据存储请求由微型计算机代为处理，设备通过一个唯一ID标识，管理中心验证存储权限后，微型计算机将数据加入缓冲池，待数据量满足一定规模或者到达某个周期时间，加密后的温度数据就上传到数据库，数据哈希和相关交易记录存储到区块链中。

![The intranet flowchart](https://ieeexplore.ieee.org/mediastore_new/IEEE/content/media/9424/8736410/8621042/imran4-2894573-large.gif)

这是一个借用区块链的自动化系统，与传统方案相比的优点是，保持了不可变的存储记录，阻止了未经授权的存储请求，可以有效抵御非法授权的攻击。

#### Log-House Construction

```
T. Kumar, A. Braeken, V. Ramani, I. Ahmad, E. Harjula, and M. Ylianttila, 
“SEC-BlockEdge: Security Threats in Blockchain-Edge based Industrial IoT Networks,” 
presented at the 2019 11th International Workshop on RNDM, Nicosia, Cyprus, Oct. 2019, p. 7.
```

这篇论文提出了一个木屋建造场景作为典型的 IIoT 用例，整个工业过程主要由下图所示的几个关键流程组成

![木屋建造场景](https://ieeexplore.ieee.org/mediastore_new/IEEE/content/media/8944089/8949080/8949107/Paper06-fig-1-source-large.gif)

在收割阶段，传感器/执行器和其它低功耗设备将感知、收集和处理与原材料和收割过程相关的信息，这些信息将用于其它阶段。运输、制造和存储阶段的日志将会被其它阶段利用。分发阶段需要保证由正确的车辆通过正确的路线准时送达。整个过程涉及传感器/执行器、边缘设备/服务器、服务/网络提供者、其它第三方等多种角色，需要保证每个角色都只能访问需要访问的资源以及提供需要提供的必要信息。作者并没有给出一个具体的实验实例。

#### 电子门锁与自动售货机

```
V. A. Siris, D. Dimopoulos, N. Fotiou, S. Voulgaris, and G. C. Polyzos, 
“Trusted D2D-Based IoT Resource Access Using Smart Contracts,” 
in 2019 IEEE 20th International Symposium on WoWMoM, Washington, DC, USA, Jun. 2019, pp. 1–9, 
Accessed: Apr. 03, 2020. [Online]. Available: https://ieeexplore.ieee.org/document/8793041/.
```

这篇论文介绍了两个用例。

第一个用例涉及的是旅馆或公寓的短租房间的电子门锁（门锁作为一种 IoT 资源）。如果一个人想预定旅馆或公寓的房间几个晚上，可以从智能手机（客户端）向处理门锁授权请求的授权服务器（AS）发送请求，AS建立一个智能合约来接收预付款，一旦验证客户已支付，AS将向客户提供必要的凭据（数字钥匙或者称作访问令牌），在确定的时间内，客户可以通过利用手机通过蓝牙等通信手段发起请求，验证权限后可以打开门锁。

第二个用例是从自动售货机购买物品。假设自动售货机具有持续的网络连接，但并不与区块链直接交互，一个人通过手环或手机从自动售货机购买物品。这种情况下，客户端（手环）付款后获得支付凭据，然后通过D2D通信向自动售货机验证，售货机验证支付后出货。

#### 总结

我们选定 Smart home 中的摄像头用例，木屋制造所代表的供应链场景，以这两个场景为例，证明我们使用区块链完成物联网访问控制的普适性。

### 2.2 测量值

不论是访问控制系统，还是信誉系统，都需要一些测量值来证明我们的结论，本部分讨论实验应该获得哪些测量结果。

#### 测量什么

对于**访问控制系统**，主要目的是证明我们方案的有效性，因此应当模拟上一小节所示的各种场景，完成各种不同情况的访问控制。由于我们使用的主要工具是智能合约，应当获得并记录合约部署的 Gas 消耗和时间消耗，尽管 Quorum 私链中 Gas 的消耗并没有意义，但作为一种常用测量值和人们通常关注的方面，还是要简单的给出结果。最后，进行一次访问控制的时间消耗是访问控制系统最重要的一个参数，我们可以确定 ABAC 模型相比于传统的 ACL 是更加物联网友好的，但对于同样的 ABAC 设计实现，究竟哪种设计实施访问控制比较快，还有待验证，因此应当复现其它的一些方案，并进行对比。

关于**信誉系统**，首先要确定以什么样的模型进行输入，从而真实的反应实际系统中恶意行为发生的情况，测量值应包括信誉值随时间的变动曲线，各种参数的最优取值，以及阻塞时间的变动趋势。

#### 如何获取

合约部署消耗很容易可以得到，可以通过 Remix 或 Truffle 部署的命令返回得到。

访问控制时间作调用 AccessControl() 合约函数的交易执行时间，可以在测试脚本中编写代码计算发起请求和得到返回值之间的时间，但要注意这一结果受实验环境影响很大。

输入模型我们之后阅读论文来查找，信誉值、阻塞时间都在智能合约中通过 Event 发出，只要即时监控即可。

关于惩罚函数的值在确定相关权重值后也可以得到，

## 3. 进行实验

部署合约

事实上，实际的测试过程接下来就可以执行测试了，Subject 首先根据设备的区块链地址从 MC 中获取设备的 ACC 合约地址，或者直接从网络获取设备管理者公布的 设备 ACC 合约地址，然后发起调用 accessControl() 函数，传入参数为要访问的资源和要执行的操作。返回的结果为 true 或 false，同时，调用者地址、结果、结果说明、访问时间等会通过事件发送出去，可以在 Js 中通过调用事件查看。
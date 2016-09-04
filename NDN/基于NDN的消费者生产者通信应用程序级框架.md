

# **Consumer / Producer Communication with ApplicationLevel Framing in Named Data Networking**

## **ABSTRACT**：

NDN是一个通用的网络层协议，提供了一组丰富的功能:网络存储、多路径转发,多播交付,以数据为中心的安全性等。在网络层以上,系统库简化了开发人员开发应用程序的任务，提供了一个易于使用的但功能强大的API来利用NDN的功能。本文提出了一个消费者/生产者的编程接口的设计,加上几种机制以支持应用程序级框架。通过NDN的数据检索协议，使得NDN应用程序编程更容易、更快捷。

Categories andSubject Descriptors

C.2 [COMPUTER-COMMUNICATIONNETWORKS]: 网络结构与设计; 网络协议; 分布式系统

**Keywords**

NDN; API; Transport; 数据恢复;

##  

## **1.      INTRODUCTION**

​	今天的互联网架构基于IP——一个用于创建点对点的通信网络通用的网络层,数据包传递给特定的目的地,可使进程与进程之间通信。首先我们要介绍Socket（套接字）的概念。它令一个运行中的进程绑定到通信信道（tunnel）上,并代表了一个当前在进程[1,2]之间进行数据传输过程状态的容器。

​	随着时间的推移,互联网已经从一个网络互联主机的网络转变成了广义互联信息对象网络;这些对象的范围从电影文件,Facebook内容,twitter消息,到传感器数据和经过身份验证的设备驱动命令。这个根本性的变化表明,互联网网络层将会更加有机地使用本地数据传播协议，这个协议更适合处理信息对象而非通信端点。

​	为了满足这种新环境,NDN作为一个新兴的体系结构,提出了取代IP的host-based的解决方案:对通过网络[3、4、5]的包移动的信息对象进行命名。在NDN网络,消费者发送的兴趣包携带应用程序级命名以请求信息对象,随后网络沿着兴趣包的路径原路返回所请求的数据包。NDN通过公开可验证签名保护数据,并根据需要加密(Section 2)。

​	正如在[ 6 ]中所解释的，网络应用程序基于ADU(ApplicationData Units)工作—为每个用例用最合适的方式表示单位数据。例如，一个视频回放应用程序通常处理视频帧中的单元数据；一个多用户游戏的ADU是代表用户当前状态的对象；对于一个智能家庭应用程序，ADU可能代表传感器读数。NDN通过使用ADU可使应用程序通信。

​	作为一种实现网络的新的方式，NDN为应用程序引进了新的设计模式。为了让通过网络有效，需要考虑多种设计选择，从名字的结构和安全模型到更加基本的问题，如数据分割。为了获取内容，除了数据丢失和错误修正的常规的问题，也需要有新的考虑，如网络中的当前的缓存和数据验证的问题。为了简化应用程序开发，应提供什么样的应用程序接口? 为了支持这种接口，应该提供怎样的协议？显然，因为NDN不支持socket模型中的相互通信的两个进程之间的虚拟信道，所以不能使用socket的抽象和相关协议。在本文中,我们设计的新的API及其相关协议套件,可以扮演NDN网络中的socket-equivalent角色。我们的工作总结为：

1. 一个消费者/生产者的编程模型,专门针对数据传播NDN网络。

2. 相关的数据检索和内容分割协议。

3. 一系列的支持机制如一个清单和negative确认,利用下述API协议以方便操作的应用程序。

   我们实现了消费者/生产者API和协议,并验证使用该API开发了几个试点应用。我们的评估的重点是,运行,行为正确,和实际环境应用，单生产者为多消费者生产ADU时的定量测量以计算开销。

   文章的剩余部分按照如下格式组织: Section II 简要的介绍了NDN结构。Section III包含了编程结构抽象和编程模型的详细解释。Section IV 和V 提供一个重要概念和内容抓取协议的设计。Section VI中说明了本文的关联工作。Section VII进行了总结。



## **2.     BACKGROUND**

​	与IP相比 NDN是一种完全不同的工作方式。为了帮助读者掌握其核心概念,在本节中,我们首先描述一个toy application,然后通过这个例子来解释NDN基础知识。

## **2.1   A Simple-Video Application**

​	我们使用一个Simple-Video应用程序作为示例来帮助说明一般应用程序背后的基本概念和开发人员可能需要解决的问题。Simple-Video生产两个单独的数据流：视频和音频,并给消费者一个选择:同时观看视频与音频,只听音频,或只看视频。视频和音频由数据帧的流组成,应用程序应该允许消费者取回个人的帧。消费者应用程序可以,例如, 当用户点击“静音”按钮时停止获取音频帧,或为了赶上实际的视频直播而在一个暂停后跳过一些视频帧。一般来说,一个视频帧对于一个网络数据包来说可能太大,因此,视频应用程序需要将一帧划分为多个数据包。

​	现有技术之一,MPEG-DASH[7],只能过以下两种方法生产内容:1.混合音频/视频流。2.区分两种流,但是通过相同的时间进行分段。片段通过HTTP的方式从 origin mediaserver或intermediateHTTP caching servers.取得。虽然，目前存在各种各样的基于不同等级的帧的产生和获取的方法, 但这些方法中总有细分数据帧和重新组装恢复帧这类繁琐和同质化的工作。就MPEG-DASH而言,所有这些低层细节由HTTP / TCP协议处理机制解决。

### **2.2   Named Data Networking**

​	NDN网络具有两种类型的包:兴趣包和数据包。消费者发送兴趣包,即表达接收特定的数据。生产者产生数据包来满足其接受到的兴趣包。这两种类型的包均会携带数据的name,这个name能在一个包唯一地标识一个信息对象。NDN中的数据name提供给特定的应用程序,并具有多个组件。它可通过网络取回包，它还包含应用程序的特定信息以促进包的处理。先提出一个说明性的例子;使用以下命名模式的Simple-Video。数据名称开头为可路由路径“/com/youtube/”表示将所有携带这个名字前缀的兴趣包路由到这个数据生产者。下一个组件是媒体资源的标识符。这个组件之后的组件将视频和音频帧分离到单独的名称空间(即名称子树)。视频和音频帧都是顺序命名。每个视频帧可能包含多个部分,每个部分也顺序命名,而每个音频帧只由一个部分组成。

​	类似于IP,NDN网络提供数据报传递功能。消费者希望数据抓取服务是可靠的;他们可能还需要通过调整兴趣包传输来控制数据包的流。

​	Data可以按需生产,也可以独立于消费者需求生产。生产者和消费者之间的天然的异步性将产生两者之间的协调矛盾，如数据可用性,数据细节等。Interest Selectors是一个促进多个消费者获取数据,其得到的数据可能来自多个生产者的可用机制。任何Interest都可能携带的可选Selectors，可以指定附加条件来进行内容检索(除了name matching)。例如,当一个消费者发送了某名称前缀的Interest,之后接收到了非理想的数据包P，它可以重发一个Interest，这个Interest携带一个Exclude selector。这个selector可能包含P的确切名称或一些P的细节(如P的哈希值)。

### **2.3   NDN inside a Node**

​	NDN Forwarding Daemon(NFD)是一个应用程序和节点内部网络接口之间多路复用器[8]。NFD不区分应用程序和网络接口,只认为它们都是Faces。NFD有一个途径数据包的Opportunisticcaching内容存储库。NFD也可能有一个face到本地存储库(例如Repo-NG[9])，它提供了管理数据包存储的方式。

​	为了使数据可用,生产者进程通过NFD注册它的命名前缀，才能收到它的data的Interest。本地NFD添加前缀到FIB(转发表)，也将前缀转发到下一路由器。消费者进程不执行任何登记操作——它只发送Interest到本地NFD，然后向生产者转发Interest,无论其在本地或远端。



## 3.   PROGRAMMING MODEL

在本节中,我们定义消费者和生产者抽象上下文环境与我们的设计理念背后的基本原理。

### 3.1 Socket is inappropriate

​	我们编程抽象的设计目标是保证应用程序开发人员能足够自由的处理ADU的同时, 最小化生产和检索任意大小ADU的复杂性。在TCP / IP网络,类似任务由socket API管理。Socket是一个IP主机进程间的虚拟通道的数据传输参数的容器。Socket会创建双向数据管道，但是服务器和客户端的应用程序使用socket或多或少有一些细微的区别(如listen() 和accept() 函数)。Socket不被配置到通道上(如bind()或者 connect())不能使用。为了支持“时间异步”或者说通信双方之间的延迟限度, 应用程序开发人员经常依赖于更适合排队和传递消息的高层抽象。

​	NDN是一个基于pull-based的数据传播协议，因此应用程序消费数据的方式不同于产生数据的方式。所以，二者需要不同的数据传输参数。生产者应用,一般来说,更关心ADU分段, 保护和缓存/存储数据包和传入Interest的多路分解。另一方面,消费者应用更关心ADU完整性(对每个ADU都能抓取其所有的数据包),获取可靠性,接收数据的验证,以及通过Interests生成速度进行流和拥塞控制。

通过这些观察，我们将设计两种编程抽象:一种用于消费者应用程序,另一种用于生产者应用程序。

### 3.2 Design goals 

​	通过互联网传送数据时,较大的ADU必须分段,因为网络数据包大小受MTU限制。TCP / IP和NDN处理数据分段有两个主要差异。首先,因为TCP将所有应用程序数据视为字节流,所以TCP分段将忽略ADU的边界,因此ADU只能在分段重新组装后进行识别 (Figure1)。NDN数据包携带独立ADU的名字或ADU段,因此这些数据包可以直接匹配应用程序的数据单元。

​	第二,应用程序可以在数据传输时保证不同程度的观察和控制。在图1所示的简单例子中,如果使用TCP / IP发送几个连续的ADU，若网络上的某一个部分在运输过程中丢失,所有的后续ADU,即使它们已经到达目的地,也将阻塞交付给应用程序。这是著名的head-of-line(HOL)阻塞问题。另一方面,如果使用NDN，面临相同的部分损失问题时,所有成功收到ADU可以立即交付给应用程序而无需等待恢复丢失的部分。

#### 3.2.1 Goals for theconsumer abstraction

​	为了确定消费者抽象的设计目标,我们做一个初始的假设,一般来说,独立的应用程序希望根据自己的优先级进行ADU抓取。因此我们描述设计目标为应用程序可能希望在处理ADU时获得何种支持。鉴于我们仍然在对这个新消费者/生产者的API进行测试,当前的设计目标可能随着时间的推移，当我们对应用程序的需求有更深的理解时进行进一步修订。生产者抽象也是如此。

目前，我们相信新的消费者抽象模型应该支持以下应用程序模式。

1. ADU顺序抓取,并且允许在必要时丢失流上的任意ADU。这点可支持处理实时流媒体的应用程序。
2. 并行抓取ADU以加速内容传输。这可以应用在web download 和torrent方面。
3. 获取独立的动态ADU。物联网等应用程序需要这点。





### 3.3  Producer context 

​	producer context用于在常见的前缀下发布数据(图4)。它是通过调用给定的名称前缀参数的producer()原语进行初始化。与TCP / IP的服务器端socket不同, producer context在没有连接到网络,没有任何的Interest的情况下，准备发布数据。除了需求驱动的情况,我们的consumer / producer 模型不要求consumer和producer同时被“连接”。因此，数据发布行为可以在任何时间发生,包括生产者断开连接的情况。在我们的Simple-Video示例中,生产者按照自己的节奏发送数据,并且提前于消费者的获取。

​	一个应用程序进程调用produce() operation开始数据发送（传送名称后缀和应用程序帧(ADU)内容）。在Simple-Video 例子中，后缀名是帧数。在一般情况下,名称后缀参数允许应用程序开发人员重用相同的producer context，以发布任意的名称子树数据。在Simple-Video 例子中，一个context用于发送视频帧,另一个用于音频帧。生产者名称树中的context定位如图4所示。

当以下操作皆完成时，produce() operation结束

（1） 应用程序帧(ADU)已经分段为一组数量适当的数据包。

（2） 段号附加到每个包的名字上。

（3） 每个包都保证安全(如signed)。

（4） 递交数据到send缓存区，并且离开context。

​	虽然一些生产者应用程序可能想写永久存储的数据包（NDNFSor Repo-NG [9, 11]. ），默认情况下,段是暂时存储在send缓冲区（内存存储数据包）的。

​	context的发送缓冲区不同于socket发送缓冲区。具体有两个方面：首先,socket发送缓冲区用来转传unacknowledged segments 。而producer context的发送缓冲区是作为传入的Interest包所指示的data包的临时缓存。换句话说,发送缓冲区软化了数据生产和获取之间的时间异步。第二,在socket中,数据包ACK后去除,而在一个context中,数据包的驱逐基于内存环境，例如,当应用程序调用producer()的时候，缓存已经写满，并且替换方式是FIFO。

​	为了获得Interest所指引的data,producer context.必须attach到本地NFD。这可以通过调用attach()操作。到达的Interest进入接收缓冲区,并且等待轮次，直到它们与发送缓冲区的数据包匹配。如果Interest通过name和Interest选择器匹配data包成功, 就表示这个Interest在发送缓冲区中被满足了。如果没有找到匹配的数据包,那么需要告知应用程序这个Interest。

​	在某些情况下,对于特定producer context， Interest的到达速度可能太高了以致于不能尽快处理。在其他情况下,请求的数据不能在Interest的生存周期内产生。代替消费者盲目超时的方式,应用程序可以使用nack()操作给Interest一个NACK来使其满足(4.1节),这样消费者可以用最直接的方式来处理这种情况。

### 3.4 Consumer context	  

​	consumer context abstraction是一个将名称前缀与使用者特定传输参数进行结合的容器。Consumer context控制Interest的传输和数据包获取的整个流程。它是通过调用 consumer()原语进行初始化。传入的参数有两个:(1)一个名称前缀。(2)一个数据检索协议。

​	注意,一般情况下,名称前缀并不是一个ADU的完整名称。因为给定NDN命名空间都是以名称树的形式,所以应用程序开发人员可以重用一个consumer context 以反复获取相同名称前缀的多个ADU。在Simple-Video示例中,可以使用一个context获取所有的视频帧,另一个context获取所有音频帧。名称树中consumer contexts 的位置图5所示。

​	当一个应用程序调用consume() operation时，数据检索开始,并将命名后缀作为输入参数。在Simple-Video示例中,命名后缀是帧数。命名后缀参数允许应用程序开发人员重用相同的上下文获取多个ADU(图5)。在 context中,数据检索协议(Section 5)产生Interest并处理传入的数据包的其他相关事件(图3)。

​	数据检索的停止基于以下三个条件之一:

> ​	(1)上一个ADU已成功的进行数据包获取,验证和重组(如果需要的话);
>
> ​     	(2)发生了不可修复的抓取错误;
>
> ​	(3)调用了stop() operation。
>



## 4.SUPPORTING MECHANISMS

​	为了支持前文描述的有效的consumer / producer 通信,我们引入了两个新的机制:negative acknowledgements 和 manifests。本节详细讨论这些机制。	

### 4.1 Negative acknowledgement

​	NDN中，consumer应用程序通过发送Interest,从网络获取所需的data。如果在整个过程中Interest未找到匹配的data,它将到达producer context,要么从send buffer找到匹配的数据包,要么通知应用程序生成请求的数据。后一种情况发生在一些特定的数据的请求和第一次产生时。			

​	由于NDN是一个基于pull-based的网络协议,所以它也面临一些常见的轮询相关问题,这一点类似于HTTP[12]。HTTP client可以“short poll”HTTP 服务器 (即定期发送请求)以获得最新数据。被请求的数据还没有生成的时候，HTTP服务器的响应是一个空。等到请求客户端超时后，poll request将再次重复。为了避免HTTP客户端过于频繁的生成请求（因为这可能导致服务器和网络过载）,常用的的方法是HTTP long polling 。Long polling的方法是HTTP请求将会在服务器保持等待或“hanging”状态,直到被请求的数据可以发送回Client。

​	Long polling 适用于HTTP,因为底层TCP连接能确保HTTP请求可靠地传送到服务器,同时HTTP客户端一直在等待数据。就其本身而言，NDN网络层不保证Interest的可靠传输,更重要的是,未解决的NDN Interest会消耗路由器的资源(通过占据 PIT条目),所以Long polling技术不是一个可行的解决方案。为了有效地处理NDN动态生成数据环境下的轮询问题,必须满足两个条件:

​	(1)consumer应用程序必须确定它的Interest已成功达到生产者。

​	(2)应用程序可以根据当前的情况调节轮询频率。

​	negative acknowledgement (NACK) 可以满足这两个条件。我们定义了一个NDN数据包的子类型,当请求的data不可用时生成这个数据包。NACK携带一个ERROR code,重传计时器值,和其他可选的由应用程序定义的值域。它将告知消费者,1)Interest的要求producer已经收到。2)错误代码中包含了对consumer的建议和下一步动作的期望。目前,两个错误代码定义如下:

> 1. RETRY-AFTER—提示数据检索协议应该基于NACK的超时重传域来重新安排Interest的传输。这个算法与 Retry-After HTTP 和 SIP header field [13, 14]很相似。NACK 的 Retry-After域不会改变 Interest 的管道容量
>
>
>
> 1. NO-DATA —提示在consumer端的数据检索协议终止操作。 	
>

​       NACK 必须类似于Data数据包进行sign，同时必须采取额外的措施[15]以防止恶意用户发起拒绝服务攻击迫使生产者应用程序生成过量的NACK。

​	由于NACK也是NDN数据包,所以它们可以在中间NDN路由器进行缓存。同样的NACK包也可以用来满足多个消费者请求相同的数据的Interest。一个已经缓存了的NACK过时当且仅当它的生命周期(如FreshnessPeriod字段，由producer context赋值)超时。	作为一个经验法则,NACK包的生命周期不得超过重传中包含的超时值,否则消费者将会在超时重传之后，收到缓存过的NACK,所以将再等待一个超时时间。还必须记住,一个数据包可以在超时之前待在每个router hop 中,并且在生产者和消费者之间可以存在multiple router hops。因此我们建议设置NACK的FreshnessPeriod值为应用程序指定的重传计时值的十分之一。

### 4.2 Manifest

​	一个结构良好的的NDN应用程序应该充分利用“多对多缓存“的交流模式。为了使consumer更好的了解他们抓取数据的生成进度,producer应用程序可以打包必要的meta-information分发给consumer。

​	Manifest[16]是一种以分发目录的方式加速consumer应用程序执行的手段。目录可以包含普通的NDN命名,或数据包名称的hash值（如摘要）。使用携带数据名称与包相关摘要的目录的主要好处是可以去掉数据包签名加密操作。代替签名的方式,数据生产者对每一个新产生的数据封包计算一个简单的hash,将携带摘要的命名填充入manifest,只在manifest签名。Consumer应用程序通过抓取manifest并比较目录中摘要的方来验证数据包。携带目录命名的manifests需要在数据抓取之前获取,所以引入了额外的往返延迟[16]。

​	我们建议嵌入一个manifest以拒绝不受欢迎的潜在数据包[17]。 当ADU分段时，producer context可以调用produce() API 原语来执行该操作。基本思想是确定一个公约——将manifest命名作为数据包产生时的第一段。所以,consumer可以简单地通过Interest管道获取manifest和数据。如图6所示。当ADU的size过大时导致这个名字所辖的所有的片段不能进入一个manifest包时,可以使用多个manifest包与数据包周期交错的方式解决。

​	Manifest embedding 可使consumer 应用程序获得在同样的Interest滑动窗口下获取manifest和数据包的机会(滑动兴趣窗口包括已经发送但尚未满足的Interest,以及计划的Interest传输调度的时刻)。通过在数据包中设置KeyLocator字段指向相应的embedded manifest的方式,一个consumer应用程序能够在收到数据包的同时立即进行验证而不用没有等待其余的数据包。Manifest 是作为一个NDN数据包子类型实现的。除了目录名字,manifest可以携带各种键值对形式的meta-information如:

##### • Current data production rate.

​	直播型应用程序可以受益于了解当前数据包产生速度(打包),并利用这些信息调节Interest包。

##### • Other available versions.

​	在多版本内容环境下工作的应用程序可以不使用耗费时间的迭代Interest选择器的方式来找到ADU的可用版本。

##### • First and Last ADU sibling.

​	在大多数情况下,生产者知道一些较大信息对象(例如视频)的ADU总数。Simple-Video应用程序使用last ADU name标识视频的结束(如 frame#2500)。





































​	


​	

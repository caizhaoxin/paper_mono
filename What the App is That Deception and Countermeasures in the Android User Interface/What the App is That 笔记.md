### 背景介绍

目前安卓系统被广泛应用，用户经常会将自己的敏感信息告诉自己信任的app，而信任的出发点往往只是app的界面。基于这个原因，恶意软件可以利用安卓的漏洞来替换或者说用透明的界面来覆盖当前运行的app，从而窃取用户的信息。

###### 例子

比如你在玩一个恶意游戏（malicious games），然后你切换到一个银行app，被游戏app（仍在后台运行）检测到（如读取log或者检索/proc），那么他会像办法霸占你的top activity并模仿成银行的app界面，而你毫无察觉，这时，你输入银行账户信息，ok，完蛋.......

### 解决方案 & 贡献

1. 系统地研究了安卓系统上能被用于界面欺骗攻击的手段，包括前人论述过的和作者自己发现的
2. 根据1所研究的各种攻击手段，利用静态分析技术开发了个工具去检测有欺骗嫌疑的app，这是从应用商层面的
3. 鉴于2的方法可能会出现假阳性（比如一些关锁app功能使用的手段会与某些恶意app类似），作者开发了一款部署在安卓系统本身上的GUI攻击防御工具，从用户层面去防止这种情况的发生
4. 实验，证明这种方式有效

### 背景知识

#### 安卓基础

1. 安卓基于linux内核运行
2. 大多数app都从谷歌官方库（google play）或者其他其三方库下载，具体的通过下载apk然后安卓
3. 每个app都运行在各自的沙箱内
4. 安卓app由四大组件组成：Activity/Service/Broadcast Receiver/content Provider
5. 一些敏感操作，比如调用摄像头，查看相册等都需要获得用户的许可，当然这仅限于非系统app，比如大多数在应用商店下载的app，系统app就是本身安卓设备厂商自带的，比如小米手机上的屏幕锁屏等。

#### 安卓界面绘制

三大系统组件：

1. View: 比如按钮、文本框....
2. Activity： 可以看成View组件的控制器，控制其逻辑实现
3. Window： 可以看成是囊括view组件的容器，比如界面下栏有返回、首页、导航这三个view，它们被囊括在一个window里面，这个是系统控制的，开发者和用户无需理会

#### GUI Confusion Attacks：

GUI Confusion Attacks就是恶意软件通过构造某些正常软件的界面，从而替代正常应用的界面来与用户交互，以达到获取用户敏感信息等目的。

作者在文章中系统性的分析了各种实施这种攻击的手段，主要有两大类，一类是某种特定的API，第二类就是某些进阶手段，我大概总结下：

##### 特定的API（作者成为攻击向量，attack vector）

![1631682426232](E:%5Cpaper_mono%5CWhat%20the%20App%20is%20That%20Deception%20and%20Countermeasures%20in%20the%20Android%20User%20Interface%5C1631682426232.png)

###### 1. Draw on top

- UI-intercepting draw-over：简单来说就是将假的界面绘制在主活动界面的上面，大概原理就是重新设置窗口的Z-order，比如addView API中将flag参数设成PRIORITY_PHONE

- Non UI-intercepting draw-over：与UI-intercepting draw-over相类似，也是讲假的界面绘制在表面，但这次绘制的界面是透明的，这样用户以为实在跟正常界面交互，但其实是跟假的在交互

- Toast message：这个功能是不需要任何权限的，只要调用就可以在任何界面上绘制出来，比如下面这个can't login in就是个例子。

  ![1631683539328](E:%5Cpaper_mono%5CWhat%20the%20App%20is%20That%20Deception%20and%20Countermeasures%20in%20the%20Android%20User%20Interface%5C1631683539328.png)

###### 2、 App switch

- startActivityAPI ：调用startActivityAPI打开新活动，一般情况下新打开的界面不会立即绘制到应用界面之上，但通过设置参数组合还是可以做到的，即使主程序不是当前调用这个API的程序
- Screen pinning ：锁屏
- moveTaskToAPIs ：与startActivityAPI 相对应的另一种将新活动覆盖到主活动之上的API
- killBackgroundProcessesAPI：杀掉其他应用的进程
- Back / power button (passive) ：通过禁用或模拟后退按钮，使用户点击后以为已经切换了界面，但实际上仍在与恶意软件进行交互
- Sit and wait (passive) ：在恶意软件在后台运行时，偷偷把界面换成某些正常软件的界面，比如银行app界面，当用户想切换到银行app时，很容易污点了这个恶意软件

###### 3、 Fullscreen

所谓Fullscreen模式就是app会霸占整个屏幕，比如某些软件，王者/PUBG会霸占整个屏幕。恶意软件可以利用这点，创建一个虚假的主屏幕，包括虚假的状态栏和虚假的导航栏。因此，恶意应用程序会给用户一种她正在与正常操作系统交互的假象，其输入会被恶意利用。

##### 其他技术

- 通过阅读系统日志：通过阅读系统日志，得知用户当前与什么app进行交互，比如银行，这时恶意软件就抓住时机，利用上述的API来进行攻击
- 利用getRunningTask API：获取当前系统正在运行的app
- 通过读取/proc文件：linux系统将所有正在运行的进程都作为一个文件放入/proc里，当然这是个虚拟文件，通过检索这个文件获取当前正在运行的app

#### State exploration of the Android GUI API

分析了上述的API在实施GUI攻击时的各项参数选择，比较枯燥繁杂我这里不展开，有兴趣地可以去看看原文

#### Detection via static analysis

作者针对上述的攻击手段而设计的静态分析方法这里不展开，我就聊聊他实验的结果。

作者通过自己设计的静态分析技术对现存的app进行GUI攻击检测实验，主要抽取了四组对象：

- benign1：随机抽取500个google play的应用
- benign2：抽取500个google play的热门应用
- app-locker：抽取20个锁屏软件，鉴于前面所说的这一类（app locker）有恶意GUI攻击app的代码特征
- malicious：抽取1260个来自 Android Malware Genome project的恶意app

###### 实验结果

![1631688760480](E:%5Cpaper_mono%5CWhat%20the%20App%20is%20That%20Deception%20and%20Countermeasures%20in%20the%20Android%20User%20Interface%5C1631688760480.png)

- 首先从bengn1和2，可以发现其实市面上大部分app没有GUI攻击的倾向，即使有也只是某些与恶意软件相类似的比如app-locker。而有这种倾向的主要是一些热门软件，因为这些软件的功能非常丰富，或多或少地被检测出有攻击倾向，但这一类软件会利用这种攻击的概率也不高，毕竟用的人多，”群众“的眼睛是雪亮的
- 对于恶意软件集，作者只检测了25个，作者说大多数这些软件都想干坏事情，所以就不检测那么多。但25个里面有21个真阳，说明这个攻击还是有点实用的。

当然，作者也提到，这个静态分析攻击仅仅只是起了辅助作用，比如上面的热门软件也检测出不少例子（因为静态分析仅仅只是从代码的层面去看问题）， 但却没有GUI攻击的目的，也就是所谓的假阳性。所以一款app究竟有没有GUI攻击的目的，还是需要应用厂商有专门的人工检测..........

##### UI DEFENSE MECHANISM（on-device defense system）

为了缓解纯静态分析带来的问题，比如上面所说的出现假阳性，作者还发明了一款防御系统来帮助用户检测恶意软件。

作者通过修改系统，告诉用户他们当前所使用的软件是属于哪个厂商那个开发者。因此首先需要解决三个问题：

- 如何知道当前用户实际在跟那个app交互
- 如何知道这款app的开发者是谁
- 如何更好的展示

首先最后一个问题好解决，只要不影响到用户正常的操作就可以了，并且不能受任何非系统程序控制

对于第一第二个问题，因为安卓的生态比较开放，用户可以从google play下载app，也可以从第三方应用市场下载，那这样子谁来为app的签名背书？作者用了和浏览器相同的做法，也就是TLS/SSl证书证明。也就是说像这样。

![1631690210676](E:%5Cpaper_mono%5CWhat%20the%20App%20is%20That%20Deception%20and%20Countermeasures%20in%20the%20Android%20User%20Interface%5C1631690210676.png)

开发者需要申请合法有效的域名签名（HTTPS）来对自己开发的安卓app进行签名。用户打开app时看到上图所示的类似”Facebook， Inc“的标记就知道我在跟谷歌打交道，那就放心了。否则就要打起十二分精神——我是否信任它。

初次之外还有一个比较有意思的问题——fullscreen。恶意软件用过fullscreenAPI，在全屏的情况下整了个假的导航栏，然后整了个假的类似上面的脸书标志怎么办？这是作者加了secret，在手机启动时给用户选择一个图案，这个团时用户和安卓系统本身的秘密，没有任何第三方知道，比如我选择了个”小狗“图案，然后我打开facebook应该是上面的第一张图，但是恶意软件用过fullscreen手段弄成了第三幅图，这是我就可以发现这个软件有问题了！

##### 实验：

实验目的：验证自研的on-device defense system的有效性。

Group：通过在Amazon Mechanical Turk招聘实验人员

group 1：使用未修改过的安卓系统

group 2：使用修改过的安卓系统，但不知其如何工作，也不知他们的设备将会收到GUI攻击

group 3：使用修改过的安卓系统，知道其如何工作也知道将会收到攻击

Tasks：

B1 & B2：这两个任务打开的都是正常的facebook app

Astd：对app实施GUI攻击

Afull：在fullscreen的情况下实施GUI攻击

结果如下：

![1631690856314](E:%5Cpaper_mono%5CWhat%20the%20App%20is%20That%20Deception%20and%20Countermeasures%20in%20the%20Android%20User%20Interface%5C1631690856314.png)

具体的数据不说了，可以自己看。其实很明显，中招概率：无这个实时防护机制 > 有防护机制但用户无防备 > 有防护机制&用户有意识准备。

总而言之，实验结果得知，这东西还是有效果的........

##### Limitations

- 在group 2中，即使没有提高告诉用户设备会受到攻击，但用户看到这个玩意多多少少会警惕，从而对实验结果造成影响
- 可能有时网络延迟造成影响

##### 总结

- 系统性的总结了可能发生的GUI攻击类型
- 开发了一款工具去研究如何使用主要的安卓GUI API去发动这样的攻击，并探索执行这类操作所使用的参数
- 开发了两层防护机制：
  - ​	其一：利用静态分析技术在去检测有GUI攻击嫌疑额apk，并用户辅助应用库去审查app应用
  - ​    其二：开发了名为on-device defence system去帮助用户检测恶意软件
- 最后实验证明on-device defence system的有效性
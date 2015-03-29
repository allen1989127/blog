---
layout: post
title: "技术填充生活之该死的酒店网络登陆系统"
date: 2015-3-29 20:00:00
categories: [Linux, Network, Python, 技术改变生活]
---

在写这篇博客的开头，首先丫的骂一下公司，第二次TNND把爷扔到银商这边过来了，而且每次都是新项目，一扔就扔一大段时间，还能不能好好的干活了。

算了，每次过来也不能算是没有收获，至少能收获一点出差报销不是，哈哈。

原归正题，准备进入正式的博客状态。

首先说明一下这个博客的标题，从这篇博客开始，将会陆续更新（虽然说更新的速度会相当的慢，嘿嘿）一整个叫做技术填充生活的博客，这个系列的博客的宗旨是给技术宅们提供一个能在平时生活中可以真正的去应用的技术，而不是一直处在一个光说不练的状态。当然了，这系列的博客并不适合没有一定技术基础的朋友们，**不是**用来教大家如何配置路由器啥的直接找个技术宅男友就可以搞定的事情，如果需要上诉知识请在网上搜如何搞定一个技术宅做仆人，哈哈。

虽然说是突发奇想，不过在平时觉得无所事事的日子里至少给自己点事做，同时发扬下开源的分享精神，想想还是有点小兴奋的。

那就让我们进入正题吧。

为什么说酒店的网络登陆系统该死呢？当然，平时我们并不需要在时间上连续的网络资源，基本上大部分的酒店登录系统都会在隔一段时间要求重新登陆一遍才可以继续上网，这样做自然无可厚菲，就是担心客人在不使用网络的情况下依然不断的占用着网络资源，对于一般人来说这也不会有大的影响。但是作为一个节操岗岗的技术渣渣来看，这是在对我的挑战，特别是在最近需要在docker上部署一个andoid source code的镜像时，一般都是在睡觉时挂着继续下载，不然五六十G的镜像，何时是个头。这个时候就有了这么个重大的需求：自动查询当前的网络状态，并且在发现网络断开的情况下自动进行登陆。

当有这种需求的时候，就可以开始思考该怎么做了，对于习惯了GUI的人来说，想到的方法应该是在界面上自动去做键盘输入和鼠标的动作，这种做法是不是有点2b了，当然，这只是我的yy猜测，请允许我的yy行为吧。

还是回归正确的做法吧，现在的酒店登陆系统和公司的验证系统基本都是基于网页登陆，简单的做个解释，当你的电脑接入路由的情况下，路由器会将你电脑的数据包重定向到一台web服务器上（利用iptables等工具），然后进行登陆验证，验证成功后web服务器会给路由器发一个通知，你的机器对应的ip数据可以放行，无需在做重定向，这就是最简单的web登陆系统的验证流程。

所以呢。。。是不是觉得很明确了，web登陆，不是就是基于http的协议吗？高级一点，就是基于https，但是路由器都是酒店的，基于https有毛毛用阿，这样也给我们进行抓包分析提供了巨大的便利，毕竟是明文麻，一眼基本就能看出来数据该怎么组包了。接下来我会基于我现在所在的酒店作为范例进行讲解，酒店是上海景宏商务酒店，在张江商业广场对面，如果真的闲得蛋疼可以来直接尝试下我的脚本^_^

接下来就真的真的是正文了。

做过web开发的同学们肯定知道，做登陆的页面一般都是一个post的form然后提交到服务端就ok了，不过一般做服务端开发的时候并不怎么去关注http的协议，不过当我们要自己向服务端发送登陆数据包的时候，自然要关注这个东东。

首先，需要进行实际登陆的操作然后进行本地抓包，对了，忘记说明了，因为本人并不怎么使用windows的操作系统，所有用到的工具均是在linux下使用的，如果需要在windows上进行操作，完全可以在网上找到相应的工具。我这里使用的抓包工具是超级瑞士军刀tcpdump，相信大部分人都听说过，就是由于它过于强大以及对网络的不熟悉不敢使用，其实在这个主题上用tcpdump是相当简单实用的，gogogo。。。用起来。

给几个前提，在进行实际登陆的动作的时候，我已经在浏览器的地址栏上知道验证web服务器的ip为192.168.0.216，还有我的无线网卡设备名为wlp3s0，接下去的事情就比较简单了，在终端中输入下面的命令：

{% highlight shell %}
sudo tcpdump -i wlp3s0 host 192.168.0.216 -A > tcpdump
{% endhighlight %}

简单说明一下这条命令，从PC的无线网卡获取本机与web服务器进行交互的数据，同时数据是以ASCII码进行输出（-A），并且重定向到tcpdump的文件中。

接下去做一次登陆操作，你会在tcpdump文件中查看到一些ASCII数据，我只列出有用的一条数据

{% highlight c %}
17:03:25.945071 IP FreeMan.44676 > 192.168.0.216.http: Flags [P.], seq 1:781, ack 1, win 229, options [nop,nop,TS val 266364 ecr 306008079], length 780: HTTP: POST /auth/Sea/checkin.php HTTP/1.1
E..@.o@.@.....Py.......PZ...........-......
...|.=P.POST /auth/Sea/checkin.php HTTP/1.1
Host: 192.168.0.216
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:36.0) Gecko/20100101 Firefox/36.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.0.216/auth/Sea/login.php
Connection: keep-alive
Content-Type: multipart/form-data; boundary=---------------------------20153011372120023553538499134
Content-Length: 303

-----------------------------20153011372120023553538499134
Content-Disposition: form-data; name="account_in"

8432
-----------------------------20153011372120023553538499134
Content-Disposition: form-data; name="password_in"

270059
-----------------------------20153011372120023553538499134--

{%endhighlight %}

我假设看这篇博客的你是有对网络有一些熟知的。

FreeMan.44676 > 192.168.0.216.http：这条数据中可以看出这个数据包是从本地发送到web服务器的http端口

HTTP: POST /auth/Sea/checkin.php HTTP/1.1：这条数据中可以看出这是一个基于HTTP 1.1协议的POST消息，同时，这个数据包是发送到/auth/Sea/checkin.php的action中进行处理

接下来会看到一些看不懂的数据，跟乱码似的，这个不需要理它，这些是IP以及TCP包头数据，无法在ASCII中进行正确的显示，如果真的愿意对这个数据包进行研究的话可以通过tcpdump的-X来进行抓包分析，具体我就不展开了。接下来我们可以跳到POST打头的数据，这里开始就是真正的HTTP数据包，这里面有大量的HTTP的header数据，我们可以跳过，因为这些都是一些key-value数据，后面处理也就是都填到对应的数据头中去就行了。直接到Content-Type

Content-Type: multipart/form-data; boundary=---------------------------20153011372120023553538499134：http协议在头的部分基本上都是这样的key-map形式，这条数据说明了这个http报文的数据格式，就是这是基于multipart/form-data进行提交的数据，这个对于网页开发的人员来说会比较熟悉一些，这个不了解也不是那么重要，不过boundary后面的一串数据就需要进行一定的了解，这是一段数据的分割界线，在后面的数据中大概能看出来帐号和密码的数据是通过该分界进行分割的吧。

接下来就是数据内容部分了，8432为房间号，270059是酒店用户登陆密码，我们将用python进行开发，所以这些数据基本拷下来就可以了。

上面对我们进行自动登陆的数据包进行了简单的分析，并没有做一个很详细的分析，如果需要进行彻底理解的话，大家动动手查看一下http的协议进行了解。

接下来移步到python脚本上，请允许我先上代码：

{% highlight python %}
import urllib2

from urllib2 import URLError

class Login:

    def __init__(self, name, password, url):
        self.name = name
        self.password = password
        self.url = url

    def login(self, action, ref_action):
        headers = {
            r'User-Agent': r'Mozilla/5.0 (X11; Linux x86_64; rv:36.0) Gecko/20100101 Firefox/36.0',
            r'Accept': r'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            r'Accept-Language': r'en-US,en;q=0.5',
            r'Accept-Encoding': r'gzip, deflate',
            r'Referer': r'http://192.168.0.216/auth/Sea/' + ref_action,
            r'Content-Type': r'multipart/form-data; boundary=---------------------------409036126110349932270895253',
            }

        data = '-----------------------------409036126110349932270895253\r\nContent-Disposition: form-data; name="account_in"\r\n\r\n' + self.name + '\r\n-----------------------------409036126110349932270895253\r\nContent-Disposition: form-data; name="password_in"\r\n\r\n' + self.password + '\r\n-----------------------------409036126110349932270895253--\r\n'

        req = urllib2.Request(self.url + action, data, headers)
        try :
            response = urllib2.urlopen(req)
        except URLError, e:
            print e
            return False

        if response.getcode() != 200:
            return False

        success = False
        for line in response.readlines():
            if 'Sign in successfully!' in line:
                success = True
                break

        return success
{% endhighlight %}

其实这个代码一眼看上去是极其简单的，就是将刚刚分析的数据包封成请求数据，然后利用urllib2发送出去就行了，就是有一个需要注意的，在http协议中，回车并不是单一的\n，而是\r\n。

拼装号数据后通过urlopen将数据发送出去即可，接下来就是处理返回的数据，当获得的返回码为200，即为成功返回数据之后，对取到的数据进行分析，就是对返回的页面数据判断是否返回的是登陆成功的页面即可。

其实到这里，基本上已经可以结束了，不过我还是大概再讲一下基于我自己需求的整个脚本流程吧，还是请允许我先上代码，嘿嘿

{% highlight python %}
vpnc = NMControl(VPN_NAME)
hellot = HelloTwitter()
login = Login(NAME, PASSWORD, URL)

while 1:
    #first need to checkout the network is ok? ping google is easiest
    res = hellot.hey()
    if res == 0:
        print ''
        print 'network is all right at', time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(time.time()))
        print ''
        print ''
        print ''
        time.sleep(12)
        continue

    print 'network is down, let\'s start rebuild it'

    vpnc.down()

    while 1:
        if not login.login(ACTION, REF_ACTION):
            time.sleep(2)
            continue
        else:
            break

    print 'success to login'

    vpnc.up()
{% endhighlight %}

我的流程需求其实是这样的，首先要判断网络是否连通，当然这个连通包括vpn是否连上已经可以进行翻墙了，一旦发现网络是断开的，就进行登陆，登陆成功后重建vpn，接下来就是不断检测的流程了。

流程知道了，一切都很简单了，这里再简单介绍一下怎么去简单的重连vpn，我使用的是linux的NetworkManager进行网络设备的管理，对应kde有相应的插件，也有对应的命令行客户端nmcli可以进行操作，我的vpn是在kde环境下通过GUI已经配置好了，然后通过：

{% highlight shell %}
nmcli c up yunti
nmcli c down yunti
{% endhighlight %}

两条命令就可以启动和关闭vpn了，其中，c表示connection，表示对一个链路的操作，up和down很明显，表示的是启动还是断开，yunti是vpn的别名。

接下来就很容易理解了吧，先去ping一下twitter检测网络的连通性，如果ping通了，说明网络没问题，间隔12秒后再继续检测（为什么是12呢？好问题，因为我打球时候的背号是12,哈哈），如果没有ping通，就先断开vpn，当连接着vpn去查一下路由表可以看到登陆数据包是无法发送到web服务器上的，所以这个动作是必要的。接下去就是登陆了，这个就不再敖述了哈。登陆成功之后就启动vpn，接下来继续进行网络连通性的检测。

好了。写到这里我想基本上知道了要做这个登陆脚本的基本流程了吧，当然，在这之上还有更多的需求可以再不断的加上去，当然，通过这篇博客不是说让大家知道这个流程，其实最主要的是需要掌握抓包以及分析网络数据包的能力，这样才能更好的搞掉各种验证的机制不是。

希望这个博客能够帮助到各种出差的码农或者烦躁于各种验证的技术屌丝，可以在[我的github](https://github.com/allen1989127/FuckLogin)上获取到这个脚本的项目工程，虽然简单，也还是可以贡献出来的，嘿嘿。

如果有什么问题，可以在twitter上@scanz1989127，我会在有时间的情况下尽快回答，大家也可以进行互相探讨交流，同时也促进我的进步。
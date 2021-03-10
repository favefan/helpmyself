# 主机连接闪讯或VPN时，WSL2内的网络操作出错为TSL类或其他网络错误

## 环境

宿主机操作系统：Windows 10 1909 专业版 64位

宿主机运行的嫌疑程序：Netkeeper（闪讯）5.3.9.5221.23

WSL2操作系统：Ubuntu 20.04 LTS

WSL2中出现问题的程序：`apt`、`docker pull` ...

## 问题复现

成功连接闪讯后，启动WSL2-Ubuntu，

### 1. apt

```bash
$ sudo apt update
Hit:1 https://download.docker.com/linux/ubuntu focal InRelease
Ign:2 https://mirrors.ustc.edu.cn/ubuntu focal InRelease
Ign:3 https://mirrors.ustc.edu.cn/ubuntu focal-security InRelease
Ign:4 https://mirrors.ustc.edu.cn/ubuntu focal-updates InRelease
Ign:5 https://mirrors.ustc.edu.cn/ubuntu focal-backports InRelease
Err:6 https://mirrors.ustc.edu.cn/ubuntu focal Release
  Could not wait for server fd - select (11: Resource temporarily unavailable) [IP: 202.141.160.110 443]
Err:7 https://mirrors.ustc.edu.cn/ubuntu focal-security Release
  Could not wait for server fd - select (11: Resource temporarily unavailable) [IP: 202.141.160.110 443]
Err:8 https://mirrors.ustc.edu.cn/ubuntu focal-updates Release
  Could not wait for server fd - select (11: Resource temporarily unavailable) [IP: 202.141.160.110 443]
Err:9 https://mirrors.ustc.edu.cn/ubuntu focal-backports Release
  Could not wait for server fd - select (11: Resource temporarily unavailable) [IP: 202.141.160.110 443]
Reading package lists... Done
E: The repository 'https://mirrors.ustc.edu.cn/ubuntu focal Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'https://mirrors.ustc.edu.cn/ubuntu focal-security Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'https://mirrors.ustc.edu.cn/ubuntu focal-updates Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'https://mirrors.ustc.edu.cn/ubuntu focal-backports Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

### 2. docker pull

```bash
$ docker pull mysql
Using default tag: latest
Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout
```

## 问题解决

### 1. 查看Netkeeper的MTU

在Windows的`powershell`命令行中输入以下命令执行：

```powershell
> netsh interface ipv4 show subinterface
```

结果类似于：

```powershell
   MTU  MediaSenseState   传入字节  传出字节      接口
------  ---------------  ---------  ---------  -------------
  1480                1  956363711  334417521  Netkeeper
4294967295                1          0    1176112  Loopback Pseudo-Interface 1
  1500                1     287647     158299  WLAN
  1500                5          0          0  以太网
  1500                5          0          0  本地连接* 1
  1500                5          0          0  本地连接* 2
  1500                1    1008699  135703769  vEthernet (WSL)
```

可以看到Netkeeper对应的MTU为1480

### 2. 在WSL2中设置MTU为1中的Netkeeper中的MTU值

```bash
sudo ip link set dev eth0 mtu 1480
```

执行后即可生效，`apt`和`docker pull`命令的执行均没有问题了。

如果经常使用VPN等可以将上述命令放入`.bashrc`等文件中。

## 问题溯“原“

### 参考资料

* [WSL2/VM TLS Handhake failed](https://sbulav.github.io/wsl/wsl2-vm-tls-handshake-failed/)
* [[Issues]WSL2 fails to make HTTPS connection if Windows is using VPN #4698](https://github.com/microsoft/WSL/issues/4698)
  * [**[r-l-x](https://github.com/r-l-x)** commented on 13 Oct 2020](https://github.com/microsoft/WSL/issues/4698#issuecomment-707365664)
* [为什么PPPoE的最佳MTU是1492？ - zdho的回答 - 知乎 ](https://www.zhihu.com/question/54689598/answer/203006929)
* [PPPOE拨号下MTU设置](https://www.cnblogs.com/p2liu/archive/2011/02/14/6048797.html)
* [ TCP Problems with Path MTU Discovery](https://tools.ietf.org/html/rfc2923#page-3)

### 名词解释

* **MTU**：最大传输单元（Maximum Transmission Unit，MTU）用来通知对方所能接受数据服务单元的最大尺寸，说明发送方能够接受的有效载荷大小。
  是包或帧的最大长度，一般以字节记。如果MTU过大，在碰到路由器时会被拒绝转发，因为它不能处理过大的包。如果太小，因为协议一定要在包(或帧)上加上包头，那实际传送的数据量就会过小，这样也划不来。大部分操作系统会提供给用户一个默认值，该值一般对用户是比较合适的。来源：[最大传输单元 - 百度百科](https://baike.baidu.com/item/%E6%9C%80%E5%A4%A7%E4%BC%A0%E8%BE%93%E5%8D%95%E5%85%83/9730690)
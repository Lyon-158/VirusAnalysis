## 浅析Trojan/Killav.G木马病毒

#### 微软大佬对病毒的解析 https://www.microsoft.com/en-us/wdsi/threats/malware-encyclopedia-description?Name=Trojan%3aWinNT%2fKillav.G

——Analysis by Ric Robielos

#### 微软办法：下载Microsoft Safety Scanner<mark>无法解决</mark>

#### 解决方法：卡巴斯基或者360安全卫士杀毒，本人采用了360

微软对病毒的描述

```python
Threat behavior
#Trojan：WinNT/Killav.G是一个恶意系统驱动程序，充当PWS：Win32 / OnLineGames变种的恶意组件
Trojan:WinNT/Killav.G is a malicious system driver that acts as a malicious component for PWS:Win32/OnLineGames variants
#PWS：Win32/OnLineGames.ZDV 是 PWS：Win32/OnLineGames.ZDV！dll 的组件检测，密码窃取木马。
PWS:Win32/OnLineGames.ZDV is a detection for the dropper component of PWS:Win32/OnLineGames.ZDV!dll, password-stealing trojan.
```

可以注意到，病毒表现为关闭 Windows Defender的几乎所有服务，通过了解，会窃取账号密码信息



#### 以下是个人分析和手动"杀毒"过程

#### 一、中毒起因及火绒方案

今天是2022/11/29，昨天给Need For Speed Carbon 下载游戏Trainer的时候上了外 网，电脑中了Trojan/killav.g木马病毒。

**火绒杀软**提示在C:\User\username\App Data\Local\Updates文件夹下，<mark>**已经处理** </mark>

##### 众所周知，**已处理 = 删除直接危害文件**，然而**根源还在**

身为安全专业的我尝试能否手动根除该病毒。上网找资料，几乎没有 Killav.G 的合理解决办法(国内)

![image-20221129165012097](http://homeofle.cn:1080/i/2022/11/29/6385c7cbc5154.png)

#### 二、尝试手动排查

根据杀软的日志，原以为病毒文件在 C:\User\username\App Data\Local\Updates文件夹下.

Updates目录结构如下：

> import.reg	（修改注册表，关闭Windows Defender，也删除了很多相关注册表，导致Defender完全不能启动）
>
> Run.vbs	（被执行程序调用脚本之一）
>
> Windows	（不清楚是什么，反正不是好东西）
>
> Windows.bat (杀软报毒，是直接发作文件，但不是最主要文件)

import.reg 写了对**WinDefender注册表**的操作

![IMG_20221129_125529](D:\qqReseve\MobileFile\IMG_20221129_125529.jpg)

---

run.vbs **被调用**的简单的运行脚本，调用了 Windows 和 Windows.bat

![IMG_20221129_125705](D:\qqReseve\MobileFile\IMG_20221129_125705.jpg)

---



在删除 `Updates` 文件夹后【**真的后悔没做备份**看源码】，重启系统。发现 `wscript.exe` 仍会调用执行 run.vbs 的情况，说明 `~\Updates`只是被调用的一小部分【杀软不能解决一切安全问题】

![image-20221129172619401](http://homeofle.cn:1080/i/2022/11/29/6385d03bda87b.png)

于是**尝试把wscript.exe从System32 中移出**，看看有**哪些程序或服务未正常执行**，缩小排查范围



把wscript.exe从System32 中挪出来后 {将\System32下的改名为wscript.txt（以保证再次重启时，无法调用到wscript.exe）}<mark>后面恢复了回来</mark>

重启后在 **事件查看器** 中，发现有服务启动错误的报错信息，摘录如下：

```assembly
#移除wscript.exe后，非正常运行的服务
HwOs2ECx64	#系华为matebook的电脑管家服务之一

rsEngineSvc	【后经过确认，系CheatEngine捆绑垃圾软件RSV Antivirus的附属服务】
rsWSC	  上同【后经过确认，系垃圾软件RSV Antivirus的附属服务】

Temp_Monitor_Service #系Microsoft服务
HRWSCCtrl #系Huorong服务
```

##### 于是，我提出猜想，病毒的源头应该是系统的某服务，而非应用程序

我尝试在注册表编辑器 `Regedit` 中删除病毒服务，路径如下 

`计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services`

找到对应的服务名称后，**一定单击左键看 `Description`描述，避免误删！**

根据杀软报毒时间，以下为嫌疑名单：【27日21时首次报毒】【非严谨推理】

![image-20221129171644422](http://homeofle.cn:1080/i/2022/11/29/6385cdfebcce9.png)



身为程序员的我应该找到了病毒的服务，没有找到是什么**exe**或者可执行程序，**在每次开机的时候都会调用 wscript.exe 去读取 run.vbs**，无奈，只能**参考Microsoft的英文病毒介绍**

下面看介绍：

```python
Trojan：WinNT/Killav.G 通常位于您的计算机中，文件名为“Windows/System32/drivers/name.sys”。

Trojan：WinNT/Killav.G 通过创建以下注册表项注册为系统服务：

In subkey: *HKLM\SYSTEM\CurrentControlSet\Services\ahnurl*
Sets value: "*Type*"
With data: "*dword:00000001*"
Sets value: "*Start*"
With data: "*dword:00000002*"
Sets value: "*ErrorControl*"
With data: "*dword:00000001*"
Sets value: "*ImagePath*"
With data: "*<system folder>\drivers\ahnurl.sys*"
Sets value: "*DisplayName*"
With data: "*ahnurl*"
```



> Trojan:WinNT/Killav.G可以删除或终止安全相关进程和文件【这里就解释了为什么WindowsDefender用不了了】

但是我按照官网的解释找来找去，找不到啊，Services注册表翻遍了，有两个驱动感觉有嫌疑，但是杀软不报毒。。

不知道是不是火绒报毒名称有误还是什么，按照微软的方法，解决不了问题。

经过反复尝试，无奈，咬牙下载360安全卫士



通过本次对病毒文件的浅显的阅读，发现表面的文件依赖关系大概如下：

某某可执行程序或伪装的驱动程序，在每次开机时被调用，执行import.reg，从而运行run.vbs，Windows和 Windows.bat 命令行脚本；每开机一次，修改一次 Windows Defender注册表，防止其正常工作（我的已经被移除了注册表）

限于本人能力有限，其他的具体技术细节无法看出，只能借助杀软解决问题了。

还有，**不是所有时候、所有的杀软都靠谱**

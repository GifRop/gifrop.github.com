title: 谈谈Aladdin HASP SRM的一般破解思路
date: 2014-05-16 09:42:40
tags: [加密狗]
---
看到论坛有人在讨论这个,我也来推一把.<!-- more -->
无狗  
能力太差搞不定.  

有狗  
直接脱壳  
关于修IAT,先找到HOOK后的IAT表的位置,随便取一个函数,新建EIP,然后F7的跟进去,找到一个固定的地方,会出现正常的函数,写回去就好了.  
记得upk论坛里有这个脱壳脚本的,关键在于,这里需要注意几个API是被壳模拟掉的.  
  
印象最深的是这几个:  
  GetCommandLineA  
  GetStartupInfoA  
  GetCurrentProcess  
  GetProcAddress  
其他的如果还有你不知道是什么函数的,直接找个同类型编译器编译的无壳程序,对比上下文代码一般都能找到.  
  
壳都脱了,下面来说破解.  
破解很简单.直接下载一份狗的SDK,编译一个demo程序.  
载入OD你们会发现,在打开,读取,写入,关闭等操作的函数调用前后都有一段奇怪的代码,  
(我这里没有备份的二进制字节码了.好几年没碰了).把这个放到需要破解的程序里一搜就知道那些函数是那些了.很简单.  
直接HOOK掉读取函数,(没脱壳的程序先HOOK掉CreateFileW函数,在My_CreateFileW中再Hook读写函数)  
在my_read函数里直接修改输入参数,把狗里的数据全部读取出来写入文件中就OK了.  
  
解析来把全部狗操作的函数都HOOK掉,重定向到对文件的操作.  
比如  
Open_dog => CreateFile  
Read_dog => ReadFile  
....  
或者  
你把数据写在dll里,弄成个Buffer,按照地址便移索引.把dll加到程序的导入表中........  
Do what you want to do..  
  
最后你会发现,现在完全不需要狗了.  
就是这么简单.  
  
Have a fun....  
  
--keykernel  
20140515  

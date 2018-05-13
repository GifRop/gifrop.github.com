title: KeyGen FileSplit 2.34.424R
date: 2014-05-14 17:14:52
tags: [KeyGen]
---
某群某人发了个软件说搞不定求破解，接着ZeNiX就放了个注册成功的图片.<!-- more -->  
![](http://ww2.sinaimg.cn/large/8061a41egw1egeptvd7kjj20d507k0tc.jpg)  
对于这种好玩的必然是要紧随大牛，OLLYDBG调试起。  
改个返回值，秒杀，but...没显示注册给谁,唔，不好玩，直接上IDA。  
我们主要关注的是411C70这个函数,函数比较简单,直接上伪码:  
{% codeblock lang:cpp %}
int CheckKey(char *user_name, const char *key_code)
{
    int name_len = 0,tmp = 0;
    int j = 0,i = 4;
    int r_key1_value = 0,g_key3 = 0; 
    int key1 = 0,key2 = 0, key3 = 0, key4 = 0;
    if ( sscanf(key_code, "%d-%d-%d-%d", &key1, &key2, &key3, &key4) &lt; 4 )
        return 0;
    key2 = (BYTE)key2;
    if ( key1 == 3535 && key2 == 35)
	    return 0;
    if ( key1 == 453 && key2 == 6272 && key3 == 7043 && key4 == 4 )
        return 0;
    name_len = strlen(user_name) - 1;
    do
    {
        tmp = (user_name[j] + user_name[name_len - j]) << 7;
        tmp = tmp | 1;
        j = (j + 1) % (signed int)(name_len + 1);
        r_key1_value = tmp % 10 + 10 * r_key1_value;
        --i;
    }
    while ( i );
    if ( key1 != r_key1_value
        || (g_key3 = vm_decfuc((int)&vm_data, key1, key2, (int)user_name), g_key3 != key3)
        || (key2 + key3 + key1) % 0xAu != key4 )
        return 0;
    return 1;
}
{% endcodeblock %}
  
通过这个函数我们知道:key2是随机的，只有8位参与计算，(key1+key2+key3)%10 = key4.  
仔细看到我注释的一个函数名vm_decfuc,这个是一个小型的虚拟机.  
弄个IDA的函数图:  
![](http://ww3.sinaimg.cn/large/8061a41egw1eges9h8d5tj20jz06gq3k.jpg)  
虚拟机的Dispatcher:  
{% codeblock lang:x86asm %}
00411985   lea eax,dword ptr ds:[esi+esi\*2]   ; EAX=00000003
00411988   mov edx,dword ptr ds:[edi+eax\*4]   ; EDX=0000000E
0041198B   lea eax,dword ptr ds:[edi+eax\*4]   ; EAX=004312A4
0041198E   cmp edx,0x1D         			  ; FL=CS
00411991   ja FSplit.00411BDD
00411997   jmp dword ptr ds:[edx*4+0x411BF0]  ;VMDispatcher
{% endcodeblock %}

看一个Handler Vm_Add:
{% codeblock lang:x86asm %}
00411ADA   mov ecx,dword ptr ds:[eax+0x4]     ; ECX=00000000
00411ADD   mov edx,dword ptr ds:[eax+0x8]     ; EDX=00000003
00411AE0   mov eax,dword ptr ss:[esp+edx*4+0x8]      ; EAX=00000FAC
00411AE4   mov edx,dword ptr ss:[esp+ecx*4+0x8]      ; EDX=000016A1
00411AE8   lea ecx,dword ptr ss:[esp+ecx*4+0x8]      ; ECX=0012F290
00411AEC   add edx,eax                        ; FL=P, EDX=0000264D
00411AEE   inc esi                            ; FL=0, ESI=00000002
00411AEF   mov dword ptr ds:[ecx],edx
00411AF1   mov ecx,edx                        ; ECX=0000264D
00411AF3   jmp FSplit.00411985
{% endcodeblock %}

截取的vm_data数据:  
{% codeblock lang:x86asm %}
00 00 00 00 03 00 00 00 AC 0F 00 00 
0E 00 00 00 00 00 00 00 03 00 00 00 
05 00 00 00 03 00 00 00 02 00 00 00 
1D 00 00 00 00 00 00 00 00 00 00 00 
0D 00 00 00 02 00 00 00 01 00 00 00 
13 00 00 00 00 00 00 00 00 40 00 00 
16 00 00 00 03 00 00 00 00 00 00 00 
......
{% endcodeblock %} 
  
结构很简单，一个Handler使用了3个DWORD数据:00000000、00000003、00000FAC.  
00000000为操作码，00000003、00000FAC为操作数。  
其中1B操作码是用于判断跳转的,还原出代码为:  
{% codeblock lang:cpp %}
int getkey3(int key1,int bkey2,char* user_name)
{
    BYTE bN = 0;
    int i= 0,flg = 0;
    int nl =  strlen(user_name);
    key1 = key1 + 0xfac;
    while(i < nl)
    {
    	bN = user_name[i];
    	flg = key1 & 0x4000;
    	if(flg)
    	{
    		key1 = key1 & 0x3FFF;
    		key1 = key1 << 1;
    		key1 = key1 | 1;
    	}
    	else
    	{
    		key1 = key1 << 1;
    	}
    	bN ^= bkey2;
    	key1 ^= bN;
    	i++;
    }
    return key1;
}
{% endcodeblock %} 

(我们可以看到,取个合适长度的用户名是很重要的,^_^)
最后，我们的keygen就完成了：  
![](http://ww4.sinaimg.cn/large/8061a41egw1egew4x3boij20ay05eaa9.jpg)  
注册的效果:  
![](/photo/keygen.png)  
PS:cpp已上传。
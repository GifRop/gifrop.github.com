title: Nisy cm_nf v0.4分析
date: 2014-06-06 10:06:47
tags: [CrackMe]
---
上午开了一上午的会，中午好不容易休息一下，想起Nisy的cm，运行点注册，卡住了，时间略久(sleep(10000))，遂分析了一下。<!-- more -->
直接祭出神器ida，载入脱壳后的文件。有个叫fn的函数出现在我们眼前。进去看看。有个和键盘输入相关的函数调用，  
查找引用发现：  
{% codeblock lang:cpp %}
SetWindowsHookExA(2, fn, hmod, GetCurrentThreadId())
2显然是WH_KEYBOARD，这里面做了手脚
{% endcodeblock %}
接着一串赋值和解密，研究了一下，是函数名GeDlgIemTextA：  
{% codeblock lang:x86asm %}
0040193D                 mov     al, 37h
0040193F                 mov     cl, 26h
00401941                 mov     [esp+18h+var_E], al
00401945                 mov     [esp+18h+var_9], al
00401949                 mov     [esp+18h+var_3], al
0040194D                 pop     edi
0040194E                 mov     [esp+14h+ProcName], 4
00401953                 mov     [esp+14h+var_F], cl
00401957                 mov     [esp+14h+var_D], 7
0040195C                 mov     [esp+14h+var_C], 2Fh
00401961                 mov     [esp+14h+var_B], 24h
00401966                 mov     [esp+14h+var_A], 0Ah
0040196B                 mov     [esp+14h+var_8], cl
0040196F                 mov     [esp+14h+var_7], 2Eh
00401974                 mov     [esp+14h+var_6], 17h
00401979                 mov     [esp+14h+var_5], cl
0040197D                 mov     [esp+14h+var_4], 3Bh
00401982                 mov     [esp+14h+var_2], 2
00401987                 mov     [esp+14h+var_1], 0
0040198C                 xor     eax, eax
0040198E                 pop     esi
0040198F
0040198F loc_40198F:                             
0040198F                 mov     cl, [esp+eax+10h+ProcName]
00401993                 xor     cl, 43h
00401996                 mov     [esp+eax+10h+ProcName], cl
0040199A                 inc     eax
0040199B                 cmp     eax, 0Fh
0040199E                 jb      short loc_40198F
004019A0                 push    offset LibFileName ; "user32.dll"
004019A5                 call    ds:GetModuleHandleA
004019AB                 lea     ecx, [esp+10h+ProcName]
004019AF                 push    ecx             ; lpProcName
004019B0                 push    eax             ; hModule
004019B1                 call    ds:GetProcAddress
004019B7                 mov     GeDlgIemTextA, eax
004019BC                 add     esp, 10h
{% endcodeblock %}
继续看钩子函数fn，GetKeyboardState是获取当前按键状态。ToAscii翻译到字符。  
{% codeblock lang:cpp %}
if ( Char[0] >= 'A' && Char[0] <= 'Z' ) //判断字符是否是'A' - 'Z',是则调用sub_401770函数。也就是说，
        sub_401770(); //只有在输入大写字母的时候会调用这个函数。
{% endcodeblock %}
sub_401770函数关键代码:  
{% codeblock lang:x86asm %}
004017EB                 cmp     [ebp+var_108], 8 ; 第一位到第八位
004017F2                 jnb     short loc_401812
004017F4                 mov     ecx, [ebp+var_108]
004017FA                 movsx   edx, [ebp+ecx+var_104]
00401802                 mov     eax, [ebp+var_10C]
00401808                 add     eax, edx        ; 累加
0040180A                 mov     [ebp+var_10C], eax
00401810                 jmp     short loc_4017DC
00401812 ; ---------------------------------------------------------------------------
00401812
00401812 loc_401812:                             
00401812                 cmp     [ebp+var_10C],272
0040181C                 jnz     short loc_401848
0040181E                 movsx   ecx, [ebp+var_104] ; key的第一位
00401825                 cmp     ecx, 4E
00401828                 jnz     short loc_401848
0040182A                 lea     edx, [ebp+var_104]
00401830                 push    edx
00401831                 call    sub_401690
00401836                 add     esp, 4
00401839                 lea     eax, [ebp+var_104]
0040183F                 push    eax
00401840                 call    sub_4015F0
{% endcodeblock %}
从这个函数中我们能看到几个关键的函数和第一个字符为0x4E 即'N',前八位的和为0x272。  
进入4015F0和401690函数:  
{% codeblock lang:x86asm %}
004015F0    83EC 0C         sub esp,0xC
004015F3    C64424 00 17    mov byte ptr ss:[esp],0x17
004015F8    C64424 01 2B    mov byte ptr ss:[esp+0x1],0x2B
004015FD    C64424 02 22    mov byte ptr ss:[esp+0x2],0x22
00401602    C64468 4F 7B    mov byte ptr ds:[eax+ebp\*2+0x4F],0x7B
00401607    8A08            mov cl,byte ptr ds:[eax]
00401609    68 48288A08     push 0x88A2848
0040160E    68 497C8A08     push 0x88A7C49
00401613    68 4A2F8A08     push 0x88A2F4A
00401618    68 4B408A08     push 0x88A404B
0040161D    68 44648A08     push 0x88A6444
00401622    68 452E8A08     push 0x88A2E45
00401627    68 464C7F8C     push 0x8C7F4C46
0040162C    C600 48         mov byte ptr ds:[eax],0x48
0040162F    4C              dec esp
00401630    CC              int3
00401631    BD 0FC40048     mov ebp,0x4800C40F
00401636    4C              dec esp
00401637    0C CF           or al,0xCF
00401639    B4 46           mov ah,0x46
0040163B    30A3 C74198BC   xor byte ptr ds:[ebx+0xBC9841C7],ah
00401641    0E              push cs
00401642    4C              dec esp
00401643    C108 68         ror dword ptr ds:[eax],0x68
00401646    4C              dec esp
00401647    1C 24           sbb al,0x24
00401649    A1 4F4C4C1D     mov eax,dword ptr ds:[0x1D4C4C4F]
{% endcodeblock %}
401690中出现了几个关键的函数004016B7  
{% codeblock lang:x86asm %}
004016B6    FF15 80224200   call dword ptr ds:[0x422280]             ; kernel32.VirtualProtect
004016BD    8B1D 84224200   mov ebx,dword ptr ds:[0x422284]          ; kernel32.WriteProcessMemory
004016C3    8B2D 88224200   mov ebp,dword ptr ds:[0x422288]          ; kernel32.GetCurrentProcess
{% endcodeblock %}
这个组合很容易的就想到了，代码需要解码：  
{% codeblock lang:x86asm %}
004016D6                 sub     edi, offset sub_4015F0
004016DC
004016DC loc_4016DC:     
004016DC                 mov     ecx, [esp+18h+arg_0]
004016E0                 mov     al, [esi]
004016E2                 mov     dl, [ecx+1]            ; key[1]
004016E5                 xor     al, dl                        ; xor code,key[1]
{% endcodeblock %}
可以很明显的看出key[1]是用于解码的。  
仔细看下00401602附近的代码:  
{% codeblock lang:x86asm %}
004015FD    C64424 02 22    mov byte ptr ss:[esp+0x2],0x22 
00401602    C64468 4F 7B    mov byte ptr ds:[eax+ebp*2+0x4F],0x7B   
00401607    8A08            mov cl,byte ptr ds:[eax]
00401609    68 48288A08     push 0x88A2848
{% endcodeblock %}
在4015FD是mov byte [esp+0x2],0x22 ，下面代码也是mov byte, 那应该后面会有[esp+0x3]之类  
的，也就是说 C64424 02 22 和 C64468 4F 7B   401602 的 0x68应该是0x24.  
0x68 xor 0x24 = 0x4C. 即'L',测试下:  
{% codeblock lang:x86asm %}
004015F0    83EC 0C         sub esp,0xC
004015F3    C64424 00 17    mov byte ptr ss:[esp],0x17
004015F8    C64424 01 2B    mov byte ptr ss:[esp+0x1],0x2B
004015FD    C64424 02 22    mov byte ptr ss:[esp+0x2],0x22
00401602    C64424 03 37    mov byte ptr ss:[esp+0x3],0x37
00401607    C64424 04 64    mov byte ptr ss:[esp+0x4],0x64
0040160C    C64424 05 30    mov byte ptr ss:[esp+0x5],0x30
00401611    C64424 06 63    mov byte ptr ss:[esp+0x6],0x63
00401616    C64424 07 0C    mov byte ptr ss:[esp+0x7],0xC
0040161B    C64424 08 28    mov byte ptr ss:[esp+0x8],0x28
00401620    C64424 09 62    mov byte ptr ss:[esp+0x9],0x62
00401625    C64424 0A 00    mov byte ptr ss:[esp+0xA],0x0               ；解密成:That's Ok!
0040162A    33C0            xor eax,eax
0040162C    8A4C04 00       mov cl,byte ptr ss:[esp+eax]
00401630    80F1 43         xor cl,0x43
00401633    884C04 00       mov byte ptr ss:[esp+eax],cl
00401637    40              inc eax
00401638    83F8 0A         cmp eax,0xA
0040163B  ^ 7C EF           jl short cm_nf_v0.0040162C
0040163D    8B0D D4F04200   mov ecx,dword ptr ds:[0x42F0D4]
00401643    8D4424 00       lea eax,dword ptr ss:[esp]
00401647    50              push eax
00401648    68 ED030000     push 0x3ED
0040164D    51              push ecx
0040164E    FF15 2C244200   call dword ptr ds:[0x42242C]             ; user32.SetDlgItemTextA
00401654    8B4424 10       mov eax,dword ptr ss:[esp+0x10]
00401658    8078 09 58      cmp byte ptr ds:[eax+0x9] ,0x58           ;key[9]
0040165C    75 26           jnz short cm_nf_v0.00401684
0040165E    8078 0A 54      cmp byte ptr ds:[eax+0xA], 0x54          ;key[10]
00401662    75 20           jnz short cm_nf_v0.00401684
00401664    8D5424 10       lea edx,dword ptr ss:[esp+0x10]
00401668    C74424 10 00000>mov dword ptr ss:[esp+0x10],0x0
00401670    52              push edx
00401671    6A 00           push 0x0
00401673    6A 00           push 0x0
00401675    68 60154000     push cm_nf_v0.00401560
0040167A    6A 00           push 0x0
0040167C    6A 00           push 0x0
0040167E    FF15 90224200   call dword ptr ds:[0x422290]             ; kernel32.CreateThread
00401684    83C4 0C         add esp,0xC
00401687    C3              retn
{% endcodeblock %}
像那么回事了，That's Ok! 了。
![](http://ww1.sinaimg.cn/large/8061a41egw1eh469ca6xgj206j04e74c.jpg)

但是没弹框。。。看看401675的这个线程函数，也被加密了。在401690中一起解密的，用的是key[2]  
{% codeblock lang:x86asm %}
00401560    8B0D D4F04200   mov ecx,dword ptr ds:[0x42F0D4]
00401566    81EC 00010000   sub esp,0x100
0040156C    8D4424 50       lea eax,dword ptr ss:[esp+0x50]
00401570    38AF 50505000   cmp byte ptr ds:[edi+0x505050],ch
00401576    38BA 53505001   cmp byte ptr ds:[edx+0x1505053],bh
0040157C    AF              scas dword ptr es:[edi]
0040157D    45              inc ebp
0040157E    60              pushad
0040157F    74 12           je short cm_nf_v0.00401593
00401581    50              push eax
00401582    F1              int1
00401583    80A0 1250D590 2>and byte ptr ds:[eax+0x90D55012],0x24
0040158A    085F EE         or byte ptr ds:[edi-0x12],bl
0040158D    04 74           add al,0x74
0040158F    57              push edi
00401590    5F              pop edi
00401591    EE              out dx,al
00401592    14 74           adc al,0x74
00401594    53              push ebx
00401595    53              push ebx
00401596    80D1 AA         adc cl,0xAA
00401599    D4 50           aam 0x50
0040159B    50              push eax
0040159C    50              push eax
0040159D    25 1407DD2C     and eax,0x2CDD0714
004015A2    74 54           je short cm_nf_v0.004015F8
{% endcodeblock %}
依瓢画葫芦，我们得到的key[2] = 'P'.  
解出来的代码:  
{% codeblock lang:x86asm %}
00401560    8B0D D4F04200   mov ecx,dword ptr ds:[0x42F0D4]
00401566    81EC 00010000   sub esp,0x100
0040156C    8D4424 00       lea eax,dword ptr ss:[esp]
00401570    68 FF000000     push 0xFF
00401575    50              push eax
00401576    68 EA030000     push 0x3EA
0040157B    51              push ecx
0040157C    FF15 30244200   call dword ptr ds:[0x422430] ; user32.GetDlgItemTextA
00401582    A1 D0F04200     mov eax,dword ptr ds:[0x42F0D0]
00401587    85C0            test eax,eax
00401589    74 58           je short cm_nf_v0.004015E3
0040158B    0FBE5424 07     movsx edx,byte ptr ss:[esp+0x7]   ;key[7]
00401590    0FBE4424 03     movsx eax,byte ptr ss:[esp+0x3]   ;key[3]
00401595    03D0            add edx,eax
00401597    81FA 84000000   cmp edx,0x84
0040159D    75 44           jnz short cm_nf_v0.004015E3
0040159F    57              push edi
004015A0    8D7C24 04       lea edi,dword ptr ss:[esp+0x4]
004015A4    83C9 FF         or ecx,0xFFFFFFFF
004015A7    33C0            xor eax,eax
004015A9    F2:AE           repne scas byte ptr es:[edi]
004015AB    F7D1            not ecx
004015AD    49              dec ecx
004015AE    5F              pop edi
004015AF    83F9 0C         cmp ecx,0xC
004015B2    75 2F           jnz short cm_nf_v0.004015E3
004015B4    807C24 0B 5A    cmp byte ptr ss:[esp+0xB],0x5A  ;key[B]
004015B9    75 28           jnz short cm_nf_v0.004015E3
004015BB    807C24 08 41    cmp byte ptr ss:[esp+0x8],0x41  ;key[8]
004015C0    7D 21           jge short cm_nf_v0.004015E3
004015C2    6A 00           push 0x0
004015C4    68 CCB04200     push cm_nf_v0.0042B0CC ; ASCII "Great!"
004015C9    68 C0B04200     push cm_nf_v0.0042B0C0 ; ASCII "Good Job!"
004015CE    6A 00           push 0x0
004015D0    FF15 34244200   call dword ptr ds:[0x422434] ; user32.MessageBoxA
004015D6    8B0D D0F04200   mov ecx,dword ptr ds:[0x42F0D0]
004015DC    51              push ecx
004015DD    FF15 38244200   call dword ptr ds:[0x422438] ; user32.UnhookWindowsHookEx
004015E3    33C0            xor eax,eax
004015E5    81C4 00010000   add esp,0x100
004015EB    C2 0400         retn 0x4
{% endcodeblock %}
从这个代码会看出key的其他几位,key[12] = 0x5A ,key[8] 小于0x41 ，key[3]+key[7] = 0x84 ,中间有一串常用代码，计算串长度的:
{% codeblock lang:x86asm %}
004015A4    83C9 FF         or ecx,0xFFFFFFFF
004015A7    33C0            xor eax,eax
004015A9    F2:AE           repne scas byte ptr es:[edi]
004015AB    F7D1            not ecx
004015AD    49              dec ecx
004015AE    5F              pop edi
{% endcodeblock %}
说明key的长度必须为0xC.  
现在我们得到的信息有:  
{% codeblock lang:x86asm %}
长度为12
1:N                       ;cmp ecx,0x4E
2:L
4:B                       ；+ 8  = 0x84拆成两个B
5:V
6:V
7:V
8:B                       ；前8位和为0x272h，可以得到567为VVV
9:0                       ;小于0x41 ,我用了0
10:X                      ;cmp byte ptr ds:[eax+0xA],0x58
11:T                      ;cmp byte ptr ds:[eax+0x9]
12:Z                      ;0x5A  cmp byte ptr ss:[esp+0xB],0x5A
{% endcodeblock %}
得到了一组key:NLPBVVVB0XTZ .
![](http://ww2.sinaimg.cn/large/8061a41egw1eh469sf8dqj207x05w0su.jpg)
Win7上测试也OK，but，为什么其他人不行，奇怪。不搞了。  
有个很有意思的地方在于:输入注册码就好了.不要点击注册. 或者直接复制注册码进去把最后一位重新输入.因为只有大写才校验.

搞完之后，看到喜欢很久的女生结婚了，就很不高兴了。{:cry:}

PS：乱搞一通，大牛们请直接路过。


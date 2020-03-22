---
title: 武汉国家安全周线下Writeup
date: 2017-08-01 15:32:00
tags:
---

## 逆向体验

这次的逆向题都是老套路, 并没有很亮的保护技术. 所以也就有机会让我这个菜鸡也能去分析清楚. 所以也就有了以下的writeup. 这里贴出此处逆向题的题目:[传送门](http://pan.baidu.com/s/1kU6CaFl)

## Re200

程序没有加壳, 也没有任何反跟踪技术. 就是赤裸裸的代码逆向. 这里贴出main函数

``` c++
int __cdecl main(int argc, const char **argv, const char **envp)
{
  if ( argc != 2 || strlen(argv[1]) > 0x18 )
  {
    result = 0;
  }
  else
  {
    memcpy(&dword_404410, argv[1], strlen(argv[1]));
    v3 = sub_401000();
    Handles = 0;
    v13 = 0;
    v14 = 0;
    v15 = 0;
    v16 = 0;
    v17 = 0;
    InitializeCriticalSection(&CriticalSection);
    v4 = 0;
    while ( 1 )
    {
      v5 = CreateThread(0, 0, StartAddress, (LPVOID)v4, 0, &ThreadId);
      *(&Handles + v4) = v5;
      if ( !v5 )
        break;
      if ( ++v4 >= 6 )
      {
        WaitForMultipleObjects(6u, &Handles, 1, 0xFFFFFFFF);
        v6 = 0;
        v7 = 0;
        do
        {
          v8 = v7 ^ dword_404428[v7];
          ++v7;
          v6 ^= v8;
        }
        while ( v7 < 24 );
        qmemcpy(v19, byte_403160, 0x18u);
        v9 = 0;
        v19[24] = byte_403160[24];
        v20 = 0;
        v21 = 0;
        v22 = 0;
        do
        {
          v10 = v19[v9] ^ dword_404428[v9];
          v9 += 6;
          *((_BYTE *)&v17 + v9 + 2) = v6 ^ v3 ^ v10;
          *((_BYTE *)&v17 + v9 + 3) ^= v6 ^ v3 ^ byte_404423[v9];
          *((_BYTE *)&ThreadId + v9) ^= v6 ^ v3 ^ byte_404424[v9];
          *((_BYTE *)&ThreadId + v9 + 1) ^= v6 ^ (unsigned __int8)(v3 ^ byte_404425[v9]);
          *((_BYTE *)&ThreadId + v9 + 2) ^= v6 ^ v3 ^ byte_404426[v9];
          *((_BYTE *)&ThreadId + v9 + 3) ^= v6 ^ v3 ^ byte_404427[v9];
        }
        while ( v9 < 24 );
        printf("The hint: %sn", v19);
        break;
      }
    }
    result = 0;
  }
  return result;
}
```

可以看出. 在第一个分支这里有一个简单的判断. 这实在太简单不过了. 就是一个参数的限制. 但是在和其他选手交流的时候还是会发现有不少人会不知道这个地方, 以为是程序的反调试. 但其实只是不够细致而且逆向的经验太少所致. 

``` c++
if ( argc != 2 || strlen(argv[1]) > 0x18 )
```

这里的约束条件也很简单. 就是要求传入1个参数. 并要求传入参数的长度不大于0x18. (事实上, 这里的正确参数的长度是0x18. 我们在做这些题的时候可以从这些判断条件里得到一些有意思的信息). 

之后memcpy对缓冲区进行初始化. 在V3这里有一个函数. 我们点进去看就可以知道是一个生成随机数的函数. (这里需要一定的逆向经验能识别出这个是一个生成随机数的函数, 事实上这里的这个函数也非常好识别. )当我们遇到随机数时, 不必慌张. 因为一个真正保护好的随机数是不可能逆出来的. 而如果必须要逆的话, 那只能说这个随机数是存在漏洞的. 事实上我们可以查看这个随机数生成函数. 可以发现函数生成的随机数赋值给了v3. 而v3的数据类型是char. 所以我们这里就可以放心了. 我们可以使用爆破的方法, 穷举256种可能得到正确结果. 

其后是一个while(1)循环. 循环终止条件是进入到if ( ++v4 >= 6 )分支里然后跳出循环. 这里就有一个关键的地方

``` c++
v5 = CreateThread(0, 0, StartAddress, (LPVOID)v4, 0, &ThreadId);
```

通过分析可以知道这里会通过循环而创建6个线程. 每个线程内部会从CreateThread的参数StartAddress所指定的地方开始执行. 这里我们需要进入StartAdress里去分析一下. (这里的分析过程可能不是很明显. )当看到那三个熟悉的函数和熟悉的参数的时候, 就可以很能确认是一个MD5生成的函数. 而在一开头有一段处理.  就是只取了输入参数的4位数进行MD5生成. 这时我们使用OD进行动态调. 就可以很直观的知道. 这个函数是将输入分为了6段, 每段4位数, 4位数参与生成MD5值, 也就是整个循环下来生成了6段MD5值. 这里尤其有一段需要特别注意. 在生成MD5之后, 会进行一个比较. 将生成的MD5与已有的MD5值进行比较. 这就很能体现出问题了. 由于是4位数生成的MD5. 所以我们使用MD5Crack就可以轻松破解得到整个的程序参数输入应该是

``` bash
1y8u 93B9 0Mkh 56bv F7um 5o9z
```

接下来就简单多了. 由于这里线程结束的时间各个不确定. 所以我们在后面的分析中会发现. 上面我们解出的MD5是依旧需要分6组, 然后我们需要爆破人工进行全排列参与下面的运算. 

这里主要是2个运算

``` c++
do
   {
      v8 = v7 ^ dword_404428[v7];
      ++v7
      v6 ^= v8;
   }
```

这里是生成v6参与下面的异或运算. 这里也很好实现. 

``` c++
do
        {
          v10 = v19[v9] ^ dword_404428[v9];
          v9 += 6;
          *((_BYTE *)&v17 + v9 + 2) = v6 ^ v3 ^ v10;
          *((_BYTE *)&v17 + v9 + 3) ^= v6 ^ v3 ^ byte_404423[v9];
          *((_BYTE *)&ThreadId + v9) ^= v6 ^ v3 ^ byte_404424[v9];
          *((_BYTE *)&ThreadId + v9 + 1) ^= v6 ^ (unsigned __int8)(v3 ^ byte_404425[v9]);
          *((_BYTE *)&ThreadId + v9 + 2) ^= v6 ^ v3 ^ byte_404426[v9];
          *((_BYTE *)&ThreadId + v9 + 3) ^= v6 ^ v3 ^ byte_404427[v9];
        }
```

这里就是会进行一个比较大的异或运算了. 这里也可以很轻松用代码复现出来. 这里关键是要实现v6的全排列和v3的爆破. 

之后与最后的密文进行比较. 得到hint. hint的意思是flag是输入的md5值. 这道题就这么分析完毕了. 

## Tryagain

这道题是我做的第一个Linux的逆向. 而且也是第一个x64的逆向程序. 一开始的时候会因为各种胆怯所以没什么太多的思路. 到晚上的时候一搞发现一切都是如此简单明白. 

``` c++
v5 = *MK_FP(__FS__, 40LL);
__printf_chk(1LL, "Input flag:", a3);
sub_4010B0(&v4);
if ( (unsigned __int8)sub_400DA0(&v4) )
  puts("Right.");
else
  puts("Wrong.");
```

这里值得学习的是这里第一句话开启了栈保护. 然后就是两个关键函数需要识别. 这里第一个函数很好弄啦. 其实是一个scanf函数. 但是很奇怪IDA为什么没有识别出来. 等以后搞了IDC脚本再想办法怎样能增强IDA的函数识别能力了. 

接下来关键的就是sub_400DA0这个函数了. 点进去你会发现函数量很大而且有的函数十分复杂. 这里我们来一个个进行分析. 

首先就是一个判断

``` c++
do
    {
      if ( (unsigned __int8)((*((_BYTE *)src + v5) & 0xDF) - 65) > 0x19u )
        return 0LL;
      ++v5;
    }while(v5!=4)
```

这里很好理解. 就是输入的参数和0xDF进行与操作之后减去65的值要不大于0x19. 所以我们的输入必须满足这个条件才能继续进入下面的验证. 事实上这是一种非常巧妙的筛选出英文字母的方法. 值得学习!总结一下就是说输入的前4位必须是英文字母. 

随后发现这个三段的函数, 明显取输入的前4位生成MD5

``` c++
sub_401560(&v32, src);
sub_401E20(&v32, &v22, 4LL);
sub_401F30(&v32, &v30);
```

再继续分析, 知生成的MD5值又作为参数传入了下面的一个关键的函数sub_4013F0

``` c++
do
      {
        v10 = (__int64)v6 + v9;
        v11 = (__int64)v7 + v9;
        v9 += 8LL;
        sub_4013F0(v11, (__int64)&v29, v10);
      }
```

可能这里有点绕. 但是你如果边动手边理解的话, 肯定是很轻松搞定的. 这里我们就需要逆向的经验啦. 识别出这个函数的内容. 事实上这里是一个DES加密函数. 没有经验的初学者如果是硬刚的话, 就真的实会刚出硬伤的. 

下面又是一个关键函数

``` c++
sub_400A00(v6, v13, v12);
```

这里也需要经验识别. 才能知道原来这是进行Base64的加密. 然后继续到下方

``` c++
for ( i = 117;
            ;
            i = *(&v23 + v16 - 6 * ((unsigned __int64)(0x0AAAAAAAAAAAAAAABLL * (unsigned __int128)v16 >> 64) >> 2)) )
      {
        *((_BYTE *)v15 + v16) = *((_BYTE *)v13 + v16) ^ i;
        if ( v14 <= (signed int)++v16 )
          break;
      }
```

这里是一个简单的异或运算. 我们将这个异或运算逆回去就可以了. (友情小提示, 这里搜索0x0AAAAAAAAAAAAAAABLL还可以知道这个异或运算在XDCTF2015中有过类似实现)

之后的运算就很简单了. 这样的一个程序也就这样逆向完毕了. 这样的一个Linux的x64程序也就这样搞定了. 

## Hurryup

gui打开jar包，找到getflag那个函数 能看到密钥是个数字。密钥本来的生成算法是以apk本身签名为key解一个aes的加密

直接暴力这个数字

``` c++
from Crypto.Cipher import AES
for i in xrange(0xffffffff):
  if i % 10000 == 0: print i
  enc = AES.new((str(i)+'x00'*16)[:16], mode= AES.MODE_CBC, IV= "0102030405060708")
  buf = enc.decrypt("tQJWYl+8C4cO2jwq782P5qc/9FXVF6cedVTtozSnyFk=".decode('base64'))

if 'flag' in buf:
  print buf
```

得到密钥9953，再解密，得到flag
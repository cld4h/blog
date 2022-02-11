---
title: lrlr
date: 2019-09-17T19:11:45+08:00
tags: ["lrlr","bbencode"]
draft: false
---

题目来源

[2019 字节跳动 bytectf crypto lrlr](https://adworld.xctf.org.cn/media/uploads/task/207689abfdd54b5382ef06739ec1e318.zip)

参考

[Team: W&M](https://xz.aliyun.com/t/6324#toc-25)

[Jarvis oj crypto](https://196011564.github.io/2018/12/11/CTF-JarvisOJ-Crypto-[61dctf]bbencode/index.html)

## 先全文摘抄大佬的 Write Up

通过old的1000组可以预测python随机数，
https://github.com/tna0y/Python-random-module-cracker

一共2轮aes加密，既然密钥可以预测出来，自然就能解密得到clist。

![lrlr.png](lrlr.png)

```py
from Crypto.Util.number import getPrime, bytes_to_long, long_to_bytes
from Crypto.Cipher import AES
import random
from randcrack import RandCrack

rc = RandCrack()
with open('old') as file:
    old = [int(i) for i in file.read().strip().split('\n')]

index = 0
for i in range(index,624):
    rc.submit(old[index])
    index+=1

for i in range(1000-624):
    assert(old[index]==rc.predict_getrandbits(32))
    index+=1


with open('cl') as file:
    nlist = [eval(i) for i in file.read().strip().split('\n')]

with open('new') as file:
    clist=[i.decode('hex') for i in file.read().strip().split('\n')]

key1=[]
key2=[]
key3=[]
for i in range(24):
    key1.append(rc.predict_getrandbits(128))
for i in range(24):
    key2.append(rc.predict_getrandbits(128))
for i in range(24):
    key3.append(rc.predict_getrandbits(128))


tmp1=[]
for i in range(24):
    handle = AES.new(long_to_bytes(key3[i]), AES.MODE_CBC, "\x00"*16)
    tmpstate=handle.decrypt(clist[i])
    tmp1.append(tmpstate)

tmp2=[]
for i in range(24):
    handle = AES.new(long_to_bytes(key2[i]), AES.MODE_CBC, "\x00"*16)
    tmpstate=handle.decrypt(tmp1[i])
    tmp2.append(tmpstate)

# tmp3=[]
# for i in range(24):
#     handle = AES.new(long_to_bytes(key1[i]), AES.MODE_CBC, "\x00"*16)
#     tmpstate=handle.decrypt(tmp2[i])
#     tmp3.append(tmpstate)

c=[]
for i in tmp2:
    c.append(bytes_to_long(i))

for i in range(17):
    print 'n = %d'%nlist[i]
    print 'e = 17'
    print 'c = %d'%c[i]
    print '\n'

```

然后有了24组n,c；e=17。随便选择17组去广播攻击

![rsa_solve](rsa_solve.png)

最后一步类似jarvis oj上的bbencode原题，循环编码，判断flag开头的字符串

```py
def bbencode(n):
    a = 0
    for i in bin(n)[2:]:
        a = a << 1
        if (int(i)):
            a = a ^ n
        if a >> 256:
            a = a ^ 0x10000000000000000000000000000000000000000000000000000000000000223L
    return a

result = 61406796444626535559771097418338494728649815464609781204026855332620301752444
for i in range(10000):
    result = bbencode(result)
    if("666c6167" == str(hex(result))[2:10]):
        print i
        print hex(result)[2:-1].decode('hex')

```

## 下面写下细节

通过 `oldtest()` 中 `random.getrandbits()` 函数生成的随机数可以用 `randcrack` 库推断到之后 `lrandout()` 中调用 `stateconvert()` 和 `gen_new_states()` 时生成的128位随机数。

此随机数直接作为AES-128加密的密钥对 `clist[]` 进行加密，IV值为128位0 。

加密后的结果Append到 `states[]` 后，用的时候再通过 `stateconvert` 进行第二次AES加密(初始时(也就是 `newtest` 中第一个循环) `states[]` 数组中为长度为24的 `clist[]` 且 `stateselse == 24` 所以直接调用 `stateconvert` 一次加密得到输出)

我们现在已知二次AES加密后的结果；已知 `nlist[]`；需要先由随机数生成规律推断出作为密钥的随机数，通过AES解密得到 `clist[]`；然后看 `gen_states` 函数发现 `clist[]` 中的元素是同一个消息(`init_state`) 以 $e=17$ 为指数模不同的 $n$ 值所得。此处可以进行低指数广播攻击。

### 低指数广播攻击

[参考 一叶飘零 的 Crypto-RSA-公钥攻击小结](https://xz.aliyun.com/t/2731#toc-10)
[参考 ashutosh1206 的 Hastad's Broadcast Attack](https://github.com/ashutosh1206/Crypton/tree/master/RSA-encryption/Attack-Hastad-Broadcast)
所谓“低指数”，指的是指数 $e <= \text{广播成员数量(加密的消息数量)}$
<!--假设 Alice 想要给 3 个不同的人发送相同的消息 $M$, 对每个人使用同样的公钥指数 $e$ 和不同的 $n$-->

#### 中国剩余定理

对于下面的一元线性同余方程组：

{{< rawhtml >}}
$$
\begin{equation*}
\left\lbrace
\begin{aligned}
	x \equiv & a_1 \bmod m_{1}\\
	x \equiv & a_2 \bmod m_{2}\\
	& \vdots\\
	x \equiv & a_n \bmod m_{n}\\
\end{aligned}
\right.
\end{equation*}
$$
{{< /rawhtml >}}

假设$m_1,m_2,\ldots,m_n$中任意两数互质，则对任意整数$a_1,a_2,\ldots,a_n$，上述方程组有解，通解的构造方法如下：

1. 设$M = m_1\times m_2\times \cdots \times m_{n} = \prod\limits_{i=1}^{n} m_{i} $ 是整数$m_1,m_2,\ldots,m_{n}$ 的乘积
2. 设$M_{i} = M / m_{i}, \forall i \in \{1,2,\ldots,n\}$
3. 设$t_{i} = M_{i}^{-1}$ 为$M_{i}$ 模$m_{i}$ 的数论倒数
4. 方程组的通解形式为：
$$
x = a_1t_1M_1+a_2t_2M_2+\cdots+a_{n}t_{n}M_{n} = kM+\sum_{i=1}^{n} a_{i}t_{i}M_{i},\qquad \forall k \in \mathbb{Z}.
$$
在模$M$ 的意义下方程组只有一个解：
$$
x=\sum_{i=1}^{n} a_{i}t_{i}M_{i}
$$

因此，对于本题

{{< rawhtml >}}
$$
\begin{equation*}
\left\lbrace
\begin{aligned}
m^{17} \equiv & c_1 \bmod n_1\\
m^{17} \equiv & c_2 \bmod n_2\\
& \vdots\\
m^{17} \equiv & c_{17} \bmod n_{17}\\
\end{aligned}
\right.
\end{equation*}
$$
{{< /rawhtml >}}

(要求$n_1,n_2,\ldots,n_{17}$两两互素，若不满足两两互素，可进行[共模攻击](https://github.com/ashutosh1206/Crypton/tree/master/RSA-encryption/Attack-Common-Prime))有

$$
m^{17} = \sum_{i=1}^{17} c_{i}N_{i}N_{i}^{-1} mod N
$$

其中，

$
N = n_1\times n_2\times \cdots \times n_{17} = \prod\limits_{i=1}^{17} n_{i}
$；

$
N_{i} = N / n_{i}
$；

$
N_{i}^{-1} \cdot N_{i} \equiv 1 \bmod n_{i}
$

因为$m < n_{i} \quad \forall i \in \{1,2,\ldots,17\}$，所以$ m^{17} < N = \prod\limits_{i=1}^{17} n_{i}$ 直接开方即可。

#### hastad_unpadded.py

[参考](https://github.com/ashutosh1206/Crypton/blob/master/RSA-encryption/Attack-Hastad-Broadcast/hastad_unpadded.py)
只需关注 `crt` 函数和 `hastad_unpadded`  函数即可

```py
#!/usr/bin/env python2.7
from Crypto.Util.number import GCD, bytes_to_long, long_to_bytes
import gmpy2

def crt(list_a, list_m):
    """
    Reference: https://crypto.stanford.edu/pbc/notes/numbertheory/crt.html
    Returns the output after computing Chinese Remainder Theorem on
    x = a_1 mod m_1
    x = a_2 mod m_2
    ...
    x = a_n mod m_n
    input parameter list_a = [a_1, a_2, ..., a_n]
    input parameter list_m = [m_1, m_2, ..., m_n]
    Returns -1 if the operation is unsuccessful due to some exceptions
    """
    try:
        assert len(list_a) == len(list_m)
    except:
        print "[+] Length of list_a should be equal to length of list_m"
        return -1
    for i in range(len(list_m)):
        for j in range(len(list_m)):
            if GCD(list_m[i], list_m[j])!= 1 and i!=j:
                print "[+] Moduli should be pairwise co-prime"
                return -1
    M = 1
    for i in list_m:
        M *= i
    list_b = [M/i for i in list_m]
    assert len(list_b) == len(list_m)
    try:
        list_b_inv = [int(gmpy2.invert(list_b[i], list_m[i])) for i in range(len(list_m))]
    except:
        print "[+] Encountered an unusual error while calculating inverse using gmpy2.invert()"
        return -1
    x = 0
    for i in range(len(list_m)):
        x += list_a[i]*list_b[i]*list_b_inv[i]
    return x % M


def test_crt():
    """
    Checking the validity and consistency of CRT function
    """
    list_a = [[2, 3], [1, 2, 3, 4], [6, 4]]
    list_m = [[5, 7], [5, 7, 9, 11], [7, 8]]
    soln_list = [17, 1731, 20]
    try:
        for i in range(len(list_a)):
            assert crt(list_a[i], list_m[i]) == soln_list[i]
    except:
        print "[+] CRT function broken. Check the function again!"


def hastad_unpadded(ct_list, mod_list, e):
    """
    Implementing Hastad's Broadcast Attack
    """
    m_expo = crt(ct_list, mod_list)
    if m_expo != -1:
        eth_root = gmpy2.iroot(m_expo, e)
        if eth_root[1] == False:
            print "[+] Cannot calculate e'th root!"
            return -1
        elif eth_root[1] == True:
	    # 注意，eth_root 为元组(mpz对象,布尔值)，需用eth_root[0]提取mpz对象
	    # 也可用gmpy2.digits(eth_root[0])
	    # 以字符串形式返回正确的结果
	    # https://gmpy2.readthedocs.io/en/latest/mpz.html#mpz-functions
            return long_to_bytes(eth_root[0])
    else:
        print "[+] Cannot calculate CRT"
        return -1

test_crt()
```

对于此题，我们传入 `hastad_unpadded(ct_list, mod_list, e)` 函数的三个参数分别为

* `ct_list = clist`
* `mod_list = nlist`
* `e = 17`

(`clist` 和 `nlist` 通过前面大佬给的WP中的代码即可得到，用到了 `randcrack` 库)

通过上述代码我们成功得到了 `init_state` 的值

![init_state](init_state.png)

与前面大佬给的一致.

### bbencode

获得 `init_state` 后就与 Jarvis OJ 上的题目完全一致了
主要参考 [Jarvis oj crypto](https://196011564.github.io/2018/12/11/CTF-JarvisOJ-Crypto-[61dctf]bbencode/index.html)

大致的意思是将 `bbencode` 的输出再传给 `bbencode` ，反复作用，每一次都查看一下结果中前四个字节是不是“flag” ，如下：
```py
    if("666c6167" == str(hex(result))[2:10]):
```
"666c6167" 正好是 flag 的ascii码

![flag](flag.png)

由于for循环执行10000次，每2695次迭代会得到一次flag，所以会输出三次flag


**1\.linux内核调试工具crash并不能直接显示函数参数，而这个对调试又非常重要**
 下面是工作中一个实际的问题，我们的进程hang在如下一个内核栈中了，通过栈回溯可知是打开了一个nfs3的网盘文件或者目录，已知客户机器的NAS盘不可访问了，只要访问就会hang住，但我们的进程理论上是不会访问该NAS盘的，那么如何知道open打开的是什么文件呢，这时候就迫切的需要知道do\_sys\_open的filename参数了(此时多么希望VS， gdb能够出手相救)。
 但在x64位linux系统中，前6个参数使用的是rdi, rsi, rdx, rcx, r8, r9寄存器来传递，超出的才会用栈来传递，filename是第2个参数，会用rsi来传递，这下就GG了，函数经过这么多层调用，到当前栈帧的时候rsi早就是n手货了，而且rsi又不是保留寄存器，下层函数不会给他提供VIP待遇做保存，绝望，绝望，就是这么的绝望。
![](https://oss-ata.alibaba.com/article/2024/09/86b94458-986f-4a19-946b-7f337769e04d.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
 

![](https://oss-ata.alibaba.com/article/2024/09/9036b4d7-3ab0-4917-82f1-0e5ab624d8d0.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
 

 

**2\.希望之光：通过函数参数被局部变量缓存获取**

#### 1\> 上帝关上了一扇门，必然会为你打开另一扇窗


 虽然寄存器不会被缓存，但局部变量会啊，局部变量是用栈保存的，函数调用栈未返回之前，栈都会一直给你当宝一样存着，so换个思路，假设，我们假设这个函数参数filename传递给了某个路人局部变量，那么我们找到这个路人，不就相当于找到了filename么。
 
#### 2\> 思路打开，上帝就立马给你送来了这个路人。


 我们看do\_sys\_open，开头就把filename丢给了tmp这个路人，因此我们只要去栈上把tmp逮捕归案就能收工下班。
![](https://oss-ata.alibaba.com/article/2024/09/3c10cc6c-3f2d-4faf-a08f-12d015238cde.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
 

#### 3\> 逮捕方案1："\-FF"


 crash的bt命令提供了"\-FF"参数，宣称可以打印局部变量。但是吗，往往宣称是一回事，实际又是另一回事。"bt \-FF"一敲，甭说放大镜，用电子显微镜也找不到tmp在哪。
![](https://oss-ata.alibaba.com/article/2024/09/de6c8129-8242-4ae1-889b-012e0bd5eb4f.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
![](https://oss-ata.alibaba.com/article/2024/09/1357183a-0708-4740-88b2-6a4673ae540b.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
#### 4\> 逮捕方案2：速请汇编大仙


既然tmp是个局部变量，又会作为第二个参数传递给do\_filp\_open，那么我们找到汇编大仙call do\_filp\_open的地方，就能找到tmp了。
 来吧，dis照妖镜，do\_sys\_open原形毕露如下，tmp是调用getname返回的，返回值rax立马给了r14(汇编里返回值都是用rax传递)，不妙不妙，还是不给留栈上啊，难怪"bt \-FF"瞎眼了。虽然但是呢，上帝又给留了一扇窗(PS: 这上帝给的有点多)，r14是保留寄存器，享受终生大保健VIP待遇，下层函数调用一定会给留个位的，走，去do\_filp\_open栈上逛逛。
 ![](https://oss-ata.alibaba.com/article/2024/09/432c90e9-62d2-460a-81db-1630fc483fe4.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1) 
dis给do\_filp\_open一照，嘿，上帝对咱是true love了，第2个push就把r14留do\_filp\_open栈上了，下班收工指日可待了这不。
![](https://oss-ata.alibaba.com/article/2024/09/f879dc82-5426-4d01-a09b-af81078cc478.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
调转枪头再来看一眼前面让我们绝望的"bt \-FF"的栈，根据汇编基础知识，linux x64调用函数先将返回地址压栈，调用do\_filp\_open后再将rbp压栈，最后就是我们朝思暮想的r14了，所以从do\_filp\_open栈底数第3个就是tmp了，抓!!!
 ![](https://oss-ata.alibaba.com/article/2024/09/307a4367-8c50-4932-958b-a766f3162264.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
 原来是用的第三方库openssl里面会打开一个编译机上的openssl.cnf文件，刚好在编译机的/mnt目录下，而/mnt目录是NAS等网盘默认挂载路径，当NAS出现异常时(如在有文件被占用情况下卸载NAS)，此时访问NAS文件目录都会hang住，导致进程hang。
![](https://oss-ata.alibaba.com/article/2024/09/a001d283-90bc-4a1c-a1ca-b4db731c5593.png?x-oss-process=image/resize,m_lfit,w_1600/auto-orient,1/quality,Q_80/format,avif/ignore-error,1)
 


**3\.后记**

 上面的例子上帝开的窗有点多，通过保存了函数参数的局部变量，咱们很快就真相大白了，但是如果万一万一非常点背，我们要找的函数参数就是没有在局部变量中保存，那怎么办呢。这种情况一般是不会有的，因为重要的参数只要有需要，就会传递到某一层函数局部变量中，耐心点找找就会有的，如果确实没有，可能就需要更耐心的分析整个调用栈，看看哪里会有些勾勾搭搭的局部变量，寄存器没有被破坏，也终究是可以抓出来的。
 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！

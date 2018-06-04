# 比原链如何在单机solonet测试环境下挖到矿 #  
                                  
***
（本人是初学者，行文简单和出错在所难免，请大家包涵。）

好多人学习和研究比原，但苦于在主网环境不敢实际操作怕带来损失，在单机solo环境又挖不出代币无法进行测试。本人在学习过程中也遇到了类似的问题，简单研究了一下写了如下方法。


1. **提前做好文件和钱包的备份。提前做好文件和钱包的备份。提前做好文件和钱包的备份。**  
  在不同的操作系统上，数据目录的位置也不同  
  苹果系统(darwin):~/Library/Bytom  
  Windows(windows): ~/AppData/Roaming/Bytom  
  其它（如Linux）:~/.bytom  
  为了方便，可以把该目录下文件直接拷贝备份，然后**清空该目录（必须要清除原来文件）**。

2. 挖矿的最终工作量证明在**bytom/consensus/difficulty/difficulty.go**
    
   //CheckProofOfWork checks whether the hash is valid for a given difficulty.
   
 func CheckProofOfWork(hash, seed *bc.Hash, bits uint64) bool {
	
compareHash := tensority.AIHash.Hash(hash, seed)
	
return HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0

}

这个函数中，最终比较的是使用AI友好算法生成的随机哈希**tensority.AIHash.Hash(hash, seed)使用HashToBig（）**转化为一个大数，
与预先设定好的一个难度的大数比较大小，这个大数由难度系数**bits**通过**CompactToBig(bits)**函数得出。即，这个函数最终比较**HashToBig(compareHash)**
和**CompactToBig(bits)**的大小，小于预定难度即工作量证明通过.

   其中使用AI友好算法生成随机哈希相关论文的下载地址： <https://github.com/Bytom/bytom/wiki/download/tensority-v1.2.pdf>，

原理介绍  
[https://mp.weixin.qq.com/s?src=11&timestamp=1527638586&ver=907&signature=MPke2SaX2swCC9XFeZ8pE1ydQqsiCjflONTQa7778N-sHtmgSskd8WDN48MoEcRbCY5QYCi4GZi87WihdxNtLOrZ2CH5G8*prn5nWbFdDPuLbH6N-uOqwFzvyZftmnB7&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1527638586&ver=907&signature=MPke2SaX2swCC9XFeZ8pE1ydQqsiCjflONTQa7778N-sHtmgSskd8WDN48MoEcRbCY5QYCi4GZi87WihdxNtLOrZ2CH5G8*prn5nWbFdDPuLbH6N-uOqwFzvyZftmnB7&new=1 "论文原理介绍")

大家感兴趣可以去看，这里不再累述。

   使用难度系数bits控制生成难度的大数的算法简介如下：

> CompactToBig converts a compact representation of a whole unsigned integer
 N to an big.Int. The representation is similar to IEEE754 floating point
numbers. Sign is not really being used.
>
	-------------------------------------------------
	|   Exponent     |    Sign    |    Mantissa     |
	-------------------------------------------------
	| 8 bits [63-56] | 1 bit [55] | 55 bits [54-00] |
	-------------------------------------------------

 >	N = (-1^sign) * mantissa * 256^(exponent-3)
  Actually it will be nicer to use 7 instead of 3 for robustness reason.

这个算法本人没有详细研究，根据我的理解就是使用类似于**IEEE754**浮点数科学计数法而使用的二进制的大数科学计数法，即一个很大的数转化为
一个小数乘以10的n次方的256进制表示，最后又把表示转化为十进制保存。

在使用单机**solonet**测试网络时很难通过该工作量证明挖到矿，于是考虑在不影响整个程序功能的情况下尝试做小的修改可以在单机跑起来，
有幸在比原技术微信群得到了几位大牛的指点，考虑修改难度系数bits,该系数在**bytom/config/genesis.go**文件中，初始值**Bits:2161727821137910632,**

为了更改**bits**,对代码**bytom/consensus/difficulty/difficulty.go**做如下修改
    
    //在文件开头的import中加入
    log "github.com/sirupsen/logrus"
    
    //修改函数CheckProofOfWork
	func CheckProofOfWork(hash, seed *bc.Hash, bits uint64) bool {

	compareHash := tensority.AIHash.Hash(hash, seed)
	log.Info("HashToBig compareHash=", HashToBig(compareHash))
	//log.Info("CompactToBig     bits=", CompactToBig(bits))
	
	if HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0 {
		log.Info("proof of work true")
		return true

	} else {
		bits = BigToCompact(HashToBig(compareHash))
		log.Info("proof of work false:bits=", bits)
		return false
		}
	}
这么做的目的是为了得到适合本机的难度系数。


3.改完代码保存修改文件，编译**bytom/cmd/bytomd/main.go** 为**bytomd.exe**
清除系统原来在**user/Administrator/AppData/Roaming**中文件。（如果需要，记得**备份备份备份**）

4.运行**bytomd init --chain_id solonet** 和**bytomd node --minging**命令，在dashboard创建账户，几秒之后在命令行窗口会看见系统的运行信息，接着会得到类似下面的信息

    time="2018-05-18T11:57:11+08:00" level=info msg="false:bits=2305843009219929325"
    time="2018-05-18T11:57:15+08:00" level=info msg="HashToBig compareHash=94470935200285873636399503116117805657110739921871232222716960430331350467616"
    time="2018-05-18T11:57:15+08:00" level=info msg="false:bits=2305843009227381927"
    time="2018-05-18T11:57:16+08:00" level=info msg="bk peer num:0 sw peer num:0 []"
    time="2018-05-18T11:57:19+08:00" level=info msg="HashToBig compareHash=108924585367465724541234689522322789151382457823652479112133953258841671282258
    time="2018-05-18T11:57:19+08:00" level=info msg="false:bits=2305843009229476129"
    time="2018-05-18T11:57:23+08:00" level=info msg="HashToBig compareHash=14146428415876784558544863438470231118789865847836167107575618549870912614213"
    time="2018-05-18T11:57:23+08:00" level=info msg="false:bits=2305843009215743640"
    time="2018-05-18T11:57:26+08:00" level=info msg="bk peer num:0 sw peer num:0 []"
    time="2018-05-18T11:57:28+08:00" level=info msg="HashToBig compareHash=78934284042892087673033389266663287288908738737936067679514124013738943934592"
    time="2018-05-18T11:57:28+08:00" level=info msg="false:bits=2305843009225130808"
    time="2018-05-18T11:57:32+08:00" level=info msg="HashToBig compareHash=55786918064653409850632601440227606224276562392668699194595776719016165547098"
    time="2018-05-18T11:57:32+08:00" level=info msg="false:bits=2305843009221776966"
    time="2018-05-18T11:57:36+08:00" level=info msg="HashToBig compareHash=83666295500599103004856267657480156049537933326077567233384721054295899257191"
    time="2018-05-18T11:57:36+08:00" level=info msg="false:bits=2305843009225816433"
    time="2018-05-18T11:57:36+08:00" level=info msg="bk peer num:0 sw peer num:0 []"
    time="2018-05-18T11:57:40+08:00" level=info msg="HashToBig compareHash=22266069092978536339665292252171173599340934438903703233524897266085035107388"

信息显示虽然挖矿失败了，但是通过运算我们可以得到很多适合本机难度的bits，每一个**time="2018-05-18T11:57:36+08:00" level=info msg="false:bits=2305843009225816433"**
中的bits都适合本机难度，这个**bits**上面对应的**level=info msg="HashToBig compareHash=**后面这个值越大，难度系数就越低。

5.挑选一个**bits**，拷贝下来**bits**的值，在源代码中**bytom/config/genesis.go**文件中，初始值**Bits:      2161727821137910632**,修改为你自己的难度系数。
比如修改为**Bits:2305843009228571441,**

6.再次执行步骤3和步骤4.现在你应该可以在单机solo模式下挖矿了。

7.另外，对**bytom/consensus/general.go** 文件中

    //config parameter for coinbase reward
	CoinbasePendingBlockNumber = uint64(100)//需要交易确认数
	subsidyReductionInterval   = uint64(840000)
	baseSubsidy                = uint64(41250000000)//每个块的比原数，单位：诺
	InitialBlockSubsidy        = uint64(140700041250000000)

	// config for pow mining
	BlocksPerRetarget     = uint64(2016)//区块难度每2016块调整
	TargetSecondsPerBlock = uint64(150)//150秒出块，初期不准的
	SeedPerRetarget       = uint64(256)

	// MaxTimeOffsetSeconds is the maximum number of seconds a block time is allowed to be ahead of the current time
	MaxTimeOffsetSeconds = uint64(60 * 60)
	MedianTimeBlocks     = 11

	PayToWitnessPubKeyHashDataSize = 20
	PayToWitnessScriptHashDataSize = 32
	CoinbaseArbitrarySizeLimit     = 128

	BTMAlias = "BTM"

这些字段修改也会影响挖矿的产量速度，比如**baseSubsidy                = uint64(41250000000)**指的是每块的产量412.5个btm,可以改大一点。
**CoinbasePendingBlockNumber = uint64(100)**交易需要的确认数可以改小一些。
具体请自行研究。这一步不是必须的。

8.如果感觉以上步骤比较麻烦，可以跳过步骤2，3和4，从步骤5开始直接修改bits的值，然后执行步骤6。即少修改一次代码和编译过程，直接使用步骤5中我已经测试好的bits.  
以上操作是针对使用命令行方式启动比原节点进行修改的，针对钱包桌面版的原理基本相同，需要使用钱包桌面版的请自行研究做相应替换。  

9.最后再次说明一定提前备份好btm里自己的重要数据，主要是key,最好整个bytom文件夹做一下备份。这个方法适合于有一定go语言基础的
程序员们，当然技术大牛们看了会见笑，觉得太简单了。费劲写这么多的主要目的是因为本人喜欢比原链，希望更多的人能够加入进来学习比原
研究比原，都为比原的发展贡献自己的力量，尽自己的一小点力量。而且在单机模拟环境下练习转账等操作也不会有实际的损失和风险。
请不要尝试用修改的文件连接主网，会有未知的风险。最后感谢微信比原技术测试群还有群友**Freewind**，**Freewind**是一个即聪明又勤奋的人，
没有他就没有这篇文章，大家感兴趣可以去看看他写的<https://github.com/freewind/bytom.win>关于比原的文章。



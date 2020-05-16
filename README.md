# 落格拼音整句引擎

落格输入法 X 和 落格输入法 macOS 2 中使用的整句拼音算法引擎。
现在起开源了，希望能和大家一起学习成长。

	TLDR;
	这个引擎目前还没有完全完工，还在测试最后的训练算法，我正在努力让它跑的更快……目前 10 GB 语料总计需要大约 10 小时处理时间。

---

## 简介

该引擎使用 Python 实现，包含了模型生成工具和整句计算样例算法，
使用 LMDB 存储转移数据，使用 SQLite 存储拼音数据，支持全拼、简拼以及全拼和简拼的混合拼音。

对于 `fangan` 这类拼音，引擎不进行处理，目前该引擎不包含拼音拆分算法，
但对于 `xian`、`qie`、`jian` 这类既能按一个拼音处理（如"先"、"且"、"键"），也能按两个拼音拆分的（如"西安"、"企鹅"、"吉安"），
引擎在模型生成之初就进行了兼容，我在统计最初就额外进行了处理，具体见训练部分相关代码注释。

由于拼音查询使用 SQLite，所以引擎天生支持模糊音，`fangan` 这一类拼音也可以根据需要由开发者自行处理（其实常见的没几个）

## 算法模型

该引擎使用隐马尔科夫模型和统计语言模型混搭概念实现，3-Gram（Tri-Gram）转移矩阵，使用 DAG（动态规划）进行求解。
Gram 单位为"词汇"，这样大大降低了算法遍历次数（相对于以字为单位来说），平滑使用简单的最大似然法处理，转移不存在则退回低阶，都不存在则使用一阶最小值。
训练时使用 `fast jieba` 进行分词，并用 `OpenCC` 统一转换为简体字处理。
考虑到中文输入时大多数情况下并不是从一句话开头进行打字，训练时不对语句进行起止标记。

## 数据结构

引擎需要两个数据文件进行计算，一个是 拼音->字词 的数据库（发射），另一个是 字词之间转移（转移） 的数据库。

### 发射数据库

发射数据库使用 SQLite ，好处是无需额外做前缀查询等处理，直接使用 SQL 语句即可，坏处是查询速度可能相对较慢，
对此我对 SQLite 进行了一系列参数优化，目前性能不错，也希望各位能提供更好的思路和算法。

为了加速模糊音和简拼处理，这里我对拼音进行了特殊的编码处理，使其用一个整数进行表达，并可进行声母和韵母（严格来讲是除了声母的部分）拆分合并。

为了加快 SQLite 查询，除了参数优化外，数据结构也是重点，首先，我根据词汇字数创建表，从 1 个字`w1`，一直到 8 个字`w8`（目前限制最大 8 字词）。
其次，每个拼音为一个字段，比如`w3`表:
```sql
CREATE TABLE "w3" (
	"p1"	INTEGER,
	"p2"	INTEGER,
	"p3"	INTEGER,
	"Words"	BLOB
);
```
如你所见，words 字段只有一个，它合并了所有这个拼音下可能出现的词汇，并按照 Uni-Gram 转移进行排序，比如：`你_泥_匿_铌` 类似这样。
_排序是为了方便单独查询，其实不排也就那样……_

总之，为了后续能方便和 LMDB 兼容以及文件体积考虑，`Words` 字段使用 `GB18030` 进行二进制编码，该编码对中文为两个字节，缩小体积，
另外 Python 对文本编码处理速度~~蜜汁~~极快，所以无需担心这方面问题。

> 注意，如果使用其他编程语言实现该算法，应当注意这个问题，比如落格输入法用 Swift 实现查询算法后，速度比 Python 版本慢一倍，
>究其原因，就是 String 与二进制文本编码转换太慢，最终我选择了直接操作二进制……

### 转移数据库

转移数据库使用 LMDB，写入速度一般，但读取的速度是极快的，将词汇转移以 `你好_我是_落格输入法` 这样的格式平铺存储，
由于 LMDB 要求存入二进制数据，所以 Key 同样使用 `GB18030` 编码处理，大大缩小了中文的存储体积， Value 则是 `Double` 类型的二进制文件。

## 模型训练

由于通常我们没有巨大内存的计算设备，训练模型专门为内存进行了优化，为了充分利用多核性能，训练器采用多进程并发方案来加速语料清洗和模型生成，这需要你根据自身设备对训练参数进行修改，比如最多进程数以及最大总可用内存数（推荐进程数为 CPU 核心数量-1，比如你的 CPU 具有 4 个核心，那么就设置为 3，给主进程留一个位置；推荐内存限制为你物理内存的一半，各个进程会平均分配内存，由于存在延时检测机制，所以并不会严格按照设定的内存大小限制）。

### 步骤
1 从 `articles` 目录中生成预处理好的语料
`data_produce.gen_data_txt()`会从 `articles` 中读取所有文本（`utf8`或`gb18030`）并进行清理，去掉所有英文和数字以及标点符号，并每一句拆分成一行进行存储，最终生成的`data.txt`将被放入`result_files`目录中。


2 从生成的 data.txt 文件统计转移
`get_transition_from_data.process()` 会逐行读入`data.txt`文件并对其使用 jieba 进行分词，然后分别统计 1、2、3 Gram 词汇转移数量，最终会在`result_files/temp`目录中生成未合并的临时文件。
然后再由主线程统一读取合并写入到数据库。


3 对统计得出的转移词频进行修剪以缩小体积并用最大似然法平滑
`get_smooth_transition.process()` 会读取统计结果并对其进行修剪，你需要根据自己的实际情况进行调参，基本概念就是去掉对应 Gram 下所有总统计数量的平均数的倍数以下的条目。
	比如 1 Gram 总共有 `100` 条，其中所有条目出现次数总和为 `10000` 次，那么平均数就是 `10000/100= 100`，如果我们比例设置为 0.5，那么就去掉所有统计数量低于`100*0.5=50`的条目。
最终，写出三个转移矩阵 json 文件以供使用和查看（因为修剪后体积足够小，就写成 json 方便 debug）以及一个同样修剪过的拼音与对应中文词汇的发射矩阵。


4 用平滑后的结果生成用于计算的二进制数据库，一个 SQLite 用来查拼音到词汇，一个 LMDB 用来查词汇转移概率（以 10 为底的对数）
`database_generator.writeLMDB()`

## 拼音编码

我对拼音进行了特殊编码，以便于把拼音转换成整数进行表达，这样每个拼音占用 16 位。声母占用高 8 位，韵母（严格来讲是所有非声母部分组合）占用低 8 位，这样一个拼音就占用 16 位，且可以方便地进行拆分组合，模糊查询。

	比如要查询 zh 的简拼，那么只需要在 SQLite 中查询所有大于 zh的低8位 0 mask 且 小于 zh的低8位 1 mask 即可。
python 字符编码那些事
==

本文旨在整理字符编码的信息以及 python 对字符编码的支持情况。

#### what is ASCII ?

回顾一下 ASCII 码是怎么一回事。

1968 年, American Standard Code for Information Interchange
制定了一个常用字符与数字的转换关系，这个关系就以机构名称的首字母命名，
即 ASCII，ASCII 码中使用了 1-127 的数字。

ASCII 码的问题是: 127 个的数字所能代表的字符数极为有限，存在大量显示不了的字符，
例如西欧的重音符，中国汉字字符等。

1980 年的时候，大部分电脑是 8 位的，也就是能使用的数字范围为 0-255，ASCII 码使
用了其中的一半。很多不同厂商的电脑于是把 128-255定义为其它的重音符号，不同机器
的定义不同，当机器间进行文件交换时出现了不少障碍。

然而，这也仅仅是解决了西欧的重音字符；如果要显示俄语字符时，又要重新定义；假如
要同时定义西欧的重音符以及俄语的西里尔字母时，就完全不够了。即，可以定义一个
KOI8 编码系统，用来存储所有俄语文档，另外定义一个 Latin1 编码系统来存储所有法
语文档，但如果想要在一个法语文档中引用几个俄语字符，就完全办不到了。

为了解决以上问题，出现了两个努力方向，ISO 10646 编码与 unicode 编码，两者后来
合并到了一起。

#### what is unicode ?

从上面我们看出增大用于表示字符的数字区间才能解决问题， 事实上 unicode 也确实是
这样干的。最初 unicode 采用 16 个比特位来表示数字，数量为 2^16=65536，预期能够
包含所有字符，但后来发现其实还不够。最终进化为今天的 unicode 标准，使用
0-1114111 这个数字范围，即十六进制 0x0-0x10ffff。

在介绍 unicode 之前，首先来明确一下几个名词：

字符：文档的最小组成单元
码点： 一个 16 进制整数，用来代表一个字符

unicode 所定义的规则即是将每个字符对应到一个码点。码点的写法示例：U+12CA，其对
应的十六制数为 0x12ca。

unicode 码表就是像这样一张表:

    0061    'a'; LATIN SMALL LETTER A
    0062    'b'; LATIN SMALL LETTER B
    0063    'c'; LATIN SMALL LETTER C
    ...
    007B    '{'; LEFT CURLY BRACKET

字符在电脑中的显现形式是码点，但通常我们人类所熟悉的是其在屏幕或纸上的呈现形式，
叫作字形 (glyph)。以怎样的字形呈现一个字符，是 GUI 工具的责任，应用开发通常不
用关心。

3. what is encoding(utf-8, gbk)

a unicode string is a sequence of code points

encoding is the rules for translating a unicode string into a sequence of bytes.

首先想到的一种 encoding 方式如下，
   P           y           t           h           o           n
0x50 00 00 00 79 00 00 00 74 00 00 00 68 00 00 00 6f 00 00 00 6e 00 00 00
   0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23

缺点：
1. 不同的处理器处理字节顺序不同，所以兼容性不好
2. 浪费空间，每 0x00 很多，因为单个字符占用大小超不过 255
3. 与已有 strlen() C 函数不兼容
4. 许多网络标准，处理不了嵌入的 0 字节

utf-8 -> ascii
0-255 之间它们是一致的
超出 255 则抛错 UnicodeEncodeError

有些 encoding 中， a-z 的码点直接是不连续的，如 IBM's EBCDIC，就没法一一对应

utf-8: 
规则：
1. 如果码点小于 128，则使用一个字节的值来表示
2. 如果码点大于 128, 则使用 2, 3 或 4 个字节来表示它，每个字节的值界于 128 - 255 之间
特点：
1. 所有的 unicode 码点都能处理
2. a unicode string 转化成的字节序列中没有嵌入 0 值字节。
3. ascii 文档，也同时是 utf-8 文档。
4. 紧凑的，常用的字符，基本都可以使用 1 到 2 个字节来表示
5. 如果字节坏了，是可能找到下一个仍旧完好的码点的位置的。随机的 8 位数据通常不会
看起来像 utf-8 编码。

更多 encoding 参见 codecs 模块 https://docs.python.org/2/library/codecs.html#module-codecs
4. python2, python3 unicode support

python2: 
unicode 类型
unicode(<str>, [encoding])，相当于 str.encode([encoding])

unicode 是由 unichr 组成的序列
unichr() 函数可以把一个整型数转化为一个 unichr
ord() 函数可以把一个 unichr 转化为一个整型数

unicode 类型支持传统 str 类型的各种操作

python3: 
str 类型就是过去的 unicode 类型，
而过去的 str 类型变为 bytes 类型。
使用 chr 与 ord 在 unicode 码点与整型数之间进行转换
已经没有 unicode 类型与 unichr 函数。

unicode literal:
所展示的是 unicode 字符的码点的 16 进制形式

码点直接落实为二进制串，其值与 utf-8, gbk 等编码之后的二进制串有区别！
参见 unicode wikipedia, utf-8 的编码方式，了解如何在码点二进制串与
utf-8 二进制串之间进行转换。

如果二进制串以 0 开头，则表明当前码点使用了一个字节
如果二进制器以 1 开头，则表明当前码点使用了多个字节:
假设有 n 个字节，则首字节以 n-1 个 1 开头，另外
所有参与码点值计算的二进制串都以 10 为前缀，故而一个多字节 utf-8 二进制串的组成如下：
(n-1)*1 + 10 + (8 - n - 1)*x    10 + 6*x   10 + 6*x  ...
x 为参数码点计算的二进制位

以汉字"赵"为例， 其码点为 0x8d75 对应的二进制串为 '0b1000110101110101', 共 16 位
在 unicode 码表之中，需采用三个字节才能表示刚好 16 位二进制位，其为
2 * 1 + 10 + 4x     10 + 6x     10 + 6x
即 11101000 10110101 10110101


在 python2 中，直接展示 unicode ，会显示其码点值序列 如 s，会展示以 '\u' 开头的码点值
In [58]: sys.getdefaultencoding()
Out[58]: 'ascii'
在 python3 中，弱化 unicode literal，直接展示 str，会是其字形序列
In [5]: sys.getdefaultencoding()
Out[5]: 'utf-8'


不过，使用 ord() 来循环打印 unicode literal，都可打印其码点值对应的十进制

5. convert

编码与解码的过程，即由 unicode 码点序列依据特定编码方式（如 utf-8, gbk, gb2312)转化为
二进制序列的过程。

以及其逆过程。

unicode -> encode -> binary(str in python2)
binary(str in python2) -> decode -> unicode

6. unicodedata

unicodedata 模块
包含 unicode 码表的信息，码点的名称及其类型等。

7. write to file. should as str.
unicode 的输出，先 encode() 成相应编码；
unicode 的输入，拿到输入先 decode() 成 unicode 码列；

数据库与文件，都有读入时自动转化为 unicode 的接口。
json.loads 与 json.dumps 也可以做相应事情。

一个约定是 unicode 文件以 U+FEFF 开头。 called byte-order mark (BOM)
utf-16 等需要， utf-8 不需要这个功能，不过也可以统一进标准。
    U+FEFF

8. unicode filenames

使用 sys.getfilesystemencoding() 查询当前文件系统采用的编码

在使用 open 操作时，指定 unicode 的文件名，其会自动转化为系统相应的编码。no bother
os.stat() os.listdir() 等也都有自动行为，根据你传入的文件名的类型，来决定输出的结果的类型。
传入 unicode，就收到 unicode, 传入 bytes 就收到 bytes.

9. str + unicode, it guess the encoding of the result. python2. python3?

10. principle

always use unicode inside, when input, decode it to unicode, when output, encode it to bytes
其实都  bytes, unicode 是码点二进制序列，而 bytes 是编码过的二进制序列。

对于从网上拿到的数据，要格外小心，先检查有没有非法字符再根据其编码进行相应转化。

11. StreamRecoder

if you have an input file f that’s in Latin-1, you can wrap it with a StreamRecoder to return bytes encoded in UTF-8:

12. 不知道文件编码，但书籍其是与 ascii 兼容的，且只想拿到并更改其中的 ascii 部分。
使用 surrogateescape when open

13. 在 codecs 文档中，除了标准的编码类型外，要注意这两个 raw_unicode_escape unicode_escape

In [49]: s
Out[49]: u'\u8d75\u4e91'

In [50]: print s
赵云

In [51]: s.encode('unicode-escape')
Out[51]: '\\u8d75\\u4e91'

In [52]: s.encode('raw-unicode-escape')
Out[52]: '\\u8d75\\u4e91'

In [53]: s.encode('utf-8')
Out[53]: '\xe8\xb5\xb5\xe4\xba\x91'

In [54]: b='\u8d75\u4e91'

In [55]: b
Out[55]: '\\u8d75\\u4e91'

In [56]: b.decode('unicode-escape')
Out[56]: u'\u8d75\u4e91'

https://docs.python.org/3/howto/unicode.html
https://docs.python.org/2/howto/unicode.html

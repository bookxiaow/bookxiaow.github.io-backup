字符集与字符编码

字符编码方案：规定了一组字符集，并且定义了将字符映射到编码的方案。例如ASCII, Unicode, GB2312等

字符编码方案包含两个层次：编码方式和实现方式。

# Unicode
以Unicode编码方案为例，

## 编码方式

从编码方式上来看，Unicode使用16位的编码空间，每个字符用一个16位的编码表示，这样共能表示65536个字符。
16位的Unicode字符组成Unicode的基本多文种平面（BMP）,在此之外还有16个平面，因此总共占用了21位来表示Unicode字符。

|平面	|始末字元值	|中文名称	|英文名称|
|---|---|---|---|
|0号平面 |	U+0000 - U+FFFF |基本多文种平面	|Basic Multilingual Plane,简称BMP|
|1号平面 |	U+10000 - U+1FFFF	| 多文种补充平面	|Supplementary Multilingual Plane,简称SMP|
|2号平面 |	U+20000 - U+2FFFF |	表意文字补充平面	|Supplementary Ideographic Plane,简称SIP|
|3号平面 |	U+30000 - U+3FFFF |	表意文字第三平面（未正式使用[1]）	|Tertiary Ideographic Plane,简称TIP|
|4号平面至13号平面	| U+40000 - U+DFFFF	|（尚未使用）	| |
|14号平面 |	U+E0000 - U+EFFFF |	特别用途补充平面	|Supplementary Special-purpose Plane,简称SSP|
|15号平面 |	U+F0000 - U+FFFFF |保留作为私人使用区（A区）[2]	|Private Use Area-A,简称PUA-A|
|16号平面 |	U+100000 - U+10FFFF |保留作为私人使用区（B区）[2]	|Private Use Area-B,简称PUA-B|

一般使用U+后接一个16进制数表示一个Unicode字符编码。

对任何一种字符编码方案来讲，每个字符的字符编码是确定的，比如U+0041表示'A'。

Unicode的前128个字符与ASCII编码一致。

## 实现方式

从实现方式上来看，Unicode编码存放在计算机内存时会有不同的格式。

Unicode的实现方式称为Unicode转换格式（Unicode Transformation Format,简称UTF），比如UTF-8, UTF-16, UTF-32

### UTF-8

UTF-8是可变长度字元编码，又称前缀码。

UTF-8针对Unicode编码的不同范围，使用不同的字节数来编码。

- 对于U+0000 ~ U+007F共128个字符，使用单字节编码；
- U+0080 ~ U+07FF，使用2字节编码
- U+0800 ~ U+D7FF和U+E000 ~ U+FFFF，使用3字节编码
- U+10000 - U+10FFFF，使用4字节编码

那对于一串UTF-8编码的字节流，如何确定每个码元占用的字节数呢？对于UTF-8编码中的任意字节B，

- 如果B的第一位为0（0x00-0x7F)，则B独立的表示一个字符(ASCII码)；
- 如果B的第一位为1，第二位为0(0x80-0xBF)，则B为一个多字节字符中的一个字节(非ASCII字符)；
- 如果B的前两位为1，第三位为0(0xC0-0xDF)，则B为两个字节表示的字符中的第一个字节；
- 如果B的前三位为1，第四位为0(0xE0-0xEF)，则B为三个字节表示的字符中的第一个字节；
- 如果B的前四位为1，第五位为0(0xF0-0xFF)，则B为四个字节表示的字符中的第一个字节；

### UTF-16

UTF-16使用1个或2个16bit的码元序列来表示所有的Unicode编码，如下所示：

16进制编码范围 | UTF-16表示方法（二进制） | 10进制码范围 | 字节数量
--- | --- | --- | ---
U+0000---U+FFFF | xxxxxxxx xxxxxxxx yyyyyyyy yyyyyyyy | 0-65535 | 2
U+10000---U+10FFFF | 110110yyyyyyyyyy 110111xxxxxxxxxx | 65536-1114111 |4

在[U+0000, U+FFFF]范围内有个区间[U+D800, U+DFFF]，不对应任何Unicode字符。因为我们看[U+10000,U+10FFFF]范围的UTF-16编码，共使用2个16bit码元来表示一个字符：

1. 码位减去0x10000,得到的值的范围为20比特长的0..0xFFFFF.
2. 高位的10比特的值（值的范围为0..0x3FF）被加上0xD800得到第一个码元或称作高位代理（high surrogate），值的范围是0xD800..0xDBFF.由于高位代理比低位代理的值要小，所以为了避免混淆使用，Unicode标准现在称高位代理为前导代理（lead surrogates）。
3. 低位的10比特的值（值的范围也是0..0x3FF）被加上0xDC00得到第二个码元或称作低位代理（low surrogate），现在值的范围是0xDC00..0xDFFF.由于低位代理比高位代理的值要大，所以为了避免混淆使用，Unicode标准现在称低位代理为后尾代理（trail surrogates）。

所以我们看到有编码值为[U+D800, U+DFFF]范围内的，就知道它是32bit 编码的一部分，而不是一个单独的字符。

对于常用的BMP的Unicode字符，UTF-16均使用2字节表示，因此在这个范围内可以看做是定长的编码方式。

对于ASCII字符，UTF-16也是采用2字节编码，因此**UTF-16与ASCII编码不兼容！**

# 最后

互联网工程工作小组（IETF）要求所有互联网协议都必须支持UTF-8编码。

互联网邮件联盟（IMC）建议所有电子邮件软件都支持UTF-8编码。

# 最后的最后

如何使用正则表达式来匹配Unicode字符

## 匹配字符

expression | 含义
--- | ---
\X | 匹配一个Unicode字符

# 附：

[Unicode编解码工具](http://www.endmemo.com/unicode/unicodeconverter.php)

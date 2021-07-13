---
title: "Golang类型转换 二"
date: 2021-07-13T15:30:41+08:00
draft: false
categories:
  - "Golang类型"

#Comment
gitment: true
# Theme-Defined params
#lead: "大家好，我们是搜狗商业技术团队！" # Lead text
#comments: false # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
---
strconv拾遗  
<!--more-->
上次学习了strconv包里的Atoi/Itoa，这次解决一下包里剩下的类型转换  

strconv包可以大致分成string/数字型转换，rune/sting转换。  常用的string/数字型转换又可以分三大类，Parse/Format/Append，简单理解就是数字型解析成字符串、数字型格式化成字符串和数字型拼接三种方法  
### Parse 

Parse系列函数

|         |  输入 |     输出      | 
| :--------- | :--: | -----------: | 
| ParseBool     | s string  |  bool,error | 
| ParseInt   |  s string, 基数 int, 位数 int   |int64,error |
| ParseUint |  s string, 基数 int, 位数 int | int64,error |  
| ParseFloat |  s string, 位数 int  | float64, error|  
| ParseComplex | s string, 位数 int   | complex128, error |  

其中ParseInt()/ParseUint()在[类型转换一](/post/inside-golang/strconv1/)中有介绍过主要流程，其他流程大同小异。   
#### ParseBool 
将字符串转换成布尔类型，因为bool结果只有两个，需要转换的目标也有限，所以这个函数超简单，直接枚举长的像bool型的字符串，对应到bool结果上。不过“t”/"f"也能进行映射成true/false 是第一次见。
````
func ParseBool(str string) (bool, error) {
	switch str {
	case "1", "t", "T", "true", "TRUE", "True":
		return true, nil
	case "0", "f", "F", "false", "FALSE", "False":
		return false, nil
	}
	return false, syntaxError("ParseBool", str)
}
````

#### ParseFloat 
ParseFloat稍微复杂点，不过也是一样的套路：检查系统位数->检查极限字符->拆分字符串->遍历字符串  
字符串“1.1”大致流程图  
![字符串“1.1”大致流程图](/img/strconv1/strconvFloat1.png)  
ParseFloat首先选择32/64位操作，32位转换和64位转换差别不大。以64位为例，具体使用内部函数atof64();   
atof64()里面有三个重要函数，分别是special()名如其功能，就是检查是否存在特殊情况比如无穷或非数，但它只能识别infinity/inf/nan;   
readFloat()将字符串拆解成mantissa uint64 尾数数, exp int指, neg 是否负数, trunc 是否溢出, hex bool是否十六进制, i int 占用字节数, ok bool 转换是否成功。有了这些后面就可以直接处理了。  
如果经过readFloat拆解是十六进制，则使用atofHex()
如果不是就可能使用atof64exact()和eiselLemire64算法，对于‘1.1’ atof64exact()就够了，atof64exact()也不复杂使用float64对尾数进行转换，对指数进行处理再与尾数相乘即可。

顺便看下16进制浮点数如何转换10进制浮点的，比如‘0x1a.2p1’ 0x是16进制标识，1a.2是尾数，p1就是e1。先将1a.2转换成二进制11010.001 指数是1小数点就向右移一位110100.01，再将2进制转换成10进制，110100=>52 .01 =>.25 52.25 atofHex()其实就是做的类似事情（ParseFloat()对16进制处理时候在readFloat()对p做了单独识别，而且p是判断16进制浮点数的条件。）​
````
    v := "1.1"
	if s, err := strconv.ParseFloat(v, 32); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}
	//float64, 1.100000023841858  #ParseFloat输出是float64所以按32位处理会影响精度
	if s, err := strconv.ParseFloat(v, 64); err == nil {
		fmt.Printf("%T, %v\n", s, s)
	}
    //float64, 1.1
     v := "0xD.2p1"
    if s, err := strconv.ParseFloat(v, 64); err == nil {
		fmt.Print(s)
	}
    //26.25
````

#### ParseComplex
ParseXXX系列还有一个复数转换，复数不怎么用，常见用在几何或者物理方面。好在ParseComplex不复杂，基本就是parseFloatPrefix处理字符串,也就是ParseFloat()执行两次
。
ParseComplex返回complex128的复数，需要注意是ParseComplex(s string, bitSize int)  bitSize是complex的位数是64/128，函数默认是采用parseFloatPrefix(s,64)处理也就是atof64(s)，所以处理complex64一定要指定bitSize对应目标位数，不然处理浮点型实虚数时会有精度缺失问题  

````
    v := "1.1+1i"
	c1 ,_:= strconv.ParseComplex(v, 128)
	fmt.Print(c1)
	//(1.1+1i)

	c2 ,_:= strconv.ParseComplex(v, 64)
	fmt.Print(c2)
	//(1.100000023841858+1i)
````


### Format
strconv包里还有另一类函数，FormatXX 将数字转换成字符串   

|         |  输入 |     输出      |   
| :--------- | :--: | -----------: |
| FormatBool     |  b bool  |  string | 
| FormatInt   | i int64, 基数 int |  string|
| FormatUint |  i uint64,基数 int|string |  
| FormatFloat |  f float64, fmt byte, prec, bitSize int | string|  
| FormatComplex |  c complex128, fmt byte, prec, bitSize int  | string |  

FormatBool()很简单，就是直接返回“true/false”字符串；
````
func FormatBool(b bool) string {
	if b {
		return "true"
	}
	return "false"
}

````
FormatInt()和FormatUint()上次也学习过了[Atoi](/post/inside-golang/strconv1/)

#### FormatFloat  
FormatFloat(),需要四个参数f float64, fmt byte 格式参数(fmt必须是byte，所以用单引号), prec 小数保留位数(如-1是全保留), bitSize int基数  
fmt可供选择：  
````
// The format fmt is one of
// 'b' (-ddddp±ddd, a binary exponent), 二进制表达式
// 'e' (-d.dddde±dd, a decimal exponent),   十进制表达式
// 'E' (-d.ddddE±dd, a decimal exponent),
// 'f' (-ddd.dddd, no exponent),    普通的浮点表达式
// 'g' ('e' for large exponents, 'f' otherwise),    大数情况使用的标识
// 'G' ('E' for large exponents, 'f' otherwise),
// 'x' (-0xd.ddddp±ddd, a hexadecimal fraction and binary exponent), or 十六进制表达式
// 'X' (-0Xd.ddddP±ddd, a hexadecimal fraction and binary exponent).
````
FormatFloat使用内部函数genericFtoa() ，genericFtoa()做的事情就是根据参数计算出原始float的十进制表达式，fmt是格式参数，指定计数格式，并且这个参数直接取决后续计算过程。
精度参数prec小数点位置（如果是-1默认全部长度），而且这时候使用Grisu3算法计算浮点数，据说是普通精度算法的四倍，具体如何计算暂时没有了解。
如果指定了精度长度就用普通的精度算法计算。篇幅限制就不展开genericFtoa()函数；  
FormatComplex()和ParseComplex()一样，相当于执行两次FormatFloat()
````
	var v float64
	v = 3.1415926535

	s64 := strconv.FormatFloat(v, 'E', 2, 64)
	fmt.Printf("%T, %v\n", s64, s64)
	//string, 3.14E+00
	
	v:= (1.1+1i)
	fmt.Print(strconv.FormatComplex(v,'f',-1,64))
	//(1.1+1i)
````


#### Append
最后一类AppendXX函数，将数字类型进行拼接，需要注意不是计算，最终输出数组。因为最终拼接结果是字符串数组，所以拼接前需要和FormatXX一致的转换过程，如AppendFloat()底层处理逻辑就与FormatFloat()是相同逻辑


#### 总结
1. strconv包有三类函数 ParseXX 将字符串转换成数字类型； FormatXX 将数字类型转换成字符串 ；AppendXX 数字类型的拼接函数。这三类函数解决常见转换问题；  
2. 精度问题 strconv转换函数大多默认返回float64，会导致输入float32的结果精度失真，使用前应注意；  
3. strconv还有处理rune/ASCII/图形符号之间的转换QuoteXX系列，以及rune/图形判断函数，逻辑也都不复杂，可以直接使用；


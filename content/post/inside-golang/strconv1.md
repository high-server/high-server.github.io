---
title: "Golang类型转换 一"
date: 2021-06-01T16:08:39+08:00
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
Atoi/Itoa函数实现
<!--more-->
## 常用类型转换

基于工作中对golang数据类型使用的总结。在长期使用PHP过程中对数据类型的感觉就是“PHP眼里什么是数据类型？不存在的！万物皆可划等号”从这点来讲PHP的确是世上最好的语言。弱类型对开发者使用有着很大的便利，但也存在一定风险，比如类型操作不统一会在迭代过程中被覆盖，造成精度缺失甚至程序崩溃的问题。  
golang作为强类型语言，在初用golang时，类型操作上会遇到过不少问题，日常使用中整理如下内容。  
本人Go语言小学生，不足之处欢迎各路大神指正。


### 第一跤  
一个美丽的清晨，开始了第一个Go线上业务的开发。简单的需求，从数据库取出时间戳与比当前时间比对。  
大概这个样子，原始数库时间戳字段直接被定义成int(php嘛)，直接上手比较，没有意外，Xcode贴心的小浪线出来了，编译不通过。
````
func main() {
	var a int
	a = 1621251016
	fmt.Print(a > time.Now().Unix())
}

//invalid operation: a > time.Now().Unix() (mismatched types int and int64)
````
心头一紧，都是整数型为啥不行。打开golang.org/pkg/time包……  func (t Time) Unix() int64。好吧，对于php就算是string也可以自动转换和int型进行比较。  
但是Go为什么不行呢，查询文档如下解释。
````
[@builtin]# go doc builtin.int
package builtin // import "builtin"

type int int
    int is a signed integer type that is at least 32 bits in size. It is a
    distinct type, however, and not an alias for, say, int32.

func cap(v Type) int
func copy(dst, src []Type) int
func len(v Type) int
[@builtin]# go doc builtin.int64
package builtin // import "builtin"

type int64 int64
    int64 is the set of all signed 64-bit integers. Range: -9223372036854775808
    through 9223372036854775807.
````
很严格，int至少32位，也就是所在系统位数32/64，特意强调“distinct”特殊的类型，不等于int32/int64的别名！  
第一次类型操作就这么刺激，Go在整数型之间都不会自动转换可以想象其他类型间转换都需要明确定义和指定操作。


### 常用数据类型转换
#### AtoI 
字符串与整数类型转换
最常用的数据类型转换可能就是string与int互转，Go提供了strconv包解决，Atoi和Itoa方法（ascii to integer/integer to ascii）  
转换前，回顾下常用进制表示，采用前缀区分：
````
2进制 0b1010...  
8进制 0o1234.../01234...  
16进制 0x12A...  
````
Atoi(s string) 对字符串转换过程中核心函数是ParseUint()，Atoi()相当于使用ParseInt()对十进制字符转换。十进制转换过程中只需要对字符串数组进行遍历，并按位乘上基数，最后补全符号即可。但是对于2/8/16进制转换就需要ParseInt()处理了。

````
//十进制
func Atoi(s string) (int, error) {
    ...
        n := 0
		for _, ch := range []byte(s) {
			ch -= '0'
			if ch > 9 {
				return 0, &NumError{fnAtoi, s0, ErrSyntax}
			}
			n = n*10 + int(ch)
		}
		if s0[0] == '-' {
			n = -n
		}
    ...
}
````
ParseInt()实质上是对转换函数做预处理和结果校验。Atoi()和ParseInt()都会对字符串进行切片，剔除“+/-”符号后交给ParseUint()处理，ParseUint()会根据进制前缀检查字符所属进制。剩下的就根据各自进制特点计算各位整型值并乘上基数，再相加。比如16进制字符串转换成十进制数值。  
如果超出进制范围，则直接返回最大值和error信息。    

````
func ParseUint(s string, base int, bitSize int) (uint64, error) {
    ......
    for _, c := range []byte(s) {
		var d byte
		switch {
		case c == '_' && base0:
			underscores = true
			continue
		case '0' <= c && c <= '9':
			d = c - '0'
		case 'a' <= lower(c) && lower(c) <= 'z':
			d = lower(c) - 'a' + 10
		default:
			return 0, syntaxError(fnParseUint, s0)
		}

		if d >= byte(base) {
			return 0, syntaxError(fnParseUint, s0)
		}

		if n >= cutoff {
			// n*base overflows
			return maxVal, rangeError(fnParseUint, s0)
		}
		n *= uint64(base)

		n1 := n + uint64(d)
		if n1 < n || n1 > maxVal {
			// n+v overflows
			return maxVal, rangeError(fnParseUint, s0)
		}
		n = n1
	}
    ......
}
````

![Atoi主要流程图](/img/strconv1/atoib.png)
  
ParseUint()里最后用到underscoreOK() 对下划线检查，很有意思，必须是在数字中间，而且是相同进制的数字字符，不能连续使用下划线。查了一下数字中间下划线只是为了增加可读性，没有什么特殊含义。  
但是需要注意base参数需要设置为0，而且bitSize要设置正确，不然溢出则返回最大值。 
````
func main() {
	var b string
	b = "12345_12345"
	s, _ := strconv.ParseUint(b, 0, 16)
	fmt.Print(s, math.MaxUint16)
}
// 65535 65535
````

#### Itoa
int型与字符串类型转换  
Itoa()直接使用FormatInt()  
其中对10进制0-99之间的字符串转换，直接在枚举常量digits和smallsString字符串中用截取出来。

````
const digits = "0123456789abcdefghijklmnopqrstuvwxyz"
const smallsString = 
    "00010203040506070809" +
	"10111213141516171819" +
	"20212223242526272829" +
	"30313233343536373839" +
	"40414243444546474849" +
	"50515253545556575859" +
	"60616263646566676869" +
	"70717273747576777879" +
	"80818283848586878889" +
	"90919293949596979899"
// 0-99
func small(i int) string {
	if i < 10 {
		return digits[i : i+1]
	}
	return smallsString[i*2 : i*2+2]
}

````
对于其他情况采用formatBits()解决，Itoa()直接将参数转化成int64，在做计算截取。通过smallsString 200以内的枚举数字字符串就可以快速映射整数型对应的字符串。  

````
func formatBits(dst []byte, u uint64, base int, neg, append_ bool) (d []byte, s string) {
    ......
    for us >= 100 {
			is := us % 100 * 2
			us /= 100
			i -= 2
			a[i+1] = smallsString[is+1]
			a[i+0] = smallsString[is+0]
		}

		// us < 100
		is := us * 2
		i--
		a[i] = smallsString[is+1]
		if us >= 10 {
			i--
			a[i] = smallsString[is]
		}
    ......
}
````

![Itoa主要流程图](/img/strconv1/Iotab.png)

如图：计算偏移时打印得到51/54 对着smallsString数了半天 第127个元素怎么就是51呢？?不是3！ 看了看smallsString 又看了看fmt.Print，默默的加上了string(smallsString[127]) = 3   
````
i -= 2
a[i+1] = smallsString[127] 
//fmt.Print(smallsString[127]) = 51 
a[i+0] = smallsString[126] 
//fmt.Print(smallsString[126]) = 54
````

### 总结
1. Go 语言是强类型语言操作类型时需要注意转换，并且确保转换精度准确；
2. 字符串与整数型间转换都以int64/uint64位为标准输入输出；
3. Atoi/Itoa设计并不复杂，作为基础函数学习是很好的例子。
4. 数据类型转换不仅限于字符整数，后续还会整理unsafe/error/复合类型的转换和操作。


## 1.初识Python

1989 年，作者决心开发一个新的解释程序

1991年，第一个Python解释器诞生

## 2.安装Python

官网下载：https://www.python.org/downloads/

- 添加到path
- 修改默认路径

cmd 输入python

```shell
C:\Users\Lenovo>python
Python 3.10.6 (tags/v3.10.6:9c7b4bd, Aug  1 2022, 21:53:49) [MSC v.1932 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

**Python解释器**：用来翻译Python代码，并提交给计算机执行。

### 2.1测试Python

```shell
>>> print("hello world")
hello world
```

### 2.2安装Pycharm

官网：https://www.jetbrains.com/zh-cn/pycharm/

安装完成记得主动配置一下编译器的位置！

注：

1. Python最常见的开发环境是：Pycharm
2. Python需要以工程为单元，想要写代码，需要县创建一个工程

## 3.Python基础语法

### 3.1 掌握字面量的含义

**字面量：**在代码中，被写下来的固定的值，称之为**字面量**

### 3.2 了解常见的字面量类型

六种数据类型：Number、String、List、Tuple、Set、Dictionary——数字、字符串、列表、元素、集合、字典

数字支持：int、float、complex、bool——整数、浮点数、复数、布尔

### 3.3 基于print完成字面量的输出

```python
print(10)
print("Strings")
```

### 3.4 注释

- 单行注释：以 # 开头，一般和文字之间**用一个空格隔开**
- 多行注释：以**一对三个双引号**引起来的内容

```python
"""
这里就是注释了
"""
```

### 3.5 变量

**变量：**在程序运行时，能储存计算结果或能表示值的抽象概念（记录数据）	变量名称=变量的值

```python
money = 50
print(money)

money = money-10
print("剩余:",money)
```

### 3.6 数据类型

**type()语句的使用方式：**

```python
# 使用print直接输出类型信息
print(type("这是一句话"))
print(type(666))
print(type(12.234))

# 使用变量存储type()语句的嗯结果
string_type = type("这是一句话")
int_type = type(666)
float_type = type(12.345)
print(string_type)
print(int_type)
print(float_type)

# 使用type()语句,查看变量中存储的数据类型信息
name = "这是一句话"
name_type =type(name)
print(name_type)
```

**变量有没有类型？**我们通过type(变量)可以输出类型，这是查看变量的类型还是数据的类型？

答：查看的是变量存储的数据类型，因为变量无类型，但是它存储的数据有。

### 3.7 数据类型转换

**为什么要转换数据类型：**

- 从文件中读取的数字，默认是字符串，我们需要转换成数字类型
- input()语句，默认结果是字符串，若需要数字也需要转换
- 将数字转换成字符串用以写出到外部系统

**常见的转换语句：**转换语句都是带有结果的，可以用print直接输出或用变量存储值

int(x)：将X转换为整数	float(x)：将x转换为浮点数	str(x)：将对象x转换为字符串

```python
# 将数字类型转换字符串
num_str = str(11)
print(type(num_str),num_str)

float_str = str(12.345)
print(type(float_str),float_str)

# 将字符串转换成数字
num = int("11")
print(type(num),num)

num2 = float(12.345)
print(type(num2),num2)

# 整数转浮点数
float_num = float(11)
print(type(float_num),float_num)

# 浮点数转整数
int_num = int(12.345)
print(type(int_num),int_num)

"""
<class 'str'> 11
<class 'str'> 12.345
<class 'int'> 11
<class 'float'> 12.345
<class 'float'> 11.0
<class 'int'> 12
"""
```

### 3.8 标识符

**什么是标识符：**变量的名字、方法的名字、类的名字 等等

**标识符命名规则：**英文、中文、数字、下划线、大小写敏感、不可使用关键字、**不能以数字开头**

**命名规范：**见名知意、下划线命名法、英文字母全小写

### 3.9 运算符

**算数运算符：**`+`加、`-`减、`*`乘、`/`除、`//`取整除、`%`取余、`**`指数

**赋值运算符：**`+=`、`_=`、`*=`、`/=`、`%=`、`**=`、`//=`

### 3.10 字符串扩展

#### 字符串定义

**字符串的三种定义方式：**

- 单引号定义法
- 双引号定义法
- 三引号定义法：三引号定义法，和多行注释的写法一样，同样支持换行操作。使用变量接收它，它就是字符串，不使用变量接收它，就可以作为多行注释使用。

```python
# 单引号定义法
name = 'torch'
print(type(name))
# 双引号定义法
name = "torch"
print(type(name))
# 三引号定义法
name = """
torch
torch
"""
print(type(name))
"""
<class 'str'>
<class 'str'>
<class 'str'>
"""
```

**字符串的引号嵌套：**

- 单引号定义法，可以内含双引号
- 双引号定义法，可以内含单引号
- 可以使用转义字符（\）来将引号接触效用，变成普通字符串

==强烈建议习惯以双引号为字符串定义==

#### 字符串拼接

一般，字面量和变量或变量和变量之间会只用拼接

```python
name= = “torch”
print("my name is:"+ name +"lee")
```

**注：无法和非字符串类型进行拼接**

#### 字符串格式化

- %表示：我要占位
- s表示：将变量变成字符串放入占位的地方

**多个变量占位，要用括号括起来，并按照占位顺序填入**

```python
name = "torch"
message = "my name is %s" % name
print(message)

class_num = 10
avg_sal = 12345
message = "班级为：%s，平均工资为：%s" % (class_num,avg_sal)
print(message)
"""
my name is torch
班级为：10，平均工资为：12345
"""
```

**常用格式化符号：%s、%d、%f**		转换字符串、转换整数、转换浮点数

#### 格式化的精度控制

用辅助符号“m.n”来控制数据的宽度和精度

- m 控制宽度，要求是数字（很少使用），**设置的宽度小于数字本身，不生效**
- .n 控制小数点精度，要求是数字，**会进行小数的四舍五入**

示例：

- %5d：表示将整数的宽度控制在5位，如数字11，被设置为5d，就会变成:空格空格空格11，用三个空格补足
  宽度。
- %5.2f：表示将宽度控制为5，将小数点精度设置为2
  小数点和小数部分也算入宽度计算。如，对11.345设置了%7.2f后，结果是:空格空格11.35。2个空格补足宽度，小数部分限制2位精度后，四舍五入为.35
- %.2f：表示不限制宽度，只设置小数点精度为2，如11.345设置%.2f后，结果是11.35

#### 字符串格式化方法2

**快速写法：**f"内容{变量}"的格式来快速格式化，==不理会类型，不做精度控制==，原样输出	format

#### 对表达式进行格式化

**表达式：**一条具有明确执行结果的代码语句

==在无须使用变量进行数据存储的时候，可以直接格式化表达式，简化代码==

```python
print("1 * 1 的结果是：%d" % (1 * 1))
print(f"1 * 1 的结果是：{1 * 2}")
```

#### 练习

```python
"""
练习题：股价计算小程序
"""
name = "例子公司"
# 当前股价
stock_price = 19.99
# 股票代码
stock_code = "003032"
# 每日增长系数
stock_price_daily_growth_factor = 1.2
# 增长天数
growth_days = 7

print(f"公司:{name},股票代码:{stock_code},当前股价:{stock_price}")
print("每日增长系数是:%.1f,经过%d天的增长后,股价达到了:%.2f" % (stock_price_daily_growth_factor,growth_days,stock_price*stock_price_daily_growth_factor**growth_days))

"""
公司:例子公司,股票代码:003032,当前股价:19.99
每日增长系数是:1.2,经过7天的增长后,股价达到了:71.63
"""
```

### 3.11 数据输入

#### 获取键盘输入

- 数据输入：input	数据输出：print
- 使用input()语句可以从键盘获取输入
- 使用一个变量接收（存储)input语句获取的键盘输入数据即可

```python
sth = input("input sth here:")
print(f"you input:{sth}")
"""
input sth here:123
you input:123
"""
```

注：无论输入什么内容，不转换，获取到的数据永远都是字符串类型！

## 4.Python判断语句

### 4.1 布尔类型和比较运算符

**布尔类型：**是和否	true 1 false 0

定义变量存储布尔类型数据：变量名称 = 布尔类型字面量

**比较运算符：**>  <  ==  !=

### 4.2 if语句的基本格式

```python
age = 22
if age >= 18:
    print("我成年了")
    print("即将步入大学生活")	# 注意缩进 四空格缩进的为if中的条件执行语句 平齐的为正常语句
print("时间过的真快")
"""
我成年了
即将步入大学生活
时间过的真快
"""
```

练习:

```python
price = 10
print("欢迎来到儿童游乐场，儿童免费，成人收费")
age = int(input("请输入你的年龄:"))
if age >= 18:
    print(f"您已成年，游玩需要补票{price}元")
print("祝您游玩愉快")

"""
欢迎来到儿童游乐场，儿童免费，成人收费
请输入你的年龄:30
您已成年，游玩需要补票10元
祝您游玩愉快
"""
```

### 4.3 if else语句

注：执行语句缩进！==可以直接把input放入if的条件中进行判断==

```python
price = 10
print("欢迎来到儿童游乐场，儿童免费，成人收费")
age = int(input("请输入你的年龄:"))
if age >= 18:
    print(f"您已成年，游玩需要补票{price}元")
else:
    print("你可以免费游玩")
print("祝您游玩愉快")
"""
欢迎来到儿童游乐场，儿童免费，成人收费
请输入你的年龄:5
你可以免费游玩
祝您游玩愉快
"""
```

### 4.4 if elif else 语句

```python
price = 10
if int(input("输入身高(cm):")) < 120:
    print("身高小于120cm，可以免费。")
elif int(input("输入VIP等级(1-5):")) >3:
    print("vip等级大于3，可以免费")
elif int(input("输入今天是几号:")) == 1:
    print("今天1号，免费玩")
else:
    print(f"不满足免费，交钱{price}！")
"""
输入身高(cm):135
输入VIP等级(1-5):3
输入今天是几号:1
今天1号，免费玩
"""
```

练习：

```python
final = 10
if int(input("请输入第一次猜想的数字：")) == final:
    print("一次猜对！")
elif int(input("不对，再猜一次：")) == final:
    print("两次猜对")
elif int(input("不对，最后一次：")) == final:
    print("三次猜对")
else:
    print(f"全部猜错了，我想的是{final}")
"""
请输入第一次猜想的数字：1
不对，再猜一次：2
不对，最后一次：3
"""
```

### 4.5 判断语句的嵌套

嵌套的关键点，在于**空格缩进**

通过空格缩进，来决定语句之间的**层次关系**

### 4.6 判断语句案例

```python
import random

num = random.randint(1,10)
guess_num = int(input("输入你要猜测的数字："))

if guess_num == num:
    print("一次猜中！")
else:
    if guess_num > num:
        print("猜大了")
    else:
        print("猜小了")
    guess_num = int(input("第二次输入猜测的数字："))

    if guess_num == num:
        print("两次猜中！")
    else:
        if guess_num > num:
            print("猜大了")
        else:
            print("猜小了")
        guess_num = int(input("第三次输入猜测的数字："))

        if guess_num == num:
            print("三次次猜中！")
        else:
            print("未猜中，结束")
"""
输入你要猜测的数字：5
猜小了
第二次输入猜测的数字：6
两次猜中！
"""
```

## 5.Python循环语句

### 5.1 while循环的基础语法

```python
# 死循环！
i = 1
while True:
    i *= 2
    print(f"{i}")
```

注：条件需要提供布尔型结果；空格缩进；规划好终止条件

**练习：1加到100的和**

```python
i = 1
final = 0
while i <= 100:
    final += i
    i += 1
print(f"{final}")
"""
5050
"""
```

### 5.2 while循环的基础案例

```python
import random
num = random.randint(1,100)
count = 0
flag = True
while flag:
    guess_sum = int(input("请输入你猜测的数字："))
    count += 1
    if guess_sum == num:
        print("猜中了！")
        flag = False
    else:
        if guess_sum > num:
            print("猜大了")
        else:
            print("猜小了")
print(f"你一共猜了{count}次")
"""
请输入你猜测的数字：50
猜小了
请输入你猜测的数字：75
猜大了
请输入你猜测的数字：65
猜小了
请输入你猜测的数字：70
猜大了
请输入你猜测的数字：68
猜大了
请输入你猜测的数字：66
猜中了！
你一共猜了6次

进程已结束,退出代码0
"""
```

### 5.3 while的嵌套应用

同判断语句的嵌套一样，循环语句的嵌套，要注意空格缩进。

### 5.4 while循环案例——九九乘法表

**在print语句中，加上end=''即可输出不换行**

**制表符 \t 用来对齐单词或者字符**

```python
# 定义外部循环
i = 1
while i <= 9:
    # 定义内部循环的控制变量
    j = 1
    while j <= i:
        print(f"{j}*{i}={j*i}\t",end='')
        j += 1
    i += 1
    print()  # 仅仅输出一个换行
"""
1*1=1	
1*2=2	2*2=4	
1*3=3	2*3=6	3*3=9	
1*4=4	2*4=8	3*4=12	4*4=16	
1*5=5	2*5=10	3*5=15	4*5=20	5*5=25	
1*6=6	2*6=12	3*6=18	4*6=24	5*6=30	6*6=36	
1*7=7	2*7=14	3*7=21	4*7=28	5*7=35	6*7=42	7*7=49	
1*8=8	2*8=16	3*8=24	4*8=32	5*8=40	6*8=48	7*8=56	8*8=64	
1*9=9	2*9=18	3*9=27	4*9=36	5*9=45	6*9=54	7*9=63	8*9=72	9*9=81	
"""
```

### 5.5 for循环的基础语法

#### 基础语法

**for循环是一种轮询机制，是对一批内容进行逐个处理**

如果在for循环外部访问临时变量：

- 实际上是可以访问到的
- 在编程规范上，是不允许，不建议这么做的，必要的话需要在for循环外部定义一个全局变量

遍历字符串，依次取出，遍历循环，**理论上讲Python的for循环无法构建无限循环**，无法定义循环条件，只能被动取出数据处理

```python
name = "i am torch"
for x in name:
    print(x)
"""
i
 
a
m
 
t
o
r
c
h
"""
```

**练习：统计字符串中字母出现的次数**

```python
name = "torch is a good student"
count = 0
for x in name:
    if x == "o":
        count += 1
print(f"字符串中一共有{count}个o")    
"""
字符串中一共有3个o
"""
```

#### range语句

语法中的：待处理数据集，严格来说，称之为“序列类型”，**序列类型：其内容可以一个个一次取出的一种类型**

语法：

1. range(num)：从0开始，到num结束（不含num本身）
2. range(num1,num2)：从num1开始，到num2结束（不含num2本身）
3. range(num1,num2,step)  step为步长，从num1开始，到num2结束（不含num2本身），步长以step值为准

练习：

```python
count = 0
for x in range(1, 100):
    if x % 2 == 0:
        count += 1
print(f"1到100(不含100本身)范围内，有{count}个偶数")
"""
1到100(不含100本身)范围内，有49个偶数
"""
```

### 5.6 for循环的嵌套使用

**注意：**while和for循环是可以嵌套的

练习：嵌套九九乘法表

```python
for x in range(1, 10):
    for y in range(1, x+1):
        print(f"{y}*{x}={x*y}\t", end='')
    print()
"""
1*1=1	
1*2=2	2*2=4	
1*3=3	2*3=6	3*3=9	
1*4=4	2*4=8	3*4=12	4*4=16	
1*5=5	2*5=10	3*5=15	4*5=20	5*5=25	
1*6=6	2*6=12	3*6=18	4*6=24	5*6=30	6*6=36	
1*7=7	2*7=14	3*7=21	4*7=28	5*7=35	6*7=42	7*7=49	
1*8=8	2*8=16	3*8=24	4*8=32	5*8=40	6*8=48	7*8=56	8*8=64	
1*9=9	2*9=18	3*9=27	4*9=36	5*9=45	6*9=54	7*9=63	8*9=72	9*9=81
"""
```

### 5.7 循环中断：break和continue

**应用场景：在循环中，因为某些原因，临时结束本次循环**

- **continue关键字**用于：中断本次循环，直接进入下一次循环；可以用于for循环和while循环，效果一致
- **break关键字**用于：直接结束循环；可以用于for循环和while循环，效果一致

在嵌套循环中，只能作用在所在的循环上，无法对上层循环起作用！

### 5.8 综合案例

发工资：某公司账户余额有1W元，给20名员工发工资

- 员工编号1-20，从编号1开始，依次领取工资每人可以领取1000元；
- 领工资时，财务判断员工的绩效分（1-10随机生成），如果低于5，不发工资，换下一位
- 如果工资发完了，结束发工资

```python
import random

salary = 10000
for i in range(1,21):
    num = random.randint(1, 10)
    if num < 5:
        print(f"员工{i},绩效分{num},低于5，不发工资，下一位")
        continue
    if salary >= 1000:
        salary -= 1000
        print(f"向员工{i}发工资1000元，账户余额还剩{salary}元")
    else:
        print(f"余额不足,当前余额{salary}元，不足以发工资，下个月再来")
        break
"""
向员工1发工资1000元，账户余额还剩9000元
向员工2发工资1000元，账户余额还剩8000元
向员工3发工资1000元，账户余额还剩7000元
向员工4发工资1000元，账户余额还剩6000元
向员工5发工资1000元，账户余额还剩5000元
向员工6发工资1000元，账户余额还剩4000元
向员工7发工资1000元，账户余额还剩3000元
员工8,绩效分3,低于5，不发工资，下一位
向员工9发工资1000元，账户余额还剩2000元
员工10,绩效分1,低于5，不发工资，下一位
向员工11发工资1000元，账户余额还剩1000元
向员工12发工资1000元，账户余额还剩0元
余额不足,当前余额0元，不足以发工资，下个月再来
"""
```

## 6.函数

### 6.1 函数体验

函数：是组织好的，可重复使用的，用来实现特定功能的代码段；提高代码的复用性，减少重复代码，提高开发效率

```python
# 统计字符串的长度，不使用内置函数len
str1 = "torch"
str2 = "torchlee"
str3 = "python"


# 用函数来优化过程
def my_len(data):
    count = 0
    for i in data:
        count += 1
    print(f"字符串{data}的长度是:{count}")

my_len(str1)
my_len(str2)
my_len(str3)

"""
字符串torch的长度是:5
字符串torchlee的长度是:8
字符串python的长度是:6
"""
```

### 6.2 函数的基础定义语法

函数的定义：

```python
def 函数名(传入参数):
    函数体
    return 返回值
```

注意：

1. 参数如果不需要，可以省略
2. 返回值如果不需要，可以省略
3. 函数必须先定义后使用

### 6.3 函数的传入参数

传入参数的功能是：在函数进行计算的时候，接受外部（调用时）提供的数据

```python
def add(x, y):
    result = x + y
    print(f"{x} + {y}的计算结果是:{result}")

add(100,500)
```

- 函数定义中，提供的x和y称之为：形式参数（形参），表示函数声明将要使用2个参数
  - 参数之间使用逗号进行分割
- 参数调用中提供的数据，称之为：实际参数（实参），表示函数执行时真正使用的参数值
  - 传入的时候按照顺序传入数据，使用逗号分隔
- **传入参数的数量是不受限制的**
  - 可以不使用参数
  - 也可以仅使用任意n个参数

### 6.4 函数练习题

```python
def temperature(data):
    if data < 37.5:
        print(f"您的体温是{data},体温正常，请进！")
    elif data >= 37.5:
        print(f"您的体温是{data},体温异常!")

temperature(39)
```

### 6.5 函数的返回值定义语法

```python
def 函数名(传入参数):
    函数体
    return 返回值
变量 = 函数(参数)

def add(a, b):
    result = a + b
    return result

r = add(5, 6)
print(r)

```

**注：函数体在遇到return后就结束了，所以写在return后的代码不会执行**

### 6.6 函数返回值之Note类型

```python
def say_hi():
    print("hello")

result = say_hi()
print(f"无返回值函数，返回的内容是：{result}")
print(f"无返回值函数，返回的内容类型是：{type(result)}")

"""
hello
无返回值函数，返回的内容是：None
无返回值函数，返回的内容类型是：<class 'NoneType'>
"""
```

None作为一个特殊的字面量，用于表示：空、无意义，有其非常多的应用场景。

- 用在函数无返回值上
- 用在if判断上
  - 在if判断中，None等同于False
  - 一般用于在函数中主动返回None，配合if判断做相关处理
- 用于声明无内容的变量上
  - 定义变量，但暂时不需要变量有具体值，可以用None来代替

### 6.7 函数的说明文档

```python
def add(x, y):
    """
    add函数可以接收两个参数，进行相加的功能
    :param x: 数字1
    :param y: 数字2
    :return: 返回相加的结果
    """
    result = x + y
    print(f"相加的结果是：{result}")
    return result
```

- param用于解释参数
- return用于解释返回值

### 6.8 函数的嵌套调用

```python
def func_b():
    print("---2---")

def func_a():
    print("---1---")
    func_b()
    print("---3---")

func_a()

"""
---1---
---2---
---3---
"""
```

### 6.9 变量在函数中的作用域

- 局部变量的作用：在函数体内部，临时保存数据，调用完成之后立即销毁变量
- 全局变量的作用：在函数体内外都能生效的变量
- **使用 global + 全局变量 的定义，可以使函数内的局部变量和全局变量联系在一起**

### 6.10 函数综合案例

```python
# 定义全局变量money name
money = 5000000
name = None

# 要求输入姓名
name = input("请输入姓名：")


# 定义查询函数
def query(show):
    if show:
        print("----------查询余额----------")
    print(f"{name},您好，您的余额为：{money}元。")


# 定义存款函数
def saving(num):
    global money  # money定义为全局变量
    money += num
    print("----------存款----------")
    print(f"{name},您好，存款{money}元成功。")
    # 调用查询余额函数
    query(False)


# 定义取款函数
def get_money(num):
    global money
    money -= num
    print(f"{name},您好，取款{num}元成功。")
    query(False)


def main():
    print("----------主菜单----------")
    print(f"{name},您好，欢迎来到ATM，请选择操作：")
    print("1.查询余额")
    print("2.存款")
    print("3.取款")
    print("4.退出")
    return input("请输入您的选择：")


while True:
    Keyboard_input = main()
    if Keyboard_input == "1":
        query(True)
        continue  # 通过continue进行下一次循环
    elif Keyboard_input == "2":
        num = int(input("输入您想存入的金额："))
        saving(num)
        continue
    elif Keyboard_input == "3":
        num = int(input("输入您想取出的金额："))
        get_money(num)
        continue
    else:
        print("退出程序……")
        break
```

## 7.数据容器

### 7.1 数据容器入门

**数据容器：一种可以容纳多分数据的数据类型，容纳每一份数据称之为1个元素；每一个元素，可以是任意类型的数据，如字符串，数字，布尔等。**

数据容器根据特点的不同：

- 是否支持重复元素
- 是否可以修改
- 是否有序等

**分为五类：列表（list）、元组（tuple）、字符串（str）、集合（set）、字典（dict）**

### 7.2 list列表的定义语法

列表的定义：

```python
# 字面量
变量名称 = [元素1, 元素2,...]

# 定义空列表
变量名 = []
变量名 = list()
```

列表内的没一个数据，称之为元素：

- 以[]为表示
- 列表内每一个元素之间用逗号隔开

```python
temp_list = ['no1','no2']
my_list = ['torch','tom','john',123,666,True,temp_list]
print(my_list)
print(type(my_list))


"""
['torch', 'tom', 'john', 123, 666, True, ['no1', 'no2']]
<class 'list'>
"""
```

注意：列表可以一次存储多个数据，且可以为不同的数据类型，支持嵌套

### 7.3 列表的下标索引

**使用下标索引，从列表中取出特定的数据**

```python
my_list = ['torch','tom','john',123,666,True,temp_list]
print(my_list[0])
print(my_list[2])
print(my_list[4])
# 负数是倒序输出
print(my_list[-2])
print(my_list[-4])
print(my_list[-6])
"""
torch
john
666
True
123
tom
"""
```

### 7.4 列表的常用操作方法

列表方法：

- 定义
- 用下标索引获取值
- 插入元素
- 删除元素
- 清空列表
- 修改元素
- 统计元素个数

**列表的查询功能（方法）**：函数是一个封装的代码单元，可以提供特定功能，如果将函数定义为class，那么函数会称之为：方法

**方法和函数功能一样，有传参和返回值，只是方法的使用格式不同：**

函数：num = add(1,2)

方法：student = Student()	num = student.add()

```python
temp_list = ['no1', 'no2']
my_list = ['torch', 'tom', 'john', 123, 666, True]

# 查找元素在列表内的下标索引值，如果不存在会报错ValueError
index = my_list.index('tom')
print(index)

# 修改特定下标索引的值
my_list[2] = "newName"
print(f"修改后的列表{my_list}")

# 指定下标位置插入新元素
my_list.insert(1, "insertName")
print(f"插入后的列表{my_list}")

# 在列表的尾部追加 单个 新元素
my_list.append("appendName")
print(f"追加后的列表{my_list}")

# 在列表的尾部追加 一批 元素
my_list.extend(temp_list)
print(f"追加新列表后的列表{my_list}")

# 删除指定下标的元素
del_list = ['no1','no2','no3','no4','no5']
print(f"原始列表{del_list}")
del del_list[2]
print(f"删除元素后的列表{del_list}")

element = del_list.pop(2)
print(f"通过pop删除了{element}元素，删除后的列表{del_list}")

# 删除元素在列表中第一个匹配项 remove相当于从前往后去搜索，只能删除一个符合的元素！！
del_list.remove("no5")
print(f"定向删除后的列表{del_list}")

# 清空列表
del_list.clear()
print(f"清空后的列表{del_list}")

# 统计列表中某个元素的数量
count_list = ['no1','no1','no1','no2','no2','no3']
count = count_list.count("no1")
print(f"列表中 no1 的数量为:{count}")

# 统计列表中全部元素的数量
all = len(count_list)
print(f"{count_list}列表一共有{all}个元素")

"""
1
修改后的列表['torch', 'tom', 'newName', 123, 666, True]
插入后的列表['torch', 'insertName', 'tom', 'newName', 123, 666, True]
追加后的列表['torch', 'insertName', 'tom', 'newName', 123, 666, True, 'appendName']
追加新列表后的列表['torch', 'insertName', 'tom', 'newName', 123, 666, True, 'appendName', 'no1', 'no2']
原始列表['no1', 'no2', 'no3', 'no4', 'no5']
删除元素后的列表['no1', 'no2', 'no4', 'no5']
通过pop删除了no4元素，删除后的列表['no1', 'no2', 'no5']
定向删除后的列表['no1', 'no2']
清空后的列表[]
列表中 no1 的数量为:3
['no1', 'no1', 'no1', 'no2', 'no2', 'no3']列表一共有6个元素
"""
```

|        使用方式         |                      作用                      |
| :---------------------: | :--------------------------------------------: |
|    list.append(元素)    |               向列表追加一个元素               |
|    list.extend(容器)    |     将数据容器内容依次取出，追加到列表尾部     |
| list.insert(下标，元素) |           在指定下标 插入指定的元素            |
|     del list[下标]      |              删除列表指定下标元素              |
|     list.pop(下标)      |              删除列表指定下标元素              |
|    list.remove(元素)    |       从前往后，删除此元素的第一个匹配项       |
|      list.clear()       |                    清空列表                    |
|    list.count(元素)     |          统计此元素在列表中出现的次数          |
|    list.index(元素)     | 查找指定元素在列表的下标，找不到报错ValueError |
|        len(list)        |              统计容器内有多少元素              |

**列表的特点：**

- 可以容纳多个元素（上限为2的63次方减一）
- 可以容纳不同类型的元素
- 数据是有序存储的
- 允许重复数据存在
- 可以修改

### 7.5 列表的常用功能练习

```python
age_list = [21,25,21,23,22,20]

age_list.append(31)
age_list.append([29,33,30])
age_list.pop(0)
age_list.pop(-1)
age_list.index(31)
```

### 7.6 列表的循环遍历

**while循环：**

```python
indes = 0
while index < len(列表)
	元素 = 列表[index]
    对元素进行处理
    index += 1
```

**for循环：**

```python
for 临时变量 in 数据容器:
    处理
```

```python
def list_while_func():
    """
    使用while循环遍历列表的演示函数
    :return:
    """
    my_list = ["torch","torchlee","hello"]
    index = 0
    while index < len(my_list):
        element = my_list[index]
        print(f"列表的元素:{element}")
        index += 1


def list_for_func():
    """
    使用for循环遍历列表的演示函数
    :return:
    """
    my_list = [1,2,3,4,5]
    for element in my_list:
        print(f"列表的元素有：{element}")


list_while_func()
list_for_func()
"""
列表的元素:torch
列表的元素:torchlee
列表的元素:hello
列表的元素有：1
列表的元素有：2
列表的元素有：3
列表的元素有：4
列表的元素有：5
"""
```

**while循环和for循环的对比：**

- 在循环控制上：
  - while循环可以自定循环条件，并自行控制
  - for循环不可以自定循环条件，只可以一个个从容器内取出数据
- 在无限循环上：
  - while循环可以通过条件控制做到无限循环
  - for循环理论上不可以，因为被遍历的容器容量不是无限的
- 在使用场景上：
  - while循环适用于任何想要循环的场景
  - for循环适用于，遍历数据容器的场景或简单的固定次数循环场景

取出列表内的偶数

```python
my_list = [1,2,3,4,5,6,7,8,9,10]
final_list = []

index = 0
while index < len(my_list):
    if my_list[index] % 2 == 0:
        final_list.append(my_list[index])
    index += 1

print(final_list)
"""
[2, 4, 6, 8, 10]
"""
```

### 7.7 元组的定义和操作

**定义元组：定义元组使用小括号，且使用逗号隔开各个数据，数据可以是不同的数据类型**

```python
(元素,元素,……….元素)
变量名 = ()
变量名 = tuple()
```

```python
# 定义元组
t1 = (1, "Hello", True)
t2 = ()
t3 = tuple()
print(f"t1的类型是{type(t1)},内容是{t1}")
print(f"t2的类型是{type(t2)},内容是{t2}")
print(f"t3的类型是{type(t3)},内容是{t3}")


# 定义单个元素的元组 注意一个数据的元组后面一定要加逗号！！
t4 = ("hello", )
print(f"t4的类型是{type(t4)},内容是{t4}")

# 元组的嵌套
t5 = ((1,2,3),(4,5,6))
print(f"t5的类型是{type(t5)},内容是{t5}")

# 下标索引取出内容
num = t5[1][2]
print(f"从元组中取出了:{num}")

# 元组的操作：index
t6 = ("no1","no2","no3")
index = t6.index("no3")
print(f"在元组t6中查找“no3”的下标是：{index}")

# 元组的操作：count
t7 = ("no1","no2","no2","no2","no2","no3")
num = t7.count("no2")
print(f"在元组t7中统计“no2”的数量为：{num}")

# 元组的操作：len函数统计元组元素数量
t8 = ("no1","no2","no2","no2","no2","no3")
print(f"t8元组中的元素有{len(t8)}个")

# 元组的遍历：while
index = 0
while index < len(t8):
    print(f"t8的元素有：{t8[index]}")
    index += 1

# 元组的遍历：for
for element in t8:
    print(f"t8的元素有：{element}")

"""
t1的类型是<class 'tuple'>,内容是(1, 'Hello', True)
t2的类型是<class 'tuple'>,内容是()
t3的类型是<class 'tuple'>,内容是()
t4的类型是<class 'tuple'>,内容是('hello',)
t5的类型是<class 'tuple'>,内容是((1, 2, 3), (4, 5, 6))
从元组中取出了:6
在元组t6中查找“no3”的下标是：2
在元组t7中统计“no2”的数量为：4
t8元组中的元素有6个
t8的元素有：no1
t8的元素有：no2
t8的元素有：no2
t8的元素有：no2
t8的元素有：no2
t8的元素有：no3
t8的元素有：no1
t8的元素有：no2
t8的元素有：no2
t8的元素有：no2
t8的元素有：no2
t8的元素有：no3
"""
```

**注意事项：**

- 不可以修改元组的数据，会报错！
- 可以修改元组内list的内容（修改元素、增加、删除、反转等）

**元组的特点：**

- 可以容纳多个数据
- 可以容纳不同类型的数据
- 数据使有序存储的
- 允许重复数据存在
- 不可以修改
- 支持for循环

```python
# 定义一个元组
stu_info = ('torch',11,['football','music'])

# 查询年龄下标位置
age = stu_info.index(11)
print(f"年龄下标位置为{age}")

# 查询姓名
print(f"姓名：{stu_info[0]}")

# 删除football
stu_info[2].remove('football')

# 增加coding爱好
stu_info[2].append('coding')
print(stu_info)

"""
年龄下标位置为1
姓名：torch
('torch', 11, ['music', 'coding'])
"""
```

### 7.8 字符串的定义和操作

**字符串的下标（索引）**

和其他容器，列表、元组一样，字符串也可以通过下标进行访问

- 从前向后，下标从0开始
- 从后向前，下标从-1开始

**同元组一样，字符串也是一个无法修改的数据容器**

**字符串的替换：**

- 语法：字符串.replace(字符串1，字符串2)
- 功能：将字符串内的全部：字符串1，替换为字符串2
- 注意：不是修改字符串本身，而是**得到了一个新字符串**

**字符串的分隔：**

- 语法：字符串.split(分隔符字符串)
- 功能：按照指定的分隔符字符串，将字符串划分为多个字符串，并存入列表对象中
- 注意：字符串本身不变，而是得到了一个列表对象

**字符串的规整操作：**

- 去前后空格：语法：字符串.strip()
- 去前后指定字符串：字符串.strip(字符串)

**其他操作：**

- 统计字符串内某字符串的出现次数：字符串.count(字符串)
- 统计字符串的字符个数：len(字符串)

**字符串的遍历：仍然是通过while和for来进行遍历**

```python
# 练习：分割字符串
str = "torch is a king king king"
print(str.count("king"))
print(str.replace(" ", "|"))
print(str.split("|"))
```

### 7.9 数据容器（序列）的切片

序列的常用操作——切片

序列支持切片，即：列表、元组、字符串，均支持进行切片操作

切片：从一个序列中，取出一个子序列

**语法：序列[起始下标:结束下标:步长]**

表示从序列中，从指定位置开始，依次取出元素，到指定位置结束，得到一个新序列：

- 起始下标表示从何处开始，可以留空，留空视作从头开始
- 结束下标（不含）表示何处结束，可以留空，留空视作截取到结尾
- 步长表示，依次取元素的间隔
  - 步长1表示，一个个取元素
  - 步长2表示，每次跳过一个元素取
  - 步长N表示，每次跳过N-1个元素取
  - 步长为负数表示，反向取（注意，起始下标和结束下标也要反向标记）

```python
# 对list进行切片，从1开始，4结束，步长1
my_list = [0, 1, 2, 3, 4, 5, 6]
result = my_list[1:4]
print(f"结果1：{result}")

# 对tuple进行切片
my_tuple = (0, 1, 2, 3, 4, 5, 6)
print(f"结果2：{my_tuple[:]}")

# 对str进行切片
my_str = "0123456"
print(f"结果3：{my_str[::2]}")
print(f"结果4：{my_str[::-1]}")

# 对列表进行切片
my_list = [0, 1, 2, 3, 4, 5, 6]
print(f"结果5：{my_list[3:1:-1]}")

# 对元组进行切片
my_tuple = (0, 1, 2, 3, 4, 5, 6)
print(f"结果6：{my_tuple[::-2]}")

"""
结果1：[1, 2, 3]
结果2：(0, 1, 2, 3, 4, 5, 6)
结果3：0246
结果4：6543210
结果5：[3, 2]
结果6：(6, 4, 2, 0)
"""
```

### 7.10 集合的定义和操作

```python
# 定义集合
my_set = {"no1","no2","no3","no1","no2","no3","no1","no2","no3"}
my_set_empty = set()
print(f"my_set的内容是：{my_set},类型是{type(my_set)}")
print(f"my_set_empty的内容是：{my_set_empty},类型是{type(my_set_empty)}")

my_set.add("no4")
my_set.remove("no1")
print(f"取出了：{my_set.pop()}")
my_set.clear()


"""
my_set的内容是：{'no1', 'no3', 'no2'},类型是<class 'set'>
my_set_empty的内容是：set(),类型是<class 'set'>
"""
```

**集合的常用操作——修改：**

首先，因为集合是无序的，所以集合不支持：下标索引访问，但是集合和列表一样，是允许修改的，所以我们来看看集合的修改方法

- 添加新元素：
  - 语法：集合.add(元素)，将指定元素添加到集合内
  - 结果：集合本身被修改，添加了新元素
- 移除元素：
  - 语法：集合.remove(元素)
  - 集合本身被修改，移除了元素
- 从集合中随机取出元素：
  - 语法：集合.pop(功能)，从集合中随机取出一个元素
  - 会得到一个元素的结果，同时集合本身被修改，元素被移除
- 清空集合：
  - 语法：集合.clear()
- 取两个集合的差集：
  - 语法：集合1.difference(集合2)，功能：取出集合1和集合2 的差集（集合1有而集合2没有的）
  - 得到一个新集合，集合1和集合2不变
- 消除两个集合的差集：
  - 语法：集合1.difference_update(集合2)
  - 集合1被修改，集合2不变
- 两个集合合并：
  - 语法：集合1.union(集合2)
  - 将集合1和集合2组成新集合，得到新集合，集合1和集合2不变
- 统计集合元素数量：len()
- **集合的遍历：可以用for循环**

集合的特点：

- 可以容纳多个数据
- 可以容纳不同类型的数据
- 数据是无序存储的
- 不允许重复数据存在
- 可以修改（增加或删除元素等）
- 支持for循环

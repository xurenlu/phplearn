一个PHP "bug"引发的思考

##Bug 示例
不废话，上代码一枚:
```php
<?php
$a = 1;
$b = & $a;
echo ++$a+$a++;
```
大家猜结果是什么?

如果你没有特别研究过,99%的人会说,结果是4.但是，运行一下吧。真正的结果是5.

怎么可能!PHP出错了吧!

没有的。PHP的特性决定了这个代码的运行结果就是5.一点也没错。下面我们一起来弄清楚，为什么结果是5.

## 从C语言的角度分析++a+a++
php是用C写的,并且常见运算符操作也是学习的C的。那么,如果是c,这段代码是如何运行的?
```
/** example.c */
#include<stdlib.h>
#include<stdio.h>
int main(int argc,void ** argv){
    int a=1;
    printf("%d\n",++a+a++);
}
```

运行一下:
```
$gcc ./example.c;./a.out
4
```
先复习一下:为什么是这个结果.
这里按运算符优先级和顺序，会先计算++a,再计算a++，最后把两个表达式的结果相加。
++a是把a+1,然后值返回。现在a是2,返回值也是2；a++是返回a的值,再把a+1;a是3,但是
返回值是2;所以最后是计算2+2,得到4.
php代码在不涉及到引用操作符(&)时也是这样的过程.

## 值变量与引用变量

要了解这个为什么加了$b=&$a结果就变成5了，就需要我们先了解一下,最关键的$b=&a
这一句做了些什么，也就是,&这个符号干了些什么;

看这两段代码:

+ 代码1:
```php
$a=1;
$b = &$a;
$b =2;
print $a."\n"; //2; a的值跟着b变了;
```

+ 代码2：

```php
$a=1;
$b = &$a;
$a =2;
print $b."\n"; //2; a的值跟着b变了;
```

这两段代码指出一个事实:当进行$b=&$a操作时,$a和$b都变成了指针。$a或$b任意一个变量的值变化
，另一个的值也会同步变化。PHP源码是如何做到这个的呢?

我们看看在PHP内部，变量是如何定义的.

##zval的定义:

(zend.h 287行)
```c
typedef struct _zval_struct zval;
```

(zend.h 318行)
```c
struct _zval_struct {
    /* Variable information */
	zvalue_value value;		/* value */
	zend_uint refcount__gc;
	zend_uchar type;	/* active type */
	zend_uchar is_ref__gc;
};
```
zval 结构体定义了refcount\_\_gc和is_ref\_\_gc两个成员变量.refcount\_\_gc标识这个zval被
多少个PHP变量引用.若refcount__gc值变为0,则会触发内存回收.而is_ref__gc则用来标识这个变
量是不是一个引用类型。当使用$b=&$a时,代表$a的zval的is_ref会被置为1，同时ref_count会加1.
下面给出《Thinking in PHP internal》一书里的一幅图:

![Alt text][1]


接下来，我们再看看$b=&$a执行后，如果unset掉一个变量,另一个会怎么样呢?
用代码验证:
```php
<?php
$a = 1;
$b = &$a;
unset($b);
echo $a;
```
执行一下:
```
$php var_mem.php
1
```
输出结果是1,证明,如果我们unset掉了$a,变量$b是还在的;这个我们可以推测出来,在unset掉其
一个一个变量时,另一个变量还存在,值不变;但是这个值还是一个引用类型的变量吗?
再看下面的代码:
```php
<?php
$a = 1;
$b = &$a;
unset($b);
echo $a."\n";
echo ++$a+$a++;
```
运行一下:
```
$php var_mem.php
1
4
```
我们看到，现在输出的++$a+$a++已经变成了4。$a不再是一个引用类型的变量了.
好了,现在我们应该能得到几个共识:

- $b=&$a; 将$a和$b内部的变量指向了同一个zval;这两个变量都变成了引用类型的变量,
- $b=&$a; 指向同一个zval后，$b或$a任意一个变量变化，另一个变量都变了;(因为本来就指向的是同一个zval)
- $b=&$a;unset($b);unset掉一个变量后,另一个的值还在,但是从引用类型又
  变回了值类型。ref_count引用计数也减1了

更多内容,请参见php官网上[PHP 引用](http://www.php.net/manual/zh/language.references.php)介绍.
## OP code 分析
有了上面的基础 ,现在我们开始剖析一下opcode,我们接下来构造这样两个文件,看看opcode有何不同;

left.php:
```php
<?php
$a = 1;
$b = 9999;
$c = (++$a) + ($a++);
echo $c; //4
```

right.php:
```php
<?php
$a = 1;
$b = &$a;
$c = (++$a) + ($a++);
echo $c;//5
```

我们导出opcode来看看二者有何不同.由于opcodesdumer和parsekit都不支持php5.4,所以用vld来做这个工作:
```
php -dvld.active=1 ./left.php > left.op 2>&1
php -dvld.active=1 ./right.php > right.op 2>&1
```

left.op:
```
function name:  (null)
number of ops:  12
compiled vars:  !0 = $a, !1 = $b, !2 = $c
line     # *  op                           fetch          ext  return  operands
---------------------------------------------------------------------------------
   2     0  >   EXT_STMT
         1      ASSIGN                                                   !0, 7
   3     2      EXT_STMT
         3      ASSIGN                                                   !1, 9999
   4     4      EXT_STMT
         5      PRE_INC                                          $2      !0
         6      POST_INC                                         ~3      !0
         7      ADD                                              ~4      $2, ~3
         8      ECHO                                                     ~4
   5     9    > RETURN                                                   1

16branch: #  0; line:     2-    5; sop:     0; eop:    9
path #1: 0,
```

right.op:
```
function name:  (null)
number of ops:  12
compiled vars:  !0 = $a, !1 = $b
line     # *  op                           fetch          ext  return  operands
---------------------------------------------------------------------------------
   2     0  >   EXT_STMT
         1      ASSIGN                                                   !0, 7
   3     2      EXT_STMT
         3      ASSIGN_REF                                               !1, !0
   4     4      EXT_STMT
         5      PRE_INC                                          $2      !0
         6      POST_INC                                         ~3      !0
         7      ADD                                              ~4      $2, ~3
         8      ECHO                                                     ~4
   5     9    > RETURN                                                   1

17branch: #  0; line:     2-    5; sop:     0; eop:    9
path #1: 0,
```
这两个php文件的opcode差另有两点:

1. 对应$b=9999 和$b=&$a这一行;就是在调ASSIGN_REF这个opcode时的操作数不同。

2. 倒数第二行,一个是16branch,一个是17branch.
从opcode看起来,差别不大,left.php和right.php似乎并不能导致++$a+$a++ 的结果有啥变化.
不过我们可以确定计算过程确实是:
>先计算++$a,计算结果赋值给$2这个临时变量,再计算$a++,计算结果赋值给~3这个环境变量.
>最后把$2,~3这两个环境变量的结果相加保存给~4这个环境变理再输出。

好了,再细心一点，我们还发能发现,opcode里，**++a和a++产生的中间变量的类型是不同的**。变量有!1,!2,!3,~1,~2,~3，还有$1,$2这么几种前缀。
!1,!2用来表示我们php中明确引用过的变量!0是$a,!1是$b...～1和~2,$2这些都是编译过程中产生的中间变量,$a++产生了一个叫~3的中间变量，
这个和add（加法运算）,echo操作产生的中间变量一样的,但是++$a却产生了一个$2的中间变量.这里面有什么文章么？
我们稍后再看。


看完opcode,我们只能找到两个点,一是branch的不同，一个是++a产生的临时变量的不同。
还是没有头绪。连为什么当$a是引用类型时,++$a+$a++的值会是5.断点调试无法打出$a++结果是什么,所以,我想试试这个方案是否可行:

```php
<?php
$a = 1;
$b = &$a;
echo (++$a). ($a++);
```
运行之:

```
$php concat.php
32
```
好了，鉴于这个concat.php的opcode和right.php的opcode除开一个是调用ADD操作
，一个是调用CONCAT操作，我们认为他们的执行流程是不变的，则可以认为++$a
返回的结果是3(在OPcode中的$2这个临时变量),$a++的结果是2,这样如果执行加法操作，确实得到5.

**现在为什么左侧的++$a 是3呢?**
我是这样推测的。引擎先执行++a,然后a的值变成了2,并把$a的指针存到了临时变量,接下来
计算a++,这时是先把a的值拷贝到临时变量,再把a的值改成3.接下来取两个临时变量相加，
由于加号左侧(++$a)对应的临时变量(在刚才的opcode中是用$2表示的，打开right.op复习一下:)是指向$a的指针
,此时已经随着a的值变成3了。所以最终参与加法计算的是一个最终指向$a的指针代表的变量，一个是
另一个是值为2的临时变量.
好了，现在要让这个推测是正确的，就需要证明：++$a运算产生的临时变量($2)是指向$a的指针,而
$a++运算产生的临时变量(~3,参见right.op)是存储了2这个值.虽然我们从OPcode中看到++$a产生的
$2变量的表示方式和其他的都不同,可是我们没有弄明白为什么不同啊。

## ++$a和$a++的内存占用

我想了一个办法来证明,那就是，如果++$a产生的临时变量只是指向$a的指针,而$a++是产生了一个
变量来真正地存储值,那么,++$a耗费的内存,应该比$a++耗费的内存要小。验证一下:

postinc.php
```php
<?php
gc_disable();
echo (memory_get_usage())."\n";
$a=1;
echo (memory_get_usage())."\n";
$b=$a++;
echo (memory_get_usage())."\n";
```
```
230856
231048
231192
```
计算一下差值 ,即分别得到$a=1和$b=$a++的内存开销:
```
192
144
```

preinc.php
```php
<?php
gc_disable();
echo (memory_get_usage())."\n";
$a=1;
echo (memory_get_usage())."\n";
$b=++$a;
echo (memory_get_usage())."\n";
```
执行结果:
```
230856
231048
231144
```

再看看查值,得到$a=1和$b=++$a的内存开销 :
```
192
96
```

可以看到，执行 ``` $b=++$a ``` 引起了96个字节的内存开销,
而执行``` $b=$a++ ``` 引起了144个字节的内存开销,似乎可以从
侧面印证++$a运算中的临时变量比$a++引起的临时变量更省内存的。

## 从zend 源代码看++$a 和$a++的根源

接下来，我们看一下++$a 和 $a++ 实际执行的代码的不同。
通过查阅PHP源码，我们找到了这两段代码实际对应的代码:

```c
/** 对应 ++a 操作 */
static int ZEND_FASTCALL  ZEND_PRE_INC_SPEC_CV_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    USE_OPLINE

	zval **var_ptr;

	SAVE_OPLINE();
	var_ptr = _get_zval_ptr_ptr_cv_BP_VAR_RW(EX_CVs(), opline->op1.var TSRMLS_CC);

	if (IS_CV == IS_VAR && UNEXPECTED(var_ptr == NULL)) {
		zend_error_noreturn(E_ERROR, "Cannot increment/decrement overloaded objects nor string offsets");
	}
	if (IS_CV == IS_VAR && UNEXPECTED(*var_ptr == &EG(error_zval))) {
		if (RETURN_VALUE_USED(opline)) {
			PZVAL_LOCK(&EG(uninitialized_zval));
			AI_SET_PTR(&EX_T(opline->result.var), &EG(uninitialized_zval));
		}

		CHECK_EXCEPTION();
		ZEND_VM_NEXT_OPCODE();
	}

	SEPARATE_ZVAL_IF_NOT_REF(var_ptr);

	if (UNEXPECTED(Z_TYPE_PP(var_ptr) == IS_OBJECT)
	   && Z_OBJ_HANDLER_PP(var_ptr, get)
	   && Z_OBJ_HANDLER_PP(var_ptr, set)) {
		/* proxy object */
		zval *val = Z_OBJ_HANDLER_PP(var_ptr, get)(*var_ptr TSRMLS_CC);
		Z_ADDREF_P(val);
		fast_increment_function(val);
		Z_OBJ_HANDLER_PP(var_ptr, set)(var_ptr, val TSRMLS_CC);
		zval_ptr_dtor(&val);
	} else {
		fast_increment_function(*var_ptr);
	}

	if (RETURN_VALUE_USED(opline)) {
		PZVAL_LOCK(*var_ptr);
		AI_SET_PTR(&EX_T(opline->result.var), *var_ptr);
	}

	CHECK_EXCEPTION();
	ZEND_VM_NEXT_OPCODE();
}

```


对应$a++的c代码(在OPCODE里命名为POST_INC);
```c
/** 对应$a++ */
static int ZEND_FASTCALL  ZEND_POST_INC_SPEC_CV_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    USE_OPLINE

	zval **var_ptr, *retval;

	SAVE_OPLINE();
	var_ptr = _get_zval_ptr_ptr_cv_BP_VAR_RW(EX_CVs(), opline->op1.var TSRMLS_CC);

	if (IS_CV == IS_VAR && UNEXPECTED(var_ptr == NULL)) {
		zend_error_noreturn(E_ERROR, "Cannot increment/decrement overloaded objects nor string offsets");
	}
	if (IS_CV == IS_VAR && UNEXPECTED(*var_ptr == &EG(error_zval))) {
		ZVAL_NULL(&EX_T(opline->result.var).tmp_var);

		CHECK_EXCEPTION();
		ZEND_VM_NEXT_OPCODE();
	}

	retval = &EX_T(opline->result.var).tmp_var;
	ZVAL_COPY_VALUE(retval, *var_ptr);
	zendi_zval_copy_ctor(*retval);

	SEPARATE_ZVAL_IF_NOT_REF(var_ptr);

	if (UNEXPECTED(Z_TYPE_PP(var_ptr) == IS_OBJECT)
	   && Z_OBJ_HANDLER_PP(var_ptr, get)
	   && Z_OBJ_HANDLER_PP(var_ptr, set)) {
		/* proxy object */
		zval *val = Z_OBJ_HANDLER_PP(var_ptr, get)(*var_ptr TSRMLS_CC);
		Z_ADDREF_P(val);
		fast_increment_function(val);
		Z_OBJ_HANDLER_PP(var_ptr, set)(var_ptr, val TSRMLS_CC);
		zval_ptr_dtor(&val);
	} else {
		fast_increment_function(*var_ptr);
	}

	CHECK_EXCEPTION();
	ZEND_VM_NEXT_OPCODE();
}

```

接下来我们解读一下上面两代c代码。事实证明,$a++ 是生成了临时变量，将结果拷贝到临时变量
，然后再对原操作数进行了+1操作(而且还是优先用汇编来实现的);设置返回值的代码是这段:

```c
    retval = &EX_T(opline->result.var).tmp_var;
	ZVAL_COPY_VALUE(retval, *var_ptr);
	zendi_zval_copy_ctor(*retval);
```

而++$a对应的逻辑，是先将变量所在的值修改,接下来，如果返回值被使用的话，将结果变量指向操作数自己;设置结果的代码如下:

```c
    if (RETURN_VALUE_USED(opline)) {
		PZVAL_LOCK(*var_ptr);
		AI_SET_PTR(&EX_T(opline->result.var), *var_ptr);
	}
```
PZVAL\_LOCK这一句是修改了var\_ptr的引用计数。【什么时候再释放这个引用计数呢,回头再查查】
双方设置返回值的方式确实不同，$a++是拷贝值，而++$a是返回变量。并且$a++是一定设置返回值的，
而++$a则不一定。
接下来则需要解决另外一个疑问，当$a不是引用的时候(没有用$b=&$a这一句时候),为啥没有出现
那个问题呢?
我们注意到两者的c代码中有这么一句:

```c
SEPARATE_ZVAL_IF_NOT_REF(var_ptr);
```

这一句相关的宏展开是:

```c
#define SEPARATE_ZVAL(ppzv)    					\
	do {										\
		if (Z_REFCOUNT_PP((ppzv)) > 1) {		\
			zval *new_zv;						\
			Z_DELREF_PP(ppzv);					\
			ALLOC_ZVAL(new_zv);					\
			INIT_PZVAL_COPY(new_zv, *(ppzv));	\
			*(ppzv) = new_zv;					\
			zval_copy_ctor(new_zv);				\
		}										\
	} while (0)

#define SEPARATE_ZVAL_IF_NOT_REF(ppzv)		\
	if (!PZVAL_IS_REF(*ppzv)) {				\
		SEPARATE_ZVAL(ppzv);				\
	}
```

可以看到,当变量不是引用类型的变量时，zend在实际计算求值之前，会先生成了一个新的变量，然后
把旧变量的值拷贝给新的变，再参与计算。（这实际上是php的写时拷贝特性）。正好解释了，当
$a不是引用型变量时，为什么不会出问题。


##扩展思考:表达式长度更长时,会多次出现这个"bug"吗?
刚刚我们想明白了<?php $a=1;$b=&$a;echo ++$a+$a++;结果为什么是5.现在再看一看，
当类似的表达式长度扩展到更长时，会是怎样的?

```php
<?php
/** file:example_more_add_alg.php */
$a = 1;
$b = & $a;
echo ++$a.$a++.++$a.$a++;
```
出乎意料的是,现在,输出的结果是"**3244**",只有第一个++$a多了一次++运算，
后续的部分，并未出现类似的问题呢?我一开始没有看zend源代码的时候，推测是因为,
PHP每次计算都只能对两个操作数做计算。并且，在有些时候,当表达式扩展得更长的时候,他不是先
根据优先级，先把所有的++操作都计算了，然后最后再拼接。而是，计算一下右侧的一个
++表达式的值，然后和左侧已经计算的部分拼接，存成一个临时变量（存的是值，不是指针）
，然后再取右侧的一个++表达式出来计算，然后再和左侧的一起做一次拼接计算。依次类推。
列出来就是:
```
第一次运算 ++$a.$a++.++$a.$a++
第二次运算 temp_pointer($a).$a++.++$a.$a++
第三次运算 temp_pointer($a).$temp_var1. ++$a.$a++
第四次运算 $temp_var2.++$a.$a++;
第五次运算 $temp_var2.temp_pointer($a).$a++;
第六次运算 $temp_var3.$a++
第七次运算 $temp_var3.$temp_var4
```

是不是这样呢，我们再看一下opcode:
```
php -dvld.active=1 ./example_more_add_alg.php 
compiled vars:  !0 = $a, !1 = $b
line     # *  op                           fetch          ext  return  operands
---------------------------------------------------------------------------------
   3     0  >   EXT_STMT
         1      ASSIGN                                                   !0, 1
   4     2      EXT_STMT
         3      ASSIGN_REF                                               !1, !0
   5     4      EXT_STMT
         5      PRE_INC                                          $2      !0
         6      POST_INC                                         ~3      !0
         7      CONCAT                                           ~4      $2, ~3
         8      PRE_INC                                          $5      !0
         9      CONCAT                                           ~6      ~4, $5
        10      POST_INC                                         ~7      !0
        11      CONCAT                                           ~8      ~6, ~7
        12      ECHO                                                     ~8
   6    13    > RETURN                                                   1

branch: #  0; line:     3-    6; sop:     0; eop:    13
path #1: 0,
```
我们可以注意到，标号为5号的opcode是PRE\_INC,就是++$a操作,紧跟着的#6 opcode是POST\_INC，也就是
$a++,接下来才执行CONCAT也就是字符串拼接操作。但是接下来，却是按步就班的执行一个PRE\_INC或
POST\_INC,然后紧接着就执行一个CONCAT;也许您没有意识到这里问题。那我们想一想PHP的
运算符的执行顺序,++操作是优先于+和"."的！（打开 http://www.php.net/manual/zh/language.operators.precedence.php 复习一下！）
按说应该执行完所有的++$a操作(PRE_INC)，再执行加号啊.

类似的“不遵守运算符优先级”的例子，在php管方也给出过一个:
```php
if (!$a = foo())
```
这时,运算符优先级是先计算foo(),再生效“=”,将foo()的值传给$a,最后生效“!”.此时没有遵守
运算符的优先级.官方手册给出的优先级是,等号的优先级是最低的。

## 扩展思考之二:PHP opcode中,运算符最多是二元的吗?
现在，再来看一下opcode在zend中的定义，这个定义也证明了PHP计算最多二元的。
```c
/** php5.4.18 Zend/zend_compile.h 106行起 */
struct _zend_op {
    opcode_handler_t handler;
	znode_op op1;
	znode_op op2;
	znode_op result;
	ulong extended_value;
	uint lineno;
	zend_uchar opcode;
	zend_uchar op1_type;
	zend_uchar op2_type;
	zend_uchar result_type;
};
```
这里定义的_zend_op 确定只有两个操作数。那么,PHP的kwyg三元操作符是如何实现的呢？
我们查php关于运算符号的优先级的手册(http://www.php.net/manual/zh/language.operators.precedence.php)
能找到的三元运算符只有一个,"?:",让我们写个代码，读读他的opcode;
```php
<?php
$a = 1;
$b = $a?$a:2;
```
这段代码的opcode是:
```
compiled vars:  !0 = $a, !1 = $b
line     # *  op                           fetch          ext  return  operands
---------------------------------------------------------------------------------
   8     0  >   EXT_STMT
         1      ASSIGN                                                   !0, 1
   9     2      EXT_STMT
         3    > JMPZ                                                     !0, ->6
         4  >   <157>                                            $1      !0
         5    > JMP                                                      ->7
         6  >   <157>                                            $1      2
         7  >   ASSIGN                                                   !1, $1
  20     8    > RETURN                                                   1

branch: #  0; line:     8-    9; sop:     0; eop:     3; out1:   4; out2:   6
branch: #  4; line:     9-    9; sop:     4; eop:     5; out1:   7
branch: #  6; line:     9-    9; sop:     6; eop:     6; out1:   7
branch: #  7; line:     9-   20; sop:     7; eop:     8
path #1: 0, 4, 7,
path #2: 0, 6, 7,
```
可以看到，一共产生了9个opcode，并且是用了两个跳转语句来实现这个三元计算符的。



[1]: http://share.162cm.net/files/201308/80313400.png "说明"



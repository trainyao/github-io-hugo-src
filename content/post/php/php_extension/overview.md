

@(PHP)
###PHP扩展分享
###什么时候需要写php扩展
* 有需要使用php调用C/C++函数库
* 实现php没有的功能
* 优化php性能

###php扩展简介
php提供一个工具用来生成php扩展的代码,类似代码生成器,工具目录`source/to/php_source_code/ext/ext_skel`,这个是供类unix平台调用的脚本,php源码也提供了一个供windows平台调用的脚本,用来生成php扩展代码.该脚本是个php脚本,在`source/to/php_source_code/ext/ext_skel_win32.php`
	
php将扩展代码统一放到`source/to/php_source_code/ext/`下,这样当开发者按照工具生成对应格式的php扩展源码,并统一放到这个目录下,php源码就会自动识别,并可以选在在编译时加上开发者的扩展.
###先找个题目
总所周知php在连接字符串会进行多次初始化临时变量,和内存复制.假设有一个变态的需求需要重复一个字符串'a' n次,并且在系统中多次出现,由于n已知,为了提高性能,可以考虑用扩展实现
###动手
####1. 编写扩展原型文件,使用工具生成扩展代码
	
php没有规定原型文件的**文件后缀**,但有规定原型文件的**格式**.
如果你的扩展需要暴露出接口函数供php代码调用,原型文件只需包含接口函数的原型.接口函数原型包括函数输入参数的类型和名字,还有函数返回的类型.其中类型支持php的变量类型.如bool,int,string,float等等..
根据上面的需求,创建一个名为`repeatString`的函数,返回类型为`string`,接受一个`string`类型的`$inputString`字符串,和一个`int`类型的`$repeatCount`,表示输出`$inputString`被重复`$repeatCount`次的字符串.
首先进入ext目录		
	
```
cd path/to/php_source_code/ext && \
vim repeatString.proto
```

编写内容
```
string repeatString(string inputString, int repeatCount)
```
保存,使用ext_skel工具生成扩展框架,可以指定扩展名和扩展的原型文件
```
./ext_skel --extname=repeatString --proto=repeatString.proto
```
你会看到ext目录下会多了一个`repeatString`的文件夹,里面放的就是你扩展的所有所需文件
需要留意的有几个文件:

* config.m4 -- *unix平台下配置文件,这里是官方解释:

> 扩展的 config.m4 文件告诉 UNIX 构建系统哪些扩展 configure 选项是支持的，你需要哪些扩展库，以及哪些源文件要编译成它的一部分。对所有经常使用的 autoconf 宏，包括 PHP 特定的及 autoconf 内建的，请查看 Zend Engine 2 API 参考 章节。

* config.w32 -- windows平台下扩展的配置文件
* php_repeatString.h -- 扩展头文件
* repeatString.c -- 扩展核心代码c文件
* tests -- 单元测试文件
	
####2. 编写核心代码
进入repeatString.c文件,可以看到这个文件都做了些什么:
* 引用了扩展必要的头文件
* 保证`php_repeatString.h`头文件内定义的全局变量线程安全
* 定义扩展所需的ini配置变量名
* **实现扩展方法  -- 我们接下来重点关注这个**
* 实现MINIT MSHUTDOWN RINIT RSHUTDOWN 方法 
* 定义扩展方法入口 和 扩展初始化方法入口
可以看到我们需要实现的方法`PHP_FUNCTION(repeatString)`, 它是一个宏,是php提供的一个接口,用来定义php接口函数.我们在里面实现我们需要的逻辑:

```
PHP_FUNCTION(repeatString)
{
        char *inputString = NULL;
        int argc = ZEND_NUM_ARGS();
        size_t inputString_len;
        zend_long repeatCount;
        if (zend_parse_parameters(argc, "sl", &inputString, &inputString_len, &count) == FAILURE)
                return;
        char * repeatBuffer = (char *)emalloc(inputString_len * count + 1);
        char * pointer = repeatBuffer;
        while(count--) {
                memcpy(pointer, inputString, inputString_len);
                pointer += inputString_len;
        }
        *pointer = '\0';
        RETURN_STRINGL(repeatBuffer, repeatedLength);
}
```
	
####3. 让php加载扩展
	
php加载php扩展的方式有2种:
* 作为扩展模块,写进php.ini内,在php启动时加载`.so`文件
* 在编译php期间加入
我们先用第二种方式实现,首先要让```./configure```文件找到我们的模块,进入```config.m4```文件,可以看到这两行代码控制着```./configure```命令是否能传参```--enable-repeatString```,从而将我们的扩展编译进php,把注释去掉实现我们的目的
```
dnl "dnl"表示注释,去掉第一第三行的`"dnl"
dnl PHP_ARG_ENABLE(repeatString, whether to enable repeatString support,
dnl Make sure that the comment is aligned:
dnl [  --enable-repeatString           Enable repeatString support])
```
退回到php源码根目录,重新生成配置文件(```./buildconf```需要安装autoconf)
```
cd ../../ && ./buildconf
```
由于我用的是git pull下来的仓库,这里会提示git仓库不应该执行```./buildconf```命令,应该在release包执行,我们加上```--force```就好
```
./buildconf --force w
```
这个命令将会重新生成```./configure```命令,输入
```
./configure --help | grep repeat
```
可以看到,我们的扩展已经能被```./configure```认出来,运行
```powershell
./configure --disable-all --enable-repeatString && make
```
重新编译php,接下来就可以测试一下我们的扩展了
```php
<?php
$input = 'input';
$count = 3;
var_dump(repeatString($input, $count));
```
###说一些细节
####1. php接口函数解析参数
php提供方法```zend_parse_parameters```对php接口函数期待的参数进行获取,判断类型和初始化.例如在:
	
```
if (zend_parse_parameters(argc, "sl", &inputString, &inputString_len, &count) == FAILURE)
		return;
```
	
中,该方法接受一个```argc```参数,表示接口函数有几个期待参数;
	
第二个参数```"sl"```表示接口函数第一个参数期待```string```类型,第二个参数期待```long int```类型,在使用```ext_skel```生成扩展源码时,我们的原型文件指定了参数的类型,所以这里的```"sl"```是对应生成的
	
后面的参数是接口函数的参数的容器,当```zend_parse_parameters```方法执行完之后,php会将php脚本解析的zend_value变量对应的值传入容器,我们在c代码里面就可以进行使用了.
	
我们甚至可以在初始化参数时进行判断,来做一些针对不同传参的不同的逻辑.比如传一个字符串一个整形时做些什么,传两个都是字符串类型时又做些什么.
	
对于`"sl"`更多类型可以在[这里](http://php.net/manual/zh/internals2.funcs.php)看到,或者在PHP源码包的`README.PARAMETER_PARSING_API`文件内有详尽的描述
	
####2. php API
可以看到我们在实现```repeatString```扩展时用到了`emalloc`方法来分配内存,其实这是一个类似c函数`malloc`的宏,主要实现了php内存管理.也就是说,用php提供的接口函数去分配内存,不用担心出现内存泄露的问题,在php结束的时候,该部分内存会自动进行释放.这部分是php这个框架帮我们做的
	
更多的phpAPI方法/宏可以看[这里](http://php.net/manual/zh/internals2.ze1.zendapi.php).
	
####3. php接口函数返回值
可以看到我们在接口函数结尾返回的时候用了`RETURN_STRINGL`宏,这也是php提供的一个接口,可以将c变量封装回zend_value返回到php脚本,或者说php执行环境中.
	
####4. 扩展全局变量
刚才说到repeatString.c做了一个动作: "保证`php_repeatString.h`头文件内定义的全局变量线程安全",具体实现的方法只有一句:
	
```c
/* If you declare any globals in php_repeatString.h uncomment this:
ZEND_DECLARE_MODULE_GLOBALS(repeatString)
*/
```
	
这里有句注释说,如果在`php_repeatString.h`定义了全局变量,取消这一句注释,看一下这个宏做了些什么:
	
```c
// zend_API.h
#ifdef ZTS
// 这里
#define ZEND_DECLARE_MODULE_GLOBALS(module_name)							\
// 初始化一个模块-线程id
	ts_rsrc_id module_name##_globals_id;
#define ZEND_EXTERN_MODULE_GLOBALS(module_name)								\
	extern ts_rsrc_id module_name##_globals_id;
// 调用这个宏分配一个模块-线程id,并注册模块全局变量构造函数和析构函数
#define ZEND_INIT_MODULE_GLOBALS(module_name, globals_ctor, globals_dtor)	\
	ts_allocate_id(&module_name##_globals_id, sizeof(zend_##module_name##_globals), (ts_allocate_ctor) globals_ctor, (ts_allocate_dtor) globals_dtor);
// 这个宏用于取全局变量的值,当多线程模式时,根据生成的模块-线程id获取全局变量的值
#define ZEND_MODULE_GLOBALS_ACCESSOR(module_name, v) ZEND_TSRMG(module_name##_globals_id, zend_##module_name##_globals *, v)
#define ZEND_MODULE_GLOBALS_BULK(module_name) TSRMG_BULK(module_name##_globals_id, zend_##module_name##_globals *)
#else
// 非多线程模式下,只简单的初始化一个全局变量结构体
#define ZEND_DECLARE_MODULE_GLOBALS(module_name)							\
	zend_##module_name##_globals module_name##_globals;
#define ZEND_EXTERN_MODULE_GLOBALS(module_name)								\
	extern zend_##module_name##_globals module_name##_globals;
// 非多线程模式下,只简单调用构造函数对结构体初始化
#define ZEND_INIT_MODULE_GLOBALS(module_name, globals_ctor, globals_dtor)	\
	globals_ctor(&module_name##_globals);
	
// 非多线程模式下,访问全局变量只是简单读取结构体内的值
#define ZEND_MODULE_GLOBALS_ACCESSOR(module_name, v) (module_name##_globals.v)
#define ZEND_MODULE_GLOBALS_BULK(module_name) (&module_name##_globals)
#endif
```
看了几个php源码自带的php扩展,多数都是在`MINIT`方法里调用`ZEND_INIT_MODULE_GLOBALS`方法初始化全集变量,而`ext_skel`框架却没有生成备注的`ZEND_INIT_MODULE_GLOBALS`代码,比较奇怪.
###总结
1. 一般写php扩展,目的可能是: 做php做不到的东西,或者想优化php性能
2. php提供一个框架让我们一键生成php扩展所需的源码文件
3. php扩展生效有两种方式: 1: 编译时加入 2: 配置文件php.ini运行时加载动态库

资料地址:

> [用C/C++扩展你的PHP](http://www.laruence.com/2009/04/28/719.html)
	
补充: 
分享时出的问题,忘了原来是php提供的扩展生成工具的shell脚本有bug,具体可以看[这里](https://segmentfault.com/q/1010000004493105)
	
    

---
layout: post
title: php内核学习（1）-生命周期与执行过程
date: 2016-03-28
categories: php
tags: [php内核]
description: 总结下php源码学习过程中的一些点
---

## PHP四层架构

开始之前，先看看PHP的核心架构，如图：

![](http://emonmit.github.io/img/php_struct.jpg)

我们自顶向下的来看这架构：

* 首先是Application，就是我们的上层应用——平时写的PHP程序，可以是web应用或者php脚本；
* 接着是SAPI（Server Application Programming Interface），服务端应用编程接口，sapi通过一系列钩子函数，使得php可以和外围交互数据，并且通过`sapi成功的将php本身和上层应用解耦隔离`，php可以不再考虑如何针对不同的应用进行兼容，而应用本身也可以针对自己的特点实现不同的处理方式；
* 再下来是Extensions——扩展层，围绕着Zend引擎，扩展层通过组件式的方式提供各种基础服务，我们常见的各种内置函数（如array系列）、标准库等都是通过扩展来实现，我们自己也能自定义一些扩展来达到扩展功能、优化性能等目的；
* 最下面就是Zend了，整体用纯c实现，是php的内核部分，在这一层，`它将php代码翻译成opcode并提供相应的处理方法、实现了基本的数据结构、内存分配及管理、提供了相应的api方法工外部调用，时一切的核心，所有的外围功能均围绕zend实现。`

## PHP的生命周期

结合php四层架构，我们来看一看一个php程序从执行开始都需要经历什么。

最常见的四种启动php的方式如下：

- 直接以CLI/CGI模式调用
- 多线程模块
- 多进程模块
- Embedded（嵌入式，在自己c程序中调用Zend Engine）

无论用哪种方式启动的，除了执行脚本本身逻辑之外，都会依次经过`Module init、Request init、Request shutdown、Module shutdown`四个过程。两种init和两种shutdown各会执行多少次，各自执行的频率有多少，取决于php是用什么SAPI与宿主通信的。

以命令行运行一个PHP程序为例：

![](http://emonmit.github.io/img/php_lifespan.png)

图中可以看出，在命令行敲下 “php -f test.php” 之后，会有这些操作：

- **MINIT** 这个过程在扩展被载入时调用，是`模块初始化阶段（MINIT）`，回调所有模块的MINIT函数，模块在这个阶段可以进行一些初始化工作，如注册常量，定义模块使用的类等。在整个SAPI生命周期内，该过程`只进行一次`。
- **RINIT** 每次请求之前，都会进行`模块激活`（RINIT请求开始），当请求到达以后，PHP会初始化执行脚本的基本环境，例如创建一个执行环境，包括保存PHP运行过程中变量名称和变量值内容的符号表，以及当前所有的函数以及类等信息的符号表。然后PHP会调用所有模块的RINIT函数，在这个阶段，各个模块也可以执行一些相关的操作。一个经典的例子是Session模块的RINIT，如果在php.ini中启用了Session模块，那在调用该模块的RINIT时就会初始化$_SESSION变量，并将相关内容读入；RINIT方法可以看作是一个准备过程， 在程序执行之间就会自动启动。
- **Execute test.php**  执行test.php阶段，主要是把PHP文件翻译成Opcodes，然后再PHP虚拟机下执行。
- **RSHUTDOWN & MSHUTDOWN** 	请求处理完后就进入了结束阶段，一般脚本执行到末尾或者通过调用exit()或die()函数， PHP都将进入结束阶段。和开始阶段对应，结束阶段也分为两个环节，一个在请求结束后停用模块(RSHUWDOWN，对应RINIT)， 一个在SAPI生命周期结束（Web服务器退出或者命令行脚本执行完毕退出）时关闭模块(MSHUTDOWN，对应MINIT)。

## Execute *.php

现在我们来深究下php文件的执行，我们都知道，编程语言最终转化成能被计算机理解的汇编语言要经过语法分析、词法分析、生成中间代码、生成目标代码等过程。PHP也不例外，先看一张php编译的流程图：

![](http://emonmit.github.io/img/opcode.jpg)

由图可见，经过词法分析、语法分析之后，`最终只生成了中间代码opcode（PHP的一种内部数据结构）`，并没有继续生成目标代码，故而PHP也被称为解释型语言。

下面我们再来详细看下从源程序生成opcode的过程。
以`Hello World!`为例：

	<?php
	    echo "Hello World!";
	?>

### Scanning（Lexing）,将PHP代码转换为语言片段（Tokens）
PHP原来使用的是Flex，之后改为re2c，源码目录下的*Zend/zend_language_scanner.l*是re2c的规则文件，如果安装了re2c的话，我们还可以修改并生成新的规则文件。*Zend/zend_language_canner.c*会根据规则文件，来对输入的PHP代码进行词法分析，从而得到一个一个的**词**。

	<?php
	    $code = '<?php echo "Hello World!"; ?>';
	    $tokens = token_get_all($code);
	    foreach($tokens as $key => $token) { 
	        $tokens[$key][0] = token_name($token[0]);
		}
		var_dump($tokens);
	?>

这里用`token_name`函数将解析器代号修改成了符号名称说明，更加容易理解，关于解释器代号列表可以参见[手册](http://php.net/manual/zh/tokens.php)。

执行结果如下：

	array(7) {
	  [0]=>
	  array(3) {
	    [0]=>string(10) "T_OPEN_TAG"
	    [1]=>string(6) "<?php "
	    [2]=>int(1)
	  }
	  [1]=>
	  array(3) {
	    [0]=>string(6) "T_ECHO"
	    [1]=>string(4) "echo"
	    [2]=>int(1)
	  }
	  [2]=>
	  string(1) ""
	  [3]=>
	  array(3) {
	    [0]=>string(26) "T_CONSTANT_ENCAPSED_STRING"
	    [1]=>string(14) ""Hello World!""
	    [2]=>int(1)
	  }
	  [4]=>
	  string(1) ""
	  [5]=>
	  array(3) {
	    [0]=>string(12) "T_WHITESPACE"
	    [1]=>string(1) " "
	    [2]=>int(1)
	  }
	  [6]=>
	  array(3) {
	    [0]=>string(11) "T_CLOSE_TAG"
	    [1]=>string(2) "?>"
	    [2]=>int(1)
	  }
	}

我们可以发现，源码中的`空格也会原样返回`，而且他的，都被转换成一个包含三部分（`解释器代号、源码中的原内容、源码中第几行`）的数组。

### Parsing, 将Tokens转换成简单而有意义的表达式
语法分析阶段首先会丢弃Tokens Array中多余的空格，然后将剩余的Tokens转换成一个个的表达式。在PHP源码中，词法分析器最终调用的是re2c规则定义的`lex_scan`函数，而提供给Bison的函数则为zendlex。 而yyparse被zendparse代替。

	> Bison是一种通用目的的分析器生成器。它将LALR(1)上下文无关文法的描述转化成分析该文法的C程序。 使用它可以生成解释器，编译器，协议实现等多种程序。 Bison向上兼容Yacc，所有书写正确的Yacc语法都应该可以不加修改地在Bison下工作。 它不但与Yacc兼容还具有许多Yacc不具备的特性。

	> Bison分析器文件是定义了名为yyparse并且实现了某个语法的函数的C代码。 这个函数并不是一个可以完成所有的语法分析任务的C程序。 除此这外我们还必须提供额外的一些函数： 如词法分析器、分析器报告错误时调用的错误报告函数等等。 我们知道一个完整的C程序必须以名为main的函数开头，如果我们要生成一个可执行文件，并且要运行语法解析器， 那么我们就需要有main函数，并且在某个地方直接或间接调用yyparse，否则语法分析器永远都不会运行。

### Compilation, 将表达式编译成Opocdes

在该阶段，Tokens被编译成一个个op_array，之前说了opcode其实是php内部实现的一种数据结构，在源码中，可以看到（version : php5.6）：

	struct _zend_op {
		opcode_handler_t handler; //执行时调用的处理函数
		znode_op op1; //操作数1
		znode_op op2; //操作数2
		znode_op result; //结果
		ulong extended_value; //额外的信息
		uint lineno; //源码中的行数
		zend_uchar opcode; //opcode代码
		zend_uchar op1_type; //操作数1类型
		zend_uchar op2_type; //操作数1类型
		zend_uchar result_type; //结果类型
	};

而编译完成的opcode是存在op_array中的，看下op_array的源码：

	struct _zend_op_array {
	/* Common elements */
	zend_uchar type;
	const char *function_name; // 如果是用户定义的函数则，这里将保存函数的名字
	zend_class_entry *scope;
	zend_uint fn_flags;
	union _zend_function *prototype;
	zend_uint num_args;
	zend_uint required_num_args;
	zend_arg_info *arg_info;
	/* END of common elements */

	zend_uint *refcount;

	zend_op *opcodes; // opcode数组
	zend_uint last;

	zend_compiled_variable *vars;
	int last_var;

	zend_uint T;

	zend_uint nested_calls;
	zend_uint used_stack;

	zend_brk_cont_element *brk_cont_array;
	int last_brk_cont;

	zend_try_catch_element *try_catch_array;
	int last_try_catch;
	zend_bool has_finally_block;

	/* static variables support */
	HashTable *static_variables;

	zend_uint this_var;

	const char *filename;
	zend_uint line_start;
	zend_uint line_end;
	const char *doc_comment;
	zend_uint doc_comment_len;
	zend_uint early_binding; /* the linked list of delayed declarations */

	zend_literal *literals;
	int last_literal;

	void **run_time_cache;
	int  last_cache_slot;

	void *reserved[ZEND_MAX_RESERVED_RESOURCES];
	};

保存好op_array后，由excute方法逐条执行，在此阶段，就会调用之前提到的`opcode_handler_t handler`里存储的函数指针。

	ZEND_API void zend_execute(zend_op_array *op_array TSRMLS_DC)
	{
		if (EG(exception)) {
			return;
		} 
		zend_execute_ex(i_create_execute_data_from_op_array(op_array, 0 TSRMLS_CC) TSRMLS_CC);
	}

**扩展**

	PHP有三种方式来进行opcode的处理:CALL，SWITCH和GOTO。
	PHP默认使用CALL的方式，也就是函数调用的方式，由于opcode执行是每个PHP程序频繁需要进行的操作，可以使用SWITCH或者GOTO的方式来分发， 通常GOTO的效率相对会高一些，不过效率是否提高依赖于不同的CPU。

到这里，我们的编译流程就结束了，我们的代码最终会被Parsing成：

	ZEND_ECHO     'Hello World%21'

另外，我们可以借助`vld插件`来看程序的opcodes，在这里就不详细展开了。

## 参考资源
- 《PHP核心技术与最佳实践》
- [Veda原型 - PHP内核探索](http://www.nowamagic.net/librarys/veda/detail/1285)
- [红黑联盟 - 深入理解PHP代码的执行过程](http://www.2cto.com/kf/201404/290863.html)
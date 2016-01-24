---
layout: post
title: "Python r'\' SyntaxError 问题分析"
date: 2016-01-24 20:42:51 +0800
comments: true
categories: Python
---
##0.Python r"\" SyntaxError 问题分析 
这是之前偶尔碰到的问题, 发现跟自己的对python理解不一致, 于是网上和python手册看了下, 网上说是Python的Bug, 但是pythonh手册说是就是不支持这种操作的, 正好最近在看python源码, 于是就通过源码分析了下.  

##1. 问题:
 "\"在python字符串中算特殊字符, 起转义作用, \n转义n变回车, \\转义\等等这个应该没啥问题. 那么正常情况下a="\" 是存在语法问题, 因为\转义了右边的", 所以字符串少一个右引号,错误原因很明显了. 但是python(很作死的)有个r操作符(全称 ), 就是对字符串内的内容按字面意思解析, 不做特殊处理. 所以如果a=r"\n" 输出的就是"\\n".而不是回车(为什么是"\\", goto 1 再看看). 那么现在问题来了, 按r的功能, a=r"\", 应该没问题, 因为在r的场子里, \是不能转义", 但是现在Pyton解析器还是给你一个语法错误:**SyntaxError**

##2. 源码如下:
 其实我刚才说错了, 这不是一个语法错误, 这是个词法错误.(略装B的说法: 语法词法概念模糊的可以网上稍微看下).为什么这么说, 跟了下python的源码,发现问题出在Parser\tokenizer.c内(Python2.5的源码), python对字符串对象的词法解析部分.代码如下:

    /*String*/
	letter_quote:
    if (c == '\'' || c == '"') {
    	Py_ssize_t quote2 = tok->cur - tok->start + 1;
    	int quote = c;
    	int triple = 0;
    	int tripcount = 0;
    	for (;;) {
    		c = tok_nextc(tok);
    		if (c == '\n') {
    			if (!triple) {
    				tok->done = E_EOLS;
    				tok_backup(tok, c);
    				return ERRORTOKEN;
    			}
    			tripcount = 0;
    			tok->cont_line = 1; /* multiline string. */
    		}
    		else if (c == EOF) {
    			if (triple)
    				tok->done = E_EOFS;
    			else
    				tok->done = E_EOLS;
    			tok->cur = tok->inp;
    			return ERRORTOKEN;
    		}
    		else if (c == quote) {
    			tripcount++;
    			if (tok->cur - tok->start == quote2) {
    				c = tok_nextc(tok);
    				if (c == quote) {
    					triple = 1;
    					tripcount = 0;
    					continue;
    				}
    				tok_backup(tok, c);
    			}
    			if (!triple || tripcount == 3)
    				break;
    		}
    		else if (c == '\\'  ) {
    			tripcount = 0;
    			c = tok_nextc(tok);
    			
    			if (c == EOF) {
    				tok->done = E_EOLS;
    				tok->cur = tok->inp;
    				return ERRORTOKEN;
    			}
    		}
    		else
    			tripcount = 0;
    	}
    	*p_start = tok->start;
    	*p_end = tok->cur;
    	return STRING;
    }

###几个地方需要说明一下
- letter_quote这个跳转标签,请关注之最后揭晓.
- tok_nextc从输入流中获取下一个字符.
- tok_backup将字符放回输入流去.
- triple 和tripcount这两个变量跟三引号处理有关,triple说明现在是否处于三引号模式(等于1时,说明当前处于三引号模式)tripcount记录连续的引号数.

##3 分析代码
 `if (c == '\'' || c == '"')`当字符是'或"都将进入字符串对象的词法解析过程.所以python支持'和"两种引号字符。在for循环中一共有4个if, 其实就是说明python的字符串中4类特殊字符.  

 1. \n. 如果在非三引号模式下, 检测到回车后,设置下相应的错误码, 然后返回失败, 相关的错误定义如下:  
 `#define E_EOFS  23  /* EOF in triple-quoted string */`
 `#define E_EOLS  24  /* EOL in single-quoted string */`  
 看这代码, 貌似在单引号模式下不能输回车.但是有点python经验的都知道, 行尾加个\就可以输入回车,在下一行重头再来.是的,请记住,要输入\ . 从中也可以看出,在三引号模式下, 回车是可以随便输的.  

 2. EOF文件尾.(输入流停水了). 这个简单除暴了. 根据当前模式,设置下错误码, 然后返回失败.

 3. quote引号. 先跳过`if(tok->cur - tok->start == quote2)`单看`if(!triple || tripcount == 3)`如果没在三引号模式下, 又"摸到"个quote, 那当前这个字符串词法解析就完了, break出去.一个字符串就ko了(尼玛!略简单的说).回过头说下`if (tok->cur - tok->start == quote2)` 这个, 就是判断三引号的地方了.跟本文核心内容无关,就不扯了哈.

 4. '\\', 终于到这货了. 处理也很简单, 核心就是`c = tok_nextc(tok)`,就是把下个字符从输入流里取了,其他什么都不管.(因为还在词法阶段,没法解析转义字符).那么现在很多问题找到答案:
 ***
 - 加个\单行模式可以输入回车了. 因为python在解析到一个\后,直接把后面的回车符从输入流里取出来了, 同时也说明, 回车一定要紧跟\后面, 因为\只取他后面的一个字符.
 - 之前所说的"\"为什么失败, 应为解析到\后, \把后面"取走了.那么后面的解析时, 面对的就是\n. 所以Syntax EOL in single-quoted string

###4 原始字符串操作符"r"登场
python对string对象的词法解析就这样了. 那么说了这么多,现在可以看下r操作符了, 在词法阶段, 他的处理很偷懒,代码如下:  

    case 'r':
    case 'R':
	    c = tok_nextc(tok);
	    if (c == '"' || c == '\'')
	    {
	    	goto letter_quote;
	    }
shit,就是一个goto到字符串解析部分了. (?:啥表情也不做, 就把活甩给别人了? r:词法分析阶段, 老子能干嘛!!!). 因为r原始操作符, 应该属于python语法上内容, 所以词法分析阶段, 他的处理方式就是常规字符串的处理方式). 所以r在操作"\"时,也要报个Syntax EOL in single-quoted string.
   
###5 后话
其实解决这个问题还是很简单的,处理r操作符时, 加个标示变量  

    int bInRMode = 0;
    case 'r':
    case 'R':
    	c = tok_nextc(tok);
    	f (c == '"' || c == '\'')
    	{
    		bInRMode = 1; //设置下标示位
    		goto letter_quote;
    	}
在处理\时, 

    else if (c == '\\'  ) {
		tripcount = 0;
		c = tok_nextc(tok);
		if (c == EOF) {
			tok->done = E_EOLS;
			tok->cur = tok->inp;
			return ERRORTOKEN;
		}
		if( bInRMode && c !='\n' ){
			tok_backup(tok ,c);
		}
	}
在解析\时, 需要特殊处理下. 应为r模式下, \应该是普通字符, 不应该有取下个字符的功能. 所以我加了个代码把取的字符又塞回去了. 本来在else if (c == '\\')这里可以直接处理的, 但是如果不对回车做特殊处理,那单引号模式下就不能通过\输回车了, 所以对\后接\n,又不采用r的功能了.
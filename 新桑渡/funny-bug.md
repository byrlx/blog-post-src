#+OPTIONS: ^:nil
#+OPTIONS: toc:t H:2
#+AUTHOR: Luis Xu
#+EMAIL: xuzhengchaojob@gmail.com
#+TITLE: 一个有趣的bug

本文有涉及android log机制相关的知识，相关知识内容可参考这三篇文章：
[logger](http://byrlx.github.io/2013/07/17/Android-Log-%E7%B3%BB%E7%BB%9F-%E4%B9%8B-Logger.html),
[liblog](http://byrlx.github.io/2013/07/13/Android-Log-%E7%B3%BB%E7%BB%9F%E4%B9%8B-liblog.html),
[logcat](http://byrlx.github.io/2013/07/10/Android-Log-%E7%B3%BB%E7%BB%9F%E4%B9%8B-logcat.html)。

* 1. 故事背景

在项目中，为了改良一个工具，有对android jelly bean版本的logger+liblog这块代码做二次加工。
其中有修改了system/core/include/cutils/logprint.h文件里的AndroidLogEntry这个结构体，
以及kernel/drivers/staging/android/logger.h
在其中添加了一个成员，如下所示。
 
	struct logger_entry {
		__u16		len;
		__u16		hdr_size;
		__s32		pid;
		__s32		tid;
		__s32		sec;
		__s32		my_modify; //这里是我添加的成员。
		__s32		nsec;
		kuid_t		euid;
		char		msg~[0]~;
	};

	typedef struct AndroidLogEntry_t {
		time_t tv_sec;
		long tv_nsec;
		android_LogPriority priority;
		int32_t my_modify; //这里是我添加的成员。
		int32_t pid;
		int32_t tid;
		const char * tag;
		size_t messageLen;
		const char * message;
	} AndroidLogEntry;
 
后来Kitkat出来之后，就一股脑的把修改后的代码全都merge了过去。
结果build出来的版本一直报segmentfault。

用gdb解析产生的core
dump文件，发现出问题的函数是liblog里的一个API。
 
	int android_log_processBinaryLogBuffer(struct logger_entry *buf,
			AndroidLogEntry *entry, const EventTagMap* map, char* messageBuf,
			int messageBufLen)
 
这个API的左右是把从kernel buffer里读出的一条log（用logger_entry表示）
转换为人类能读懂的log（用AndroidLogEntry表示）。传入的logger_entry是正确的，
但是转换出的AndroidLogEntry却发生了问题。从gdb里能看到，my_modify的值等于pid，
而pid等于tid...依次类推，tag变量指向了一个非法地址。
导致后面的代码在访问 entry->tag 时发生了问题。

为什么说这个bug有趣的呢？因为在KK上使用JB版本的log工具logcat，却能够正常运行，
而该工具也使用了同样的API及结构体，走了相同的解析流程。
而且，如果在logcat代码中该API后面人工造一个segment fault的话，
发现转换出来的AndroidLogEntry的成员也出现了相同的“偏移”，但是为什么logcat就没问题呢？
下面是logcat中的源码。
 
	if (dev->binary) {
		err = android_log_processBinaryLogBuffer(buf, &entry, g_eventTagMap,
				binaryMsgBuf, sizeof(binaryMsgBuf));
		//printf(">>> pri=%d len=%d msg='%s'\n",
		//    entry.priority, entry.messageLen, entry.message);
	} else {
		err = android_log_processLogBuffer(buf, &entry);
	}
	if (err < 0) {
		goto error;
	}

	if (android_log_shouldPrintLine(g_logformat, entry.tag, entry.priority)) {
		if (false && g_devCount > 1) {
			binaryMsgBuf~[0]~ = dev->label;
			binaryMsgBuf~[1]~ = ' ';
			bytesWritten = write(g_outFD, binaryMsgBuf, 2);
			if (bytesWritten < 0) {
				perror("output error");
				exit(-1);
			}
		}

		bytesWritten = android_log_printLogLine(g_logformat, g_outFD, &entry);

		if (bytesWritten < 0) {
			perror("output error");
			exit(-1);
		}
	}
 
可以看到在android_log_shouldPrintLine()函数中也使用了entry.tag当参数，根据上面的分析，
这个值是非法的，那为什么logcat没报错呢？

* 2. 为什么会有segment fault

在解释为什么logcat没报错之前，先来看下为什么会有segment fault发生。

原因很简单，在Kitkat版本上，liblog层使用的头文件有改动。
即在system/core/include下多了一个log目录，原先在system/core/include/cutils/下面相关的log
头文件logger.h/logprint.h/log.h/logd.h都移到了该目录下面。

而我们在从JB向KK的代码库做移植的时候，却把JB里cutils下面相关的头文件也移了过来
(在KK上该目录下已经没有这些文件)。
正是因为这样，我们的代码才没出现编译错误（否则，在刚开始编译就会因为找不到头文件报错），
结果就是，我们的代码使用的是cutils下面的结构体，而liblog使用的是log目录下的结构体。
就AndroidLogEntry来说，我们自己定义的变量并没有被改到log目录下logprint.h中。
 
	// system/core/include/log/logprint.h
	typedef struct AndroidLogEntry_t {
		time_t tv_sec;
		long tv_nsec;
		android_LogPriority priority;
		int32_t pid;
		int32_t tid;
		const char * tag;
		size_t messageLen;
		const char * message;
	} AndroidLogEntry;

	// system/core/include/cutils/logprint.h
	typedef struct AndroidLogEntry_t {
		time_t tv_sec;
		long tv_nsec;
		android_LogPriority priority;
		int32_t my_modify; //这里是我添加的成员。
		int32_t pid;
		int32_t tid;
		const char * tag;
		size_t messageLen;
		const char * message;
	} AndroidLogEntry;

	int android_log_processBinaryLogBuffer(struct logger_entry *buf,
			AndroidLogEntry *entry, const EventTagMap* map, char* messageBuf,
			int messageBufLen)

所以在我们的程序中，当调用上面的函数时，传入的entry是被改造过的类型，
而在函数内部使用的却是原生的类型。导致参数被“悄悄”转换了类型。

				AndroidLogEntry
				/			\
	实际传入的参数			对liblog来说却长这样

	--------------			--------------
	| 	tv_sec	 |			| 	tv_sec	 |
	--------------			--------------
	| 	tv_nsec	 |			| 	tv_nsec	 |
	--------------			--------------
	| 	pri		 |			| 	pri		 |
	--------------			--------------
	|  my_modify |			| 	pid		 |
	--------------			--------------
	| 	pid		 |			| 	tid		 |
	--------------			--------------
	| 	tid		 |			| 	tag		 |
	--------------			--------------
	| 	tag		 |			| messageLen |
	--------------			--------------

通过上图可以看到，在函数内部对参数的修改都是“错位”的，
修改entry->pid的值，其实是修改的entry->my_modify的内容。
entry->tag则是messageLen的值。
当该函数返回后，entry又“恢复”了原样，当访问entry->tag时，
因为tag被“错误”的设成了messageLen的值（该值一般很小），
所以对它的访问才出现“非法地址访问”。

* 3. 为什么JB上的logcat却能正常运行？

在第一部分讲过，在logcat的代码中有调用android_log_shouldPrintLine()这个函数，
其中的一个参数就是entry.tag，按照第二部分的分析，该值是非法的，
那为什么logcat没有报错呢？

通过分析这个函数的实现，可以看出原因。
该函数只是简单调用了filterPriForTag()这个函数，
而filterPriForTag()函数通过for循环来使用这个tag参数。
 
	int android_log_shouldPrintLine (
			AndroidLogFormat *p_format, const char *tag, android_LogPriority pri)
	{
		return pri >= filterPriForTag(p_format, tag);
	}

	static android_LogPriority filterPriForTag(
			AndroidLogFormat *p_format, const char *tag)
	{
		FilterInfo *p_curFilter;

		for (p_curFilter = p_format->filters
				; p_curFilter != NULL
				; p_curFilter = p_curFilter->p_next
			) {
			if (0 == strcmp(tag, p_curFilter->mTag)) {
				if (p_curFilter->mPri == ANDROID_LOG_DEFAULT) {
					return p_format->global_pri;
				} else {
					return p_curFilter->mPri;
				}
			}
		}

		return p_format->global_pri;
	}

如果该循环一次都没有执行呢？那tag就永远调用不到了。
通过分析logcat的代码，发现默认情况下（简单使用logcat命令），
p_format->filters的值确实为NULL，所以这里的tag才没有被使用，
错误也没发生。

通过使用logcat的“-s“参数可以给filters赋值，
使用指定'-s'选项的logcat，segment fault果然发生了。

* 4. END

这个bug的分析还是比较有趣的一个过程，它让我更深一步理解了一个道理：

”能正常运行不代表你的程序没bug，只是你还没有触及到它“。

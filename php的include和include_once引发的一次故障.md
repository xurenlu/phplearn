在php with apc环境下,include和include_once的差异引发的故障

13年年末,我们team出了一次故障,故障的过程是这样的:
OPS在进行正常的上线操作中,接到QA的通知说,可以发布了,于是往线上发布了新的rpm包,
这时QA通知说,再稍等一下,有一个bug需要再确认一下,于是OPS操作人员暂停了发布,等待
QA确认;
需要说明一下,平时php项目的发布,是分为安装rpm包和刷新apc缓存两个过程的,两者紧密相连.
但是这次发布时,QA通知暂停时,操作人员暂停了操作没有继续进行刷新apc的动作,但是也没有
将代码包回退,于是,就产生了问题,线上出现了大量的代码报错.
    
事后追踪,最终,从php的技术层面讲,故障是由于不了解include和include_once的差异引起的.
    
我们剖析一下故障过程;
    
上线之前,首页入口文件(/home/admin/web/htdocs/index.php)代码:(以下代码均为简化版)

```php
    <?php
        include_once("lib/ClassA.php");
```

上线之前,文件夹的文件示意:

```
--index.php
--lib
    └--ClassA.php
```
    
上线之后:首页入口文件(/home/admin/web/htdocs/index.php)的代码:

```php
<?php
    include_once("library/ClassA.php");
```

上线之后,文件夹的文件示意:
```
--index.php
--library
     └--ClassA.php
```

解决说,这次上线,就是移动了库文件ClassA.php的位置,从lib目录移动到了library目录;
    
###这么简单的一个改动,引发了致命的故障!为什么!???怎么可能??
    
事实上,*故障就是这么产生的*.线上还是报错了:
```
Fatal error:can't find class 'ClassA'
```

由于线上使用了apc扩展,并且设置了apc.stat=0,apc并不检测文件内容的改动.在文件内容发生修改时,
如果不手动刷新缓存或是重启fpm,新的代码是不会生效的.在我们刚才介绍的操作人员上线了新的代码包
但是没有刷新apc缓存(也没有重启fpm)的情况下,index.php实际生效的内容还是老的内容.

    但是,同样因为没有刷新apc缓存,lib/ClassA.php的缓存也应该是还存在的啊,为什么会报错呢??
    
为了搞清楚,当被包含文件路径改动时,对应的代码缓存是否还存在,我们还做了一些实际的测试,发现,当被包含
文件移动了路径或被删除时,似乎有的时候代码缓存不会被清掉,有的时候不会?
在研究apc相关源码之前,我们team的成员提过各种想法,其中有这么几种:

1. PHP 的APC缓存是根据文件的inode来计算hash的,因此,过期与不过期,取决于inode;
2. PHP 的apc缓存是寻找软链接对应的目标文件的路径的,旧的文件不应该被删除,在实际安装中应使
用修改软链接的方式来实现;

而在查阅过了php的源代码之后,我们发现,原来跟这些都没有关系,最根本的原因是在于,php 执行引擎
对于include/require和include\_once/require\_once 的两种不同的处理上.

1. 简单说,遇到include和require,php会调用compile_filename 来编译被包含文件.
2. 而遇到include_once和require_once,则会:
    先尝试得到被包含文件的一个规整过后的路径;
    检查一下这个文件是否被包含过了,如果已经被包含过,则跳过,啥也不用干;
    如果此文件未被包含过,则将此文件加入已经包含列表,并尝试打开,如果能打开,调用zend_compile_file来编译;
    如果不能打开,则报错;

现在问题清楚了,如果是用include\_once或require\_once来包含文件的,PHP是要先尝试打开这个文件的,如果文件不存在,会给出相应级别的报错的;
相应代码如下,稍看一下应该能找到问题所在:

```c
//摘录自zend_vm_execute.h中 
//static int ZEND_FASTCALL  ZEND_INCLUDE_OR_EVAL_SPEC_TMP_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
//{ 这一段
switch (opline->extended_value) {
    		case ZEND_INCLUDE_ONCE:
			case ZEND_REQUIRE_ONCE: {
					zend_file_handle file_handle;
					char *resolved_path;

					resolved_path = zend_resolve_path(Z_STRVAL_P(inc_filename), Z_STRLEN_P(inc_filename) TSRMLS_CC);
					if (resolved_path) {
						failure_retval = zend_hash_exists(&EG(included_files), resolved_path, strlen(resolved_path)+1);
					} else {
						resolved_path = Z_STRVAL_P(inc_filename);
					}

					if (failure_retval) {
						/* do nothing, file already included */
					} else if (SUCCESS == zend_stream_open(resolved_path, &file_handle TSRMLS_CC)) {

						if (!file_handle.opened_path) {
							file_handle.opened_path = estrdup(resolved_path);
						}

						if (zend_hash_add_empty_element(&EG(included_files), file_handle.opened_path, strlen(file_handle.opened_path)+1)==SUCCESS) {
							new_op_array = zend_compile_file(&file_handle, (opline->extended_value==ZEND_INCLUDE_ONCE?ZEND_INCLUDE:ZEND_REQUIRE) TSRMLS_CC);
							zend_destroy_file_handle(&file_handle TSRMLS_CC);
						} else {
							zend_file_handle_dtor(&file_handle TSRMLS_CC);
							failure_retval=1;
						}
					} else {
						if (opline->extended_value == ZEND_INCLUDE_ONCE) {
							zend_message_dispatcher(ZMSG_FAILED_INCLUDE_FOPEN, Z_STRVAL_P(inc_filename) TSRMLS_CC);
						} else {
							zend_message_dispatcher(ZMSG_FAILED_REQUIRE_FOPEN, Z_STRVAL_P(inc_filename) TSRMLS_CC);
						}
					}
					if (resolved_path != Z_STRVAL_P(inc_filename)) {
						efree(resolved_path);
					}
				}
				break;
			case ZEND_INCLUDE:
			case ZEND_REQUIRE:
				new_op_array = compile_filename(opline->extended_value, inc_filename TSRMLS_CC);
				break;
```

接下来其实还有一个疑问需要解决,Apc不是改写了php 引擎对include系列语句的处理的吗?
是的,不过看看apc的修改,原来改过之后还是要调用php原有的代码来执行的:
(所以我们总是看到return apc_original_opcode_handler这样的语句.)
```c
static int ZEND_FASTCALL apc_op_ZEND_INCLUDE_OR_EVAL(ZEND_OPCODE_HANDLER_ARGS)
{
    APC_ZEND_OPLINE
    zval *freeop1 = NULL;
    zval *inc_filename = NULL, tmp_inc_filename;
    char realpath[MAXPATHLEN] = {0};
    php_stream_wrapper *wrapper;
    char *path_for_open;
    char *full_path = NULL;
    int ret = 0;
    apc_opflags_t* flags = NULL;

#ifdef ZEND_ENGINE_2_4
    if (opline->extended_value != ZEND_INCLUDE_ONCE &&
        opline->extended_value != ZEND_REQUIRE_ONCE) {
        return apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
    }

    inc_filename = apc_get_zval_ptr(opline->op1_type, &opline->op1, &freeop1, execute_data TSRMLS_CC);
#else
    if (Z_LVAL(opline->op2.u.constant) != ZEND_INCLUDE_ONCE &&
        Z_LVAL(opline->op2.u.constant) != ZEND_REQUIRE_ONCE) {
        return apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
    }

    inc_filename = apc_get_zval_ptr(&opline->op1, &freeop1, execute_data TSRMLS_CC);
#endif

    if (Z_TYPE_P(inc_filename) != IS_STRING) {
        tmp_inc_filename = *inc_filename;
        zval_copy_ctor(&tmp_inc_filename);
        convert_to_string(&tmp_inc_filename);
        inc_filename = &tmp_inc_filename;
    }

    wrapper = php_stream_locate_url_wrapper(Z_STRVAL_P(inc_filename), &path_for_open, 0 TSRMLS_CC);

    if (wrapper != &php_plain_files_wrapper || !(IS_ABSOLUTE_PATH(path_for_open, strlen(path_for_open)) || (full_path = expand_filepath(path_for_open, realpath TSRMLS_CC)))) {
        /* Fallback to original handler */
        if (inc_filename == &tmp_inc_filename) {
            zval_dtor(&tmp_inc_filename);
        }
        return apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
    }

    if (!full_path) {
        full_path = path_for_open;
    }
    if (zend_hash_exists(&EG(included_files), realpath, strlen(realpath) + 1)) {
#ifdef ZEND_ENGINE_2_4
        if (!(opline->result_type & EXT_TYPE_UNUSED)) {
            ALLOC_INIT_ZVAL(APC_EX_T(opline->result.var).var.ptr);
            ZVAL_TRUE(APC_EX_T(opline->result.var).var.ptr);
        }
#else
        if (!(opline->result.u.EA.type & EXT_TYPE_UNUSED)) {
            ALLOC_INIT_ZVAL(APC_EX_T(opline->result.u.var).var.ptr);
            ZVAL_TRUE(APC_EX_T(opline->result.u.var).var.ptr);
        }
#endif
        if (inc_filename == &tmp_inc_filename) {
            zval_dtor(&tmp_inc_filename);
        }
        if (freeop1) {
            zval_dtor(freeop1);
        }
        execute_data->opline++;
        return 0;
    }

    if (inc_filename == &tmp_inc_filename) {
        zval_dtor(&tmp_inc_filename);
    }

    if(apc_reserved_offset != -1) {
        /* Insanity alert: look into apc_compile.c for why a void** is cast to a apc_opflags_t* */
        flags = (apc_opflags_t*) & (execute_data->op_array->reserved[apc_reserved_offset]);
    }

    if(flags && flags->deep_copy == 1) {
        /* Since the op array is a local copy, we can cheat our way through the file inclusion by temporarily 
         * changing the op to a plain require/include, calling its handler and finally restoring the opcode.
         */
#ifdef ZEND_ENGINE_2_4
        opline->extended_value = (opline->extended_value == ZEND_INCLUDE_ONCE) ? ZEND_INCLUDE : ZEND_REQUIRE;
        ret = apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
        opline->extended_value = (opline->extended_value == ZEND_INCLUDE) ? ZEND_INCLUDE_ONCE : ZEND_REQUIRE_ONCE;
#else
        Z_LVAL(opline->op2.u.constant) = (Z_LVAL(opline->op2.u.constant) == ZEND_INCLUDE_ONCE) ? ZEND_INCLUDE : ZEND_REQUIRE;
        ret = apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
        Z_LVAL(opline->op2.u.constant) = (Z_LVAL(opline->op2.u.constant) == ZEND_INCLUDE) ? ZEND_INCLUDE_ONCE : ZEND_REQUIRE_ONCE;
#endif
    } else {
        ret = apc_original_opcode_handlers[APC_OPCODE_HANDLER_DECODE(opline)](ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
    }

    return ret;
}

```

附记:
    apc的实现原理是,劫持了zend_compile_file这个函数指针,用自己的函数替换了他.代码在apc的源码中,
    apc_main.c的apc_module_init函数中:
```c
    /* override compilation */
    old_compile_file = zend_compile_file;
    zend_compile_file = my_compile_file;
```

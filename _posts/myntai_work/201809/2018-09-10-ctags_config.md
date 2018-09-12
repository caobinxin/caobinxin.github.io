# ctags配置

vim /etc/vim/.vimrc

再在配置文件中加入set tags+=tags;/这是告诉vim在当前目录找不到tags文件时请到上层目录查找



**【生成tags文件】**
  安装成功后，要为源码文件生成tags文件，才可享受ctags为阅读代码带来的便利。

$ ctags -R
 递归的为当前目录及子目录下的所有代码文件生成tags文件

为某些源码生成tags文件，使用如下命令
 $ ctags filename.c filename1.c file.h 
 或
 $ ctags *.c *.h

 

为了使得字段补全有效，在生成tags时需要一些额外的参数，推荐的c++参数主要是：
 ctags -R --c++-kinds=+px --fields=+iaS --extra=+q
 其中：
 选项c++-kinds 用于指定C++语言的 tags记录类型,  --c-kinds用于指定c语言的，  通用格式是  --{language}-kinds
 选项 fileds 用于指定每条标记的扩展字段域
 extra 选项用于增加额外的条目:   f表示为每个文件增加一个条目，  q为每个类增加一个条目

 

 

**【使用方法】**
 在vim打开源码时，指定tags文件，才可正常使用，通常手动指定，在vim命令行输入：
 :set tags=./tags(当前路径下的tags文件)
 若要引用多个不同目录的tags文件，可以用逗号隔开

或者，设置 ~/.vimrc，加入一行，则不用手动设置tags路径：
 set tags=~/path/tags

若要加入系统函数或全局变量的tag标签，则需执行：
 ctags -I __THROW –file-scope=yes –langmap=c:+.h –languages=c,c++  –links=yes –c-kinds=+p --fields=+S -R -f ~/.vim/systags /usr/include  /usr/local/include
 并且在~/.vimrc中添加（亦可用上面描述的手动加入的方式）：
 set tags+=~/.vim/systags
 这样，便可以享受系统库函数名补全、原型预览等功能了。

如果经常在不同工程里查阅代码，那么可以在~/.vimrc中添加：
 set tags=tags;
 set autochdir

 

设置好了tags文件，在定位变量/函数的定义时，最常用的快捷键是：
 Ctrl + ]
 跳转到变量或函数的定义处，或者用命令
 :ta name
 而使用快捷组合键
 Ctrl + o/t 
 返回到跳转前的位置。

另外，ctags不会生成局部变量的索引，不过可以使用gd组合键（对光标所在处的word进行快捷查找定位）来定位，也是相当快捷的。

$ vim -t myAdd
 用vim打开文件时，添加参数-t funcName会自动打开定义该函数的文件并定位到定义首行，上面这句就是找到myAdd定义的文件打开并将光标置于定义的第一行处。

:tags 
 会列出查找/跳转过程(经过的标签列表)


 另外，附上vim环境中其他较为好用的快捷键：
 \* 定位至当前光光标所指单词的下一次出现的地方
 \# 定位至当前光光标所指单词的上一次出现的地方
 n 定位至跳至已被标记出的单词下一次出现的地方
 shift+n 定位至跳至已被标记出的单词上一次出现的地方


 关于更详细的ctags用法，vim中使用
 :help tags

 

 

此时在回头学习一下vim手册吧

```
:help usr_29
```

不过还有一个小瑕疵, 你修改程序后, 比如增加了函数定义, 删除了变量定义, tags文件不能自动rebuild, 你必须手动再运行一下命令:

```
$ ctags -R
```

使tags文件更新一下, 不过让人感到欣慰的是vim不用重新启动, 正在编写的程序也不用退出, 马上就可以又正确使用<C-]>和<C-T>了
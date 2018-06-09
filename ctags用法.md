### ctags用法

ctags -R --extra=f 强制在tags文件中显示扫描文件

ctags --lang map = c++:+.inl -R 告知ctags，以.inl为扩展名当c++文件处理

ctrl + ]： 跳转到光标下的定义

tags：列出你已经到过哪些tag

ctrl + T：往回跳

tnext：同名tag跳转

tselect tagname

ctags -R -f .tags


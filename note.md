# 中文化问题 #

在终端中启动clojure后,输入中文字符串 例如 "你好"  是无法正常显示的.这是因为clojure是建立在java下的
所以需要对java运行环境进行设置.

具体做法:


1. 终端启动时添加 -Dfile.encoding=UTF-8  选项

2. 在shell中添加环境变量   export JAVA\_TOOL\_OPTIONS=-Dfile.encoding=UTF-8

任选起一就可以了
### VIM

### 三种模式

* 命令行模式,按下ESC键进入命令行模式。
* 编辑模式，在命令行模式下输入插入命令i,附加命令a,打开命令a,修改命令c,取代命令r,替换命令s都可以进入编辑模式。
* 末行模式。在命令行模式下，用户按":"键即可进入末行模式。

### 打开文件

* `vi filename` 打开或者新建一个文件。如果不写名字，需要在保存的时候输入文件名字。

* `vi +n filename` 打开文件定将光标定位在第n行
* `vi + filename` 打开文件定位在最后一行。
* `vi +/模式字符串　filename` 进入vi后光标处于文件中第一个与指定模式串匹配的那行上。

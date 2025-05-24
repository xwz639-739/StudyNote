# 25.5.24
## GDB调试 ———— core dump 的解决
在server程序的登录功能中一直出现的一个bug，起初用GDB调试，但都没有生成core文件。这是因为系统设置导致的，可以通过`ulimit -c unlimited`或`ulimit -c fileseize`来生成，前者表示不限制core文件的大小，后者则可限制文件大小。
core文件的默认文件名为core，可以`cat /proc/sys/kernel/core_pattern`进行查看core文件生成时的文件名。通过`sysctl -w name=value`进行永久修改，以下是一些规则：
- %p -  添加pid
- %u -  添加当前uid 
- %g -  添加当前gid
- %s -  添加导致产生core的信号 
- %t -  添加core文件生成时的unix时间(由1970年1月1日计起的秒数)
- %h -  添加主机名
- %e -  添加命令名（程序文件名）
也可以通过修改`/proc/sys/kernel/core_pattern`文件进行临时修改。

在调试过程中，学会看堆栈十分重要。可以通过`bt`查看堆栈信息。一般有以下错误可能及解决方法：
- **段错误(Segmentation Fault)**
	1. 使用`bt`查看崩溃点
	2. 检查指针是否为NULL
	3. 使用Valgrind检测内存错误3
 - **堆栈显示不完整**
	1. 确保编译时使用`-g`选项
	2. 检查GDB版本是否匹配
	3. 尝试`set backtrace past-main on`显示更多堆栈帧8
- **优化代码调试**
	对于使用`-O2`或`-O3`优化的代码：
	1. 添加`-fno-omit-frame-pointer`保留帧指针
	2. 使用`__attribute__((noinline))`防止关键函数被内联

然而，不幸的是，这次程序的bug并非如此简单。首先来看错误代码：
```
std::string Login::validateLogin()
{
	MysqlConnGuard mysql_db;
    std::string query = "select pwd from test where acc = '" + _acc + "'";
    MYSQL_RES *res;
    if (mysql_db.isValid()) {
        auto res = (*mysql_db)->Query(query);
    } else {
        return "Error in query";
    }
    if (res) {
        MYSQL_ROW row = mysql_fetch_row(res);
        if (row == nullptr)
            return "no such account";
        else if (row[0] != _pwd)
            return "error account or password";
        else
            return "";
    }
}
```
 这是一段验证登录的代码（比较简陋），但是在运行时却产生了core dump（有时也是Segment Fault）。如下图所示：
 ![[image25.5.24.png]]
 从堆栈信息中可以看出，问题发生在`mysql_fetch_row()`这个函数上，而这个函数在`validateLogin()`这个函数（即上述错误代码）中进行调用。由于`mysql_fetch_row()`是官方的API函数，因此在这个函数上出错有点不可能。那么错误就只可能存在于`validateLogin()`这个函数且就在`MYSQL_ROW row = mysql_fetch_row(res)`这个语句中。

要解决这个问题，就要知道段错误是怎么产生的：
1. **空指针解引用**
	访问 `NULL` 或未初始化的指针
2. **访问已释放内存**
	操作被 `free()` 或 `delete` 后的指针
3. **数组越界**
	读写超出数组/缓冲区边界
4. **栈溢出**
	递归过深或局部变量过大
5. **非法地址访问**
	直接操作无效地址（如对齐错误的地址）
总结：段错误的本质是**程序试图读写操作系统未分配或禁止访问的内存**，通常由指针错误、内存管理不当或资源耗尽导致。

在这段代码中，很显然2~5不是产生段错误的原因：`res`显然不是数组或缓冲区，也没有进行递归操作，更没有被free()或delete过。通过之前`MYSQL_RES *res`这个语句可以看出res这个指针违背初始化，而在`auto res = (*mysql_db)->Query(query)`这个语句中，res被重复定义，不是if外的这个res，而`mysql_fetch_row()`这个函数对res进行操作，显然会产生段错误，因为对于这个函数而言res是未初始化的指针。将auto删掉后，程序就可以顺利跑通了。

至此，这个bug就完美的解决了。
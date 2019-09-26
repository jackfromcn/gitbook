# 文件内容查询

## tail命令
> tail :输出文件的最后几行。
> 用于linux查看日志的时候很方便，假如日志文件为：Console.log
用法：
1. tail Console.log
    输出文件最后10行的内容
2. tail -nf Console.log  --n为最后n行
    ```bash
    tail -200f Console.log
    ```
    输出文件最后n行的内容，同时监视文件的改变，只要文件有一变化就同步刷新并显示出来
3. tail -n 5 filename
    输出文件最后5行的内容
4. tail -f filename
    输出最后10行内容，同时监视文件的改变，只要文件有一变化就显示出来。

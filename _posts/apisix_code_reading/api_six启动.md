# apisix启动部分源码阅读
走读的是master分支的apisix源码，首先读服务启动部分代码，启动apisix命令如下：
> apisix start
对应的代码位于/apisix/cli/ops.lua文件的start函数，该函数实现如下功能
1）检查运行目录：

\```
    if env.is_root_path then
        util.die("Error: It is forbidden to run APISIX in the /root directory.\n")
    end
\```


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

2）创建日志目录：
\```
local cmd_logs = "mkdir -p " .. env.apisix_home .. "/logs"
util.execute_cmd(cmd_logs)
\```

3）检查nginx是否在运行
\```
local pid_path = env.apisix_home .. "/logs/nginx.pid"
local pid = util.read_file(pid_path)
pid = tonumber(pid)
if pid then
    local lsof_cmd = "lsof -p " .. pid
    local res, err = util.execute_cmd(lsof_cmd)
    if not (res and res == "") then
        if not res then
            print(err)
        else
            print("APISIX is running...")
        end

        return
    end

    print("nginx.pid exists but there's no corresponding process with pid ", pid,
          ", the file will be overwritten")
end
\```

4）如果命令行传入配置路径，则建立其与conf/config.yaml之间的软连接，否则采用默认配置
\```
local customized_yaml = args["config"]
if customized_yaml then
    profile.apisix_home = env.apisix_home .. "/"
    local local_conf_path = profile:yaml_path("config")

    local err = util.execute_cmd_with_error("mv " .. local_conf_path .. " "
                                            .. local_conf_path .. ".bak")
    if #err > 0 then
        util.die("failed to mv config to backup, error: ", err)
    end
    err = util.execute_cmd_with_error("ln " .. customized_yaml .. " " .. local_conf_path)
    if #err > 0 then
        util.execute_cmd("mv " .. local_conf_path .. ".bak " .. local_conf_path)
        util.die("failed to link customized config, error: ", err)
    end

    print("Use customized yaml: ", customized_yaml)
end
\```

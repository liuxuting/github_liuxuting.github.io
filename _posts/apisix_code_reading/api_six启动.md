# apisix启动部分源码阅读
走读的是master分支的apisix源码，首先读服务启动部分代码，启动apisix命令如下：
> apisix start

对应的代码位于/apisix/cli/ops.lua文件的start函数。
## start函数
1）检查运行目录：

```lua
if env.is_root_path then
    util.die("Error: It is forbidden to run APISIX in the /root directory.\n")
end
```
2）创建日志目录：
```lua
local cmd_logs = "mkdir -p " .. env.apisix_home .. "/logs"
util.execute_cmd(cmd_logs)
```

3）检查nginx是否在运行
```lua
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
```

4）如果命令行传入配置路径，则建立其与conf/config.yaml之间的软连接，否则采用默认配置
```lua
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
```

5）检查和初始化配置信息

```init(env)```

6)初始化etcd的配置信息，检查etcd节点的健康状态

```init_etcd(env, args)```

7)启动openresty

```util.execute_cmd(env.openresty_args)```

##init函数
init函数用于检查运行目录、文件描述符数量限制，配置文件中服务端ip和port等信息、初始化nginx配置信息并写入conf/nginx.conf文件中。
1）检查运行目录
```lua
if env.is_root_path then
    print('Warning! Running apisix under /root is only suitable for '
          .. 'development environments and it is dangerous to do so. '
          .. 'It is recommended to run APISIX in a directory '
          .. 'other than /root.')
end
```

2）检查文件描述符数量限制
```lua
local min_ulimit = 1024
if env.ulimit <= min_ulimit then
    print(str_format("Warning! Current maximum number of open file "
            .. "descriptors [%d] is not greater than %d, please increase user limits by "
            .. "execute \'ulimit -n <new user limits>\' , otherwise the performance"
            .. " is low.", env.ulimit, min_ulimit))
end
```

3）检查配置文件中服务端ip和port
```lua
local ports_to_check = {}

-- listen in admin use a separate port, support specific IP, compatible with the original style
local admin_server_addr
if yaml_conf.apisix.enable_admin then
    if yaml_conf.apisix.admin_listen or yaml_conf.apisix.port_admin then
        local ip = "0.0.0.0"
        local port = yaml_conf.apisix.port_admin or 9180

        if yaml_conf.apisix.admin_listen then
            ip = yaml_conf.apisix.admin_listen.ip or ip
            port = tonumber(yaml_conf.apisix.admin_listen.port) or port
        end

        if ports_to_check[port] ~= nil then
            util.die("admin port ", port, " conflicts with ", ports_to_check[port], "\n")
        end

        admin_server_addr = ip .. ":" .. port
        ports_to_check[port] = "admin"
    end
end
```

4）初始化nginx配置信息，并写入conf/nginx.conf配置文件
```lua
local sys_conf = {
    use_openresty_1_17 = use_openresty_1_17,
    lua_path = env.pkg_path_org,
    lua_cpath = env.pkg_cpath_org,
    os_name = util.trim(util.execute_cmd("uname")),
    apisix_lua_home = env.apisix_home,
    with_module_status = with_module_status,
    use_apisix_openresty = use_apisix_openresty,
    error_log = {level = "warn"},
    enabled_plugins = enabled_plugins,
    dubbo_upstream_multiplex_count = dubbo_upstream_multiplex_count,
    tcp_enable_ssl = tcp_enable_ssl,
    admin_server_addr = admin_server_addr,
    control_server_addr = control_server_addr,
    prometheus_server_addr = prometheus_server_addr,
}

if not yaml_conf.apisix then
    util.die("failed to read `apisix` field from yaml file")
end

if not yaml_conf.nginx_config then
    util.die("failed to read `nginx_config` field from yaml file")
end
...
-- fix up lua path
sys_conf["extra_lua_path"] = get_lua_path(yaml_conf.apisix.extra_lua_path)
sys_conf["extra_lua_cpath"] = get_lua_path(yaml_conf.apisix.extra_lua_cpath)

local conf_render = template.compile(ngx_tpl)
local ngxconf = conf_render(sys_conf)

local ok, err = util.write_file(env.apisix_home .. "/conf/nginx.conf",
                                ngxconf)
```

## init_etcd函数
init_etcd函数用于初始化etcd的配置信息，检查etcd节点的健康状态
1）初始化etcd的配置信息
```lua
if not yaml_conf.etcd then
    util.die("failed to read `etcd` field from yaml file when init etcd")
end
```

2）检查etcd节点的健康状态
```lua
-- check the etcd cluster version
local etcd_healthy_hosts = {}
for index, host in ipairs(yaml_conf.etcd.host) do
    local version_url = host .. "/version"
    local errmsg

    local res, err
    local retry_time = 0
    while retry_time < 2 do
        res, err = request(version_url, yaml_conf)
        -- In case of failure, request returns nil followed by an error message.
        -- Else the first return value is the response body
        -- and followed by the response status code.
        if res then
            break
        end
        retry_time = retry_time + 1
        print(str_format("Warning! Request etcd endpoint \'%s\' error, %s, retry time=%s",
                         version_url, err, retry_time))
    end

    if res then
        local body, _, err = dkjson.decode(res)
        if err or (body and not body["etcdcluster"]) then
            errmsg = str_format("got malformed version message: \"%s\" from etcd \"%s\"\n", res,
                    version_url)
            util.die(errmsg)
        end

        local cluster_version = body["etcdcluster"]
        if compare_semantic_version(cluster_version, env.min_etcd_version) then
            util.die("etcd cluster version ", cluster_version,
                     " is less than the required version ", env.min_etcd_version,
                     ", please upgrade your etcd cluster\n")
        end

        table_insert(etcd_healthy_hosts, host)
    else
        io_stderr:write(str_format("request etcd endpoint \'%s\' error, %s\n", version_url,
                err))
    end
end
```



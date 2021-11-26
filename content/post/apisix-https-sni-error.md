---
title: "Apisix 配置 HTTPS SNI 不识别问题定位"
date: 2021-11-26T16:32:00+08:00
draft: false
tags: ["Apisix", "网关"]
---

ApiSIX网关

## 问题说明

项目中配置 HTTPS 需要支持 IP 格式直接访问，Apisix 中配置证书需要使用到 SNI 识别功能，但是使用 IP 方式请求主机时 SNI 识别为空，导致配置的 SNI 证书无法被匹配。

## 问题定位

根据报错信息来定位代码逻辑部分：

```lua
2021/07/20 09:54:33 [error] 42#42: *2011 [lua] init.lua:184: http_ssl_phase(): failed to fetch ssl config: failed to find SNI: please check if the client reques
 If you need to report an issue, provide a packet capture file of the TLS handshake., context: ssl_certificate_by_lua*, client: 172.17.0.1, server: 0.0.0.0:6443
```
<!--more-->
根据提示来看是在 `init.lua`中的一个方法调用报错。查看 `init.lua`报错行位置代码逻辑。

```lua
local ok, err = router.router_ssl.match_and_set(api_ctx)
if not ok then
    if err then
        core.log.error("failed to fetch ssl config: ", err)
    end
    ngx_exit(-1)
end
```

可以看出来是在 ssl 匹配阶段报错，因为 Apisix 使用 radixtree 模式来支持范域名匹配。此方法包文件为 `apisix/ssl/router/radixtree_sni.lua` ，打开文件查看方法位置逻辑：

```lua
local sni
sni, err = ngx_ssl.server_name()
if type(sni) ~= "string" then
    local advise = "please check if the client requests via IP or uses an outdated protocol" ..
                   ". If you need to report an issue, " ..
                   "provide a packet capture file of the TLS handshake."
    return false, "failed to find SNI: " .. (err or advise)
end
```

看得出来，`advise` 字符串模板与日志输出完全吻合。

此逻辑为使用 `nginx_ssl`模块调用 `server_name` 方法，会返回 `sni` 字符串，当使用**域名**访问服务时，`sni` 值为域名，当使用 **IP** 访问服务时，`sni` 返回为`nil` 。逻辑判定返回 `sni`为非字符串时直接返回未匹配的报错。

## 解决办法

如上，通过查看 Apisix 初始化完成后生成的 `nginx.conf`文件发现，其会默认配置一套 SSL证书，所以，如果查不到时 `sni`就使用系统默认生成的那一套证书。修改判断逻辑，使其不返回错误即可使用默认证书。

```lua
local sni
sni, err = ngx_ssl.server_name()
if sni == nil then -- 这里是新加入的逻辑
    return true
end
if type(sni) ~= "string" then
    local advise = "please check if the client requests via IP or uses an outdated protocol" ..
                   ". If you need to report an issue, " ..
                   "provide a packet capture file of the TLS handshake."
    return false, "failed to find SNI: " .. (err or advise)
end
```

## 验证

再次使用 IP 直接访问服务，可以正常访问，查看证书信息为测试证书
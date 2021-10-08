## 概述
* 阐述如何使用节点内部服务之间的call调用和跨节点的call调用，它们的回调函数和被调用函数的说明
    * [远程调用](#1)
    * [服务调用](#2)
    * [参数说明](#3)
    * [调用链构建](#4)

## <b id="1">远程调用</b>
* PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, func, ...)
    * 发起跨进程的远程调用
    * ![跨节点调用](https://user-images.githubusercontent.com/50430941/133444385-15c7e09b-ca8b-4aaf-b21a-ef3a6632474d.png)

字段名|必选|类型及范围|说明|特殊规则|传参示例
---|---|---|---|---|---
nodeName|`是`|string|节点名，配置在clusterName.lua中||"BATTLE_1"
serviceName|`是`|string|远程节点服务名||".battle"
method|`是`|string|远程节点rmc_iml的key||"do_once_battle"
param|`否`|any|远程节点函数参数||20
mod|`是`|string|回调的模块名||"BATTLE"
func|`是`|string|回调的函数名||"onBattleFinish"
...|`否`|any|模式参数和回调参数|参考[参数说明](#3)|COMMON_CONST.CALL_MODE.COMMON, ...

* 远程服务端代码，return返回的数据会返回给客户端
    * session：服务端为本次请求生成的唯一key
    * param：客户端的请求参数

```
local function doOnceBattle(session, param)
    ...
    return ret
end

function __init__()
    rmc_iml.do_once_battle = doOnceBattle
end
```
* 客户端回调函数代码
    * ret：服务端返回的数据
    * ...：调用remoteCall时传入的模式参数和回调参数

```
function onBattleFinish(ret, ...)
end
```

## <b id="2">服务调用</b>
* EVPT.safeCall(addr, cmd, param, mod, func, ...)
	* 发起进程内部服务间的调用

字段名|必选|类型及范围|说明|特殊规则|传参示例
---|---|---|---|---|---
addr|`是`|string|服务名||".report"
cmd|`是`|string|sfc_iml的key||"create_report"
param|`否`|any|被调用函数的参数||10
mod|`是`|string|回调的模块名||"BATTLE"
func|`是`|string|回调的函数名||"onBattleFinish"
...|`否`|any|模式参数和回调参数|参考[参数说明](#3)|COMMON_CONST.CALL_MODE.COMMON, ...

* 被调用服务代码

```
local function createReport(source, cid, param)
    ...
    return ret
end

function __init__()
    sfc_iml.create_report = createReport
end
```
* 回调函数

```
function onBattleFinish(ret, ...)
end
```

## <b id="3">参数说明</b>
* 模式参数枚举说明，safeCall的模式参数与remoteCall一致，接下来以remoteCall的使用来说明模式参数如何使用
    * 普通模式调用
        * PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, func, COMMON_CONST.CALL_MODE.COMMON, ...)
        * 省略号是回调参数
    * 节点模式调用
        * PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, func, COMMON_CONST.CALL_MODE.NODE, session, ...)
        * 省略号是回调参数
    * 服务模式调用
        * PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, func, COMMON_CONST.CALL_MODE.SERVICE, source, cid, ...)
        * 省略号是回调参数
    * ==补充说明：当没有回调参数，且是普通模式调用的情况下，调用可简化为PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, func)==

```
CALL_MODE = {
	COMMON = 0,						-- 普通模式，后跟0个模式参数
	NODE = 1,						-- 当前服务是跨节点调用目标服务，后跟1个模式参数session(跨节点调用服务端保存上下文环境的key)
	SERVICE = 2,					-- 当前服务被其它服务call调用，后跟2个模式参数source(原服务句柄), cid(请求唯一key)
}
```
* 服务间的模式参数和回调参数使用与逻辑的服务调用顺序有关
    * 当前服务处于服务调用链的开头使用普通模式
    * 当前服务处于服务调用链的中间，且前一个服务与当前服务处于不同进程，使用节点模式
    * 当前服务处于服务调用链的中间，且前一个服务与当前服务处于同一个进程，使用服务模式

## <b id="4">调用链构建</b>
* 假设有个业务，服务的调用逻辑如下图所示
* ![混合调用](https://user-images.githubusercontent.com/50430941/136487438-e1160061-3681-4510-82a6-8e60e614c937.png)
* call调用发起后，有3种方式可以返回数据给调用者
1. 立即返回
    * srv_5服务收到请求后，并没有再发起call的调用，在被调用函数末尾把数据return回去，返回代码如下

```
local function doOnceBattle(session, param)
    ...
    return ret
end

function __init__()
    rmc_iml.do_once_battle = doOnceBattle
end
```
2. 回调函数返回
    * srv_2、srv_3、srv_4都是相同的返回方式，在call调用的回调函数末尾return返回，返回示例如下

```
local function doOnceBattle(session, param)
    PUBLIC_CLUSTER_TOOL.remoteCall(nodeName, serviceName, method, param, mod, "func", COMMON_CONST.CALL_MODE.NODE, session, ...)
end

function func(ret, COMMON_CONST.CALL_MODE.NODE, session, ...)
    ...
    return data
end

function __init__()
    rmc_iml.do_once_battle = doOnceBattle
end
```
3. 自行返回
    * 需要自行处理的情况，比如调用链需要到tx服务器去验证SDK，需要自行保存模式参数
    * 返回给PUBLIC_CLUSTER_TOOL.remoteCall，需要手动调用PUBLIC_CLUSTER_TOOL.returnBack(session, ret)
    * 返回给EVPT.safeCall，需要手动调用EVPT.sret(addr, id, ret)

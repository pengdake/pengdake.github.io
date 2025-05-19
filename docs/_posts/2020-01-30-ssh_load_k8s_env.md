---
layout: post
title:  "ssh_load_k8s_env"
---


# ssh远程登陆加载环境变量

## 背景
* k8s新建pod时，为了满足某些pod访问当前namespace下svc的需要，即要pod具有服务发现的能力，默认会将当前namespace下面的svc信息以环境变量的形式注入到容器当中，当前namespace下如果存在过多的svc，注入的环境变量的数目将变得很大。[文档链接](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)
* ssh远程登陆会清理环境变量，只会根据bash模式加载/etc/profile, ~/.bashrc等文件中的环境变量。关于不同bash模式加载哪些文件，可以通过man bash查看，这里就不在细说。

## 问题
最近遇到个需求，需要在ssh远程登陆时，能够自动加载我们在k8s资源文件中自定义的容器环境变量，这个实现很简单，只需要在容器启动时，将env命令输出的环境变量转储到/etc/profile.d/k8s_env.sh，ssh远程登陆会自动加载这里面的环境变量。但是由于当前pod所在namespaces中存在过多的svc资源，并且当前pod不需要与其它svc通信，因此传入了许多不必要的环境变量，有没有什么办法过滤掉这些不需要的环境变量呢？

## 如何避免过多关于k8s的环境变量
这个目前可以通过在pod.spec中添加enableServcieLinks变量进行控制，当设置为false时，会禁止当前namespace下的svc信息以环境变量的形式注入到pod中。需要注意的是该变量仅支持1.13及以后的版本。


## 延伸
### 服务发现的替代品
k8s原生支持两种服务发现的方式，一种就是上面所说的通过环境变量，这种方式目前存在一些问题，包括pod所依赖服务必须先于pod创建，注入环境变量过多可能会与自己定义的环境变量名重合导致错误，以及无法根据当前namespace下svc资源情况进行动态更新。因此官方推荐的方式也就是第二种方式，就是通过dns来实现服务发现，这个可以借助插件来实现，比如coredns。
### enableServiceLinks的具体作用
目前enableServiceLinks默认为true,当设置为false时，当前namespace下的svc信息不会以环境变量的形式注入到pod,但是需要注意的是master namespace(默认为default)下的kubernetes svc信息会以环境变量的形式注入到pod中。[具体参考](https://github.com/kubernetes/kubernetes/pull/68754)



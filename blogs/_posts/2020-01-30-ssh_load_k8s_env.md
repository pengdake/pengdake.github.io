---
layout: post
title:  "ssh_load_k8s_env"
---


# ssh远程登陆加载环境变量

## 背景
* k8s新建pod时，默认会将当前namespace下面的svc信息以环境变量的形式注入到容器当中，当前namespace下如果存在过多的svc，注入的环境变量的数目将变得很大。[文档链接](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)
* ssh远程登陆会清理环境变量，只会根据bash模式加载/etc/profile, ~/.bashrc等文件中的环境变量。关于不同bash模式加载哪些文件，可以通过man bash查看，这里就不在细说。

## 问题
最近遇到个需求，需要在ssh远程登陆时，能够自动加载我们在k8s资源文件中自定义的容器环境变量，这个实现很简单，只需要在容器启动时，将env命令输出的环境变量转储到/etc/profile.d/k8s_env.sh，ssh远程登陆会自动加载这里面的环境变量。但是由于当前pod所在namespaces中存在过多的svc资源，并且当前pod不需要与其它svc通信，因此传入了许多不必要的环境变量，有没有什么办法过滤掉这些不需要的环境变量呢？

## 如何避免过多关于k8s的环境变量

## 延伸

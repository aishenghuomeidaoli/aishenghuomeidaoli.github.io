---
title: 使用Virtualbox搭建Kubernetes集群
date: 2019-04-24 17:20:07
tags:
---
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

```sh
docker pull docker.io/aishenghuomeidaoli/kube-proxy:v1.14.1
docker pull docker.io/aishenghuomeidaoli/kube-apiserver:v1.14.1
docker pull docker.io/aishenghuomeidaoli/kube-scheduler:v1.14.1
docker pull docker.io/aishenghuomeidaoli/kube-controller-manager:v1.14.1
docker pull docker.io/aishenghuomeidaoli/coredns:1.3.1
docker pull docker.io/aishenghuomeidaoli/etcd:3.3.10
docker pull docker.io/aishenghuomeidaoli/pause:3.1
```

```sh
docker tag docker.io/aishenghuomeidaoli/kube-proxy:v1.14.1 k8s.gcr.io/kube-proxy:v1.14.1
docker tag docker.io/aishenghuomeidaoli/kube-apiserver:v1.14.1 k8s.gcr.io/kube-apiserver:v1.14.1
docker tag docker.io/aishenghuomeidaoli/kube-scheduler:v1.14.1 k8s.gcr.io/kube-scheduler:v1.14.1
docker tag docker.io/aishenghuomeidaoli/kube-controller-manager:v1.14.1 k8s.gcr.io/kube-controller-manager:v1.14.1
docker tag docker.io/aishenghuomeidaoli/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker tag docker.io/aishenghuomeidaoli/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag docker.io/aishenghuomeidaoli/pause:3.1 k8s.gcr.io/pause:3.1
```
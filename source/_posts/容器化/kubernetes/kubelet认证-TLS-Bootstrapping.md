---
title: kubelet认证-TLS-Bootstrapping
date: 2020-08-11 10:05:45
tags:
  - TLS-Bootstrapping
  - 证书续签
categories: ['kubernetes']
---

# [TLS-Bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)简介
新的工作节点加入集群时需要配置相关的私钥和证书，可以由管理员手动配置，也可以让kubelet自行生成相关的私钥和证书  
但是集群规模扩大后，手动配置会增加管理员工作量，kubelet自动配置则会降低PKI本身的优势  
结合这两种方式，kubernetes提出了新方案(TLS Bootstrapping)：由kubelet自行生成私钥和证书签名请求  
然后向集群的证书签署进程(CA)发起请求，获取的证书和已有的私钥文件共同存储于kubelet客户端以用于后续的通信  
但是，这样一来就认证和非认证的kubelet客户端都会发起证书签名请求，为了控制仅那些认证的客户端可以发起请求，就需要使用令牌机制(bootstrap token)来确认认证的客户端  
APIserver使用基于认证令牌对“system:bootstrappers”组内用户完成认证  
controller-manager会用到这个组的默认审批控制器来决定是否颁发证书，它依赖于由管理员将此token绑定于合适的RBAC策略，以限制其仅用于签证操作  

# apiserver
* 配置项 
    * --enable-bootstrap-token-auth：开启bootstrap认证
    * --token-auth-file：bootstrap token文件
    * --client-ca-file=FILENAME：开启客户端证书认证，指定客户端证书的根证书机构（ca证书）
    * --tls-cert-file
    * --tls-private-key-file：如果apiserver开启https服务，则需要指定私钥和证书【由ca颁发的证书】
    
* token文件范例：`02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"`

* 授权kubelet可以创建证书签名请求

    * 基于kubelet-bootstrap用户

        ```
        kubectl create clusterrolebinding kubelet-bootstrap \
        --clusterrole=system:node-bootstrapper \
        --user=kubelet-bootstrap
        ```

    * 基于system:bootstrappers组

        ```
        # enable bootstrapping nodes to create CSR
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: create-csrs-for-bootstrapping
        subjects:
        - kind: Group
          name: system:bootstrappers
          apiGroup: rbac.authorization.k8s.io
        roleRef:
          kind: ClusterRole
          name: system:node-bootstrapper
          apiGroup: rbac.authorization.k8s.io
        ```
# controller-manager
* 配置项
    * --cluster-signing-cert-file
    * --cluster-signing-key-file：用于颁发集群范围内证书的ca证书
    * --root-ca-file：根ca证书，此根证书颁发机构将包含在服务帐户的令牌密钥中
    
* [批准kubelet证书请求](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/#approval)

    * 手动批准证书签名请求

        ```
        kubectl get csr
        kubectl certificate approve <name>
        ```

    * 自动批准首次申请证书的 CSR 请求(基于[system:bootstrappers]组用户)

        ```
        # Approve all CSRs for the group "system:bootstrappers"
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: auto-approve-csrs-for-group
        subjects:
        - kind: Group
          name: system:bootstrappers
          apiGroup: rbac.authorization.k8s.io
        roleRef:
          kind: ClusterRole
          name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
          apiGroup: rbac.authorization.k8s.io
        ```
        
    * 自动批准kubelet客户端证书续签
    
        ```
        # Approve renewal CSRs for the group "system:nodes"
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: auto-approve-renewals-for-nodes
        subjects:
        - kind: Group
          name: system:nodes
          apiGroup: rbac.authorization.k8s.io
        roleRef:
          kind: ClusterRole
          name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
          apiGroup: rbac.authorization.k8s.io
        ```
        
    * 自动批准kubelet服务端证书续签(kubelet暴露https接口作为服务端)
    
        ```
        kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver --group=system:nodes
        ```

# kubelet

* 配置项
    - --bootstrap-kubeconfig：bootstrap的kubeconfig配置，主要包含bootstrap token
    - --kubeconfig：kubelet和apiserver通信的kubeconfig配置
    - --cert-dir：bootstrap获取的证书和原有的私钥文件的存储位置
    - --rotate-certificates：允许kubelet客户端证书过期前发起续签请求
    - --rotate-server-certificates：允许kubelet作为服务端，在其证书过期前发起续签请求
* 配置bootstrap-kubeconfig
```
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-cluster bootstrap --server='https://my.server.example.com:6443' --certificate-authority=/var/lib/kubernetes/ca.pem
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-credentials kubelet-bootstrap --token=07401b.f395accd246ae52d
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-context bootstrap --user=kubelet-bootstrap --cluster=bootstrap
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig use-context bootstrap
```

# kubelet证书续签-kubeadm范例

* kube-controller-manager组件【/etc/kubernetes/manifests/kube-controller-manager.yaml】

  ```
  --experimental-cluster-signing-duration=87600h0m0s  #kubelet证书有效期
  --feature-gates=RotateKubeletServerCertificate=true #默认设置，启用服务端证书轮转
  --feature-gates=RotateKubeletClientCertificate=true #默认设置，启用客户端证书轮转
  ```

* kubelet组件

  ```
  ServerTLSBootstrap: true #开启kubelet服务端证书续签
  rotateCertificates: true #默认设置，开启kubelet客户端证书续签
  ```

* rbac设置【clusterrolebinding】

  * kubeadm:kubelet-bootstrap【默认】：授权kubelet可以创建证书签名请求
  * kubeadm:node-autoapprove-bootstrap【默认】：自动批准首次申请证书的 CSR 请求
  * kubeadm:node-autoapprove-certificate-rotation【默认】：自动批准kubelet客户端证书续签
  * `kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver --group=system:nodes`：自动批准kubelet服务端证书续签

  
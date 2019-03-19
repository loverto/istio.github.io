---
title: 使用 Cert-Manager 部署自定义入口网关
description: 介绍如何使用 cert-manager 手动部署自定义入口网关。
subtitle: 自定义入口网关
publishdate: 2019-01-10
keywords: [ingress,traffic-management]
attribution: Julien Senon
---

本文提供了手动创建自定义入口[网关](/zh/docs/reference/config/networking/v1alpha3/gateway/)的说明，并基于cert-manager自动配置证书。

可以使用自定义入口网关的创建，以便具有不同的 `loadbalancer` 以隔离流量。

## 开始之前{#before-you-begin}

* 按照[安装指南](/zh/docs/setup/)中的说明设置 Istio。
* 使用 helm 设置 cert-manager` [图表](https://github.com/helm/charts/tree/master/stable/cert-manager#installing-the-chart)
* 我们将使用 `demo.mydemo.com` 作为示例，必须使用您的 DNS 解决

## 配置自定义入口网关{#configuring-the-custom-ingress-gateway}

1. 使用 Helm 使用以下命令检查是否安装了[cert-manager](https://github.com/helm/charts/tree/master/stable/cert-manager)：

    {{< text bash >}}
    $ helm ls
    {{< /text >}}

    输出应该类似于下面的示例，并显示一个 `STATUS` 为 `DEPLOYED` 的 cert-manager：

    {{< text plain >}}
    NAME   REVISION UPDATED                  STATUS   CHART                     APP VERSION   NAMESPACE
    istio     1     Thu Oct 11 13:34:24 2018 DEPLOYED istio-1.0.X               1.0.X         istio-system
    cert      1     Wed Oct 24 14:08:36 2018 DEPLOYED cert-manager-v0.6.0-dev.2 v0.6.0-dev.2  istio-system
    {{< /text >}}

1. 要创建群集的颁发者，请应用以下配置：

    {{< tip >}}
    使用您自己的配置值更改群集的 [issuer](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html#issuers)提供程序。该示例使用`route53`下的值。
    {{< /tip >}}

    {{< text yaml >}}
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-demo
      namespace: kube-system
    spec:
      acme:
        # The ACME server URL
        server: https://acme-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: <REDACTED>
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
          name: letsencrypt-demo
        dns01:
          # Here we define a list of DNS-01 providers that can solve DNS challenges
          providers:
          - name: your-dns
            route53:
              accessKeyID: <REDACTED>
              region: eu-central-1
              secretAccessKeySecretRef:
                name: prod-route53-credentials-secret
                key: secret-access-key
    {{< /text >}}

1. 如果您使用 `route53` [provider](https://cert-manager.readthedocs.io/en/latest/tasks/acme/configuring-dns01/route53.html)，您必须提供秘密才能执行DNS ACME验证。要创建密钥，请应用以下配置文件：

    {{< text yaml >}}
    apiVersion: v1
    kind: Secret
    metadata:
      name: prod-route53-credentials-secret
    type: Opaque
    data:
      secret-access-key: <REDACTED BASE64>
    {{< /text >}}

1. 创建自己的证书：

    {{< text yaml >}}
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: demo-certificate
      namespace: istio-system
    spec:
      acme:
        config:
        - dns01:
            provider: your-dns
          domains:
          - '*.mydemo.com'
      commonName: '*.mydemo.com'
      dnsNames:
      - '*.mydemo.com'
      issuerRef:
        kind: ClusterIssuer
        name: letsencrypt-demo
      secretName: istio-customingressgateway-certs
    {{< /text >}}

    记下 `secretName` 的值，因为将来的步骤需要它。

1. 要自动缩放，请使用以下配置声明新的水平 pod 自动缩放器：

    {{< text yaml >}}
    apiVersion: autoscaling/v1
    kind: HorizontalPodAutoscaler
    metadata:
      name: my-ingressgateway
      namespace: istio-system
    spec:
      maxReplicas: 5
      minReplicas: 1
      scaleTargetRef:
        apiVersion: apps/v1beta1
        kind: Deployment
        name: my-ingressgateway
      targetCPUUtilizationPercentage: 80
    status:
      currentCPUUtilizationPercentage: 0
      currentReplicas: 1
      desiredReplicas: 1
    {{< /text >}}

1. 使用 [yaml definition](/blog/2019/custom-ingress-gateway/deployment-custom-ingress.yaml)中提供的声明应用您的部署

    {{< tip >}}
    The annotations used, for example `aws-load-balancer-type`, only apply for AWS.
    {{< /tip >}}

1. 创建您的服务：

    {{< warning >}}
    The `NodePort` used needs to be an available port.
    {{< /warning >}}

    {{< text yaml >}}
    apiVersion: v1
    kind: Service
    metadata:
      name: my-ingressgateway
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
      labels:
        app: my-ingressgateway
        istio: my-ingressgateway
    spec:
      type: LoadBalancer
      selector:
        app: my-ingressgateway
        istio: my-ingressgateway
      ports:
        -
          name: http2
          nodePort: 32380
          port: 80
          targetPort: 80
        -
          name: https
          nodePort: 32390
          port: 443
        -
          name: tcp
          nodePort: 32400
          port: 31400
    {{< /text >}}

1. 创建您的 Istio 自定义网关配置对象：

    {{< text yaml >}}
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      annotations:
      name: istio-custom-gateway
      namespace: default
    spec:
      selector:
        istio: my-ingressgateway
      servers:
      - hosts:
        - '*.mydemo.com'
        port:
          name: http
          number: 80
          protocol: HTTP
        tls:
          httpsRedirect: true
      - hosts:
        - '*.mydemo.com'
        port:
          name: https
          number: 443
          protocol: HTTPS
        tls:
          mode: SIMPLE
          privateKey: /etc/istio/ingressgateway-certs/tls.key
          serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
    {{< /text >}}

1. 将 `istio-custom-gateway` 与 `VirtualService` 链接：

    {{< text yaml >}}
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: my-virtualservice
    spec:
      hosts:
      - "demo.mydemo.com"
      gateways:
      - istio-custom-gateway
      http:
      - route:
        - destination:
            host: my-demoapp
    {{< /text >}}

1. 服务器返回正确的证书并成功验证（打印 _SSL 证书验证 ok_）：

    {{< text bash >}}
    $ curl -v `https://demo.mydemo.com`
    Server certificate:
      SSL certificate verify ok.
    {{< /text >}}

**恭喜！**您现在可以使用自定义`istio-custom-gateway` [gateway](/zh/docs/reference/config/networking/v1alpha3/gateway/)配置对象。

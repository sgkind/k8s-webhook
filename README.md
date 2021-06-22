# k8s-webhook

## 安装cert-manager
### 安装
```
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```

### 验证是否安装成功
```
$ kubectl get pods - cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5d7f97b46d-k7kf9              1/1     Running   0          3d20h
cert-manager-cainjector-69d885bf55-xjrbx   1/1     Running   0          3d20h
cert-manager-webhook-54754dcdfd-58gvh      1/1     Running   0          3d20h
```
一共有cert-manager, cert-manager-cainjector和cert-manager-webhook三个pod处于running状态

## 创建webhook用的证书
创建证书用的yaml文件为，需要注意要根据service的名称或者url来修改下面配置文件中的commonName和dnsName.
```
$ cat cert.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert
spec:
  commonName: webhook-service.default.svc
  dnsNames:
  - webhook-service
  - webhook-service.default
  - webhook-service.default.svc
  secretName: selfsigned-cert-tls
  issuerRef:
    name: selfsigned
```

查看创建的证书
```
$ kubectl get Certificate
NAME   READY   SECRET                AGE
cert   True    selfsigned-cert-tls   3d19h
```

证书在创建完成后会自动生成secret资源，如下所示
```
$ kubectl get secret                            
NAME                  TYPE                                  DATA   AGE
selfsigned-cert-tls   kubernetes.io/tls                     3      3d19h
```
其类型为`kubernetes.io/tls`类型，里面包含ca机构的公钥和网站的公钥和私钥。

## 编译webhook server的镜像
### 克隆项目
```
$ git clone https://github.com/sgkind/k8s-webhook.git
```

### 编译
```
$ cd k8s-webhook/cmd/webhook
$ go build
```

### 编译镜像
```
$ docker build -t webhook:v1 .
```

## 在k8s中部署webhook server
### service
```
$ cat webhook-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  labels:
    run: webhook
spec:
  ports:
  - port: 443
    protocol: TCP
  selector:
    run: webhook
```

### webhook server
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook
spec:
  selector:
    matchLabels:
      run: webhook
  replicas: 1
  template:
    metadata:
      labels:
        run: webhook
    spec:
      containers:
      - name: webhook
        image: webhook:v1
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-secret
          mountPath: /tmp/tls_secret
      volumes:
      - name: tls-secret
        secret:
          secretName: selfsigned-cert-tls
```
注意：需要将上面配置文件中的secretName修改为自己创建的Certificate所生成的secret的名字。

## 创建ValidatingWebhookConfiguration
```
$ cat validating.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.nocsys.cn"
webhooks:
- name: "pod-policy.nocsys.cn"
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "*"
  clientConfig:
    service:
      namespace: "default"
      name: "webhook-service"
      path: /always-deny
      port: 443
    caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURTRENDQWpDZ0F3SUJBZ0lRT01NUys1bWRzclhNZUswYSsrb2xUekFOQmdrcWhraUc5dzBCQVFzRkFEQW0KTVNRd0lnWURWUVFERXh0M1pXSm9iMjlyTFhObGNuWnBZMlV1WkdWbVlYVnNkQzV6ZG1Nd0hoY05NakV3TmpFNApNRFkwTkRRM1doY05NakV3T1RFMk1EWTBORFEzV2pBbU1TUXdJZ1lEVlFRREV4dDNaV0pvYjI5ckxYTmxjblpwClkyVXVaR1ZtWVhWc2RDNXpkbU13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRQ3UKenVNVURRS0pFcmVHT1NmajJLc1QxMFR0Y3VmM1hlYnJqWE5RcFJ6b3lBQko1WnQzMlhPUm9lZmw2SG9FbnM2KwpwOVExdVVjcU1jQU94a08yMnZRTDJ6NnJWUlEwVUhSb0x3Umk3VmFrY0VvN3ZLdlo1NUkrRVhPSDE1QklKaW10CmsxSStxQ0xPbW0va0ZZU1NKSEdwU1dKZUlHZ2s4dmJTTG4yMnNlVmRFbFF6TjZON3Z2c3l0RUx2ak1ScGdhcHUKRHRYOHJrTzN4c0lhME5saGJXUFRyK2JOS2QxbWNSaCt1RGRVNTNFYXI2aTE0Zjh2VUxFRGZaRVZEUUdTSXVGQQoxd0lqWmFCRCtwZ3NQbU1yY3V3WUM1T2FUcjF1OUhJSWg5U3RFeXVPNVlMMUx5RWdzenRQNzNwNGsyUjdnNk5GCndiQW04L0QzU2U5Y0Znamo1V3N4QWdNQkFBR2pjakJ3TUE0R0ExVWREd0VCL3dRRUF3SUZvREFNQmdOVkhSTUIKQWY4RUFqQUFNRkFHQTFVZEVRUkpNRWVDRDNkbFltaHZiMnN0YzJWeWRtbGpaWUlYZDJWaWFHOXZheTF6WlhKMgphV05sTG1SbFptRjFiSFNDRzNkbFltaHZiMnN0YzJWeWRtbGpaUzVrWldaaGRXeDBMbk4yWXpBTkJna3Foa2lHCjl3MEJBUXNGQUFPQ0FRRUFjOWpaUFBOUHlGajZzU0dMSDlYTnFZMnFpU2w3aXBTR21FWXBGR2xDVkh6TUh0VmUKVEJ3dHJCVDhzOWVHSWQyaHd2c2lySUlkb2dRYTBNWGZWZml2UGlvc1BOOVd3L0drRE5RUDM2YW85V1dPRjdTVgpnU1A1OFphSjZlaGZETE9PZG5LMkw0a0Z4djBaUVM4RC9pK2FYZnZBcmlHOGJhdzZVN1RXVnBvZXA4VUdkVU54ClU3cnVhSDJBSEZ0UFF4N0VtaXVrcXJqVmRQdm0yQjBzN0RSaGRsK1VnVVR5cmliMVc4Qm5RWjFVOFlDM2JWMTMKYXk4UEFWUU5Zb2dpVUpGQUN3TFNjUHpFbmR5TFEwQVM5Uk8yOUxWeWN1M0hsSCtHaVdjWkFvQ3hOTnpSaTROVQpLMnE0NGNFa0xHbTFEY2pHd0diQlpnRzg1K1FEQU1RS0VQMWU0QT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  timeoutSeconds: 5
```

其中caBundle是secret中的ca.crt字段中的值。当前此值是手动拷贝的，也可以在注解中添加`cert-manager.io/inject-ca-from: $(CERTIFICATE_NAMESPACE)/$(CERTIFICATE_NAME)`

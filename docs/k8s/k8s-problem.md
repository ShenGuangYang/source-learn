# k8s 问题解决



## 查看证书是否过期

执行 `kubectl get pod ` 提示 `error: You must be logged in to the server (Unauthorized)` ，怀疑是权限证书问题，权限无误后查看证书是否过期

获取kube config 里面对应的 `client-certificate-data` 内容

```
cat <<EOF | base64 -d > cert.crt
client-certificate-data body
EOF
```

执行

```
openssl x509 -in cert.crt -noout -text
```

查看输出内容的证书有效期。
























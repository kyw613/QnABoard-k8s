### 키 생성및 인증서 발급후 secret 생성

```jsx

# 1. 개인 키 생성 (2048bit)
openssl genrsa -out server.key 2048

# 2. 인증서 생성
openssl req -new -x509 -key server.key -out server.cert -days 360 \
  -subj "/CN=k8s.frontback" \
  -addext "subjectAltName = DNS:k8s.frontback"

# 3. Kubernetes Secret 생성
kubectl create secret tls k8s-front-back-secret \
  --cert=server.cert \
  --key=server.key
secret/k8s-front-back-secret created

# Secret 목록 확인
student@k8s-master:~/Homework$ kubectl get secrets
NAME                    TYPE                DATA   AGE
dshub-https             Opaque              2      7d14h
k8s-front-back-secret   kubernetes.io/tls   2      10s
k8s-secret              kubernetes.io/tls   2      7d13h
my-pwd                  Opaque              1      7d15h
mydb-secret             Opaque              2      4d14h
mysql-pwd               Opaque              1      3d15h
sec-dev                 Opaque              1      7d14h
web-db-secret           Opaque              4      7d15h


```

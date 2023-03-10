## k3s란

K3S란 가벼운 Kubernetes로 쉽게 설치하고 적은 메모리/binary 파일을 사용하여 Edget/IoT 환경 혹은 CI/Dev 환경에서 k8s를 쉽게 사용할 수 있도록 도와주는 도구이다.

Rancher Labs에서 만든 Kubernetes의 또 다른 버전. Kubernetes와 비교해서는 크게 두가지의 차이점이있다.

경량화, 외부 클라우드 서비스와의 연동 기능을 최소한으로 줄이고, 고가용성(HA) 배포를 위해 기본으로 사용하던 etcd 의존성을 없애고 sqlite를 기본값으로 사용한다. 또한 Docker와 같은 의존성을 모두 삭제하고 containerd와 같은 가벼운 대체재를 사용한다. 기존 Kubernetes에서 지원하는 과거 버전의 API또한 지원하지 않는다.

설치가 무척 간단하다. 경량화를 통해 시스템 요구 사항을 극단적으로 줄인 결실이 설치 과정에서 드러나는데, 쉘 스크립트 하나로 대부분의 배포판에서 설치할 수 있다. 설치 후 자동으로 systemd 서비스 또한 만들기 때문에, 사용자가 신경 써 주어야 할 것이 거의 없다.

---

## proxmox에서 k3s

k3s의 노드가될 LXC 컨테이너 생성

- control.k8s
- worker-1.k8s
- worker-2.k8s

컨테이너에서 사용할 SSH Public Key를 생성한다.

### Ubuntu 22 템플릿 추가하기

먼저 사용할 Ubuntu 22.04 템플릿을 추가해줍니다.
LXC컨테이너는 경량화된 리눅스로 템플릿이 별도로 있습니다.

사용하고있는 스토리지의 CT 템플릿 > 템플릿으로 이동

<img width="1462" alt="스크린샷 2023-02-04 오전 12 19 14" src="https://user-images.githubusercontent.com/42893446/216640407-b06258eb-3a1b-4c08-b84b-2c6c4d705be6.png">
<img width="1462" alt="스크린샷 2023-02-04 오전 12 21 05" src="https://user-images.githubusercontent.com/42893446/216640439-79a1af3b-130e-4dc2-9b24-693ed7d2a8b0.png">

LXC 컨테이너는 보안상 권한이 없는 컨테이너로 만들어주셔야 안전합니다.

### Control Plane LXC컨테이너 생성

<kbd>Create CT</kbd> 클릭해서 CT 생성

다음과 같이 설정

<img width="723" alt="스크린샷 2023-01-18 오후 5 06 47" src="https://user-images.githubusercontent.com/42893446/213116959-52d0afa7-fa0a-42e2-86dd-59331db4011f.png">
<img width="720" alt="스크린샷 2023-01-18 오후 5 07 04" src="https://user-images.githubusercontent.com/42893446/213117009-8e8f3a94-0887-405b-8666-7544b69d0985.png">
<img width="728" alt="스크린샷 2023-01-18 오후 5 07 29" src="https://user-images.githubusercontent.com/42893446/213117080-d1a2dd3b-6b7c-4987-8cba-8300071dbc93.png">
<img width="732" alt="스크린샷 2023-01-18 오후 5 07 42" src="https://user-images.githubusercontent.com/42893446/213117122-997cffa8-a297-497f-adf2-81982e65e0bc.png">
<img width="729" alt="스크린샷 2023-01-18 오후 5 07 51" src="https://user-images.githubusercontent.com/42893446/213117166-38177263-ee66-43ff-a2ed-52b309e43b3f.png">
<img width="728" alt="스크린샷 2023-01-18 오후 5 08 14" src="https://user-images.githubusercontent.com/42893446/213117234-f86c15d6-4f25-423b-af00-b8ae5e1ea0ac.png">
<img width="724" alt="스크린샷 2023-01-18 오후 5 08 29" src="https://user-images.githubusercontent.com/42893446/213117288-e7a54ee9-c053-4e7c-b4be-bc833148dd6b.png">

### Worker 노드 LXC 컨테이너 생성

Control 노드와 동일하게 생성하고 Hostnmae을 `worker-[id].k8s`로 변경해서 생성한다.

### 컨테이너 권한부여

1. pve의 shell or SSH로 연결한다.
2. `vim /etc/pve/lxc/300.conf`
3. 다음을 추가합니다
  ```
  lxc.apparmor.profile: unconfined
  lxc.cgroup.devices.allow: a
  lxc.cap.drop:
  lxc.mount.auto: "proc:rw sys:rw"
  ```
4. 설정 파일을 저장
5. 그런다음 woker 노드에 대해서 동일한 작업을 반복합니다.
  `vim /etc/pve/lxc/[CT_ID].conf`
  
해당 설정을 LXC 컨테이너를 재부팅하여 설정을 반영합니다.

1. pve의 shell or SSH로 연결한다.
2. `pct push 300 /boot/config-$(uname -r) /boot/config-$(uname -r)`
3. 그런다음 woker 노드에 대해서 동일한 작업을 반복합니다.

### 각 LXC 컨테이너에서

#### 운영체제에서 필요한 프로그램 설치


LXC는 경량화다보니 기본적인 명령어들이 없는 경우가 있습니다. 필요한 프로그램들을 설치합니다.

```bash
apt-get install neovim git curl
```

#### create `conf-kmsg.sh`

1. conf-kmsg.sh 추가

```bash
vim /usr/local/bin/conf-kmsg.sh
```

2. 내용 추가
  
```
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
  ln -s /dev/console /dev/kmsg
fi
mount --make-rshared /
```

#### create `conf-kmsg.service`

```bash
vim /etc/systemd/system/conf-kmsg.service
```

내용 추가

```
[Unit]
Description=Make sure /dev/kmsg exists

[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/usr/local/bin/conf-kmsg.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target
```
  
#### service 실행

```bash
chmod +x /usr/local/bin/conf-kmsg.sh
```

```bash
systemctl daemon-reload
```

```bash
systemctl enable --now conf-kmsg
```

### Control Plane에 k3s 설치

```bash
curl -fsL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable traefik --node-name control.k8s
```

1. control plane의 ip 주소 가져오기

```bash
hostname -I
```

2. k3s woker node 등록 토큰을 표시

```bash
cat /var/lib/rancher/k3s/server/node-token
```

3. 클러스터에 액세스하기 위해서 `k3s.yaml`을 복사한다.

```bash
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```
  
```bash
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 644 ~/.kube/config
chown $(id -u):$(id -g) ~/.kube/config
echo 'export KUBECONFIG="/home/ubuntu/.kube/config"' >> ~/.bashrc
```
  
### woker 노드에 k3s 설치

```bash
curl -fsL https://get.k3s.io | K3S_URL=https://[control-node-ip]:6443 K3S_TOKEN=[node-token] sh -s - --node-name worker-1.k8s
```

### 설치완료

control 노드에서

```bash
kubectl get nodes
```

확인가능

---

## Ingress

`ingress`는 외부에서 쿠버네티스 클러스터에 접근하기 위한 오브젝트를 가리키며 `NGINX` `Traefik`등의 구현이 있습니다. `k3s`는 `Traefik`을 번들로 제공하고 있습니다.

- ✅ traefik ingress
- nginx ingress

두가지중에 traefik ingress 설정에 대한 자료가 아주 우수해서 traefik ingress로 구성하였습니다.

## Traefik ingress (helm)

### 테스트 애플리케이션 - 에코서버 배포

테스트응 위해 도커 애플리케이션 이미지를 만들기 번거로우니 적절히 에코서버 이미지를 가져와서 deployment를 사용하여 배포합니다.

```bash
mkdir ~/k3s
cd ~/k3s
vi echo-server.yaml
```

```yaml
# echo-sever.yml

# 서비스
---
apiVersion: v1
kind: Service
metadata:
  name: echo-server
  labels:
    app: echo-server
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app: echo-server

# 디플로이먼트
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:0.5.2
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "0.5"
              memory: "1Gi"
```

```bash
kubectl apply -f echo-server.yaml # 서비스와 디플로이먼트를 적용합니다.
```

```bash
kubectl get pods # 포드 목록을 나열합니다.
```

### Traefik을 사용해 외부로 서비스 노출하기

파드는 서비스의 형태로 배포되었지만, 호스트에서 해당 파드로 직접 접근할 수는 없습니다.
`Ingress`를 사용하여 `ingress Controller`에게 생성한 서비스를 연결해 주어야 외부에서 80 포트로 접근이 가능합니다.

```bash
cd ~/k3s
vi traefik.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-routers
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: [YOUR_DOMAIN]
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-server
                port:
                  number: 80
```

### cert-manager와 let's encrypt로 TLS 구현하기

이번에는 `cert-manager`와 `letsencrypt`를 사용하여 TLS 인증서를 적용

[Helm - cert-manager Documentation](https://cert-manager.io/docs/installation/helm/) `helm`을 이용하여 `cert-manager`를 설치합니다.

```bash
helm repo add jetstack https://charts.jetstack.io
````

```bash
helm repo update
```

```bash
curl -L -o cert-manager.yml https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```

버전이 바뀔수있으니 [Helm - cert-manager Documentation 3-install-customresourcedefinitions](https://cert-manager.io/docs/installation/helm/#3-install-customresourcedefinitions)참고하기

```bash
kubectl apply -f cert-manager.yml
```

```bash
helm install \
cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.11.0
```

```bash
kubectl get pods --namespace=cert-manager
===
NAME                                       READY   STATUS      RESTARTS   AGE
cert-manager-cainjector-5c55bb7cb4-schmh   1/1     Running     0          5m16s
cert-manager-76578c9687-jgqmp              1/1     Running     0          5m16s
cert-manager-webhook-556f979d7f-7l99s      1/1     Running     0          5m16s
cert-manager-startupapicheck-xt556         0/1     Completed   0          5m15s
```

### issuer 등록

인증서를 만들기 전, 발급자를 등록해 주어야 합니다. `letsencrypt`에 대한 `tls-issuer.yaml`를 작성합니다.

```yaml
# tls-issuer.yml
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: tls-issuer # 마음대로 지정합니다
  namespace: default
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: [YOUR_EMAIL]
    privateKeySecretRef:
      name: tls-key # 마음대로 지정합니다
    solvers:
      - selector: {}
        http01:
          ingress:
            class: traefik
```

`tls-issuer.yaml`를 등록하고 상태를 확인합니다. `Ready: True` 상태로 변경되기 까지 시간이 조금 걸릴 수 있습니다.

```bash
k create -f tls-issuer.yml
```

```bash
k get issuer -o wide
===
NAME         READY   STATUS                                                 AGE
tls-issuer   True    The ACME account was registered with the ACME server   4m31s
```

### certificate 생성

인증서를 만듭니다.

```yaml
# tls-cert.yml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-cert
  namespace: default
spec:
  secretName: tls-secret # 마음대로 작성
  issuerRef:
    name: tls-issuer # 위에서 지정한 이름
  commonName: [YOUR_DOMAIN]
  dnsNames:
    - [YOUR_DOMAIN]
    # 서브도메인 등록 가능
    # - sub.example.com
```

이제 인증서를 발급합니다.
1. 더미 인증서 제작
2. acme 핸드쉐이크
3. 실제 인증서 발급 및 적용
4. 더미 인증서 파기

해당 과정을 자동으로 처리해줍니다.

```bash
kubectl create -f tls-cert.yml
```

```bash
kubectl get certificate -o wide
===
NAME         READY   SECRET       ISSUER       STATUS                                          AGE
tls-cert     True    tls-secret   tls-issuer   Certificate is up to date and has not expired   6s
```

마찬가지로 `Ready: True` 상태가 되기까지 시간이 조금 걸릴 수 있습니다.

### Ingress Controller에 인증서 정보 전달하기

인증서 발급이 성공적으로 처리되었으니 이제 `traefik.yaml`에서 인증서와 비밀 키 정보를전달하면 TLS를 사용할 수 있습니다.

```yaml
# traefik.yml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-routers
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: [YOUR_DOMAIN]
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-server
                port:
                  number: 80
  tls:
    - secretName: tls-secret # tls-cert 에서 지정한 키 이름
      hosts:
        - [YOUR_DOMAIN]
```

`cert-manager` 플러그인이 인증서 파기 1개월 전에 자동갱신 처리해 주니 신경 쓸 필요가 없습니다. `describe` 명령어를 통해 인증서 만료일과 갱신 예정일을 확인해 볼 수 있습니다.

```bash
kubectl describe certificate tls-cert 
Name:         tls-cert
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2023-01-18T14:30:56Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:commonName:
        f:dnsNames:
        f:issuerRef:
          .:
          f:name:
        f:secretName:
    Manager:      kubectl-create
    Operation:    Update
    Time:         2023-01-18T14:30:56Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:revision:
    Manager:      cert-manager-certificates-issuing
    Operation:    Update
    Subresource:  status
    Time:         2023-01-18T14:31:22Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
          .:
          k:{"type":"Ready"}:
            .:
            f:lastTransitionTime:
            f:message:
            f:observedGeneration:
            f:reason:
            f:status:
            f:type:
        f:notAfter:
        f:notBefore:
        f:renewalTime:
    Manager:         cert-manager-certificates-readiness
    Operation:       Update
    Subresource:     status
    Time:            2023-01-18T14:31:22Z
  Resource Version:  23295
  UID:               24262cb8-631f-4506-ade0-2abba2023f61
Spec:
  Common Name:  4t.gg
  Dns Names:
    4t.gg
  Issuer Ref:
    Name:       tls-issuer
  Secret Name:  tls-secret
Status:
  Conditions:
    Last Transition Time:  2023-01-18T14:31:22Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2023-04-18T13:31:20Z
  Not Before:              2023-01-18T13:31:21Z
  Renewal Time:            2023-03-19T13:31:20Z
  Revision:                1
Events:
  Type    Reason     Age    From                                       Message
  ----    ------     ----   ----                                       -------
  Normal  Issuing    7m1s   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  7m1s   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "tls-cert-sb27z"
  Normal  Requested  7m1s   cert-manager-certificates-request-manager  Created new CertificateRequest resource "tls-cert-twd5d"
  Normal  Issuing    6m35s  cert-manager-certificates-issuing          The certificate has been successfully issued
```

## nginx ingress (helm)

helm `ingress-nginx/ingress-nginx` 차트를 사용하여 nginx ingress controller를 설정. 이를 위해서 repo 추가 repo metadata를 로드한 다음 차트를 설치합니다.

helm의 stable repo가 업데이트를 중단했고, k8s는 빠르게 업데이트 되는 중이다. `stable/nginx-ingress`는 사용하기엔 너무 옛날 버전이라서, k8s에서 따로 배포하는 ingress-nginx repo를 사용해 ingress-controller를 설정하자.

### helm repo 추가

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

```bash
helm repo update
```

### CLI로 예제

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true -n default
```

여기에서 `controller.publishService.enabled` 설정은 수신할 서비스 IP주소를 수신 리소스에 게시하도록 컨트롤러에 지시합니다.

차트가 완료되면 다양한 리소스가 `kubectl get all` 출력에 표시되어야 합니다. (컨트롤러가 온라인 상태가 되어 로드 밸러서에 IP주소를 할당하는 데 몇 분정도 걸릴 수 있습니다)

control plane의 ip로 접속하면 404 not found nginx를 볼 수 있습니다.

### k8s 파일 예제

#### nginx pod와 service 생성

```yaml
# mynginx.yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mynginx
  name: mynginx
spec:
  containers:
  - image: nginx:1.16
    name: mynginx
    resources: {}
  restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: nginxsvc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: mynginx
```

Service의 Type은 ClusterIP로 설정한다.

```bash
kubectl apply -f mynginx.yaml
```

#### ingress namespace 생성

```bash
kubectl create ns ingress-nginx
```

#### helm repo update & search

```bash
helm repo update
```

```bash
helm search repo ingress-nginx
```

#### helm install ingress-nginx

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -n default
```

namespace를 지정해야하는데 수십 수백명이 운영하는 쿠버네티스가 아니라면 하나의 네임스페이스만 사용하자 [참고](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/)

```bash
kubectl get pod -n ingress-nginx
kubectl get svc -n ingress-nginx
```

ingress 구동 

#### ingress-controller 생성 

```yaml
# mynginx-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations: 
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: mynginx-ingress
spec:
  rules:
  # - host: -> domain이 없는 경우 생략 가능
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginxsvc
            port:
              number: 80
```

```bash
kubectl apply -f mynginx-ingress.yaml
```

```bash
kubectl get ing
```

#### 연결 확인

http://ip/nginx

접속하여 nginx가 재대로 실행되는지 확인


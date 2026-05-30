# Kubernetes Cluster Operation Project

> CKA(Certified Kubernetes Administrator) 실전 시나리오 기반 쿠버네티스 클러스터 운영 실습 — 백업/업그레이드 · RBAC · 스케줄링 · 네트워킹 · 스토리지


## 자료

[Kubernetes Project HISTORY.pdf](./1_history_kubernetesproject.pdf)
---

## 개요

| 항목 | 내용 |
|------|------|
| **기간** | 2026.05 |
| **유형** | 팀 프로젝트 |
| **팀 구성** | 김동진 · 이영훈 · 장규혁 · 정성현 |
| **역할** | 조장 — 업무 분담 및 문서(docs) 작성 |
| **환경** | kubeadm 기반 쿠버네티스 클러스터 (Master 1 + Worker 2) |
| **이미지 레지스트리** | ghcr.io/mjc-t (git 레지스트리) |

---

## 목차

1. [ETCD Backup & Upgrade](#1-etcd-backup--upgrade)
2. [역할 기반 제어 (RBAC)](#2-역할-기반-제어-rbac)
3. [노드 관리](#3-노드-관리)
4. [파드 생성](#4-파드-생성)
5. [Kubernetes Controller & Networking](#5-kubernetes-controller--networking)
6. [NetworkPolicy & Ingress](#6-networkpolicy--ingress)
7. [Resource 관리](#7-resource-관리)

---

## 1. ETCD Backup & Upgrade

### 1-1. ETCD Snapshot 백업 및 복구

`https://127.0.0.1:2379`에서 실행 중인 etcd의 스냅샷을 생성하고 `etcd-snapshot.db`에 저장 후 복구

```bash
# root 권한 전환 및 etcd-client 설치
sudo -i
apt install etcd-client

# Snapshot 생성
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save etcd-snapshot.db

# Snapshot 복구
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore etcd-snapshot.db
```

> 💡 TLS 인증서 경로: CA `ca.crt`, Client `server.crt` / `server.key` (`/etc/kubernetes/pki/etcd/`)

### 1-2. Cluster Upgrade

클러스터 버전을 1.33.11-1.1 → 1.34.7-1.1로 업그레이드. Master 노드는 drain 후 작업, 완료 후 uncordon

```bash
# 1. 가용 버전 확인
sudo apt update
sudo apt-cache madison kubeadm

# 2. kubeadm 업그레이드
sudo apt-mark unhold kubeadm && \
  sudo apt-get update && sudo apt-get install -y kubeadm='1.34.7-1.1' && \
  sudo apt-mark hold kubeadm

kubeadm version
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.34.7-1.1

# 3. 노드 drain
kubectl drain master --ignore-daemonsets

# 4. kubelet & kubectl 업그레이드
sudo apt-mark unhold kubelet kubectl && \
  sudo apt-get update && \
  sudo apt-get install -y kubelet='1.34.7-1.1' kubectl='1.34.7-1.1' && \
  sudo apt-mark hold kubelet kubectl

# 5. 재시작 및 uncordon
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon master
```

> 💡 워커 노드의 kubelet은 마스터보다 최대 2단계 낮아도 호환되나, 안정성을 위해 동일 버전 권장. 워커도 동일 방식(unhold → install → hold)으로 순차 진행


---

## 2. 역할 기반 제어 (RBAC)

### 2-1. ServiceAccount + Role + RoleBinding (네임스페이스 범위)

`api-access` 네임스페이스의 모든 Pod를 watch/list/get 할 수 있는 권한 구성

```bash
# Namespace 생성
kubectl create namespace api-access

# Role 생성 (Pod에 대한 watch, list, get 허용)
kubectl create role podreader-role \
  --resource=pod \
  --verb=watch,list,get \
  --namespace=api-access

# ServiceAccount 생성
kubectl create serviceaccount pod-viewer -n api-access

# RoleBinding 생성 (Role ↔ ServiceAccount 매핑)
kubectl create rolebinding podreader-rolebinding \
  --role=podreader-role \
  --serviceaccount=api-access:pod-viewer \
  --namespace=api-access
```

### 2-2. ClusterRole + ClusterRoleBinding (클러스터 범위)

Deployment/StatefulSet/DaemonSet에 대해 create만 허용하는 ClusterRole 구성

```bash
# ClusterRole 생성 (Create만 허용)
kubectl create clusterrole deployment-clusterrole \
  --resource=deployment,statefulset,daemonset \
  --verb=create

# ServiceAccount 생성
kubectl create serviceaccount cicd-token --namespace=api-access

# ClusterRoleBinding 생성 (네임스페이스로 제한)
kubectl create clusterrolebinding deployment-clusterrolebinding \
  --clusterrole=deployment-clusterrole \
  --serviceaccount=api-access:cicd-token
```

> 💡 **Role vs ClusterRole**: Role은 특정 네임스페이스 내 권한, ClusterRole은 클러스터 전체 범위 권한

---

## 3. 노드 관리

### 3-1. 노드 비우기 (Drain / Uncordon)

worker-2를 스케줄링 불가 상태로 만들고 실행 중인 Pod를 다른 노드로 재배치

```bash
kubectl get node

# worker-2 비우기 (Pod 재배치 + 스케줄링 차단)
kubectl drain worker-2 --ignore-daemonsets

# 스케줄링 재허용
kubectl uncordon worker-2
```

### 3-2. NodeSelector를 이용한 Pod 스케줄링

노드에 라벨을 부여하고, 특정 라벨(disktype=ssd)을 가진 노드에만 Pod 배치

```bash
# 노드 라벨 부여
kubectl label node worker-1 disktype=ssd
kubectl label node worker-2 disktype=hdd
kubectl get no -L disktype

# Pod yaml 생성
kubectl run eshop-store \
  --image=ghcr.io/mjc-t/nginx \
  --dry-run=client -o yaml > eshop-store.yaml
```

```yaml
# eshop-store.yaml (nodeSelector 추가)
apiVersion: v1
kind: Pod
metadata:
  name: eshop-store
spec:
  containers:
  - image: ghcr.io/mjc-t/nginx
    name: eshop-store
  nodeSelector:
    disktype: ssd
```

```bash
kubectl apply -f eshop-store.yaml
kubectl get po eshop-store -o wide   # worker-1에 배치 확인
```

---

## 4. 파드 생성

### 4-1. 환경변수 · command · args 적용

```yaml
# pod-01.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  namespace: cka-exam
spec:
  containers:
  - image: ghcr.io/mjc-t/busybox
    name: pod-01
    env:
    - name: CERT
      value: "CKA-cert"
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(CERT); sleep 10;done"]
```

### 4-2. Static Pod

kube-apiserver를 거치지 않고 kubelet이 직접 관리하는 Pod. worker-1 노드에 생성

```bash
# worker-1 노드에서 작업
sudo -i
cd /etc/kubernetes/manifests
vi nginx-static-pod.yaml

# master 노드에서 생성 확인 (이름에 노드명이 자동 접미)
kubectl get pods -o wide   # nginx-static-pod-worker1
```

> 💡 Static Pod 경로: `/etc/kubernetes/manifests` (kubelet이 해당 디렉터리를 감지하여 자동 실행)

### 4-3. Multi-Container Pod

nginx, redis, memcached, busybox 4개 컨테이너를 하나의 Pod로 구성

```yaml
# eshop-frontend.yaml
apiVersion: v1
kind: Pod
metadata:
  name: eshop-frontend
spec:
  containers:
  - image: ghcr.io/mjc-t/nginx
    name: nginx
  - image: ghcr.io/mjc-t/redis:3.2-alpine
    name: redis
  - image: ghcr.io/mjc-t/memcached:latest
    name: memcached
  - image: ghcr.io/mjc-t/busybox:1.28
    name: busybox
    command: ["sh", "-c", "sleep 3600"]
```

> 💡 busybox는 foreground 프로세스가 있어야 컨테이너가 유지되므로 `sleep` command 필요

### 4-4. 로그 추출

```bash
# Pod 로그를 파일로 저장
kubectl logs nginx-static-pod-worker1 > /home/cka/pod-log
cat /home/cka/pod-log
```

---

## 5. Kubernetes Controller & Networking

### 5-1. Rolling Update & Rollback

nginx Pod 3개 배포 후 이미지 버전 업데이트(1.16 → 1.17), 이후 롤백

```bash
# Deployment 생성 (--record로 변경 이력 기록)
kubectl apply -f eshop-payment.yaml --record

# Rolling Update
kubectl set image deployments eshop-payment \
  nginx=ghcr.io/mjc-t/nginx:1.17 --record

# 변경 이력 확인
kubectl rollout history deployment eshop-payment

# Rollback (이전 버전으로 복귀)
kubectl rollout undo deployment eshop-payment
kubectl describe pod | grep -i image:
```

### 5-2. ClusterIP Service

클러스터 내부 통신용 서비스 (devops 네임스페이스)

```bash
kubectl create namespace devops
kubectl apply -f eshop-order.yaml   # replicas: 2

# Service 노출
kubectl expose deployment -n devops eshop-order \
  --name=eshop-order-svc --port=80 --target-port=80

kubectl get svc -n devops
curl 10.101.82.59   # ClusterIP로 접근 확인
```

### 5-3. NodePort Service

노드의 30200 포트로 외부 접근 허용

```yaml
# front-end-nodesvc.yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-nodesvc
  labels:
    run: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30200
  selector:
    run: nginx
```

```bash
kubectl apply -f front-end-nodesvc.yaml
curl 192.168.111.20:30200   # NodeIP:NodePort로 접근 확인
```

### 5-4. Pod Scale-out

```bash
# replicas를 5개로 확장
kubectl scale deploy eshop-order -n devops --replicas=5
kubectl get deploy,pod -n devops
```

---

## 6. NetworkPolicy & Ingress

### 6-1. Network Policy

특정 라벨(partition=customera)을 가진 네임스페이스에서만 Pod의 80포트 접근 허용

```yaml
# netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-customera
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: poc
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          partition: customera
    ports:
    - protocol: TCP
      port: 80
```

```bash
# 네임스페이스 라벨링
kubectl label ns customera partition=customera
kubectl label ns customerb partition=customerb
```

### 6-2. Ingress

경로(/hi) 기반 라우팅으로 서비스(hi:5678) 노출

```yaml
# ing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ping
  namespace: ing-internal
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /hi
        pathType: Prefix
        backend:
          service:
            name: hi
            port:
              number: 5678
```

### 6-3. Service & DNS Lookup

클러스터 내 Service/Pod 이름 조회 테스트 (busybox:1.28 + nslookup)

```bash
# resolver Pod + Service 생성
kubectl run resolver --image ghcr.io/mjc-t/nginx --port 80
kubectl expose pod resolver --name resolver-service --port 80

# Service DNS 조회 → 파일 저장
kubectl run -it test-nslookup --image=ghcr.io/mjc-t/busybox \
  --restart=Never --rm \
  -- nslookup 10.108.94.23 > ~/nginx.svc

# Pod DNS 조회 → 파일 저장
kubectl run -it test-nslookup --image=ghcr.io/mjc-t/busybox \
  --restart=Never --rm \
  -- nslookup 10-244-235-191.default.pod.cluster.local > ~/nginx.pod
```

---

## 7. Resource 관리

### 7-1. emptyDir Volume

Pod 내 컨테이너 간 데이터 공유 (nginx 로그 → busybox로 STDOUT 출력)

```yaml
# weblog.yaml
apiVersion: v1
kind: Pod
metadata:
  name: weblog
  labels:
    run: weblog
spec:
  containers:
  - image: ghcr.io/mjc-t/nginx:1.17
    name: web
    volumeMounts:
      - mountPath: /var/log/nginx
        name: weblog
  - image: ghcr.io/mjc-t/busybox
    name: log
    args: ["/bin/sh", "-c", "tail -n+1 -f /data/access.log"]
    volumeMounts:
      - mountPath: /data
        name: weblog
        readOnly: true
  volumes:
    - name: weblog
      emptyDir: {}
```

### 7-2. HostPath Volume

워커 노드의 디렉터리를 Pod에 마운트 (로그 수집용 fluentd)

```yaml
# fluentd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fluentd
  labels:
    run: fluentd
spec:
  containers:
  - image: ghcr.io/mjc-t/fluentd
    name: fluentd
    volumeMounts:
      - name: containersdir
        mountPath: /var/lib/docker/containers
      - name: logdir
        mountPath: /var/log
  volumes:
    - name: containersdir
      hostPath:
        path: /var/lib/docker/containers
    - name: logdir
      hostPath:
        path: /var/log
```

### 7-3. PersistentVolume (PV)

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/tmp/app-config"
```

### 7-4. PersistentVolumeClaim (PVC) + Pod 연동

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
---
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
spec:
  containers:
    - name: nginx
      image: ghcr.io/mjc-t/nginx
      volumeMounts:
      - mountPath: "/usr/share/nginx/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pv-volume
```

---

## 주요 학습 내용

| 영역 | 핵심 기술 |
|------|----------|
| **클러스터 운영** | ETCD 백업/복구, kubeadm 버전 업그레이드, drain/uncordon |
| **권한 제어** | RBAC (Role/ClusterRole, RoleBinding/ClusterRoleBinding, ServiceAccount) |
| **스케줄링** | NodeSelector, 노드 라벨링, Static Pod |
| **워크로드** | Multi-Container Pod, Deployment Rolling Update/Rollback, Scale-out |
| **네트워킹** | ClusterIP, NodePort, NetworkPolicy, Ingress, DNS Lookup |
| **스토리지** | emptyDir, hostPath, PersistentVolume, PersistentVolumeClaim |

---

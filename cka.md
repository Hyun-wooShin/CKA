# 참고사이트
https://daintree.tistory.com/16

# (cluster)role and serviceaccount and (cluster)rolebinding

## 유형1
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole
```
kubectl create clusterrole deployment-clusterrole --verb=create --resource=depolyment,statefulset,daemonset
kubectl create serviceaccount cicd-token --namespace app-team1
kubectl create clusterrolebinding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token
```

## 유형2
https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user
```
openssl genrsa -out app-manager.key 2048
openssl req -new -key app-manager.key -out app-manager.csr -subj "/CN=app-manager"
cat app-manager.csr | base64 | tr -d "\n"

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: app-manager
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1d6Q0NBVU1DQVFBd0ZqRVVNQklHQTFVRUF3d0xZWEJ3TFcxaGJtRm5aWEl3Z2dFaU1BMEdDU3FHU0liMwpEUUVCQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUURnbW5HWXN1SEJXbmpaSVlFOW5PTFJPQldXT0p1MTMwOEFsMTl0Cm5xMHN6ZDJ1WUhjU2dGNmpYampZZDJkR0NlYzlBQkM0cTduaFdYaVU1QmpsOXk3MXlRZGlsMHo5eFRLaHBGbE4KMzQ3SzJudGJ0dTEvZEdvck0ycHc3aVVHWkEyUXFBYktkSXMyVXJRZ0FMMGs3Y01hZnlqTUVJZDQ1ZldubkNGMQpVcHZsZHBoK3pxeEx2ajRUSDZZRGc1dDJibThYWldJS2gwaC8xeVd2UU1iYTJ5WXJ6M0RLZVYwR3E2dHpUZkZvCkt1eWpuNjBjVm1DS2ZhWDR1TzgzM1Nuc1hxcmRQY0p0MVl4YTU1UkU5ZU1JbGhDcVRvZ2JCZWNYSFpZYUV4NzAKcGRiVE1qVDVyalBtNnZnbTBlcUNpMVpSKzlOODB6cTJDeUloa1RIRDU3SVlrL3BUQWdNQkFBR2dBREFOQmdrcQpoa2lHOXcwQkFRc0ZBQU9DQVFFQTMvdkxmd1Izby9GbU0vdlQrRy9RUWt4TmRCcXJBejlIUHZ6K2FXVGVIVnBqCit3dHBFUHRqSDQ2cDFrSU5KUDJnWjJleHFkZlVUQXJjYnZPek9rYlA5SURCcm9LbWhsUlBuVWlYeW9BNGZoL0QKMldhVWk4VEMwNnhEZ2h1K2hBTHFzUlp4bzgxdjZ5YTdYVzdOK1FYUk42V1pxM0RVQlNKTFpXQW4rZ2JOa0ZkdQpVL014Z0dZNzFURkZLNGZETUF5amlra05udlBudVR1L3UyMzJCMmJ1REJ5VS9DaDlFVEpnVVNaYUErVXVnd0p0Ck00VVlYT1pVMGVBT0dqdXlyUXAzME1mMVAzclBZNGM4bWlzSjQzQzN5aWRkZWVTN1pkWFBkWVYvb0FsSWVETGsKZHN1MjJYQ1VNdVlSSExOUnA4cEdySVZRUGF3RncwaEZ4QXVnWk9HT1FBPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  # expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF

kubectl certificate approve app-manager
kubectl get csr
kubectl get csr app-manager -o jsonpath='{.status.certificate}'| base64 -d > app-manager.crt
kubectl config set-credentials app-manager --client-key=app-manager.key --client-certificate=app-manager.crt --embed-certs=true
kubectl config set-context app-manager --cluster=kubernetes --user=app-manager
kubectl create clusterrole app-access --verb=create,list,get,update,delete --resource=deployment,pod,service
kubectl create clusterrolebinding app-access-binding --clusterrole=app-access --user=app-manager
```

# node drain
https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/

# cluster upgrade
https://kubernetes.io/ko/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
1) 마스터 노드 drain
2) 마스터 노드의 kubeadm 업그레이드
3) 마스터 노드의 kubectl, kubelet 업그레이드
4) 마스터 노드 uncordon
5) 워커 노드 drain
6) 워커 노드의 kubeadm 업그레이드
7) 워커 노드의 kubectl, kubelet 업그레이드(kubectl이 없을 경우 kubelet만 업그레이드)
8) 워커 노드 uncordon

```
kubectl drain controlplane --ingnore-demonsets
apt update
#업그레이드 가능한 kubeadm 버전 확인
apt-cache madison kubeadm
#kubeadm 업그레이드
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.19.0-00 && \
apt-mark hold kubeadm
#컴포넌트 업그레이드 가능 버전 확인
kubeadm version
kubeadm upgrade plan
sudo kubeadm upgrade apply v1.19.0
#kubelet, kubectl 업그레이드
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.19.0-00 kubectl=1.19.0-00 && \
apt-mark hold kubelet kubectl
#kubelet 재시작
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon controlplane 


kubectl drain node --ignore-daemonsets
#kubeadm 업그레이드
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.19.0-00 && \
apt-mark hold kubeadm
#노드 업그레이드
kubeadm upgrade node
#kubelet 및 kubectl 업그레이드
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.19.0-00 kubectl=1.19.0-00 && \
apt-mark hold kubelet kubectl
#kubelet 재시작
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon node
```

# ETCD Backup & Restore
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
https://velog.io/@khyup0629/K8S-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EC%84%A4%EC%A0%95

```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>

export ETCDCTL_API=3
etcdctl snapshot restore --data-dir <data-dir-location> snapshotdb
```

# Kubernetes NetworkPolicy
https://kubernetes.io/docs/concepts/services-networking/network-policies/
https://taemy-sw.tistory.com/9

# kuybespray로 설치
https://www.whatwant.com/entry/Kubespray
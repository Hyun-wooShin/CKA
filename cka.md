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
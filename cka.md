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
```
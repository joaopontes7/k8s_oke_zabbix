
# Guia: Monitoramento Kubernetes via Zabbix

Este guia descreve como configurar o monitoramento de um cluster Kubernetes no Zabbix utilizando Service Account, permissões RBAC e template oficial via HTTP.

---

## 1. Criar ServiceAccount

```powershell
kubectl create serviceaccount zabbix-sa -n default
```

---

## 2. Criar Secret vinculado

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: zabbix-sa-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: zabbix-sa
type: kubernetes.io/service-account-token
"@ | kubectl apply -f -
```

---

## 3. Criar ClusterRole com permissões de leitura

```powershell
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: zabbix-read-all
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "nodes/stats", "pods", "services", "endpoints", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
"@ | kubectl apply -f -
```

---

## 4. Vincular a SA ao ClusterRole

```powershell
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: zabbix-read-all-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: zabbix-read-all
subjects:
- kind: ServiceAccount
  name: zabbix-sa
  namespace: default
"@ | kubectl apply -f -
```

---

## 5. Obter o token da ServiceAccount

```powershell
$tokenBase64 = kubectl get secret zabbix-sa-token -n default -o jsonpath="{.data.token}"
$token = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($tokenBase64))
$token
```

---

## 6. Testar o token com a API do Kubernetes

```powershell
Add-Type @"
using System.Net;
using System.Security.Cryptography.X509Certificates;
public class TrustAllCertsPolicy : ICertificatePolicy {
    public bool CheckValidationResult(
        ServicePoint srvPoint, X509Certificate certificate,
        WebRequest request, int certificateProblem) {
        return true;
    }
}
"@
[System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy

$headers = @{
    "Authorization" = "Bearer $token"
    "Accept"        = "application/json"
}
Invoke-WebRequest -Uri "https://<IP_API_SERVER>:6443/api/v1/nodes" -Headers $headers -Method GET
```

---

## 7. Configurar host no Zabbix

- Nome: `meu-cluster`
- Grupo: `Kubernetes`
- Template: `Kubernetes cluster by HTTP`

---

## 8. Adicionar macros no host

```
{$KUBE.API.URL} = https://<IP_API_SERVER>:6443
{$KUBE.API.TOKEN} = <cole o token obtido>
{$KUBE.API.CERT.VALIDATE} = false
```

---

## 9. Validar permissões no cluster

```powershell
kubectl auth can-i list nodes --as=system:serviceaccount:default:zabbix-sa
```

---

## 10. Verificar coleta no Zabbix

- Vá em **Monitoring > Latest Data**
- Filtre pelo host configurado
- Valide se os itens estão coletando dados

---

Pronto! Agora seu cluster Kubernetes está integrado ao Zabbix com autenticação segura via token e coleta completa por API HTTP.

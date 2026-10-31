# ☸️ Projeto K8s

Projeto de estudo de **Kubernetes**: um servidor **nginx** publicado com Deployment, Service, ConfigMap, Secret e volume persistente.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Docker Desktop](https://img.shields.io/badge/Docker_Desktop-2496ED?style=for-the-badge&logo=docker&logoColor=white)

---

## 📁 Estrutura

| Arquivo | Recurso | O que faz |
|---|---|---|
| [`deployment.yaml`](deployment.yaml) | Deployment `web-pod-deployment` | 3 réplicas do nginx com limites de CPU/memória, readiness/liveness probes, variáveis de ambiente vindas da ConfigMap e do Secret, e a página `index.html` montada da ConfigMap |
| [`service.yaml`](service.yaml) | Service `web` (NodePort) | Expõe os pods `app=web` na porta 80 (NodePort `30080`) |
| [`configmap.yaml`](configmap.yaml) | ConfigMap `web-config` | Guarda `app_name`, `app_env` e o `index.html` da aplicação |
| [`secret.yaml`](secret.yaml) | Secret `db-secret` | Senha de exemplo em base64 (`password`) |
| [`pvc.yaml`](pvc.yaml) | PersistentVolumeClaim `app-data` | Pede 1Gi de armazenamento persistente |
| [`arquivos.yaml`](arquivos.yaml) | Deployment `arquivos` | nginx que serve arquivos gravados no volume persistente `app-data` |
| [`pod.yaml`](pod.yaml) | Pod `web` | Pod simples do nginx, o primeiro exemplo do projeto |

## 🗺️ Arquitetura

```
                 ┌──────────────────────────────┐
  localhost:8080 │  Service "web" (NodePort)    │
 ───────────────▶│  selector: app=web           │
  port-forward   └──────────────┬───────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
         ┌─────────┐       ┌─────────┐       ┌─────────┐
         │  nginx  │       │  nginx  │       │  nginx  │   Deployment
         └────┬────┘       └────┬────┘       └────┬────┘   web-pod-deployment
              └─────────────────┼─────────────────┘
                     env + index.html
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
     ConfigMap web-config                  Secret db-secret


  Deployment "arquivos" ──▶ PVC app-data (1Gi) ──▶ /usr/share/nginx/html
```

## 🚀 Como rodar

**Pré-requisitos:** um cluster Kubernetes (por exemplo o do [Docker Desktop](https://docs.docker.com/desktop/kubernetes/)) e o `kubectl` configurado.

```bash
# 1. Configurações primeiro (o Deployment depende delas)
kubectl apply -f configmap.yaml -f secret.yaml

# 2. Aplicação web
kubectl apply -f deployment.yaml -f service.yaml

# 3. Volume persistente e o Deployment que usa ele
kubectl apply -f pvc.yaml -f arquivos.yaml
```

Confira se está tudo de pé:

```bash
kubectl get pods,svc,pvc
```

## 🌐 Acessando

```bash
kubectl port-forward svc/web 8080:80
```

Depois abra **http://localhost:8080**.

Para o Deployment `arquivos`:

```bash
kubectl port-forward deploy/arquivos 8081:80
```

> **ℹ️ O volume começa vazio**, então o nginx responde `403` até você gravar um arquivo nele:
>
> ```bash
> kubectl exec deploy/arquivos -- sh -c "echo '<h1>Ola do volume</h1>' > /usr/share/nginx/html/index.html"
> ```
>
> O arquivo continua lá mesmo se o pod for recriado. Esse é o objetivo do volume persistente.

## 🔐 Decodificando o Secret

```bash
# Linux / Git Bash
kubectl get secret db-secret -o jsonpath="{.data.password}" | base64 -d
```

```powershell
# PowerShell
kubectl get secret db-secret -o jsonpath="{.data.password}" | % { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

## ⚠️ Observações

- **Não aplique o `pod.yaml` junto com o Deployment.** O Pod `web` tem o mesmo label `app=web`, então o Service também manda tráfego para ele, e ele mostra a página padrão do nginx em vez da página da ConfigMap.
- **O `index.html` não substitui variáveis.** O nginx entrega o arquivo estático exatamente como está, então `${APP_NAME}` aparece literalmente na página.
- **O Secret é só de exemplo.** Base64 não é criptografia. Não suba senhas reais em repositórios públicos.
- **O NodePort `30080` pode não abrir no Docker Desktop**, porque o nó roda dentro de um container. Use o `port-forward`.

## 🧹 Limpando

```bash
kubectl delete -f .
```

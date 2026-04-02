# Guia de Instalação — Cluster K3s com ArgoCD, MetalLB e Envoy Gateway

## Pré-requisitos

- Servidor Linux (Ubuntu 24.04 recomendado)
- Acesso root ou sudo
- Conexão com a internet

---

## 1. Instalar o K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

Dar permissão ao kubeconfig:

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

Verificar se o cluster está funcionando:

```bash
kubectl get nodes
```

---

## 2. Instalar o ArgoCD

Criar o namespace:

```bash
kubectl create ns argocd
```

Instalar o ArgoCD no cluster:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Instalar o ArgoCD CLI:

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

Desabilitar o TLS interno do ArgoCD (necessário para expor via HTTP):

```bash
kubectl patch deployment argocd-server \
  -n argocd \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--insecure"}]'
```

---

## 3. Instalar o MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Aguardar os pods subirem:

```bash
kubectl get pods -n metallb-system -w
```

### Configurar o pool de IPs

> ⚠️ **Importante:** Substitua o range de IPs abaixo por endereços da subnet do seu servidor que **não** sejam distribuídos pelo DHCP.
>
> Para descobrir a subnet do seu servidor, rode: `ip addr show`

Crie o arquivo `metallb-pool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210  # Altere para IPs livres na subnet do seu servidor

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: first-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Aplicar:

```bash
kubectl apply -f metallb-pool.yaml
```

---

## 4. Instalar o Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

---

## 5. Instalar o Envoy Gateway

O K3s já vem com CRDs da Gateway API pré-instaladas, o que causa conflito. Remova-as antes:

```bash
kubectl delete crd \
  gateways.gateway.networking.k8s.io \
  gatewayclasses.gateway.networking.k8s.io \
  httproutes.gateway.networking.k8s.io \
  grpcroutes.gateway.networking.k8s.io \
  referencegrants.gateway.networking.k8s.io \
  backendtlspolicies.gateway.networking.k8s.io
```

Instalar o Envoy Gateway via Helm:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.0 \
  -n envoy-gateway-system \
  --create-namespace
```

Verificar se os pods subiram:

```bash
kubectl get pods -n envoy-gateway-system
```

---

## 6. Configurar o Gateway

Crie o arquivo `cluster-gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cluster-gateway-gateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            senai/allowed: "true"
```

Aplicar:

```bash
kubectl apply -f cluster-gateway.yaml
```

Verificar o IP atribuído pelo MetalLB ao Gateway:

```bash
kubectl get gateway -n envoy-gateway-system
```

---

## 7. Expor o ArgoCD

Adicionar a label no namespace do ArgoCD para permitir o roteamento:

```bash
kubectl label namespace argocd senai/allowed=true
```

Crie o arquivo `argocd-httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-route
  namespace: argocd
spec:
  parentRefs:
  - name: cluster-gateway-gateway
    namespace: envoy-gateway-system
  hostnames:
  - "argocd.senai.com"   # Altere para o domínio desejado
  rules:
  - backendRefs:
    - name: argocd-server
      port: 80
```

Aplicar:

```bash
kubectl apply -f argocd-httproute.yaml
```

Verificar se a rota foi aceita:

```bash
kubectl describe httproute argocd-route -n argocd | grep -A3 "Message"
```

A saída deve mostrar `Route is accepted`.

---

## 8. Configurar DNS

Em cada máquina que for acessar o cluster, adicione o IP do Gateway no arquivo `hosts` apontando para o domínio configurado.

**Linux/Mac** — `/etc/hosts`:
```
192.168.1.200   argocd.senai.com
```

**Windows** — `C:\Windows\System32\drivers\etc\hosts`:
```
192.168.1.200   argocd.senai.com
```

> 💡 Em produção, configure um servidor DNS interno (ex: Pi-hole) para que todos os usuários resolvam os domínios automaticamente sem precisar editar o `hosts` individualmente.

---

## 9. Verificar instalação completa

```bash
# Nodes
kubectl get nodes

# Pods do sistema
kubectl get pods -n metallb-system
kubectl get pods -n envoy-gateway-system
kubectl get pods -n argocd

# Gateway e IP externo
kubectl get gateway -n envoy-gateway-system

# Rota do ArgoCD
kubectl get httproute -n argocd
```

Acesse `http://argocd.senai.com` no browser para confirmar que tudo está funcionando.

---

## Senha inicial do ArgoCD

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login: `admin` / Senha: resultado do comando acima.

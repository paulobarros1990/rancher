# RKE2 + Rancher + NGINX Ingress + cert-manager (CA Interna)

Este repositório documenta **todo o passo a passo**, baseado em um cenário real, para criação de um cluster Kubernetes com **RKE2**, **MetalLB**, **NGINX Ingress Controller**, **Rancher** e **cert-manager** utilizando **CA interna**, incluindo erros encontrados e como foram corrigidos.

---

## Visão Geral da Arquitetura

- **Distribuição Kubernetes:** RKE2 (stable)
- **Masters:** 3 nós (Control Plane + etcd)
- **Workers:** 3 nós
- **Ingress Controller:** NGINX
- **LoadBalancer:** MetalLB (L2)
- **Gerenciamento:** Rancher
- **TLS:** cert-manager com CA interna

---

## 1. Provisionamento dos Servidores

### Topologia

| Hostname | Função | IP |
|--------|------|----|
| rke2-master-01 | Control Plane / etcd | 192.168.122.110 |
| rke2-master-02 | Control Plane / etcd | 192.168.122.111 |
| rke2-master-03 | Control Plane / etcd | 192.168.122.112 |
| rke2-worker-01 | Worker | 192.168.122.120 |
| rke2-worker-02 | Worker | 192.168.122.121 |
| rke2-worker-03 | Worker | 192.168.122.122 |

Pré-requisitos mínimos:
- Linux (Rocky / Alma / RHEL / Ubuntu)
- 2 vCPU (mínimo)
- 4 GB RAM (mínimo)
- Swap desabilitado
- Timezone e NTP configurados

---

## 2. Cópia da Chave SSH (Fedora → Nodes)

```bash
ssh-copy-id root@rke2-master-01
ssh-copy-id root@rke2-master-02
ssh-copy-id root@rke2-master-03
ssh-copy-id root@rke2-worker-01
ssh-copy-id root@rke2-worker-02
ssh-copy-id root@rke2-worker-03
```

---

## 3. Instalação dos Pré-requisitos

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

swapoff -a
sed -i '/swap/d' /etc/fstab

modprobe br_netfilter
cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF
sysctl --system
```

---

## 4. Instalação do RKE2

### Masters

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
systemctl enable rke2-server --now
```

Token:
```bash
cat /var/lib/rancher/rke2/server/node-token
```

### Workers

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -
systemctl enable rke2-agent --now
```

---

## 5. Instalação do MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Configuração L2:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.200-192.168.122.230
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
```

---

## 6. Instalação do NGINX Ingress Controller

### Problema Encontrado
O RKE2 **não instala automaticamente** o controller, apenas o `rke2-ingress-nginx-controller-admission`.

### Correção

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

Validação:
```bash
kubectl -n ingress-nginx get pods
```

---

## 7. Instalação do Rancher

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

helm install rancher rancher-latest/rancher   --namespace cattle-system   --create-namespace   --set hostname=rancher.lab.local
```

---

## 8. Instalação e Configuração do cert-manager (CA Interna)

### Instalação

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```

### CA Root

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: rancher-ca-issuer
spec:
  ca:
    secretName: rancher-root-ca-secret
```

### Certificate do Rancher

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  commonName: rancher.lab.local
  dnsNames:
    - rancher.lab.local
  issuerRef:
    name: rancher-ca-issuer
    kind: ClusterIssuer
```

### Correção Importante
- O `secretName` **não deve existir previamente**
- O cert-manager cria o Secret automaticamente
- Ingress deve apontar exatamente para este Secret

---

## Evidências

### 1️⃣ Cluster em Produção com HTTPS Funcionando

- Rancher acessível via HTTPS
- Certificado emitido por CA interna
- Cluster com 6 nós ativos (3 masters + 3 workers)

📸 Evidência: Rancher Dashboard com Cluster Healthy

---

### 2️⃣ Deployment de Teste para Validação do cert-manager

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test --port=80
```

Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.lab.local
    secretName: nginx-test-tls
  rules:
  - host: app.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-test
            port:
              number: 80
```

Resultado:
- Página padrão do NGINX acessível via HTTPS
- Certificado emitido corretamente

📸 Evidência: "Welcome to nginx!"

---

## Conclusão

Este ambiente valida:
- Alta disponibilidade do control plane
- Exposição via LoadBalancer (MetalLB)
- Ingress funcional com NGINX
- TLS totalmente operacional com cert-manager
- Rancher em produção com CA interna


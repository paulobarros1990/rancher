# RKE2 + Rancher (HA) – Guia Completo de Instalação (Passo a Passo)

Este documento descreve **do zero** a criação de um ambiente **Kubernetes RKE2 em Alta Disponibilidade**, com **MetalLB**, **Ingress NGINX**, **Rancher** e **TLS usando cert-manager com CA interna**.

O conteúdo foi construído a partir de uma instalação real em laboratório, incluindo **erros encontrados**, **causas raiz** e **correções aplicadas**, garantindo um **runbook confiável e reproduzível**.

---

## Visão Geral do Ambiente

- Kubernetes: **RKE2 (stable)**
- Topologia: **3 Masters + 3 Workers**
- LoadBalancer: **MetalLB (Layer 2)**
- Ingress Controller: **ingress-nginx (RKE2)**
- Gerenciamento: **Rancher**
- TLS: **cert-manager + CA interna**
- Sistema Operacional: **Rocky Linux / RHEL-like**
- Ambiente: **Lab on‑prem / virtualizado**

---

## Endereçamento de Exemplo

### Masters
| Hostname | IP |
|--------|----|
| rke2-master-01 | 192.168.122.110 |
| rke2-master-02 | 192.168.122.111 |
| rke2-master-03 | 192.168.122.112 |

### Workers
| Hostname | IP |
|--------|----|
| rke2-worker-01 | 192.168.122.120 |
| rke2-worker-02 | 192.168.122.121 |
| rke2-worker-03 | 192.168.122.122 |

### MetalLB
```
192.168.122.200 – 192.168.122.220
```

---

# 1️⃣ Provisionamento dos Servidores

1. Instalar Rocky Linux / RHEL-like
2. Configurar IP fixo
3. Definir hostname persistente
4. Atualizar o sistema

```bash
hostnamectl set-hostname rke2-master-01
dnf update -y
```

Desabilitar componentes não suportados pelo Kubernetes (LAB):

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
systemctl disable --now firewalld
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

---

# 2️⃣ Cópia da Chave SSH (Fedora → Servidores)

Na máquina de administração (Fedora):

```bash
ssh-keygen -t ed25519
ssh-copy-id root@192.168.122.110
ssh-copy-id root@192.168.122.111
ssh-copy-id root@192.168.122.112
ssh-copy-id root@192.168.122.120
ssh-copy-id root@192.168.122.121
ssh-copy-id root@192.168.122.122
```

Objetivo:
- Permitir automação
- Evitar autenticação por senha

---

# 3️⃣ Instalação dos Pré‑Requisitos

Executar em **todos os nós (masters e workers)**:

```bash
modprobe overlay
modprobe br_netfilter

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
dnf install -y curl vim chrony bash-completion
systemctl enable --now chronyd
```

---

# 4️⃣ Instalação do RKE2 (Masters e Workers)

## 4.1 Master 01

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable sh -
systemctl enable --now rke2-server
```

Token do cluster:

```bash
cat /var/lib/rancher/rke2/server/node-token
```

## 4.2 Masters adicionais

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable sh -
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://rke2-master-01:9345
token: <TOKEN>
EOF
systemctl enable --now rke2-server
```

## 4.3 Workers

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_TYPE=agent sh -
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://rke2-master-01:9345
token: <TOKEN>
EOF
systemctl enable --now rke2-agent
```

---

# 5️⃣ Instalação do MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Configuração do pool:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.200-192.168.122.220
```

---

# 6️⃣ Ingress NGINX (RKE2)

## Problema Encontrado

Inicialmente apenas o service abaixo existia:

```
rke2-ingress-nginx-controller-admission
```

Esse service **não expõe aplicações**.

## Correção

Criar `HelmChartConfig` do RKE2:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |
    controller:
      service:
        enabled: true
        type: LoadBalancer
```

Resultado esperado:

```bash
kubectl -n kube-system get svc rke2-ingress-nginx-controller
```

---

# 7️⃣ Instalação do Rancher

```bash
helm repo add rancher https://releases.rancher.com/server-charts/stable
helm repo update

helm install rancher rancher/rancher   --namespace cattle-system   --create-namespace   --set hostname=rancher.lab.local
```

---

# 8️⃣ cert-manager + TLS com CA Interna

## 8.1 Instalar cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

## 8.2 Criar CA interna

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: rancher-ca-issuer
spec:
  ca:
    secretName: rancher-root-ca-secret
```

## 8.3 Criar Certificate do Rancher

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

Ingress deve apontar para o mesmo `secretName`.

---

## Validação Final

```bash
kubectl get nodes
kubectl -n kube-system get svc rke2-ingress-nginx-controller
kubectl -n cattle-system get certificate
kubectl -n cattle-system get ingress rancher
```

Acesso:
```
https://rancher.lab.local
```

---

## Conclusão

✔ Cluster RKE2 HA funcional  
✔ Ingress exposto via MetalLB  
✔ Rancher operacional  
✔ TLS válido com CA interna  
✔ Runbook reproduzível  

---

Autor:  
Paulo Henrique Barros  
Linux | DevOps | Kubernetes | Cloud

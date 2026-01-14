# 🚀 RKE2 + Rancher (HA) – Guia Completo de Instalação (Lab Produção-Like)

Este repositório documenta **passo a passo** a criação de um ambiente **Kubernetes RKE2 em Alta Disponibilidade**, com **Ingress NGINX**, **MetalLB**, **Rancher** e **TLS via cert-manager com CA interna**.

O guia foi validado em laboratório local e inclui **problemas reais encontrados**, suas **causas raiz** e as **correções aplicadas**, servindo como **runbook confiável** para recriação do ambiente do zero.

---

## 📐 Arquitetura do Ambiente

### Control Plane (HA – etcd + control-plane)
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

### LoadBalancer (MetalLB)
```
192.168.122.200 – 192.168.122.220
```

---

## 🧱 Componentes Utilizados

- **Kubernetes**: RKE2 (canal `stable`)
- **Container Runtime**: containerd (nativo RKE2)
- **Ingress Controller**: ingress-nginx (RKE2)
- **LoadBalancer**: MetalLB (Layer 2)
- **Gerenciamento**: Rancher
- **TLS**: cert-manager + CA interna
- **SO**: Rocky Linux / RHEL-like
- **Ambiente**: Lab on-prem / virtualizado

---

## 📋 Pré-Requisitos

- Máquinas com IP fixo configurado
- Acesso SSH como `root`
- DNS local ou `/etc/hosts`
- Swap desabilitado
- SELinux permissive/disabled (lab)
- Firewall desabilitado (lab)

---

## 📌 Etapas da Instalação

### 1️⃣ Provisionamento dos Servidores
- Configuração de IP fixo
- Definição de hostname persistente
- Ajustes básicos de SO

### 2️⃣ Cópia da Chave SSH
Permite automação e administração centralizada a partir da máquina Fedora.

### 3️⃣ Instalação dos Pré-Requisitos
- Desabilitar swap
- Ajustar sysctl do Kubernetes
- Carregar módulos do kernel
- Habilitar chrony

### 4️⃣ Instalação do RKE2 (stable)
- 3 masters (server)
- 3 workers (agent)
- Cluster em HA com etcd distribuído

### 5️⃣ MetalLB
- Implementação de LoadBalancer on-prem
- Pool de IPs dedicado

### 6️⃣ Ingress NGINX (RKE2)

#### ⚠️ Problema encontrado
Inicialmente apenas o service abaixo estava presente:
```
rke2-ingress-nginx-controller-admission
```

Esse service é **apenas webhook interno** e **não expõe tráfego HTTP/HTTPS**.

#### ✅ Correção aplicada
Foi necessário aplicar uma **HelmChartConfig** do RKE2 para garantir a criação do service principal do controller como `LoadBalancer`:

```yaml
controller:
  service:
    enabled: true
    type: LoadBalancer
```

Após isso:
- `rke2-ingress-nginx-controller` passou a expor **80/443**
- MetalLB atribuiu IP externo corretamente

---

### 7️⃣ Instalação do cert-manager
Responsável pela emissão e gerenciamento de certificados TLS.

### 8️⃣ CA Interna (Recomendado para LAB)
- Criação de CA Root interna
- ClusterIssuer baseado em CA
- Evita dependência externa (Let's Encrypt)

### 9️⃣ Instalação do Rancher
- Deploy via Helm
- Integrado ao Ingress NGINX
- Acesso via hostname (`rancher.lab.local`)

---

## 🔐 TLS do Rancher – Problema e Correção

### ❌ Sintoma
- Browser apresentava:
  ```
  Kubernetes Ingress Controller Fake Certificate
  ```
- Certificate ficava em estado `DoesNotExist`

### 🔍 Causa Raiz
- Ingress apontava para um `secretName` inexistente
- O `Certificate` do cert-manager ainda não havia criado o Secret
- NGINX faz fallback automático para Fake Certificate

### ✅ Correção Definitiva
1. Criar o `Certificate` com CA interna
2. Garantir que o Secret TLS fosse criado
3. Apontar o Ingress para o **mesmo `secretName`**
4. Recarregar o ingress-nginx

Resultado:
- TLS válido
- Certificado assinado pela CA interna
- Rancher funcional sem Fake Certificate

---

## ✅ Validação Final

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

## 🧠 Observações Importantes

- O taint `CriticalAddonsOnly=true:NoExecute` nos masters é **comportamento esperado**
- cert-manager **não cria Secret manualmente**
- `secretName` deve ser **idêntico** no Certificate e no Ingress
- Ingress NGINX do RKE2 roda como **DaemonSet**
- O service `*-admission` **não expõe aplicações**

---

## 📂 Estrutura Recomendada do Repositório

```
.
├── README.md
├── runbook.sh
├── docs/
│   ├── arquitetura.md
│   ├── ingress-nginx.md
│   ├── cert-manager.md
│   ├── troubleshooting.md
```

---

## 🎯 Status do Ambiente

✔ Kubernetes HA funcional  
✔ Ingress exposto via MetalLB  
✔ Rancher acessível  
✔ TLS válido com CA interna  
✔ Ambiente reproduzível  

---

## 📌 Próximos Passos (opcional)

- Automação total via Ansible
- Backup do etcd
- Integração com LDAP/AD
- Observabilidade (Prometheus + Grafana)
- Migração futura para domínio público

---

**Autor**  
Paulo Henrique Barros  
Linux | DevOps | Kubernetes | Cloud

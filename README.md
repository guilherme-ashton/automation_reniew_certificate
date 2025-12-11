# automation_reniew_certificate
Repository containing a CI/CD workflow for automating a TLS certificate renewal process.


# 🔐 Automação Completa de Renovação de Certificados TLS no GCP

Este projeto implementa uma solução **100% automatizada**, segura e escalável para renovação e distribuição de certificados TLS usando:

- **Cloud Build**
- **Secret Manager**
- **Compute Engine (VM)**
- **Certbot (DNS Challenge)**
- **Cloud DNS**
- **Kubernetes / GKE**

Ele elimina processos manuais, reduz riscos de segurança e garante que todos os serviços tenham sempre certificados atualizados e válidos.

---

## 📌 Visão Geral da Arquitetura

O fluxo de automação funciona assim:

1. O **Cloud Build** dispara o pipeline.
2. A pipeline baixa scripts e chaves do **Secret Manager**.
3. Os scripts são enviados para uma **VM Compute Engine**, onde o Certbot é executado.
4. O Certbot valida o domínio usando **DNS Challenge**, criando automaticamente o registro TXT no **Cloud DNS**.
5. A VM gera o certificado atualizado.
6. O Cloud Build baixa os certificados gerados.
7. O certificado é publicado como **Secret TLS no Kubernetes**, e replicado para todos os namespaces que utilizam TLS.

---

## 🧠 Por que esse processo é importante?

- Substitui renovações manuais por automação total  
- Elimina erros humanos  
- Reduz risco de expiração do certificado  
- Aumenta a segurança através de IAM mínimo e controle centralizado  
- Simplifica a manutenção de múltiplos serviços em GKE  

---

## ⚙️ Fluxo Completo do Processo Automatizado

### **1. Cloud Build baixa arquivos sensíveis**
Scripts e chaves são recuperados do Secret Manager:

- `renovar-certificado.sh`
- `prepare-cert-files.sh`
- `auth-hook.sh`
- `cleanup-hook.sh`
- Chaves SSH
- Service Account do DNS Challenge

Cada script é versionado automaticamente dentro do Secret Manager.

---

### **2. Cloud Build transfere scripts e chaves para a VM**
A pipeline usa `gcloud compute scp` para copiar:

- Scripts de hook  
- Script principal do Certbot  
- Chaves necessárias  

Tudo é colocado em `/tmp` na VM.

---

### **3. Execução do Certbot via DNS Challenge**

A VM roda:

```
certbot certonly   --manual   --manual-auth-hook /tmp/auth-hook.sh   --manual-cleanup-hook /tmp/cleanup-hook.sh   -d your-domain   -d your-domain
```

O **auth-hook** cria automaticamente no Cloud DNS o registro:

```
txt-dns TXT <token>
```

Após validação, o Certbot gera:

- `fullchain.pem`
- `privkey.pem`

---

### **4. Preparação dos arquivos na VM**
O script `prepare-cert-files.sh`:

- Copia os certificados para `/tmp/letsencrypt`
- Ajusta permissões seguras (644 → público / 600 → chave privada)

---

### **5. Cloud Build coleta os certificados**
O Cloud Build baixa os arquivos da VM para o workspace:

```
/workspace/certificado.pem
/workspace/private_key.pem
```

Com permissões seguras:

```
chmod 644 certificado.pem
chmod 600 private_key.pem
```

---

### **6. Atualização dos Secrets TLS no Kubernetes (GKE)**

O Cloud Build:

1. Atualiza o secret principal (`tls-secret`) no namespace root  
2. Remove secrets antigos nos namespaces
3. Replica automaticamente o secret novo para todos os namespaces

Os Ingress Controllers passam a servir o certificado renovado imediatamente.

---

# 🔒 Melhorias de Segurança Implementadas

### ✔ Permissões mínimas da chave privada (600)  
### ✔ Remoção de logs sensíveis (`CERTBOT_VALIDATION`)  
### ✔ Role IAM custom para Certbot (DNS mínimo necessário)  
### ✔ Minimização de cópias da chave privada  
### ✔ Cloud Build endurecido (chmod + pipeline seguro)  
### ✔ RBAC no Kubernetes para impedir leitura de secrets  
### ✔ Secrets versionados no Secret Manager  
### ✔ Execução efêmera através de containers Docker do Cloud Build  

---

## 📦 Arquivos no Secret Manager

| Nome do Secret | Função |
|----------------|--------|
| `renovar-certificado-sh` | Executa Certbot |
| `prepare-cert-files` | Move certs da VM para `/tmp/letsencrypt` |
| `auth-hook-script` | Cria registro TXT no Cloud DNS |
| `cleanup-hook-script` | Legacy para remoção do TXT |
| `ssh-private-key` | Conexão segura com a VM |
| `ssh-public-key` | Par da chave privada |
| `sa-certbot` | Service Account para o DNS Challenge |

---

## 🐳 Uso de Docker no Processo

Embora nenhuma imagem Docker própria tenha sido criada, todas as etapas do Cloud Build são executadas dentro de **containers Docker oficiais**, como:

- `gcr.io/cloud-builders/gcloud`
- `gcr.io/cloud-builders/kubectl`

A VM onde o Certbot executa **não utiliza Docker**, mas todo o pipeline é containerizado pelo Cloud Build.

---

---

# Conclusão

Este projeto garante:

- Automação total da renovação de certificados TLS  
- Redução de risco operacional e de falhas humanas  
- Segurança reforçada com IAM mínimo, RBAC e controle de secrets  
- Distribuição automática do certificado para todos os serviços Kubernetes  
- Processo auditável, padronizado e de baixa manutenção  

---


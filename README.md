# Deploy de WordPress na AWS com Docker

---

## 📖 Índice
1. [🚀 Introdução](#🚀-1-introdução)  
2. [🛠️ Pré-requisitos](#🛠️-2-pré-requisitos)  
3. [☁️ Criação da Rede AWS](#☁️-3-criação-da-rede-aws)  
4. [🛡️ Configuração de Grupos de Segurança](#🛡️-4-configuração-de-grupos-de-segurança)  
5. [🏦 Preparando o Banco de Dados MySQL (RDS)](#🏦-5-preparando-o-banco-de-dados-mysql-rds)  
6. [📁 Configuração do EFS para Arquivos Estáticos](#📁-6-configuração-do-efs-para-arquivos-estáticos)  
7. [⬆️ Configuração do Auto Scaling Group](#⬆️-7-configuração-do-auto-scaling-group)  
8. [⚖️ Configurando o Load Balancer](#⚖️-8-configurando-o-load-balancer)  
9. [🐳 Verificação da Configuração no Host EC2](#🐳-9-verificação-da-configuração-no-host-ec2)  
10. [📊 Conclusão](#📊-10-conclusão)  

---

## 🚀 1. Introdução
### 1.1 Objetivo do Projeto  
Implantar um WordPress escalável e altamente disponível na AWS usando Docker, RDS (MySQL), EFS, Auto Scaling e Load Balancer.  

### 1.2 Visão Geral da Arquitetura  
- **EC2**: Instâncias com containers Docker.  
- **RDS**: Banco de dados MySQL gerenciado.  
- **EFS**: Armazenamento compartilhado para arquivos do WordPress.  
- **Load Balancer**: Distribui tráfego entre instâncias.  
- **Auto Scaling**: Escalabilidade automática baseada em demanda.  

---

## 🛠️ 2. Pré-requisitos  
- Conta AWS ativa.  
- Conhecimento básico em Docker, AWS (EC2, RDS, EFS) e WordPress.  
- Chave SSH criada na AWS (*Project2.pem*).  

---

## ☁️ 3. Criação da Rede AWS  
### 3.1 VPC  
- **Nome**: `Project2-vpc`  
- **CIDR**: `10.0.0.0/16`  

### 3.2 Internet Gateway (IGW)  
- Associe à VPC criada.  

### 3.3 Subnets  
- **Públicas**:  
  - `10.0.1.0/24` (us-east-1a)  
  - `10.0.3.0/24` (us-east-1b)  
- **Privadas**:  
  - `10.0.2.0/24` (us-east-1a)  
  - `10.0.4.0/24` (us-east-1b)  

### 3.4 NAT Gateway  
- Crie um em cada subnet pública com Elastic IP.  

### 3.5 Tabelas de Rotas  
- **Pública**: Rota `0.0.0.0/0` via IGW.  
- **Privada**: Rota `0.0.0.0/0` via NAT Gateway.  

---

## 🛡️ 4. Configuração de Grupos de Segurança  
### 4.1 Load Balancer (`Project2-CLB-SG`)  
- **Entrada**: Portas 80 (HTTP) e 443 (HTTPS) de `0.0.0.0/0`.  
- **Saída**: Portas 80/443 para o grupo `Project2-EC2-SG`.  

### 4.2 EC2 (`Project2-EC2-SG`)  
- **Entrada**: Porta 80 do `Project2-CLB-SG` e SSH (22) do mesmo grupo.  

### 4.3 RDS & EFS (`Project2-RDS&EFS-SG`)  
- **Entrada**: Porta 3306 (MySQL) e 2049 (NFS) do `Project2-EC2-SG`.  

---

## 🏦 5. Preparando o Banco de Dados MySQL (RDS)  
1. **Criação do RDS**:  
   - **Engine**: MySQL 8.0.  
   - **Instance Class**: `db.t3.micro`.  
   - **VPC**: `Project2-vpc`.  
   - **Grupo de Segurança**: `Project2-RDS&EFS-SG`.  
   - **Database Name**: `wordpress`.  

2. **Anote o Endpoint do RDS** para uso no Docker Compose.  

---

## 📁 6. Configuração do EFS para Arquivos Estáticos  
1. **Crie um Sistema de Arquivos EFS**:  
   - **Nome**: `Project2-efs`.  
   - **VPC**: `Project2-vpc`.  
   - **Mount Targets**: Subnets privadas com grupo `Project2-RDS&EFS-SG`.  

2. **Anote os IPs dos Mount Targets** para montagem nas EC2.  

---

## ⬆️ 7. Configuração do Auto Scaling Group  
### 7.1 Launch Template  
- **AMI**: Amazon Linux 2023.  
- **Instance Type**: `t2.micro`.  
- **User Data**: Script para instalar Docker, montar EFS e subir WordPress.  
  ```bash
  #!/bin/bash
      #!/bin/bash
  # Variáveis
  EFS_VOLUME="/mnt/efs"
  WORDPRESS_VOLUME="/var/www/html"
  DATABASE_HOST="-------"
  DATABASE_USER="------"
  DATABASE_PASSWORD="-------"
  DATABASE_NAME="-----"
  
  
  
  # Atualização do sistema
  sudo yum update -y
  
  
  
  # Instalação do Docker e do utilitário EFS
  sudo yum install docker -y
  sudo yum install amazon-efs-utils -y
  
  
  
  # Adição do usuário ao grupo Docker
  sudo usermod -aG docker $(whoami)
  
  
  
  # Inicialização e ativação do serviço Docker
  sudo systemctl start docker
  sudo systemctl enable docker
  
  
  
  # Criação do ponto de montagem EFS
  sudo mkdir -p $EFS_VOLUME
  
  
  
  # Montagem do volume EFS
  if ! mountpoint -q $EFS_VOLUME; then
    echo "Montando volume EFS..."
    sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport -------------:/ $EFS_VOLUME
  else
    echo "Volume EFS já montado."
  fi
  
  
  
  # Ajuste de permissões do EFS
  sudo chown -R 33:33 $EFS_VOLUME   # Usuário do Apache/Nginx no container
  sudo chmod -R 775 $EFS_VOLUME
  
  
  
  # Download do Docker Compose
  curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /bin/docker-compose
  chmod +x /bin/docker-compose
  
  
  
  # Criação do arquivo docker-compose.yaml
  cat <<EOL > /home/ec2-user/docker-compose.yaml
  version: '3.8'
  services:
    wordpress:
      image: wordpress:latest
      volumes:
        - $EFS_VOLUME$WORDPRESS_VOLUME:/$WORDPRESS_VOLUME
      ports:
        - 80:80
      environment:
        WORDPRESS_DB_HOST: $DATABASE_HOST
        WORDPRESS_DB_USER: $DATABASE_USER
        WORDPRESS_DB_PASSWORD: $DATABASE_PASSWORD
        WORDPRESS_DB_NAME: $DATABASE_NAME
  EOL
  ```

### 7.2 Auto Scaling Group  
- **Desired Capacity**: 2.  
- **Mín/Máx**: 2/3.  
- **Escalonamento**: Baseado em 70% de uso de CPU.  

---

## ⚖️ 8. Configurando o Load Balancer  
1. **Crie um Classic Load Balancer**:  
   - **Subnets**: Públicas.  
   - **Health Check**: `/wp-admin/install.php`.  

2. **Associe ao Auto Scaling Group**.  

3. **Teste o DNS do LB** no navegador.  

---

## 🐳 9. Verificação da Configuração no Host EC2  
1. **EC2 Instance Connect Endpoint**:  
   - Crie um endpoint para acessar instâncias privadas via SSH.  

2. **Comandos de Verificação**:  
   ```bash
   docker ps
   ls /mnt/efs/wordpress_data
   ``

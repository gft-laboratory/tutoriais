# tutoriais

# Procedimento de Configuração de EC2 para Agente do GitHub Actions

Este documento descreve os passos necessários para preparar uma instância EC2 com Ubuntu, instalar o Terraform e configurar o agente do GitHub Actions.

---

## 📋 Pré-requisitos

| Requisito                | Valor                                                                 |
|--------------------------|-----------------------------------------------------------------------|
| Sistema Operacional      | Ubuntu Linux 24.04                                                    |
| Tipo da Instância EC2    | `t2.medium`                                                           |
| AMI                      | `ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-20250610` |
| IAM Role (Instance Profile) | `instance-profile-agent` (com permissões para EC2, RDS, IAM Role e Policy) |
| Security Group           | Portas `80` e `443` liberadas para `0.0.0.0/0`                         |

---

## ⚙️ Instalação do Terraform na Máquina

Execute os comandos abaixo na instância EC2:

```bash
# Atualiza o sistema e instala dependências
sudo apt-get update -y
sudo apt-get install -y gnupg software-properties-common curl unzip

# Adiciona o repositório da HashiCorp
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Instala o Terraform
sudo apt-get update -y
sudo apt-get install -y terraform

# Autocomplete (opcional)
terraform -install-autocomplete

# Verifica a versão
terraform -version
```

# 🤖 Instalação e Configuração do Agente do GitHub Actions
Execute o script abaixo na instância EC2 para configurar o agente:

```bash
#!/bin/bash
# ubuntu image

apt-get update -y
apt-get install -y snapd
snap install amazon-ssm-agent --classic
systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service

# Create a folder and install
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.326.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.326.0/actions-runner-linux-x64-2.326.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.326.0.tar.gz

# Create the runner and start the configuration experience
./config.sh --url https://github.com/gft-laboratory/modules-consumer --token AEBICIZ2B5FHOADXKOV5VELIOAT66
nohup ./run.sh > log.txt 2>&1 &
```

O token fornecido expira rapidamente. A configuração deve ser realizada em tempo real (4 mãos).

# ✅ Finalização
Após seguir os passos acima, sua instância EC2 estará pronta para executar pipelines do GitHub Actions com Terraform.

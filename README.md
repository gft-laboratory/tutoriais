# tutoriais

# Procedimento de Configuração de EC2 para Agente do GitHub Actions

Este documento descreve os passos necessários para preparar uma instância EC2 com Ubuntu, instalar o Terraform e configurar o agente do GitHub Actions.

---

## 📋 Pré-requisitos

| Requisito                | Valor                                                                 |
|--------------------------|-----------------------------------------------------------------------|
| Sistema Operacional      | Ubuntu Linux 24.04 ou Amazon Linux 2                                  |
| Tipo da Instância EC2    | `t2.medium`                                                           |
| AMI                      | `ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-20250610` ou amazon-linux-2-latest-amd64 (ou ID: ami-0c55b159cbfafe1f0 na região us-east-1, ajustar conforme a região) |
| IAM Role (Instance Profile) | `instance-profile-agent` (com permissões para EC2, RDS, IAM Role e Policy) |
| Security Group           | Portas `80` e `443` liberadas para `0.0.0.0/0`                         |

---

## ⚙️ Instalação do Terraform na Máquina

Execute os comandos abaixo na instância EC2:

Debian/Ubuntu (original):
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

CentOS 7/8 ou Amazon LInux2:
```bash
# Atualiza o sistema
sudo yum update -y

# Instala dependências
sudo yum install -y yum-utils curl unzip gnupg

# Adiciona o repositório da HashiCorp
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

# Instala o Terraform
sudo yum install -y terraform

# Autocomplete (opcional, pode variar dependendo da shell)
terraform -install-autocomplete

# Verifica a versão
terraform -version
```

# 🤖 Instalação e Configuração do Agente do GitHub Actions
Execute o script abaixo na instância EC2 para configurar o agente:

Debian/Ubuntu (original):
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
# Verifique a versão mais recente: https://github.com/actions/runner/releases
curl -o actions-runner-linux-x64-2.326.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.326.0/actions-runner-linux-x64-2.326.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.326.0.tar.gz

# Create the runner and start the configuration experience
./config.sh --url https://github.com/gft-laboratory/modules-consumer --token AEBICIZ2B5FHOADXKOV5VELIOAT66
nohup ./run.sh > log.txt 2>&1 &
```

CentOS 7/8 ou Amazon Linux 2
```bash
#!/bin/bash
# Amazon Linux 2 ou CentOS 7/8

# Atualiza pacotes e instala dependências
sudo yum update -y
sudo yum install -y curl tar unzip

# Instala o Amazon SSM Agent (caso ainda não esteja presente)
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

# Cria o diretório do GitHub Actions Runner
mkdir actions-runner && cd actions-runner

# Baixa o runner (ajuste a versão conforme necessário)
# Verifique a versão mais recente: https://github.com/actions/runner/releases
curl -o actions-runner-linux-x64-2.326.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.326.0/actions-runner-linux-x64-2.326.0.tar.gz

# Extrai o conteúdo
tar xzf ./actions-runner-linux-x64-2.326.0.tar.gz

# Executa a configuração do runner (substitua o token conforme necessário)
./config.sh --url https://github.com/gft-laboratory/modules-consumer --token AEBICIZ2B5FHOADXKOV5VELIOAT66

# Inicia o runner em segundo plano
nohup ./run.sh > log.txt 2>&1 &
```
O token fornecido expira rapidamente. A configuração deve ser realizada em tempo real (4 mãos).


### Amazon Linux 2 ou CentOS 7/8 - Tornando o ./run.sh auto-executável
```hcl
sudo vi /etc/systemd/system/myagent.service
```

Adicione o seguinte conteúdo no arquivo .service:
```hcl
[Unit]
Description=GitHub Actions Runner Script
After=network.target

[Service]
Type=simple
User=ssm-user
WorkingDirectory=/home/ssm-user/actions-runner
ExecStart=/usr/bin/nohup /home/ssm-user/actions-runner/run.sh
Restart=always
StandardOutput=append:/home/ssm-user/actions-runner/log.txt
StandardError=append:/home/ssm-user/actions-runner/log.txt

[Install]
WantedBy=multi-user.target
```

### Ativando o serviço e testando
```hcl
sudo systemctl daemon-reexec           # Reinicia systemd se necessário
sudo systemctl daemon-reload           # Carrega novo serviço
sudo systemctl enable myagent.service  # Ativa para iniciar com o sistema
sudo systemctl start myagent.service   # Inicia agora
systemctl status myagent.service
```




# Troubleshooting
## Amazon Linux 2 ou CentOS 7/8 - Erro por falta do pacote SDK

```hcl
sudo yum update -y
sudo yum install -y wget yum-utils
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm
sudo yum install -y dotnet-sdk-6.0
dotnet --version
```

## Amazon Linux 2 ou CentOS 7/8 - Erro de curl: (7) Failed to connect to 169.254.169.254 port 80 after 0 ms: Could not connect to server

Se ao configurar uma pipeline, e o comando `terraform init`estiver emitindo o seguinte erro abaixo:
```hcl
Run terraform init -upgrade

Initializing the backend...
Upgrading modules...
Downloading git::https://github.com/gft-laboratory/modules-shared.git?ref=main for aurora_cluster...
- aurora_cluster in .terraform/modules/aurora_cluster/RDS-AuroraGlobal
╷
│ Error: No valid credential sources found
│ 
│ Please see https://www.terraform.io/docs/language/settings/backends/s3.html
│ for more information about providing credentials.
│ 
│ Error: failed to refresh cached credentials, no EC2 IMDS role found,
│ operation error ec2imds: GetMetadata, exceeded maximum number of attempts,
│ 3, request send failed, Get
│ "http://169.254.169.254/latest/meta-data/iam/security-credentials/": dial
│ tcp 169.254.169.254:80: connect: connection refused
```

Significa que a role associada a EC2, não é possivel ser acessada pelo usuário local. Valide com o comando abaixo, acessando a EC2 e rodando o comando `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/`

E for exibido a informação  `curl: (7) Failed to connect to 169.254.169.254 port 80 after 0 ms: Could not connect to server

Se faz necessário validar bloqueios locais da EC2.

Execute:
`sudo iptables -L -n` 
O resultado for:
<img width="1132" height="263" alt="image" src="https://github.com/user-attachments/assets/678d0fbb-2a0d-4d4f-8d61-9b72c062aa1c" />

Significa que somente o 1001 e 1002 UID tem permissão para chamar o meta-data. O ssm-user que geralmente é o UID 1003 precisa dessa liberação, assim rode o comando:
`sudo iptables -I OUTPUT -d 169.254.169.254 -m owner --uid-owner 1003 -j ACCEPT`

O resultado será:
```hcl
ACCEPT  all  --  0.0.0.0/0  169.254.169.254  owner UID match 1003
ACCEPT  tcp  --  0.0.0.0/0  169.254.169.254  owner UID match 1001
ACCEPT  tcp  --  0.0.0.0/0  169.254.169.254  owner UID match 1002
REJECT  tcp  --  0.0.0.0/0  169.254.169.254  ! owner UID match 0 reject-with icmp-port-unreachable
```

Assim teste novamente se a instancia consegue chamar o meta-data da própria EC2: `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/` e o resultado esperado é:
<img width="780" height="284" alt="image" src="https://github.com/user-attachments/assets/35da14ea-2d8c-477b-95dc-26dfe962ccca" />

Pronto, problema de acesso a role resolvida.


# ✅ Finalização
Após seguir os passos acima, sua instância EC2 estará pronta para executar pipelines do GitHub Actions com Terraform.

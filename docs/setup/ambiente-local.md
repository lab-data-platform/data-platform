# Setup do ambiente local e acesso à AWS

Guia da **camada 0**: ferramentas locais e acesso da linha de comando à conta AWS. Feito uma vez por máquina.

## Pré-requisitos

| Ferramenta | Para quê |
|---|---|
| AWS CLI v2 | Autenticar e operar a conta pelo terminal |
| Terraform ≥ 1.10 | Criar a infraestrutura |
| Python 3.12+ | Lambdas e dbt |
| Git | Versionamento |

Os comandos abaixo são para Ubuntu (WSL2, x86_64).

## 1. Instalar o AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
aws --version        # esperado: aws-cli/2.x
```

## 2. Instalar o Terraform

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
terraform version
```

## 3. Proteger a conta AWS (no console, como root)

O usuário **root** (o e-mail com que você criou a conta) só deve ser usado para tarefas administrativas da conta. Nunca crie access keys para o root.

1. Ative o **MFA no root**: menu da conta → *Security credentials* → *Multi-factor authentication*.
2. A **região do projeto** é `us-east-2` (Ohio). No plano gratuito da AWS, ela é a única região liberada pela SCP da conta ([ADR-0013](../decisoes/0013-regiao-us-east-2.md)).

## 4. Criar o acesso para a linha de comando

O tipo de conta usada **não tem IAM Identity Center (SSO) e não permite login de usuários IAM no console**. Sem login no console, o `aws login` também não está disponível. Por isso o acesso é feito por um **usuário IAM com access key**, usado só pelo CLI, e os cuidados abaixo compensam o fato de a chave não expirar.

> Se a conta passar a permitir login no console, prefira o `aws login` (AWS CLI ≥ 2.32), que gera credenciais temporárias.

### 4.1 Criar o usuário (no console, como root)

1. **IAM → Usuários → Criar usuário**, com o nome `lab-admin` → **Próximo**.
2. **Anexar políticas diretamente →** marque `AdministratorAccess` → **Próximo → Criar usuário**.
3. Abra o `lab-admin` → **Credenciais de segurança → Criar chave de acesso**.
4. Escolha **Command Line Interface (CLI)**, marque a confirmação (o alerta de "alternativas recomendadas" pode ser ignorado) → **Próximo → Criar chave de acesso**.
5. Clique em **Baixar arquivo .csv**. **O secret só aparece uma vez.**

### 4.2 Configurar o profile no terminal

```bash
aws configure --profile lab-data
# AWS Access Key ID:     <Access key ID do .csv>
# AWS Secret Access Key: <Secret access key do .csv>
# Default region name:   us-east-2
# Default output format: json

chmod 600 ~/.aws/credentials ~/.aws/config
rm <caminho-do-arquivo>.csv          # apague o .csv depois de configurar
```

### 4.3 Cuidados com a access key

- **Nunca** coloque a chave no repositório, em código, em `.tfvars` ou em print de tela. O Terraform e o dbt leem o profile direto do `~/.aws`.
- O `.gitignore` do projeto bloqueia arquivos de credencial, mas a responsabilidade é sua.
- **Rotação a cada 90 dias:** crie a chave nova, configure, teste, e depois desative e exclua a antiga.
- **Vazou?** IAM → `lab-admin` → Credenciais de segurança → **Desativar** a chave na hora, e confira o CloudTrail.
- O **root** continua só para administração da conta, com MFA e **sem** access key.

## 5. Validar o acesso

```bash
export AWS_PROFILE=lab-data          # para não repetir --profile em todo comando
aws sts get-caller-identity          # mostra Account, UserId e Arn
aws s3 ls                            # deve rodar sem erro (vazio numa conta nova)
```

O `Arn` esperado é `arn:aws:iam::<account-id>:user/lab-admin`. Se aparecer `:root`, a chave configurada é do root: exclua essa chave no console e refaça o passo 4.

Para deixar o profile fixo no terminal:

```bash
echo 'export AWS_PROFILE=lab-data' >> ~/.bashrc
```

## 6. Controle de custo

O alerta de custo é criado pelo Terraform na Etapa 1 (budget `lab-data-dev-mensal`). Não crie um budget manual no console, senão o alerta fica duplicado.

# 🚀 Projeto DevOps: Containerização de Website Estático com Docker e Deploy Manual na AWS

## 🎯 Visão Geral do Projeto

Neste projeto, o objetivo foi **containerizar um website estático (HTML, CSS e JavaScript)** usando **Docker**, realizando o **deploy manual** da aplicação em uma instância **EC2 na AWS**, utilizando o **Amazon ECR (Elastic Container Registry)** como repositório privado para a imagem Docker.

Essa abordagem aplica princípios fundamentais de **DevOps**, garantindo:

- **Portabilidade:** a aplicação roda em qualquer ambiente compatível com Docker.  
- **Isolamento:** elimina conflitos de dependências e o clássico “funciona na minha máquina”.  
- **Escalabilidade:** estabelece a base para automações futuras.  
- **Padrão da Indústria:** utiliza ferramentas amplamente adotadas no mercado.  

---

## 🔧 Ferramentas e Arquitetura

### 🧰 Ferramentas Utilizadas
- **Docker Desktop / Docker Engine (Linux)** – Construção e teste local dos containers.  
- **AWS CLI** – Interação com os serviços AWS (ECR e EC2).  
- **Conta AWS** – Provisionamento dos recursos.  
- **Visual Studio Code** – Editor de código com extensões Docker e AWS Toolkit.

### 📁 Estrutura Inicial do Projeto
```
meu-projeto/
├── website/
│   ├── index.html
│   ├── styles.css
│   ├── script.js
│   └── assets/
└── Dockerfile
```

### ☁️ Arquitetura da Solução
```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Código Local   │────▶│  Docker Image   │────▶│   Amazon ECR    │
│ (HTML/CSS/JS)   │      │  (Build Local)  │      │  (Registry)     │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                                       │
                                                       ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│    Browser      │◀────│   Amazon EC2    │◀────│ (Pull da Imagem)│
│ (Acesso Público)│      │ (Container Prod)│      └─────────────────┘
└─────────────────┘      └─────────────────┘
```

---

## 🐳 Fase 1: Containerização Local com Docker

### 1.1 Definição do `Dockerfile`
```dockerfile
# Imagem base - Nginx Alpine (leve e eficiente)
FROM nginx:alpine

# Copia os arquivos do website para o diretório padrão do Nginx
COPY website/ /usr/share/nginx/html/

# Expõe a porta 80 (documentação - o Nginx já escuta nela)
EXPOSE 80

# Comando padrão quando o container iniciar
CMD ["nginx", "-g", "daemon off;"]
```

**Lógica do Dockerfile:**
- `FROM nginx:alpine` → imagem base leve.  
- `COPY website/...` → copia os arquivos estáticos.  
- `CMD [...]` → mantém o container ativo com Nginx em modo foreground.  

### 1.2 Build e Teste Local
```bash
# Build da imagem
docker build -t meu-website:v1.0 .

# Listar imagens
docker images
```

---

## 🧪 Fase 2: Validação Local do Container

### 2.1 Execução do Container
```bash
docker run -d -p 8080:80 --name meu-website-container meu-website:v1.0
```
- `-d`: modo background  
- `-p 8080:80`: mapeia porta local → container  
- `--name`: define um nome para facilitar o gerenciamento  

### 2.2 Teste no Navegador
Acesse:  
👉 [http://localhost:8080](http://localhost:8080)

### 2.3 Limpeza Local
```bash
docker stop meu-website-container
docker rm meu-website-container
```

---

## ☁️ Fase 3: Configuração do Amazon ECR

1. Acesse o **Amazon ECR** no console AWS.  
2. Crie um **repositório privado** chamado `meu-website`.  
3. Habilite **Scan on push**.  
4. Anote a URI do repositório, ex:  
   ```
   123456789012.dkr.ecr.us-east-1.amazonaws.com/meu-website
   ```

---

## 📤 Fase 4: Push da Imagem para o ECR

### 4.1 Autenticação
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

### 4.2 Tag e Push
```bash
# Tag
docker tag meu-website:v1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
```

---

## 🖥️ Fase 5: Provisionamento da Instância EC2

### Configurações principais:
- **AMI:** Amazon Linux 2023 (Free Tier)  
- **Tipo:** t2.micro  
- **Key Pair:** `meu-website-key.pem`  
- **Security Group:**  
  - SSH (TCP/22): Meu IP  
  - HTTP (TCP/80): 0.0.0.0/0  
- **IAM Role:** `EC2-ECR-Role` com política `AmazonEC2ContainerRegistryReadOnly`

---

## 🚀 Fase 6: Deploy e Execução na EC2

### 6.1 Conexão e Instalação do Docker
```bash
chmod 400 meu-website-key.pem
ssh -i meu-website-key.pem ec2-user@<IP_PUBLICO_EC2>

sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Reentrar para aplicar o grupo
exit
ssh -i meu-website-key.pem ec2-user@<IP_PUBLICO_EC2>
```

### 6.2 Pull da Imagem do ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
```

### 6.3 Execução do Container em Produção
```bash
docker run -d -p 80:80 --name meu-website-prod --restart always 123456789012.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
```

Acesse:  
👉 **http://<IP_PUBLICO_EC2>**

---

## 🧹 Limpeza de Recursos
Para evitar custos:

1. **Container:**
   ```bash
   docker stop meu-website-prod
   docker rm meu-website-prod
   ```
2. **AWS:**
   - Exclua a instância EC2.  
   - Delete a imagem e o repositório no ECR.  
   - Remova a IAM Role e o Security Group.  

---

## 🎓 Conclusão e Conceitos Aplicados

✅ **Conceitos principais abordados:**
- Criação de **imagens Docker** com `Dockerfile`.  
- **Build, tag e push** de imagens para o **Amazon ECR**.  
- **Deploy manual** de container em **Amazon EC2**.  
- **Configuração de rede e segurança** via Security Groups.  
- **Uso de IAM Roles** para integração segura entre serviços AWS.  

Este projeto completa o ciclo **DevOps fundamental**, do **desenvolvimento local à entrega em produção** com **Docker e AWS**.  

---

💡 *Autor:* **Willian Carlos de Almeida Silva**  
📧 *Contato:* [willian.carlosalmeidasilva@gmail.com](mailto:willian.carlosalmeidasilva@gmail.com)  
🔗 *LinkedIn:* [linkedin.com/in/willian-carlos-a77105211](https://linkedin.com/in/willian-carlos-a77105211)

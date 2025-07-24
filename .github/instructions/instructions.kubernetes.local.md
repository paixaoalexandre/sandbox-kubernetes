---
applyTo: "**/ms-*/**,**/bff/**"
---

## Visão Geral

Em produção, os microserviços da arquitetura GIP (como `bff`, `ms-internal-db-consumer`, etc.) são executados como pods em um cluster Kubernetes. Para o ambiente de desenvolvimento local, essa arquitetura é emulada de forma simplificada utilizando Docker Compose e LocalStack, sem a necessidade de um cluster Kubernetes local (como Minikube ou Kind).

Os microserviços, que são aplicações Node.js/Nest.js, não rodam em pods Kubernetes individuais. Em vez disso, eles são executados como processos Node.js concorrentes dentro de um único contêiner Docker: o contêiner `localstack_gip`. Este contêiner utiliza uma imagem customizada do LocalStack que já contém o código-fonte e as dependências de todos os microserviços.

## Componentes da Arquitetura Local

1.  **`docker-compose.yaml`**: É o orquestrador principal do ambiente local. Ele define todos os serviços necessários:
    *   `localstack_gip`: O serviço central que roda o LocalStack e todos os microserviços.
    *   `redis`: Instância do Redis para cache.
    *   `zookeeper` e `kafka`: Para emular o barramento de eventos Apache Kafka.

2.  **`Localstack.Dockerfile`**: Este arquivo define como a imagem Docker `localstack_gip` é construída. É um Dockerfile multi-stage que realiza as seguintes ações:
    *   Cria stages de build separados para cada microserviço (`bff`, `ms-internal-db-consumer`, etc.). Em cada stage, ele copia os `package.json`, instala as dependências com `npm install` e depois copia o restante do código-fonte.
    *   Utiliza uma imagem base do `localstack/localstack:latest`.
    *   Copia os artefatos de build (código e `node_modules`) de cada microserviço para dentro da imagem final do LocalStack, no diretório `/src/apps/`.
    *   Instala dependências necessárias no contêiner, como o Node.js e o SAP RFC SDK.

3.  **`init-localstack.sh`**: Este é o script de inicialização executado assim que o contêiner `localstack_gip` está pronto. Ele é responsável por:
    *   Criar os recursos da AWS necessários no LocalStack (filas SQS, tópicos SNS, secrets, etc.).
    *   **Iniciar cada um dos microserviços**. Ele navega até o diretório de cada aplicação (ex: `/src/apps/bff`) e executa o comando `npm run start:dev` ou similar, utilizando `nodemon` para monitorar alterações no código.

## Passo a Passo para Executar os Microserviços

1.  **Construir a Imagem Docker**:
    O primeiro passo é construir a imagem customizada do LocalStack que contém os microserviços. O Docker Compose gerencia esse processo.
    ```bash
    docker-compose build
    ```
    Este comando irá executar as etapas definidas no `Localstack.Dockerfile`, baixando a imagem do LocalStack, instalando as dependências de cada microserviço e montando a imagem final.

2.  **Iniciar o Ambiente Local**:
    Após o build, inicie todos os serviços definidos no `docker-compose.yaml`.
    ```bash
    docker-compose up
    ```
    Este comando irá:
    *   Iniciar os contêineres `redis`, `zookeeper` e `kafka`.
    *   Iniciar o contêiner `localstack_gip`.
    *   Após o LocalStack estar pronto, o script `/etc/localstack/init/ready.d/init-localstack.sh` (que é um volume montado do `init-localstack.sh` local) será executado.
    *   O script de inicialização criará os recursos AWS e, em seguida, iniciará os processos dos microserviços em background dentro do contêiner.

## Acessando os Microserviços

As portas dos microserviços são mapeadas do contêiner `localstack_gip` para o seu `localhost` no arquivo `docker-compose.yaml`. Por exemplo:
*   **BFF**: `6060:6060` -> Acessível em `http://localhost:6060`
*   **ms-internal-db-consumer**: `5070:5070` -> Acessível em `http://localhost:5070`
*   **ms-external-rfc-consumer**: `5080:5080` -> Acessível em `http://localhost:5080`

## Desenvolvimento e Debug

Como os serviços são iniciados com `nodemon`, qualquer alteração que você fizer nos arquivos de um microserviço na sua máquina local (na pasta `apps/`) será refletida imediatamente dentro do contêiner (graças aos volumes montados), e o `nodemon` reiniciará automaticamente o serviço correspondente.

Para depurar, você pode se conectar ao contêiner `localstack_gip` e visualizar os logs de um serviço específico ou anexar um debugger Node.js à porta de inspeção, caso configurada.
```bash
# Acessar o shell do contêiner
docker exec -it localstack_main /bin/bash
```

## Execução Alternativa com Kubernetes Local (Minikube/Kind)

Para um ambiente de desenvolvimento que espelhe ainda mais de perto a produção, é possível executar os microserviços em um cluster Kubernetes local usando ferramentas como Minikube ou Kind. Esta abordagem é mais complexa que a baseada em Docker Compose, mas oferece uma simulação mais fiel do ambiente de produção.

### Ferramentas Necessárias

*   **Docker**: Como o runtime dos contêineres.
*   **kubectl**: A ferramenta de linha de comando para interagir com o cluster Kubernetes.
*   **Minikube** ou **Kind**: Ferramentas para criar e gerenciar um cluster Kubernetes de nó único localmente.
*   **(Opcional, mas recomendado) Skaffold**: Uma ferramenta que automatiza o ciclo de build, push e deploy de aplicações em Kubernetes, ideal para desenvolvimento.

### Passo a Passo (Exemplo com Minikube e kubectl)

1.  **Iniciar o Cluster Local**:
    ```bash
    minikube start
    ```

2.  **Apontar o Docker CLI para o Daemon do Minikube**:
    Para que as imagens construídas localmente fiquem disponíveis para o cluster sem a necessidade de um registro remoto, execute:
    ```bash
    eval $(minikube -p minikube docker-env)
    ```
    *Nota: Este comando deve ser executado no mesmo terminal onde você irá construir as imagens.*

3.  **Criar Manifestos Kubernetes**:
    Cada microserviço (`bff`, `ms-internal-db-consumer`, etc.) precisará de seus próprios arquivos de manifesto Kubernetes (ex: `deployment.yaml`, `service.yaml`).
    *   `deployment.yaml`: Define como a aplicação será executada, qual imagem usar, número de réplicas, variáveis de ambiente, etc.
    *   `service.yaml`: Expõe o `Deployment` como um serviço de rede, permitindo a comunicação entre microserviços e o acesso externo (via `NodePort` ou `LoadBalancer`).

    Abaixo estão exemplos genéricos que podem ser adaptados para qualquer uma das suas aplicações (ex: `app-produto`).

    **Exemplo: `product-deployment.yaml`**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: app-produto-deployment # Nome único para o Deployment
    spec:
      replicas: 1 # Quantidade de Pods que serão criados
      selector:
        matchLabels:
          app: app-produto # Seletor para conectar o Deployment ao Service
      template:
        metadata:
          labels:
            app: app-produto # Label aplicada aos Pods
        spec:
          containers:
            - name: app-produto-container
              image: app-produto-local:latest # A imagem Docker a ser usada
              ports:
                - containerPort: 3000 # A porta que a aplicação expõe dentro do contêiner
              # (Opcional) Variáveis de ambiente para a aplicação
              # env:
              #   - name: DATABASE_URL
              #     value: "postgresql://user:password@postgres-service:5432/mydatabase"
    ```

    **Exemplo: `product-service.yaml`**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: app-produto-service # Nome do serviço, usado para service discovery (ex: http://app-produto-service:3000)
    spec:
      selector:
        app: app-produto # Deve corresponder ao 'matchLabels' do Deployment
      ports:
        - protocol: TCP
          port: 3000 # Porta que o serviço expõe para outros serviços no cluster
          targetPort: 3000 # Porta no contêiner para onde o tráfego será redirecionado
      # (Opcional) Use NodePort para expor o serviço fora do cluster para debug
      # type: NodePort
    ```

4.  **Construir a Imagem Docker do Microserviço**:
    Navegue até a pasta de um microserviço (ex: `apps/bff`) e construa a imagem.
    ```bash
    # Exemplo para o bff
    docker build . -t bff-local:latest
    ```

5.  **Aplicar os Manifestos**:
    Com os manifestos criados, aplique-os ao cluster:
    ```bash
    kubectl apply -f product-deployment.yaml
    kubectl apply -f product-service.yaml
    ```

6.  **Acessar o Serviço**:
    Use `minikube service <nome-do-servico>` para obter a URL de acesso ao microserviço.

### Fluxo de Trabalho Simplificado com Skaffold

Skaffold simplifica enormemente o ciclo de desenvolvimento. Com um único arquivo de configuração (`skaffold.yaml`), ele monitora seu código-fonte, reconstrói as imagens e as reimplementa no cluster automaticamente a cada alteração. Os arquivos `deployment.yaml` e `service.yaml` continuam sendo necessários, pois o Skaffold os utiliza para saber *o que* implantar.

1.  **Criar `skaffold.yaml` na Raiz do Projeto**:
    Este arquivo define os artefatos a serem construídos e os manifestos a serem implantados.

    ```yaml
    # Exemplo de skaffold.yaml
    apiVersion: skaffold/v4beta1
    kind: Config
    build:
      artifacts:
        - image: bff-local
          context: apps/bff
        - image: ms-internal-db-consumer-local
          context: apps/ms-internal-db-consumer
    manifests:
      rawYaml:
        - ./k8s/bff-deployment.yaml
        - ./k8s/bff-service.yaml
        - ./k8s/ms-internal-db-consumer-deployment.yaml
        - ./k8s/ms-internal-db-consumer-service.yaml
    deploy:
      kubectl: {}
    ```

2.  **Iniciar o Modo de Desenvolvimento**:
    Na raiz do projeto, execute:
    ```bash
    skaffold dev
    ```
    O Skaffold irá construir as imagens, implantar os serviços e começar a monitorar os arquivos. Qualquer alteração no código de um microserviço irá disparar um novo ciclo de build e deploy para aquele serviço, de forma automática.


# Executando Microserviços "Kubernetes" Localmente

## Cenário Avançado: Minikube dentro de um Contêiner Docker

Esta seção descreve uma abordagem avançada para executar um ambiente de desenvolvimento Kubernetes completo (Minikube) dentro de um contêiner Docker, gerenciado por Docker Compose. Isso permite encapsular todo o ambiente de desenvolvimento, incluindo o próprio cluster, em uma única unidade.

**Atenção:** Esta é uma configuração complexa e pode ter implicações de segurança e performance. Requer que o contêiner tenha acesso privilegiado ao daemon Docker do host.

### 1. Componentes Necessários

*   **`docker-compose.yaml`**: Para orquestrar o contêiner de desenvolvimento.
*   **`Dockerfile`**: Para criar uma imagem customizada com todas as ferramentas necessárias (Docker CLI, kubectl, Minikube).

### 2. Configuração do Dockerfile

Crie um `Dockerfile` para preparar o ambiente:

```dockerfile
# Use uma imagem base que facilite a instalação de pacotes
FROM ubuntu:22.04

# Evita prompts interativos durante a instalação
ENV DEBIAN_FRONTEND=noninteractive

# Instala dependências básicas
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    apt-transport-https \
    ca-certificates \
    gnupg \
    lsb-release

# Instala o Docker CLI
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
RUN apt-get update && apt-get install -y docker-ce-cli

# Instala o kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
RUN install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Instala o Minikube
RUN wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
RUN install minikube-linux-amd64 /usr/local/bin/minikube

# Define um diretório de trabalho
WORKDIR /app

# Comando para manter o contêiner em execução
CMD ["tail", "-f", "/dev/null"]
```

### 3. Configuração do `docker-compose.yaml`

Este serviço `docker-compose` irá construir e executar o contêiner de desenvolvimento.

```yaml
version: "3.9"

services:
  kube-dev-environment:
    container_name: minikube_in_docker
    build:
      context: .
      dockerfile: Dockerfile  # Nome do Dockerfile criado acima
    privileged: true # Essencial para Minikube controlar o Docker do host
    volumes:
      # Mapeia o socket do Docker do host para o contêiner
      - /var/run/docker.sock:/var/run/docker.sock
      # Mapeia o diretório do Minikube para persistir o estado do cluster
      - ~/.minikube:/root/.minikube
      # Mapeia o código-fonte das suas aplicações para dentro do contêiner
      - ./apps:/app/apps
    working_dir: /app
    tty: true
```

### 4. Passo a Passo para Execução

1.  **Construir e Iniciar o Contêiner**:
    ```bash
    docker-compose up -d --build
    ```

2.  **Acessar o Contêiner**:
    ```bash
    docker exec -it minikube_in_docker /bin/bash
    ```

3.  **Dentro do Contêiner, Iniciar o Minikube**:
    Uma vez dentro do shell do contêiner, você pode iniciar o Minikube. É crucial usar o driver `--driver=docker` e especificar que não há um VM driver (`--vm-driver=none` ou similar, dependendo da versão) para que ele use o Docker do host.
    ```bash
    # Dentro do contêiner minikube_in_docker
    minikube start --driver=docker
    ```

4.  **Trabalhar com o Cluster**:
    Após o Minikube iniciar, você pode usar `kubectl` normalmente para implantar suas aplicações (que estão disponíveis em `/app/apps` graças ao volume mapeado).

---

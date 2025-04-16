## Preparação da Infraestrutura

### Etapa 1: Criação dos Arquivos Necessários

Antes de iniciar o fluxo de descoberta e recuperação de dados, precisamos preparar a infraestrutura criando os arquivos necessários:

```bash
# Criar arquivo contract-agent.config.json para o Contract Manager
cat > contract-manager/contract-agent.config.json << EOF
{
  "dataProviderConfig": [
    {
      "source": "contracts",
      "url": "mongodb://mongodb:27017",
      "dbName": "contract-manager"
    },
    {
      "source": "profiles",
      "url": "mongodb://mongodb:27017",
      "dbName": "contract-manager",
      "watchChanges": false,
      "hostsProfiles": true
    }
  ]
}
EOF

# Criar arquivos de produção para o Provider Connector
mkdir -p provider/connector/src/config
cat > provider/connector/.env.production << EOF
NODE_ENV=production
PORT=3002
SESSION_SECRET=abc
SESSION_COOKIE_EXPIRATION=24000
MONGO_URI=mongodb://mongodb:27017/provider-connector
CURATOR=https://prometheus-x.org
MAINTAINER=https://prometheus-x.org
EOF

cat > provider/connector/src/config.production.json << EOF
{
    "endpoint": "http://provider-connector:3002",
    "serviceKey": "provider-key",
    "secretKey": "provider-secret",
    "catalogUri": "http://catalog-api:3000/v1",
    "contractUri": "http://contract-manager:3001",
    "consentUri": "",
    "registrationUri": "",
    "billingUri": "",
    "credentials": [],
    "expressLimitSize": "50mb"
}
EOF

# Criar arquivos de produção para o Consumer Connector
mkdir -p consumer/connector/src/config
cat > consumer/connector/.env.production << EOF
NODE_ENV=production
PORT=3002
SESSION_SECRET=abc
SESSION_COOKIE_EXPIRATION=24000
MONGO_URI=mongodb://mongodb:27017/consumer-connector
CURATOR=https://prometheus-x.org
MAINTAINER=https://prometheus-x.org
EOF

cat > consumer/connector/src/config.production.json << EOF
{
    "endpoint": "http://consumer-connector:3002",
    "serviceKey": "consumer-key",
    "secretKey": "consumer-secret",
    "catalogUri": "http://catalog-api:3000/v1",
    "contractUri": "http://contract-manager:3001",
    "consentUri": "",
    "registrationUri": "",
    "billingUri": "",
    "credentials": [],
    "expressLimitSize": "50mb"
}
EOF

# Atualizar o Dockerfile do Catalog API para suportar M3
cat > catalog/docker/app/Dockerfile << EOF
# Use Node.js LTS versão
FROM node:lts

# Instalar dependências para compilação nativa
RUN apt-get update && apt-get install -y \\
    python3 \\
    make \\
    g++ \\
    && rm -rf /var/lib/apt/lists/*

# Configurar diretório de trabalho
WORKDIR /usr/src/app

# Copiar package.json e pnpm-lock.yaml
COPY package.json pnpm-lock.yaml ./

# Instalar pnpm globalmente e dependências do projeto
RUN npm install -g pnpm
RUN pnpm install

# Copiar resto do código
COPY . .

# Garantir permissões corretas
RUN chown -R node:node /usr/src/app

# Compilar o projeto e iniciar
CMD ["sh", "-c", "pnpm run build && pnpm run start"]
EOF
```

### Etapa 2: Compilação e Execução

Com os arquivos de configuração criados, vamos iniciar os serviços:

```bash
# Na raiz do projeto prometheus-x-poc
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

Verifique se todos os contêineres estão rodando corretamente:

```bash
docker-compose ps
```

Aguarde até que todos os serviços estejam saudáveis. Você pode verificar os logs para confirmar:

```bash
docker-compose logs -f
```

### Etapa 3: Adicionando Endpoint de Dados ao Provedor

Vamos criar um endpoint para fornecer dados de exemplo no conector do provedor. Como já temos a estrutura do projeto, vamos adicionar um endpoint no arquivo `provider/connector/src/routes/public/v1/data.public.router.# Implementação de POC com Prometheus-X: Conectando Dataspaces

## Introdução

Esta documentação fornece um guia passo a passo para implementar uma Prova de Conceito (POC) com os componentes do Prometheus-X existentes, focando na conexão de dois dataspaces para realizar o processo completo de descoberta e recuperação de dados. A estrutura do projeto já está criada, e os arquivos de configuração já estão presentes.

## Requisitos Prévios

- Docker e Docker Compose
- Node.js (versão 14 ou superior)
- Git
- Conhecimento básico de APIs REST e formatos JSON

## Visão Geral da Arquitetura

O Prometheus-X é composto por vários componentes principais:

1. **Catálogo (Catalog API)**: Serviço centralizado para registro de participantes, ativos de dados e gerenciamento de ofertas de serviços.
2. **Gerenciador de Contratos (Contract Manager)**: Gerencia verificação de políticas, geração de contratos e assinaturas.
3. **Conector de Dataspace (Dataspace Connector)**: Permite verificações de plano de controle, gestão de consentimento e trocas de dados baseadas em contratos. Temos dois conectores: um para o provedor e outro para o consumidor.

## Estrutura do Projeto

A partir da estrutura de arquivos fornecida, vemos que o projeto já está organizado com os seguintes componentes:

```
prometheus-x-poc/
├── catalog/                # Catálogo API
├── contract-manager/       # Gerenciador de Contratos
├── provider/
│   └── connector/          # Conector do Provedor
├── consumer/
│   └── connector/          # Conector do Consumidor
└── docker-compose.yml      # Arquivo principal para orquestração dos serviços
```

## Configuração do Ambiente

### Etapa 1: Configuração dos Arquivos de Ambiente

Para cada componente, já existem arquivos `.env` configurados. Vamos verificar e, se necessário, ajustar esses arquivos:

#### Arquivo `.env` do Catálogo:

```
NODE_ENV=development
PORT=3000
MONGO_USERNAME=""
MONGO_PASSWORD=""
MONGO_PORT=27017
MONGO_HOST=catalog-api-mongodb
MONGO_DATABASE=ptx-catalog
API_URL=http://localhost:3000/v1
JWT_SECRET_KEY=abc123
SALT_ROUNDS=10
CONTRACT_SERVICE_ENDPOINT=http://contract-manager:3001
API_PREFIX=
X_PTX_CONTRACT_CATALOG_KEY=
```

#### Arquivo `.env` do Contract Manager:

```
NODE_ENV="development"
MONGO_URL="mongodb://mongodb:27017/contract-manager"
MONGO_TEST_URL="mongodb://mongodb:27017/contract-manager-test"
MONGO_USERNAME=""
MONGO_PASSWORD=""
SERVER_PORT=3001
SECRET_AUTH_KEY="seu_segredo_auth"
SECRET_SESSION_KEY="seu_segredo_session"
SERVER_BASE_URL="http://contract-manager:3001"
USE_CONTRACT_AGENT=true
CATALOG_AUTHORIZATION_KEY="chave_autorizacao_catalogo"
```

#### Arquivo `.env` (Principal e Produção) do Connector:

Precisamos criar os arquivos `.env.production` para os conectores:

```bash
# Para o provider
cat > provider/connector/.env.production << EOF
NODE_ENV=production
PORT=3002
SESSION_SECRET=abc
SESSION_COOKIE_EXPIRATION=24000
MONGO_URI=mongodb://mongodb:27017/provider-connector
CURATOR=https://prometheus-x.org
MAINTAINER=https://prometheus-x.org
EOF

# Para o consumer
cat > consumer/connector/.env.production << EOF
NODE_ENV=production
PORT=3002
SESSION_SECRET=abc
SESSION_COOKIE_EXPIRATION=24000
MONGO_URI=mongodb://mongodb:27017/consumer-connector
CURATOR=https://prometheus-x.org
MAINTAINER=https://prometheus-x.org
EOF
```

### Etapa 2: Configuração do Docker Compose

Vamos criar um arquivo `docker-compose.yml` na raiz do projeto que suporta arquitetura ARM64 (Apple Silicon M3):

```yaml
version: '3'
services:
  mongodb:
    container_name: "mongodb"
    image: mongo:latest
    platform: linux/arm64
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    networks:
      - ptx-network

  catalog-api:
    container_name: "catalog-api"
    build:
      context: ./catalog
      dockerfile: docker/app/Dockerfile
    platform: linux/arm64
    ports:
      - "3000:3000"
    depends_on:
      - mongodb
    env_file:
      - ./catalog/.env
    environment:
      MONGO_HOST: mongodb
    networks:
      - ptx-network

  contract-manager:
    container_name: "contract-manager"
    build:
      context: ./contract-manager
      dockerfile: docker/app/Dockerfile
    platform: linux/arm64
    ports:
      - "3001:3001"
    depends_on:
      - mongodb
      - catalog-api
    env_file:
      - ./contract-manager/.env
    environment:
      MONGO_URL: mongodb://mongodb:27017/contract-manager
    volumes:
      - ./contract-manager/contract-agent.config.json:/usr/src/app/contract-agent.config.json
    networks:
      - ptx-network

  provider-connector:
    container_name: "provider-connector"
    build:
      context: ./provider/connector
      dockerfile: docker/app/Dockerfile
    platform: linux/arm64
    ports:
      - "4000:3002"
    depends_on:
      - mongodb
      - catalog-api
      - contract-manager
    env_file:
      - ./provider/connector/.env
    environment:
      MONGO_URI: mongodb://mongodb:27017/provider-connector
      PORT: 3002
    volumes:
      - ./provider/connector/src/config.production.json:/usr/src/app/src/config.production.json
    networks:
      - ptx-network

  consumer-connector:
    container_name: "consumer-connector"
    build:
      context: ./consumer/connector
      dockerfile: docker/app/Dockerfile
    platform: linux/arm64
    ports:
      - "4001:3002"
    depends_on:
      - mongodb
      - catalog-api
      - contract-manager
    env_file:
      - ./consumer/connector/.env
    environment:
      MONGO_URI: mongodb://mongodb:27017/consumer-connector
      PORT: 3002
    volumes:
      - ./consumer/connector/src/config.production.json:/usr/src/app/src/config.production.json
    networks:
      - ptx-network

networks:
  ptx-network:
    driver: bridge

volumes:
  mongodb_data:
```

### Etapa 3: Configuração dos Arquivos JSON dos Conectores

Os conectores precisam de arquivos de configuração JSON específicos:

#### Arquivo `src/config.production.json` do Conector do Provedor:

```json
{
    "endpoint": "http://provider-connector:3002",
    "serviceKey": "provider-key",
    "secretKey": "provider-secret",
    "catalogUri": "http://catalog-api:3000/v1",
    "contractUri": "http://contract-manager:3001",
    "consentUri": "",
    "registrationUri": "",
    "billingUri": "",
    "credentials": [],
    "expressLimitSize": "50mb"
}
```

#### Arquivo `src/config.production.json` do Conector do Consumidor:

```json
{
    "endpoint": "http://consumer-connector:3002",
    "serviceKey": "consumer-key",
    "secretKey": "consumer-secret",
    "catalogUri": "http://catalog-api:3000/v1",
    "contractUri": "http://contract-manager:3001",
    "consentUri": "",
    "registrationUri": "",
    "billingUri": "",
    "credentials": [],
    "expressLimitSize": "50mb"
}
```

### Etapa 4: Configuração do Contract Agent

Crie um arquivo `contract-agent.config.json` para o Contract Manager:

```json
{
  "dataProviderConfig": [
    {
      "source": "contracts",
      "url": "mongodb://mongodb:27017",
      "dbName": "contract-manager"
    },
    {
      "source": "profiles",
      "url": "mongodb://mongodb:27017",
      "dbName": "contract-manager",
      "watchChanges": false,
      "hostsProfiles": true
    }
  ]
}
```

### Etapa 5: Ajuste dos Dockerfiles

Para compatibilidade com Apple Silicon (M3), é necessário ajustar os Dockerfiles. Vamos modificar o Dockerfile do Catalog API:

```bash
cat > catalog/docker/app/Dockerfile << EOF
# Use Node.js LTS versão para arquitetura ARM64
FROM node:lts

# Instalar dependências para compilação nativa
RUN apt-get update && apt-get install -y \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Configurar diretório de trabalho
WORKDIR /usr/src/app

# Copiar package.json e pnpm-lock.yaml
COPY package.json pnpm-lock.yaml ./

# Instalar pnpm globalmente e dependências do projeto
RUN npm install -g pnpm
RUN pnpm install

# Copiar resto do código
COPY . .

# Garantir permissões corretas
RUN chown -R node:node /usr/src/app

# Compilar o projeto e iniciar
CMD ["sh", "-c", "pnpm run build && pnpm run start"]
EOF
```

## Preparação da Infraestrutura

Antes de iniciar o fluxo de descoberta e recuperação de dados, precisamos preparar a infraestrutura:

### Etapa 1: Compilação e Execução

Considerando que todos os arquivos estão em suas devidas pastas e os arquivos de configuração foram ajustados conforme descrito anteriormente, vamos iniciar os serviços:

```bash
# Na raiz do projeto prometheus-x-poc
docker-compose up -d --build
```

Verifique se todos os contêineres estão rodando corretamente:

```bash
docker-compose ps
```

Aguarde até que todos os serviços estejam saudáveis. Você pode verificar os logs para confirmar:

```bash
docker-compose logs -f
```

### Etapa 2: Adicionando Endpoint de Dados ao Provedor

Vamos criar um endpoint para fornecer dados de exemplo no conector do provedor. Como já temos a estrutura do projeto, vamos adicionar um endpoint no arquivo `provider/connector/src/routes/public/v1/data.public.router.ts`:

```typescript
import express from 'express';
const router = express.Router();

// Endpoint para servir dados de exemplo
router.get('/api/data', (req, res) => {
  // Dados de exemplo para a POC
  const exampleData = {
    timestamp: new Date().toISOString(),
    readings: [
      { sensor: "temp-1", value: 22.5 },
      { sensor: "humidity-1", value: 45 },
      { sensor: "pressure-1", value: 1013 }
    ],
    metadata: {
      source: "Prometheus-X POC",
      version: "1.0"
    }
  };
  
  res.json(exampleData);
});

export default router;
```

E atualize o arquivo `provider/connector/src/routes/public/v1/index.ts` para incluir essa rota:

```typescript
// Adicione a importação
import dataRoutes from './data.public.router';

// No local onde as rotas são registradas, adicione:
app.use(dataRoutes);
```

Após essas alterações, reconstrua o conector do provedor:

```bash
docker-compose up -d --build provider-connector
```

## Fluxo de Descoberta e Recuperação de Dados

Vamos implementar o fluxo completo de descoberta e recuperação de dados entre os dois dataspaces:

### Etapa 1: Criação de Credenciais

Primeiro, vamos criar credenciais para acessar as APIs. No conector do provedor:

```bash
curl -X POST http://localhost:4000/v1/private/credentials \
  -H "Content-Type: application/json" \
  -d '{
    "type": "basic",
    "key": "api-key-provider",
    "value": "provider-secret-value"
  }'
```

Anote o ID da credencial retornada (ex: `credential-id-provider`).

No conector do consumidor:

```bash
curl -X POST http://localhost:4001/v1/private/credentials \
  -H "Content-Type: application/json" \
  -d '{
    "type": "basic",
    "key": "api-key-consumer",
    "value": "consumer-secret-value"
  }'
```

Anote o ID da credencial retornada (ex: `credential-id-consumer`).

### Etapa 2: Registro de Participantes

Agora, precisamos registrar os participantes no Catálogo:

#### 2.1 Registro do Provedor de Dados

```bash
curl -X POST http://localhost:3000/v1/participants \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Provedor de Dados",
    "description": "Organização que fornece dados para o dataspace",
    "url": "http://provider-connector:3002",
    "did": "did:web:provider-connector",
    "legalName": "Provedor de Dados S.A.",
    "legalRegistrationNumber": "12345678901234",
    "headquartersAddress": {
      "street": "Rua Exemplo",
      "houseNumber": "123",
      "postalCode": "01234-567",
      "city": "São Paulo",
      "country": "Brasil"
    }
  }'
```

Anote o ID do participante retornado (ex: `provider-id`).

#### 2.2 Registro do Consumidor de Dados

```bash
curl -X POST http://localhost:3000/v1/participants \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Consumidor de Dados",
    "description": "Organização que consome dados do dataspace",
    "url": "http://consumer-connector:3002",
    "did": "did:web:consumer-connector",
    "legalName": "Consumidor de Dados S.A.",
    "legalRegistrationNumber": "98765432109876",
    "headquartersAddress": {
      "street": "Rua Teste",
      "houseNumber": "456",
      "postalCode": "04321-765",
      "city": "Rio de Janeiro",
      "country": "Brasil"
    }
  }'
```

Anote o ID do participante retornado (ex: `consumer-id`).

### Etapa 3: Registro de Recursos de Dados

O provedor precisa registrar os recursos de dados que serão disponibilizados:

```bash
curl -X POST http://localhost:3000/v1/dataresources \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Conjunto de Dados de Exemplo",
    "description": "Um conjunto de dados de exemplo para nossa POC",
    "keywords": ["exemplo", "poc", "prometheus-x"],
    "publisher": "provider-id",
    "type": "Dataset",
    "version": "1.0.0",
    "representations": [
      {
        "title": "API JSON",
        "mediaType": "application/json",
        "endpoint": "http://provider-connector:3002/api/data",
        "protocol": "http",
        "sourceType": "rest",
        "credentialId": "credential-id-provider"
      }
    ]
  }'
```

Anote o ID do recurso retornado (ex: `resource-id`).

### Etapa 4: Criação de Ofertas de Serviço

O provedor cria uma oferta para o recurso:

```bash
curl -X POST http://localhost:3000/v1/serviceofferings \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Oferta de Dados de Exemplo",
    "description": "Oferta para acesso ao conjunto de dados de exemplo",
    "provider": "provider-id",
    "dataResources": ["resource-id"],
    "policies": [
      {
        "type": "usage",
        "rule": "Uso permitido apenas para fins não comerciais"
      }
    ]
  }'
```

Anote o ID da oferta retornado (ex: `offering-id`).

### Etapa 5: Descoberta de Recursos

O consumidor pode descobrir recursos disponíveis no Catálogo:

```bash
curl -X GET http://localhost:3000/v1/serviceofferings
```

Ou pesquisar por ofertas específicas:

```bash
curl -X GET "http://localhost:3000/v1/serviceofferings?title=Oferta%20de%20Dados%20de%20Exemplo"
```

### Etapa 6: Negociação e Contratos

O consumidor inicia uma negociação para utilizar os dados:

```bash
curl -X POST http://localhost:3001/bilateral \
  -H "Content-Type: application/json" \
  -d '{
    "consumerId": "consumer-id",
    "offeringId": "offering-id",
    "message": "Solicitando acesso aos dados para nossa POC"
  }'
```

Anote o ID da negociação bilateral retornado (ex: `bilateral-id`).

O provedor aceita a negociação:

```bash
curl -X PUT http://localhost:3001/bilateral/bilateral-id/accept \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Aceitamos sua solicitação"
  }'
```

Um contrato é gerado automaticamente após a aceitação. Para verificar o contrato:

```bash
curl -X GET http://localhost:3001/contracts?bilateralId=bilateral-id
```

Anote o ID do contrato retornado (ex: `contract-id`).

### Etapa 7: Troca de Dados

Com o contrato estabelecido, o consumidor pode solicitar os dados. Primeiro, precisamos configurar uma representação de serviço no consumidor para receber os dados:

```bash
curl -X POST http://localhost:4001/v1/private/catalog/representation \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Serviço de Recebimento de Dados",
    "endpointType": "rest",
    "endpoint": "http://consumer-connector:3002/v1/data",
    "mediaType": "application/json",
    "credentialId": "credential-id-consumer"
  }'
```

Anote o ID da representação retornada (ex: `representation-id`).

Agora, o consumidor pode iniciar a troca de dados:

```bash
curl -X POST http://localhost:4001/v1/dataexchange \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "contract-id",
    "resourceId": "resource-id",
    "representationId": "representation-id"
  }'
```

Anote o ID da troca de dados retornado (ex: `exchange-id`).

Para verificar o status da troca de dados:

```bash
curl -X GET http://localhost:4001/v1/dataexchange/status/exchange-id
```

Para recuperar os dados recebidos:

```bash
curl -X GET http://localhost:4001/v1/data
```

## Monitoramento e Depuração

### Verificando Logs

Para verificar o que está acontecendo em cada serviço:

```bash
# Logs do catálogo
docker logs catalog-api

# Logs do gerenciador de contratos
docker logs contract-manager

# Logs do conector do provedor
docker logs provider-connector

# Logs do conector do consumidor
docker logs consumer-connector
```

Para acompanhar os logs em tempo real:

```bash
docker logs -f consumer-connector
```

### Verificando o Status dos Serviços

```bash
# Status de todos os contêineres
docker ps

# Estatísticas de uso de recursos
docker stats
``` acontecendo em cada serviço:

```bash
# Logs do catálogo
docker logs catalog-api

# Logs do gerenciador de contratos
docker logs contract-manager

# Logs do conector do provedor
docker logs provider-connector

# Logs do conector do consumidor
docker logs consumer-connector
```

Para acompanhar os logs em tempo real:

```bash
docker logs -f consumer-connector
```

### Depuração de Problemas Comuns

#### Problemas de Conectividade

Se os serviços não conseguirem se comunicar, verifique se os nomes de host estão corretos nos arquivos de configuração e se a rede Docker está funcionando corretamente:

```bash
# Teste de ping entre contêineres
docker exec -it consumer-connector ping catalog-api
docker exec -it consumer-connector ping provider-connector
```

#### Problemas de Banco de Dados

Se houver problemas com o MongoDB:

```bash
# Conecte-se ao MongoDB
docker exec -it mongodb mongo

# Verifique os bancos de dados
> show dbs

# Selecione um banco de dados específico
> use provider-connector

# Verifique as coleções
> show collections

# Consulte documentos em uma coleção
> db.participants.find()
```

#### Problemas com os Conectores

Se os conectores apresentarem problemas:

```bash
# Verifique se os arquivos de configuração estão corretos
docker exec -it provider-connector cat /usr/src/app/src/config.json

# Reinicie o serviço específico
docker-compose restart provider-connector
```

## Cenários Avançados

### 1. Trocas de Dados com Consentimento

Se estiver trabalhando com dados pessoais, você pode implementar o fluxo de consentimento:

```bash
# Registrando um usuário no conector do provedor
curl -X POST http://localhost:4000/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "user@example.com",
    "name": "Usuário Teste"
  }'
```

Anote o ID do usuário retornado (ex: `user-id`).

```bash
# Registrando consentimento do usuário
curl -X POST http://localhost:4000/v1/consent \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user-id",
    "resourceId": "resource-id",
    "purpose": "Análise de dados",
    "expirationDate": "2025-12-31"
  }'
```

Anote o ID do consentimento retornado (ex: `consent-id`).

```bash
# Troca de dados baseada em consentimento
curl -X POST http://localhost:4001/v1/dataexchange/consent \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "contract-id",
    "resourceId": "resource-id",
    "userId": "user-id",
    "consentId": "consent-id",
    "representationId": "representation-id"
  }'
```

### 2. Processamento de Dados na Troca

Para aplicar transformações aos dados durante a troca, você pode configurar um processador:

```bash
# Registrando um processador no conector do consumidor
curl -X POST http://localhost:4001/v1/private/processor \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Filtro de Dados",
    "description": "Filtra valores acima de um limite",
    "config": {
      "threshold": 50
    }
  }'
```

Anote o ID do processador retornado (ex: `processor-id`).

Ao solicitar a troca de dados, inclua o processador:

```bash
curl -X POST http://localhost:4001/v1/dataexchange \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "contract-id",
    "resourceId": "resource-id",
    "representationId": "representation-id",
    "processorId": "processor-id"
  }'
```

## Conclusão

Neste guia, implementamos uma POC funcional usando a tecnologia Prometheus-X que demonstra:

1. Configuração dos componentes principais (Catálogo, Contract Manager, Conectores)
2. Registro de participantes, recursos e ofertas
3. Negociação e estabelecimento de contratos
4. Troca de dados entre provedores e consumidores

O Prometheus-X oferece uma estrutura flexível e segura para a implementação de dataspaces, permitindo a troca soberana de dados entre organizações. Esta POC serve como ponto de partida para implementações mais complexas e pode ser estendida para incluir funcionalidades adicionais como:

- Autenticação e autorização avançadas
- Implementação completa de políticas ODRL
- Integração com sistemas de identidade descentralizada
- Federação de dataspaces
- Monetização de dados

## Referências

- [Documentação oficial do Prometheus-X](https://github.com/Prometheus-X-association/docs)
- [Catálogo API](https://github.com/Prometheus-X-association/catalog-api)
- [Contract Manager](https://github.com/Prometheus-X-association/contract-manager)
- [Dataspace Connector](https://github.com/Prometheus-X-association/dataspace-connector)-X segue estas etapas:

1. O consumidor solicita dados com base em um contrato válido
2. O conector do consumidor verifica a validade do contrato com o Contract Manager
3. O conector do consumidor envia a solicitação ao conector do provedor
4. O conector do provedor verifica novamente a validade do contrato
5. O conector do provedor recupera os dados da fonte configurada
6. O conector do provedor envia os dados ao conector do consumidor
7. O conector do consumidor recebe os dados e os armazena temporariamente
8. A aplicação do consumidor pode então acessar os dados

### Verificação do Status da Troca de Dados

Para verificar o status de uma troca de dados iniciada:

```bash
curl -X GET http://localhost:4001/v1/dataexchange/status/{exchange-id}
```

Onde `{exchange-id}` é o identificador retornado na criação da troca de dados.

### Recuperação dos Dados

Após a conclusão bem-sucedida da troca, o consumidor pode acessar os dados:

```bash
curl -X GET http://localhost:4001/v1/data/{exchange-id}
```

## Cenários Avançados

### 1. Trocas de Dados com Consentimento

Se estiver trabalhando com dados pessoais, você pode implementar o fluxo de consentimento:

```bash
# Registrando um usuário no conector do provedor
curl -X POST http://localhost:4000/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "user@example.com",
    "name": "Usuário Teste"
  }'

# Registrando consentimento do usuário
curl -X POST http://localhost:4000/v1/consent \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user-id",
    "resourceId": "resource-id",
    "purpose": "Análise de dados",
    "expirationDate": "2025-12-31"
  }'

# Troca de dados baseada em consentimento
curl -X POST http://localhost:4001/v1/dataexchange/consent \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "contract-id",
    "resourceId": "resource-id",
    "userId": "user-id"
  }'
```

### 2. Implementação de Cadeias de Processamento de Dados

Você pode configurar cadeias de processamento de dados (DPCPs) para transformar dados durante a troca:

```bash
# Registrando uma DPCP no Catálogo
curl -X POST http://localhost:3000/v1/dataprocessingchains \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Cadeia de Processamento de Exemplo",
    "description": "Transforma dados brutos em dados processados",
    "owner": "provider-id",
    "steps": [
      {
        "name": "Filtro",
        "description": "Remove dados inválidos",
        "order": 1
      },
      {
        "name": "Transformação",
        "description": "Converte unidades",
        "order": 2
      }
    ]
  }'

# Associando a DPCP a uma oferta de serviço
curl -X PUT http://localhost:3000/v1/serviceofferings/offering-id \
  -H "Content-Type: application/json" \
  -d '{
    "dataProcessingChainId": "dpcp-id"
  }'
```

### 3. Comunicação entre Dataspaces

Para cenários mais avançados, você pode configurar a comunicação entre diferentes dataspaces:

1. Configure dois ambientes Prometheus-X completos
2. Configure os conectores para se conectarem a ambos os ambientes
3. Registre recursos e ofertas em ambos os dataspaces
4. Implemente políticas de federação para controlar o acesso entre dataspaces

## Conclusão e Próximos Passos

Neste guia, implementamos uma POC funcional usando a tecnologia Prometheus-X, demonstrando:

1. Configuração dos componentes principais (Catálogo, Contract Manager, Conectores)
2. Registro de participantes, recursos e ofertas
3. Negociação e estabelecimento de contratos
4. Troca de dados entre provedores e consumidores

### Próximos Passos Recomendados

1. **Segurança e Autenticação**:
   - Implementar autenticação JWT completa
   - Configurar HTTPS para todos os serviços
   - Adicionar camadas de autenticação baseadas em certificados

2. **Escalabilidade**:
   - Configurar replicação do MongoDB
   - Implementar balanceamento de carga para os serviços
   - Adicionar monitoramento e alerta

3. **Integração com Outros Sistemas**:
   - Conectar a sistemas existentes usando adaptadores personalizados
   - Implementar transformação de dados para compatibilidade

4. **Interface de Usuário**:
   - Desenvolver um portal para gerenciar recursos e contratos
   - Implementar dashboards para visualização de dados
   - Criar assistentes para configuração de políticas

5. **Extensões de Funcionalidade**:
   - Implementar modelos de políticas mais sofisticados
   - Adicionar suporte para monetização de dados
   - Desenvolver verificações automáticas de conformidade

Ao seguir este guia e considerar essas extensões, você terá uma base sólida para implementar uma solução de dataspace completa usando as tecnologias Prometheus-X.

## Referências

- [Documentação oficial do Prometheus-X](https://github.com/Prometheus-X-association/docs)
- [Catálogo API](https://github.com/Prometheus-X-association/catalog-api)
- [Contract Manager](https://github.com/Prometheus-X-association/contract-manager)
- [Dataspace Connector](https://github.com/Prometheus-X-association/dataspace-connector)
- [Protocolo de Cadeia de Processamento de Dados](https://github.com/prometheus-x-association/data-processing-chain-protocol)
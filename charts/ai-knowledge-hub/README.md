# AI Knowledge Hub

## Visão Geral

Este Helm chart implanta uma stack completa para um "Knowledge Hub" baseado em IA. Ele monitora diretórios de documentação locais, gera embeddings automaticamente usando a API do Google Gemini, armazena o conhecimento no Redis Stack e fornece uma interface MCP (Model Context Protocol) para que agentes de IA possam consultar esse conhecimento.

## Arquitetura

A solução é composta por três componentes principais que trabalham em conjunto:

### 1. Redis Stack (`statefulset-redis`)
- **Função**: Atua como o "cérebro" central, armazenando tanto o conteúdo textual dos documentos quanto seus embeddings vetoriais para busca semântica.
- **Detalhes**:
  - Executa a imagem `redis/redis-stack-server`.
  - Persistência de dados habilitada (PVC).
  - Acessível internamente no cluster via `redis://ai-knowledge-hub-redis:6379`.

### 2. Ingestion Watcher (`deployment-watcher`)
- **Função**: Mantém o conhecimento atualizado em tempo real.
- **Fluxo**:
  1. Monitora diretórios especificados (mapeados do host via `hostPath`) usando `chokidar`.
  2. Detecta criação ou modificação de arquivos Markdown (`.md`).
  3. Envia o conteúdo para a API do Google Gemini (modelo `text-embedding-004`) para gerar vetores.
  4. Salva o conteúdo e o vetor no Redis.

### 3. MCP Server (`deployment-mcp`)
- **Função**: Interface padronizada para agentes de IA consumirem o conhecimento.
- **Protocolo**: Implementa o [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).
- **Transporte**: Atualmente configurado para usar `StdioServerTransport` (entrada/saída padrão), rodando dentro do container.
- **Ferramentas Expostas**:
  - `search_knowledge_base`: Permite realizar buscas semânticas na base de conhecimento.

### Detalhes Técnicos da Busca Vetorial

A implementação utiliza o **RediSearch** com índices JSON.

- **Schema do Índice (`idx:knowledge`)**:
  - `content` (TEXT): Conteúdo textual do arquivo.
  - `project` (TAG): Identificador do projeto.
  - `path` (TAG): Caminho relativo do arquivo.
  - `embedding` (VECTOR): Vetor de 768 dimensões (Float32), usando distância COSINE e algoritmo FLAT.

- **Embeddings**:
  - Modelo: `text-embedding-004` (Google Gemini).
  - Dimensões: 768.

- **Query de Busca**:
  - Utiliza KNN (K-Nearest Neighbors) para encontrar os 3 documentos mais similares.
  - Sintaxe: `[KNN 3 @embedding $vec AS score]`
  - Cliente Redis: `ioredis` (necessário para suporte correto a envio de buffers binários).

---

## Instalação e Configuração

### Pré-requisitos
- Cluster Kubernetes (k3s é ideal para mapeamento de volumes locais).
- Helm instalado.
- Uma chave de API do Google Gemini (Google AI Studio).

### 1. Configurar Segredos
Antes de instalar, crie o segredo com sua chave da API Gemini para evitar colocá-la em arquivos de configuração:

```bash
kubectl create secret generic ai-knowledge-hub-secrets \
  --from-literal=gemini-api-key=SUA_CHAVE_AQUI
```

### 2. Configurar Projetos (`values.yaml`)
Edite o arquivo `values.yaml` para definir quais pastas de documentação devem ser monitoradas. O caminho (`path`) deve ser o caminho absoluto na máquina host (onde o k3s está rodando).

```yaml
projects:
  - name: meu-projeto
    path: /home/usuario/meus-projetos/docs
    redis_prefix: proj1
```

### 3. Instalar o Chart
Na pasta raiz do chart:

```bash
helm install ai-hub .
```

---

## Instruções de Uso

### Verificando a Ingestão
Após a instalação, verifique se o Watcher está processando seus arquivos corretamente:

```bash
# Acompanhar logs do watcher
kubectl logs -l app=watcher -f
```
Você deve ver mensagens como `Processing file: ...` e `Updated ... in Redis`.

### Conectando ao MCP Server
O servidor MCP atual roda via `stdio` dentro do pod, o que é o padrão para integração direta com aplicações locais (como Claude Desktop), mas requer adaptação para acesso remoto em Kubernetes.

**Métodos de Conexão:**

1.  **Via Exec (Experimental)**: É possível conectar um cliente MCP local tunelando a entrada/saída para o pod via `kubectl exec`.
    ```bash
    kubectl exec -i $(kubectl get pod -l app=mcp-server -o jsonpath="{.items[0].metadata.name}") -- npx tsx mcp-server.ts
    ```

2.  **Adaptação para HTTP (SSE)**: Para uso em produção com agentes remotos, recomenda-se alterar o código do servidor (`mcp-server.ts` no ConfigMap) para usar `SSEServerTransport` e expor o serviço via Kubernetes Service/Ingress.

### Ferramentas Disponíveis (Tools)

O servidor expõe a seguinte ferramenta para o Agente:

#### `search_knowledge_base`
Busca informações relevantes na documentação indexada.
- **Input**:
  - `query` (string, obrigatório): A pergunta ou termo de busca.
  - `project_id` (string, opcional): Filtrar por um projeto específico.
- **Retorno**: Trechos de texto relevantes encontrados no Redis.

---

## Referências e Documentação Oficial

Aqui estão os links para a documentação oficial das tecnologias e protocolos utilizados nesta arquitetura:

- **Model Context Protocol (MCP)**
  - [Documentação Oficial](https://modelcontextprotocol.io/)
  - [Especificação do Protocolo](https://spec.modelcontextprotocol.io/)
  - [SDK para TypeScript](https://github.com/modelcontextprotocol/typescript-sdk)

- **Redis & Redis Stack**
  - [Redis Stack Docs](https://redis.io/docs/stack/)
  - [Redis Vector Search](https://redis.io/docs/stack/search/reference/vectors/)
  - [Redis MCP Server (Referência Oficial)](https://github.com/redis-inc/mcp-redis-go) - *Nota: Este é o servidor oficial da Redis Inc. em Go. O nosso é uma implementação customizada em Node.js focada em RAG.*

- **Google AI (Gemini)**
  - [Gemini API Docs](https://ai.google.dev/docs)
  - [Embeddings Guide](https://ai.google.dev/docs/embeddings_guide)

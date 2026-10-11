# 🎵 Recomendador Musical

O **Recomendador Musical** é um assistente conversacional inteligente que recomenda músicas e dialoga sobre preferências musicais utilizando **LangGraph**, modelos de linguagem via **OpenRouter** e um sistema avançado de **memória híbrida e persistência contextual**.

O projeto demonstra como construir conversas ricas em múltiplos turnos (_multi-turn_), extrair dinamicamente informações do usuário em formato estruturado (com **Zod**), persistir histórico de sessões e condensar periodicamente mensagens antigas para manter o contexto enxuto e otimizado.

---

## 🎯 Objetivos e Funcionalidades

- 🎧 **Recomendações Personalizadas**: A IA recomenda faixas, bandas e artistas específicos adaptados ao perfil, contexto de escuta e humor do usuário.
- 🧠 **Memória Híbrida em Dois Níveis**:
  - **Sessão / Curto Prazo (PostgreSQL)**: Persistência do histórico conversacional e checkpoints do LangGraph indexados por `thread_id`.
  - **Perfil / Longo Prazo (SQLite)**: Extração e armazenamento consolidado das preferências do usuário (nome, idade, gêneros favoritos, bandas, humor e contexto).
- 📜 **Sumarização Automática de Contexto**: A cada $N$ mensagens configuradas, o grafo resume a conversa passada, atualiza o resumo do usuário e poda as mensagens antigas para evitar estouro da janela de contexto de tokens.
- ⚡ **Structured Outputs Confiáveis**: Uso de esquemas Zod (`ChatResponseSchema`, `SummarySchema`) para garantir respostas padronizadas e extração determinística de entidades.
- 🔌 **Compatibilidade OpenRouter**: Suporte a dezenas de modelos de ponta (gratuitos e pagos) com roteamento inteligente de provedores e fallback configurável.
- 🛠️ **Execução Nativa em TypeScript**: Desenvolvido para Node.js 24+ sem etapa de transpilação manual (`--experimental-strip-types`).

---

## 🏗️ Arquitetura & Como Funciona

O fluxo conversacional é gerenciado por uma máquina de estados com **LangGraph** (`StateGraph`):

```
       ┌────────────────────────┐
       │         START          │
       └───────────┬────────────┘
                   │
                   ▼
       ┌────────────────────────┐
       │       chatNode         │ ◄── Extrai preferências & gera resposta (Zod)
       └───────────┬────────────┘
                   │
         [Novas preferências?]
         ├── Sim ──► ┌────────────────────────┐
         │           │  savePreferencesNode   │ ◄── Salva/mescla dados no SQLite
         │           └───────────┬────────────┘
         │                       │
         └── Não ────────────────┤
                                 │
                   [Mensagens >= Limite?]
                   ├── Sim ──► ┌────────────────────────┐
                   │           │   summarizationNode    │ ◄── Sumariza & poda histórico
                   │           └───────────┬────────────┘
                   │                       │
                   └── Não ────────────────┤
                                           ▼
                               ┌────────────────────────┐
                               │          END           │
                               └────────────────────────┘
```

### Componentes Principais

1. **`chatNode`** ([src/graph/nodes/chatNode.ts](/src/graph/nodes/chatNode.ts)):
   - Carrega o contexto prévio do usuário do banco de dados SQLite.
   - Invoca a LLM com structured output para obter a mensagem conversacional, preferências extraídas e flags de controle.
2. **`savePreferencesNode`** ([src/graph/nodes/savePreferencesNode.ts](/src/graph/nodes/savePreferencesNode.ts)):
   - Mescla novas preferências com as já existentes na tabela `user_preferences`.
3. **`summarizationNode`** ([src/graph/nodes/summarizationNode.ts](/src/graph/nodes/summarizationNode.ts)):
   - Gera um resumo executivo da conversa e remove mensagens antigas do estado usando `RemoveMessage`.
4. **`MemoryService`** ([src/services/memoryService.ts](/src/services/memoryService.ts)):
   - Configura o `PostgresSaver` e o `PostgresStore` para persistir o estado do LangGraph no PostgreSQL.

---

## ⚙️ Pré-requisitos

Antes de baixar e executar o projeto, certifique-se de ter instalado:

1. **[Node.js](https://nodejs.org/)**: Versão **>= 24.10.0** (obrigatório para suporte ao `--experimental-strip-types` e `--env-file`).
2. **[Docker](https://www.docker.com/) & Docker Compose**: Para rodar o container do PostgreSQL responsável pelos checkpoints de memória.
3. **Conta no [OpenRouter](https://openrouter.ai/)**: Para obter uma chave de API (`OPENROUTER_API_KEY`).

---

## 🔑 Configuração de Variáveis de Ambiente & Chaves de API

Copie o arquivo [.env.example](/.env.example) para criar o seu `.env`:

```bash
cp .env.example .env
```

Edite o arquivo `.env` com suas credenciais:

```env
# OpenRouter Configuration (Obrigatório)
OPENROUTER_API_KEY=sk-or-v1-sua-chave-aqui
OPENROUTER_HTTP_REFERER=http://localhost:3000
OPENROUTER_X_TITLE=Recomendador-Musical

# LangSmith Tracing & Observabilidade (Opcional)
LANGSMITH_API_KEY=sua-chave-langsmith
LANGCHAIN_TRACING_V2=false
LANGCHAIN_PROJECT=recomendador-musical
```

### Chaves de API Utilizadas:
- **`OPENROUTER_API_KEY`** *(Obrigatória)*: Chave de acesso à API do OpenRouter para invocação dos modelos LLM.
- **`LANGSMITH_API_KEY`** *(Opcional)*: Permite rastreamento detalhado de execuções de nós, tokens e latência no dashboard do LangSmith.

---

## 🤖 Modelos de IA Configuráveis

Todas as configurações dos modelos de IA e memória ficam centralizadas no arquivo [src/config.ts](/src/config.ts):

```typescript
export const config: ModelConfig = {
  apiKey: process.env.OPENROUTER_API_KEY!,
  httpReferer: '',
  xTitle: 'Recomendador Musical',
  
  // Lista de modelos suportados / fallback
  models: [
    'nvidia/nemotron-3-ultra-550b-a55b:free', // Modelo gratuito
    // 'meta-llama/llama-3.3-70b-instruct',
    // 'anthropic/claude-3.5-sonnet',
  ],

  // Roteamento de provedores no OpenRouter
  provider: {
    sort: {
      by: 'throughput', // Prioriza velocidade ('throughput' ou 'latency')
      partition: 'none',
    },
  },

  temperature: 0.7,
  maxTokens: 1000, // Limite de tokens por resposta

  memory: {
    dbUri: 'postgresql://postgres:mysecretpassword@localhost:5432/song_recommender',
  },

  // Quantidade de mensagens acumuladas antes de disparar a sumarização
  maxMessagesToSummary: 4,
};
```

### Como trocar de modelo:
Você pode alterar o array `models` no [src/config.ts](/src/config.ts) para qualquer modelo disponível no catálogo do OpenRouter (por exemplo: `meta-llama/llama-3.3-70b-instruct`, `google/gemini-2.0-flash-001`, `openai/gpt-4o-mini`, etc.).

---

## 🚀 Instalação e Execução Passo a Passo

### 1. Instalar as dependências do projeto
```bash
npm install
```

### 2. Subir o banco de dados PostgreSQL (Docker)
Inicie o container do banco de dados configurado no [docker-compose.yml](/docker-compose.yml):
```bash
npm run docker:up
```

Para checar os logs do banco:
```bash
npm run docker:logs
```

### 3. Iniciar o Chat Interativo
Execute o chat no terminal com um usuário específico:

```bash
# Usuário "fabin"
npm run chat:fabin

# Usuária "ana"
npm run chat:ana

# Ou com um usuário customizado
node --experimental-strip-types --env-file .env src/index.ts --user meu_usuario
```

Digite suas mensagens no terminal e digite `exit` para encerrar a sessão.

---

## 📜 Scripts NPM Disponíveis

| Comando | Descrição |
|---|---|
| `npm run docker:up` | Sobe o container PostgreSQL em background |
| `npm run docker:down` | Encerra e remove o container PostgreSQL |
| `npm run docker:logs` | Exibe os logs em tempo real do PostgreSQL |
| `npm run chat:fabin` | Inicia o chat interativo com o perfil `fabin` |
| `npm run chat:ana` | Inicia o chat interativo com o perfil `ana` |
| `npm run test` | Executa a suíte de testes E2E |
| `npm run test:dev` | Executa testes em modo watch |
| `npm run langgraph:serve` | Inicia o servidor local para uso com o LangGraph Studio |

---

## 🐛 Execução em Modo Debug (VS Code)

O projeto inclui o arquivo [.vscode/launch.json](/.vscode/launch.json) configurado para depuração visual:

1. Abra qualquer arquivo `.ts` e adicione **breakpoints** clicando à esquerda do número da linha.
2. Pressione **`F5`** (ou selecione **Debug Chat (fabin)** na aba *Executar e Depurar* do VS Code).
3. Interaja pelo terminal integrado do VS Code enquanto inspeciona variáveis e a pilha de execução.

---

## 📂 Estrutura de Pastas

```text
├── docker-compose.yml             # Definição do container PostgreSQL
├── langgraph.json                 # Configuração para o LangGraph CLI / Studio
├── package.json                   # Dependências e scripts do projeto
├── preferences.db                 # Banco SQLite local para preferências de usuários
├── src/
│   ├── config.ts                  # Configurações de LLM, OpenRouter e memória
│   ├── index.ts                   # Ponto de entrada CLI interativo
│   ├── graph/
│   │   ├── factory.ts             # Fábrica para compilação do grafo e serviços
│   │   ├── graph.ts               # Definição do StateGraph e anotações Zod
│   │   └── nodes/
│   │       ├── chatNode.ts        # Nó de conversa e extração
│   │       ├── edgeConditions.ts  # Condições de roteamento condicional
│   │       ├── savePreferencesNode.ts # Nó de persistência de perfil
│   │       └── summarizationNode.ts   # Nó de sumarização e limpeza de mensagens
│   ├── prompts/
│   │   └── v1/
│   │       ├── chatResponse.ts    # Prompts e esquemas Zod do chat
│   │       └── summarization.ts   # Prompts e esquemas Zod da sumarização
│   └── services/
│       ├── memoryService.ts       # Checkpointing no PostgreSQL (LangGraph)
│       ├── openrouterService.ts   # Integração com a API do OpenRouter
│       └── preferencesService.ts  # Gerenciamento de SQLite com Knex
└── tests/
    └── chat.e2e.test.ts           # Testes automatizados de ponta a ponta
```

---

## 📄 Licença

Este projeto está sob a licença [MIT](/package.json).

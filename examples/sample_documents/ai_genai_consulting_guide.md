# Guia de Soluções de IA Generativa para Projetos de Consultoria

**Versão:** 2.1  
**Área:** Engenharia de IA  
**Classificação:** Interno

---

## 1. Introdução

Este guia descreve os padrões, arquiteturas e critérios de decisão adotados pela área de Engenharia de IA para projetos de IA Generativa em clientes corporativos. O objetivo é padronizar escolhas técnicas, acelerar entregas e garantir consistência de qualidade entre equipes.

---

## 2. Arquiteturas principais

### 2.1 RAG — Retrieval-Augmented Generation

RAG é a arquitetura recomendada quando o cliente possui bases de conhecimento proprietárias que o modelo de linguagem não conhece (documentos internos, manuais, políticas, histórico de tickets). O fluxo básico é:

1. O usuário faz uma pergunta.
2. O sistema converte a pergunta em um vetor de embeddings.
3. Uma busca semântica recupera os trechos mais relevantes da base.
4. O modelo de linguagem gera a resposta usando esses trechos como contexto.

**Quando usar RAG:**
- Base de conhecimento grande (> 500 documentos) e em constante atualização.
- Necessidade de rastreabilidade (citar a fonte da resposta).
- Custo de fine-tuning inviável ou frequência de atualização alta demais.

**Quando NÃO usar RAG:**
- O conhecimento necessário é estável e pequeno o suficiente para caber no contexto do modelo.
- O cliente não tem base documental estruturada.
- Latência abaixo de 200ms é exigida (RAG adiciona uma etapa de busca).

**Stack recomendado:** PostgreSQL + pgvector, OpenAI Embeddings (`text-embedding-3-small`), FastAPI, LangChain ou LangGraph.

---

### 2.2 Arquiteturas Multi-Agente

Sistemas multi-agente são indicados quando uma tarefa complexa pode ser decomposta em subtarefas especializadas que se beneficiam de paralelismo ou de agentes com ferramentas e prompts distintos.

**Padrões de orquestração:**

| Padrão | Quando usar |
|---|---|
| Supervisor + Workers | O supervisor decide qual worker chamar com base no contexto. Ideal para fluxos com ramificações. |
| Pipeline sequencial | As etapas têm dependência estrita (A → B → C). Simples e previsível. |
| Swarm (consenso) | Múltiplos agentes independentes votam ou colaboram para produzir uma saída. Usado em revisão de código e análise de risco. |

**Framework recomendado:** LangGraph. Permite modelar o fluxo como um grafo de estados com nós (agentes/ferramentas) e arestas condicionais, com suporte nativo a checkpoints e retomada de execução.

---

### 2.3 MCP — Model Context Protocol

MCP (Model Context Protocol) é um protocolo aberto desenvolvido pela Anthropic que padroniza como modelos de linguagem se conectam a ferramentas e fontes de dados externas. Funciona como uma "tomada universal" entre o modelo e qualquer sistema externo.

**Componentes do MCP:**
- **MCP Host:** o modelo ou agente que consome as ferramentas.
- **MCP Server:** o serviço que expõe ferramentas, recursos ou prompts.
- **Transport:** comunicação via stdio (local) ou SSE/HTTP (remoto).

**Vantagens para projetos de consultoria:**
- Reutilização de servidores MCP entre projetos diferentes (um servidor de banco de dados, por exemplo, serve qualquer agente).
- Auditabilidade: cada chamada de ferramenta é explícita no protocolo.
- Reduz lock-in: o mesmo servidor MCP funciona com Claude, GPT-4o ou modelos open-source.

**Exemplo de uso em projeto:** um agente de suporte interno usa um MCP Server conectado ao Jira para abrir tickets, ao Confluence para buscar documentação, e ao banco de dados de incidentes — tudo sem código de integração no agente em si.

---

## 3. Critérios de decisão: RAG vs. Fine-tuning vs. Prompt Engineering

| Critério | Prompt Engineering | RAG | Fine-tuning |
|---|---|---|---|
| Base de conhecimento | Pequena (cabe no contexto) | Grande, dinâmica | Estável, volumosa |
| Custo de implementação | Baixo | Médio | Alto |
| Tempo até produção | Dias | Semanas | Meses |
| Rastreabilidade da fonte | Não | Sim | Não |
| Atualização de conhecimento | Imediata | Imediata | Requer retreino |
| Tom e estilo personalizados | Parcial | Parcial | Total |

**Regra geral adotada pela equipe:** começar sempre com prompt engineering. Se o contexto não couber, adicionar RAG. Fine-tuning apenas quando estilo/comportamento não consegue ser reproduzido por prompt e o volume de dados é suficiente.

---

## 4. Infraestrutura e Cloud

### 4.1 Banco vetorial

- **PostgreSQL + pgvector:** recomendado para projetos que já usam Postgres. Custo zero de infra adicional, query SQL normal com extensão `<->` para distância cosseno.
- **Pinecone / Weaviate:** recomendado quando o volume de vetores ultrapassa 10 milhões ou quando a equipe não tem experiência com Postgres.

### 4.2 Computação

- **AWS:** SageMaker para hosting de modelos open-source, Lambda para APIs leves, ECS/EKS para serviços de agentes.
- **Azure:** Azure OpenAI Service para modelos GPT/Embeddings com compliance europeu, AKS para orquestração.
- **GCP:** Vertex AI para Gemini, Cloud Run para FastAPI serverless.

### 4.3 Observabilidade

Todo agente em produção deve ter rastreamento de:
- Latência por nó do grafo (LangSmith ou Langfuse).
- Tokens consumidos por requisição.
- Taxa de fallback (quando o agente não encontra contexto relevante).
- Qualidade das respostas via RAGAS (faithfulness, answer relevancy, context precision).

---

## 5. Segurança e Governança

### 5.1 Guardrails obrigatórios em projetos enterprise

- **Input validation:** sanitizar prompts do usuário antes de enviar ao modelo. Bloquear prompt injection via checagem de padrões suspeitos.
- **Output filtering:** classificar a saída antes de retornar ao usuário. Bloquear PII (CPF, cartão, senha) e conteúdo sensível.
- **Grounding:** toda resposta gerada com RAG deve citar a fonte. Respostas sem fonte são sinalizadas como "não fundamentadas".

### 5.2 Dados do cliente

- Nenhum dado de cliente pode ser enviado a modelos externos sem consentimento explícito e contrato de processamento de dados.
- Em projetos com restrição de saída de dados, usar modelos self-hosted (LLaMA 3, Mistral) ou Azure OpenAI com rede privada.

---

## 6. Processo de entrega

### Fase 1 — Discovery (1–2 semanas)
- Mapeamento das fontes de dados disponíveis.
- Definição dos casos de uso prioritários com o cliente.
- Escolha da arquitetura (RAG, agente, pipeline).

### Fase 2 — Prova de Conceito (2–4 semanas)
- Implementação do fluxo básico end-to-end.
- Avaliação de qualidade com conjunto de perguntas de referência (ground truth).
- Aprovação do cliente para avançar.

### Fase 3 — Produção (4–8 semanas)
- Hardening: guardrails, logging, autenticação.
- Integração com sistemas do cliente (ERP, CRM, intranet).
- Deploy em cloud do cliente com CI/CD.
- Treinamento dos usuários finais.

---

## 7. Contatos da prática de IA

| Papel | Responsabilidade |
|---|---|
| AI Practice Lead | Aprovação de arquitetura e revisão técnica de propostas |
| AI Engineer (Sênior) | Implementação de agentes, RAG e integrações |
| MLOps Engineer | Infraestrutura, deploy, monitoramento em produção |
| Data Engineer | Pipelines de ingestão e atualização da base vetorial |

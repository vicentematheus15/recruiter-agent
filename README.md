# Recruiter Agent: Assistente de Carreira Multi-Agente com IA

> Projeto desenvolvido durante a **Imersão em IA da Alura**, um curso online intensivo focado em construção de sistemas baseados em agentes de inteligência artificial.

---

## 📌 Sobre o Projeto

Sistema multi-agente conversacional de desenvolvimento de carreira, construído para rodar dentro do editor **Zed** usando seu recurso nativo de agentes de IA. O usuário interage com um orquestrador central (o **Maestro**) que, dependendo da solicitação, despacha agentes especializados para buscar vagas, recomendar cursos ou conduzir entrevistas simuladas.

Todo o estado da conversa é armazenado em arquivos Markdown simples. Sem banco de dados, sem servidor, sem framework externo. A "lógica" do sistema vive nas personas e skills escritas em linguagem natural, que instruem os modelos de linguagem a se comportarem como agentes com papéis bem definidos.

---

## 🏗️ Arquitetura

O sistema é composto por um orquestrador e três agentes especializados, cada um com sua própria persona, skills e protocolo de comunicação:

```
┌─────────────────────────────────────────────────┐
│                    Usuário                        │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│            MAESTRO (Orquestrador)                │
│  · Interface principal com o usuário             │
│  · Conduz o quiz de perfil                       │
│  · Coordena os agentes via spawn_agent           │
│  · Lê e escreve arquivos em data/                │
└──┬──────────────┬──────────────┬────────────────┘
   │              │              │
   ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────────┐
│  SCOUT  │  │ CURATOR  │  │    COACH     │
│  Busca  │  │  Busca   │  │  Simulação   │
│  vagas  │  │  cursos  │  │ de entrevista│
└─────────┘  └──────────┘  └──────────────┘
```

### Como a orquestração funciona

Os agentes não se comunicam diretamente, o Maestro é o único ponto de contato com o usuário. Quando uma tarefa é necessária, o Maestro constrói um **Envelope de Despacho** em Markdown estruturado e o envia ao sub-agente via `spawn_agent` (ferramenta nativa do Zed). O sub-agente retorna um **Envelope de Resposta** padronizado com campos `estado`, `resumo`, `dados` e `erros`.

```
## DESPACHO: SCOUT
### referencia_persona
[conteúdo completo de personas/scout.md]
### tarefa
Buscar vagas de emprego para Backend em São José - SC
### perfil_usuario
[conteúdo de data/user-profile.md]
### contexto
Área: Backend | Nível: Estágio | Habilidades: Node.js, JWT...
### saida_esperada
Lista de até 5 vagas com habilidades correspondentes e faltantes
```

---

## 🧩 Agentes e Responsabilidades

### 🎼 Maestro — Orquestrador Principal
**Arquivo:** `personas/maestro.md`

Interface principal com o usuário. Ao ser iniciado, verifica se o quiz de perfil já foi respondido (`data/personality-quiz.md`). Se não, conduz o quiz com 7 perguntas feitas uma por vez. Após o quiz, gera `data/user-profile.md` com o perfil consolidado e apresenta o menu.

**Menu principal:**
| Opção | Ação |
|---|---|
| A | Buscar vagas (Scout) |
| B | Recomendar cursos (Curator) |
| C | Entrevista simulada (Coach) |
| D | Refazer o quiz |

O Maestro suporta retomada de quiz incompleto — se `personality-quiz.md` existir mas não tiver `Concluído: true`, pergunta ao usuário se deseja continuar de onde parou ou recomeçar do zero.

---

### 🔍 Scout — Agente de Busca de Vagas
**Arquivo:** `personas/scout.md` + `skills/job-search.md`

Recebe o perfil do usuário e executa buscas de vagas usando o **CLI do Firecrawl**, que agrega resultados de Indeed, Catho, LinkedIn, Glassdoor e Infojobs. Para cada vaga encontrada:

1. Executa `firecrawl search "vagas [area] [localização]" --json`
2. Para cada URL retornada, executa `firecrawl scrape <url> --format markdown` para extrair detalhes completos
3. Compara as habilidades da vaga com as habilidades do usuário (case-insensitive)
4. Produz duas listas: `habilidades_correspondentes` e `habilidades_faltantes`
5. Retorna até 5 vagas formatadas com contagem de correspondência

As habilidades faltantes são gravadas em `data/job-search-results.md` e alimentam o Curator na etapa seguinte.

---

### 📚 Curator — Agente de Recomendação de Cursos
**Arquivo:** `personas/curator.md` + `skills/course-search.md`

Lê as habilidades faltantes de `data/job-search-results.md` e busca cursos na **Alura** que as cubram. Para cada habilidade faltante:

1. Executa `firecrawl search "alura [habilidade]" --json`
2. Tenta fallback `firecrawl search "site:alura.com.br [habilidade]" --json` se a primeira busca falhar
3. Faz scrape das páginas dos cursos para extrair duração e nível
4. Classifica os cursos por nível usando heurísticas de título: `iniciante` → `intermediario` → `avancado`
5. Retorna até 5 cursos ordenados do mais básico ao mais avançado

---

### 🎤 Coach — Simulador de Entrevistas
**Arquivo:** `personas/coach.md` + `skills/interview-simulator.md`

Conduz uma entrevista simulada de 5 perguntas baseada em uma vaga selecionada. É o agente mais complexo do sistema — o Maestro o despacha **6 vezes sequencialmente**, passando o histórico acumulado de perguntas e respostas a cada chamada:

| Despacho | Conteúdo |
|---|---|
| 1 | Gera e retorna a Pergunta 1 |
| 2 | Avalia Resposta 1 → feedback + Pergunta 2 |
| 3 | Avalia Resposta 2 → feedback + Pergunta 3 |
| 4 | Avalia Resposta 3 → feedback + Pergunta 4 |
| 5 | Avalia Resposta 4 → feedback + Pergunta 5 |
| 6 | Avalia Resposta 5 → pontuação final + áreas de melhoria |

As 5 perguntas misturam **comportamentais** ("conte-me sobre uma vez em que...") e **técnicas** (específicas da área da vaga). A complexidade é calibrada pelo nível de experiência do usuário: Estágio/Júnior recebem perguntas fundamentais; Sênior recebe questões de arquitetura e design de sistemas.

Ao final, o Coach entrega uma **pontuação de 1 a 10** avaliando precisão técnica, clareza de comunicação, confiança e relevância para a vaga, mais 2-3 áreas de melhoria concretas e acionáveis.

---

## 📁 Estrutura do Projeto

```
recruiter-agent/
├── AGENTS.md                      # Instrução de inicialização: adota a persona Maestro imediatamente
│
├── personas/
│   ├── maestro.md                 # Orquestrador: quiz, menu, despacho de agentes
│   ├── scout.md                   # Busca de vagas via Firecrawl
│   ├── curator.md                 # Recomendação de cursos na Alura
│   └── coach.md                   # Simulador de entrevistas com 6 despachos sequenciais
│
├── skills/
│   ├── dispatch.md                # Protocolo de envelopes de despacho e resposta
│   ├── firecrawl.md               # Comandos do CLI Firecrawl (search e scrape)
│   ├── job-search.md              # Fluxo completo de busca e correspondência de vagas
│   ├── course-search.md           # Fluxo de busca, extração e ordenação de cursos
│   └── interview-simulator.md     # Geração de perguntas, feedback e rubrica de pontuação
│
├── data/
│   ├── personality-quiz.md        # Respostas do quiz (gerado na primeira conversa)
│   ├── user-profile.md            # Perfil consolidado com funções-alvo mapeadas
│   ├── job-search-results.md      # Últimas vagas encontradas pelo Scout
│   └── course-recommendations.md  # Últimas recomendações do Curator
│
├── plano.md                       # Aula 1: construção do Maestro e quiz
├── plano-2.md                     # Aula 2: construção do Scout
├── plano-3.md                     # Aula 3: construção do Curator
└── plano-4.md                     # Aula 4: construção do Coach
```

---

## 🗂️ Arquivos de Estado (`data/`)

Todo o estado do sistema é persistido em arquivos Markdown simples, sem banco de dados.

### `data/personality-quiz.md`
Gerado pelo Maestro durante o quiz. Considerado completo quando `Concluído: true`.
```
Área de interesse: Backend
Nível de experiência: Estágio
Preferências de trabalho: Remoto, Híbrido
Localização: São José - Santa Catarina
Soft skills: Comunicação, resolução de problemas
Objetivo de carreira: Primeiro emprego
Habilidades atuais: Node.js, JavaScript, Git, JWT, API REST
Concluído: true
```

### `data/user-profile.md`
Gerado a partir do quiz. Adiciona o campo `Funções alvo`, mapeado automaticamente pela combinação de Área + Nível (ex: Backend + Estágio → "Estagiário de Backend, Estagiário de Desenvolvimento de Software, Estagiário de API").

### `data/job-search-results.md`
Salvo pelo Maestro após o Scout retornar. Inclui vagas com habilidades correspondentes e faltantes — estas últimas alimentam o Curator.

### `data/course-recommendations.md`
Salvo pelo Maestro após o Curator retornar. Lista cursos com nome, nível, duração e link.

---

## 🛠️ Tecnologias e Conceitos

| Tecnologia / Conceito | Uso no projeto |
|---|---|
| **Zed Editor** | Ambiente de execução; fornece `spawn_agent`, `terminal` e `find_path` |
| **Firecrawl CLI** | Busca web (`firecrawl search`) e scraping (`firecrawl scrape --format markdown`) |
| **Prompt Engineering** | Personas e skills escritas em Markdown como instruções comportamentais para LLMs |
| **Arquitetura Multi-Agente** | Orquestrador + sub-agentes especializados com protocolo de handoff definido |
| **MoE (Mixture of Experts)** | Padrão de design adotado: cada agente tem uma única responsabilidade bem definida |
| **State Management em Markdown** | Persistência de estado entre conversas sem banco de dados |
| **Envelope de Despacho/Resposta** | Protocolo estruturado de comunicação entre Maestro e sub-agentes |

---

## 🚀 Como Usar

### Pré-requisitos

- [Zed Editor](https://zed.dev/) instalado
- [Firecrawl CLI](https://github.com/mendableai/firecrawl) instalado e configurado
- Variável de ambiente `FIRECRAWL_API_KEY` definida

### Iniciando

1. Clone o repositório e abra a pasta no Zed
2. Abra o painel de agentes do Zed (Agent Panel)
3. O arquivo `AGENTS.md` é lido automaticamente pelo Zed na inicialização — o agente adota a persona Maestro imediatamente
4. O Maestro saúda o usuário e verifica se o quiz já foi respondido

### Fluxo de uso

```
1. Responda o quiz de perfil (7 perguntas, uma por vez)
2. Escolha uma opção do menu:
   A → Buscar vagas compatíveis com seu perfil
   B → Ver cursos para preencher suas lacunas de habilidades
   C → Praticar uma entrevista simulada
   D → Refazer o quiz
```

---

## 📚 Contexto Acadêmico

Este projeto foi proposto e construído ao longo das **4 aulas da Imersão em IA da Alura**, cada uma introduzindo um novo agente ao sistema:

| Aula | Entregável |
|---|---|
| Aula 1 | Maestro com quiz de perfil e menu |
| Aula 2 | Scout com busca de vagas via Firecrawl |
| Aula 3 | Curator com recomendação de cursos na Alura |
| Aula 4 | Coach com simulação de entrevistas em 6 despachos |

Os arquivos `plano.md`, `plano-2.md`, `plano-3.md` e `plano-4.md` contêm as especificações técnicas detalhadas de cada aula, incluindo arquitetura, tasks, fluxos e critérios de teste.

---

## 📄 Licença

Distribuído sob a licença MIT. Veja o arquivo [LICENSE](./LICENSE) para mais informações.

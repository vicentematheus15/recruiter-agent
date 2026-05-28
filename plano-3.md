# Curator — Agente de Busca de Cursos

## Visão Geral

Agente especializado em buscar cursos na **alura.com.br** que complementem as habilidades faltantes do usuário, identificadas a partir do quiz de personalidade e das vagas encontradas pelo agente Scout. O Curator utiliza o **Firecrawl** para navegar no site da Alura, extrair informações de cursos e gerar recomendações estruturadas.

## Objetivo

- Receber como entrada a lista de habilidades faltantes (gerada pelo Scout) e o perfil do usuário.
- Realizar buscas no domínio `alura.com.br` usando o Firecrawl, filtrando por tópicos relevantes.
- Extrair título, autor, carga horária, nível, link e descrição resumida de cada curso.
- Priorizar cursos que cubram o maior número de habilidades faltantes.
- Retornar até **5** recomendações de cursos ao Maestro, que as exibirá ao usuário.

## Pré‑requisitos

- Firecrawl instalado e configurado (`FIRECRAWL_API_KEY` definida).
- Acesso ao site `alura.com.br` (não requer login para visualização de cursos públicos).

## Diretrizes para Modelos MoE

- Cada passo deve ser explicitamente descrito, indicando a ferramenta a ser usada e o formato de saída esperado.
- Saídas estruturadas devem ser listas numeradas de pares chave‑valor (sem tabelas markdown).
- Todos os caminhos de arquivo são relativos à raiz do projeto, prefixados por `data/`.
- Em caso de falha de busca ou extração, relatar o erro exato no campo `erros` e interromper a execução.
- Não gerar código Python ou scripts de shell; o agente age por meio de prompts e ferramentas do Zed.

## Arquitetura

```
┌─────────────────────────────────────────────────┐
│                  Usuário                          │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│              MAESTRO (Orquestrador)              │
│  - Coordena agentes e consolida resultados        │
└──┬──────────────┬──────────────┬────────────────┘
   │              │              │
   ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────────┐
│ SCOUT   │  │ CURATOR  │  │ COACH        │
│ (Vagas) │  │ (Cursos) │  │ (Entrevista) │
└─────────┘  └──────────┘  └──────────────┘
```

## Curator - Agente de Busca de Cursos

**Responsabilidade**: Buscar, analisar e recomendar cursos da Alura que preencham as lacunas de habilidades do usuário.

**Skills**:
- `skills/course-search.md` – fluxo completo de busca de cursos via Firecrawl, extração de metadados e montagem da resposta.
- `skills/firecrawl.md` – comandos e regras do CLI Firecrawl (já existente).

**Ferramentas do Zed**:
- `terminal` – executar comandos `firecrawl search` e `firecrawl scrape`.

**Fluxo de Trabalho**
1. **Entrada** – Receber do Maestro a lista de habilidades faltantes e o perfil do usuário (`data/user-profile.md`).
2. **Busca** – Executar:
   ```
   firecrawl search "cursos alura [habilidade]" --json
   ```
   para cada habilidade faltante (ou para combinações relevantes). Limitar a 10 resultados por busca.
3. **Raspagem** – Para cada URL de curso retornada, executar:
   ```
   firecrawl scrape <url> --format markdown
   ```
   e extrair:
   - título
   - autor
   - carga horária
   - nível (iniciante, avançado, etc.)
   - descrição resumida
   - link
4. **Filtragem** – Remover duplicados e cursos que não abordem nenhuma das habilidades faltantes.
5. **Pontuação** – Atribuir pontuação baseada no número de habilidades cobertas por curso.
6. **Seleção** – Ordenar por pontuação descrescente e selecionar até 5 cursos.
7. **Resposta** – Construir o *Response Envelope* esperado pelo Maestro:
   ```
   1. titulo: <título>
      autor: <autor>
      carga_horaria: <horas>
      nivel: <nível>
      link: <URL>
      habilidades_cobertas: [habilidade1, habilidade2]
      descricao_resumida: <texto>
   2. ...
   ```
   Incluir campo `erros` caso alguma busca ou raspagem falhe.

## Skills

### course-search.md
- **Ferramenta Zed**: `terminal` para comandos Firecrawl.
- **Busca de cursos**: `firecrawl search "cursos alura [habilidade]" --json`.
- **Extração**: `firecrawl scrape <url> --format markdown`.
- **Análise**: Parsear markdown para obter metadados (título, autor, carga horária, nível, descrição).
- **Correspondência**: Verificar quais habilidades faltantes aparecem no conteúdo do curso (case‑insensitive).
- **Ordenação**: Priorizar cursos que cubram mais habilidades.
- **Saída**: Lista numerada de até 5 cursos com campos descritos acima.
- **Tratamento de erros**: Se `search` falhar, registrar erro e abortar. Se `scrape` falhar para um curso, usar apenas os dados disponíveis do resultado de busca e anotar a falha.

## Persona

`personas/curator.md`
- Descreve o papel do Curator, ferramentas disponíveis e referência obrigatória às skills `course-search.md` e `firecrawl.md`.
- Define o *Response Envelope* que o agente deve retornar ao Maestro.
- Estabelece regras de erro e limites de resultados.

## Protocolo de Handoff do Curator

**Envelope de Despacho** (construído pelo Maestro):
```
## DESPACHO: CURATOR
### referencia_persona
[conteúdo completo de personas/curator.md]

### tarefa
Buscar cursos na Alura que preencham as habilidades faltantes: [lista de habilidades]

### perfil_usuario
[conteúdo de data/user-profile.md]

### contexto
Habilidades faltantes: [habilidade1, habilidade2, ...]

### saida_esperada
Envelope de resposta com estado, resumo, dados (lista de cursos) e erros se houver
```

**Formato de Resposta**
```
1. titulo: <título do curso>
   autor: <autor>
   carga_horaria: <horas>
   nivel: <nível>
   link: <URL>
   habilidades_cobertas: [habilidade1, habilidade2]
   descricao_resumida: <texto>
2. ...
```

## Esquemas dos Arquivos de Dados

### data/course-recommendations.md
Armazena as recomendações mais recentes do Curator:
```
Data da Busca: [AAAA-MM-DD HH:MM]
Habilidades analisadas: [habilidade1, habilidade2]

Recomendacoes:
1. titulo: ...
   autor: ...
   carga_horaria: ...
   nivel: ...
   link: ...
   habilidades_cobertas: [...]
   descricao_resumida: ...
2. ...
```

## Tasks

1. **Escrever `skills/course-search.md`** – detalhar fluxo de busca, extração e formatação.
2. **Escrever `personas/curator.md`** – papel, ferramentas, referências a skills e regras de resposta.
3. **Atualizar Maestro** – adicionar opção de despacho para o Curator (similar ao Scout) e salvar resultados em `data/course-recommendations.md`.
4. **Testar fluxo** – usuário seleciona a opção correspondente, Curator realiza buscas, recomenda cursos e Maestro exibe ao usuário.
5. **Testar tratamento de erro** – simular falha no `firecrawl search` ou `scrape` e garantir que o erro seja reportado e o fluxo retorne ao menu.

## Fluxo Resumido
```
Usuário → seleciona opção Curator
   │
   ▼
Maestro constrói envelope de despacho para Curator
   │
   ▼
spawn_agent dispara Curator com persona + contexto
   │
   ▼
Curator executa firecrawl search → firecrawl scrape
   │
   ▼
Curator gera lista de até 5 cursos e devolve envelope de resposta
   │
   ▼
Maestro salva em data/course-recommendations.md → exibe ao usuário → retorna ao menu
```

## Regras de Erro
- Se `firecrawl search` falhar, incluir mensagem em `erros` e abortar.
- Se `firecrawl scrape` falhar para um curso, usar dados da busca e marcar `scrape_failed: true`.
- Se `spawn_agent` falhar, comunicar ao usuário e retornar ao menu.
- Sempre limitar a 5 recomendações para evitar sobrecarga visual.

---

*Este plano complementa o plano‑2.md, completando o conjunto de agentes SCOUT, CURATOR e COACH dentro da arquitetura multi‑agente.*
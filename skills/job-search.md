# Job Search Skill

## Ferramentas do Zed
- `terminal` – executar comandos CLI do Firecrawl.
- `read_file` – ler `data/user-profile.md` para obter habilidades, nível e localização.
- `write_file` – salvar resultados em `data/job-search-results.md` (opcional, o agente pode retornar e Maestro grava).

## Fluxo
1. **Carregar perfil do usuário**
   - Ler `data/user-profile.md`.
   - Extrair campos: `area_de_interesse`, `localizacao`, `nivel_de_experiencia`, `habilidades` (lista separada por vírgulas).
2. **Buscar vagas**
   - Construir query: `vagas {area_de_interesse} {localizacao}`.
   - Executar:
     ```sh
     firecrawl search "vagas {area_de_interesse} {localizacao}" --json
     ```
   - O comando devolve JSON com objetos contendo `url`, `title`, `description`.
   - Se o comando falhar, retornar envelope de resposta com `estado: erro` e mensagem.
3. **Obter detalhes de cada vaga**
   - Para cada resultado (máximo 5), executar:
     ```sh
     firecrawl scrape <url> --format markdown
     ```
   - Se a extração falhar, usar `title` e `description` do resultado da busca como fallback e anotar a falha.
4. **Extrair requisitos da vaga**
   - Analisar o markdown retornado procurando padrões de habilidades (palavras‑chave). Para simplificar, considerar todas as palavras que aparecem nas seções "Requisitos", "Skills", "Tecnologias" como habilidades requeridas.
   - Normalizar a lista para minúsculas e remover duplicatas.
5. **Correspondência de habilidades**
   - Comparar a lista de habilidades requeridas com a lista do usuário (case‑insensitive).
   - `habilidades_correspondentes` = interseção.
   - `habilidades_faltantes` = diferença (requeridas – usuário).
   - `contagem_correspondencia` = "X de Y habilidades correspondem" onde X = tamanho da interseção e Y = total de habilidades requeridas.
6. **Filtragem por nível de experiência**
   - Se a descrição da vaga mencionar um nível (júnior, pleno, sênior), manter apenas vagas que coincidam com `nivel_de_experiencia` do usuário.
   - Caso não haja vagas do nível exato nos primeiros 5 resultados, expandir a busca incluindo vagas adjacentes e marcar a discrepância no campo `nivel_discrepancia` (opcional).
7. **Construir resposta**
   - Selecionar até 5 vagas que atendam aos critérios.
   - Formatar cada vaga conforme o **Formato de dados da resposta**:
     ```
     1. titulo: <title>
        empresa: <empresa>
        localizacao: <cidade ou Remoto>
        link: <url>
        habilidades_correspondentes: [h1, h2]
        habilidades_faltantes: [h3, h4]
        contagem_correspondencia: <X de Y>
     ```
   - Incluir campo opcional `nivel_discrepancia` se houver diferença de nível.
8. **Envelope de Resposta**
   ```
   ## RESPOSTA: SCOUT
   ### estado
   sucesso
   ### dados
   <lista formatada acima>
   ```
   - Em caso de erro em qualquer etapa, usar `estado: erro` e incluir mensagem detalhada.

## Tratamento de Erros
- Falha no `firecrawl search` → retornar erro imediatamente.
- Falha no `firecrawl scrape` de uma URL específica → usar fallback (title/description) e continuar.
- Falha ao ler `data/user-profile.md` → erro crítico.
- Qualquer exceção inesperada → `estado: erro` com descrição.

# job-search.md

## Objetivo
Implementa o fluxo de busca de vagas usando o CLI **Firecrawl** e gera um resumo estruturado dos resultados para o agente Scout.

## Passos
1. **Construir consulta**
   ```text
   firecrawl search "vagas <area_de_interesse> <localizacao>" --json
   ```
   - `<area_de_interesse>` e `<localizacao>` são lidos de `data/user-profile.md`.
   - O flag `--json` garante que a saída seja um JSON contendo `url`, `title`, `description` e `snippet` para cada resultado.

2. **Executar busca**
   - Use a ferramenta `terminal` com `cd` apontando para a raiz do projeto (`recruiter-agent`).
   - Capture a saída; se o comando falhar, registre o erro no campo `erros` e interrompa o fluxo.

3. **Selecionar até 5 resultados**
   - Parseie o JSON e mantenha os primeiros 5 itens.
   - Para cada URL, execute:
     ```text
     firecrawl scrape <url> --format markdown
     ```
   - Caso a extração falhe, use apenas o `title` e `description` retornados pela busca.

4. **Extrair informações da vaga**
   - **Título** – `title`.
   - **Empresa** – Tentar inferir a partir da URL (sub‑domínio) ou do título (texto antes de "-" ou "|`).
   - **Localização** – Procurar por padrões de cidade/estado ou a palavra "Remoto" no conteúdo markdown.
   - **Habilidades requeridas** – Extrair listas ou frases que contenham palavras‑chave de habilidades (ex.: "Node.js", "JavaScript", "SQL", etc.).
   - **Nível de experiência** – Detectar termos como "Estágio", "Júnior", "Pleno", "Sênior".

5. **Correspondência de habilidades**
   - Ler `data/user-profile.md` e obter a lista de habilidades técnicas do usuário.
   - Comparar (case‑insensitive) cada habilidade requerida da vaga com a lista do usuário.
   - Produzir duas listas:
     - `habilidades_correspondentes`
     - `habilidades_faltantes`
   - Calcular `contagem_correspondencia` como `X de Y` onde `X` é o número de correspondências e `Y` o total de habilidades requeridas.

6. **Filtragem por nível**
   - Se a vaga especificar um nível que não corresponde ao nível do usuário, marcar a vaga como "nível divergente" e ainda incluí‑la caso não existam vagas do nível exato nos primeiros 5 resultados.

7. **Formato de saída**
   ```text
   1. titulo: <título da vaga>
      empresa: <nome da empresa>
      localizacao: <cidade ou Remoto>
      link: <URL>
      habilidades_correspondentes: [h1, h2]
      habilidades_faltantes: [h3, h4]
      contagem_correspondencia: X de Y habilidades correspondem
   
   2. titulo: ...
   ```
   - Até 5 vagas.
   - Caso alguma extração falhe, incluir o campo `erro_extracao` com a mensagem.

8. **Envelope de resposta**
   - O Scout deve devolver um objeto JSON contendo:
     ```json
     {
       "estado": "sucesso" | "erro",
       "resumo": "<texto resumido>",
       "dados": "<texto formatado acima>",
       "erros": []
     }
     ```
   - Em caso de erro, `estado` = "erro" e `erros` lista as mensagens.

## Tratamento de erros
- Falha no comando `firecrawl search` → registrar erro e abortar.
- Falha no `firecrawl scrape` de uma URL específica → usar título/descrição da busca e anotar `erro_extracao`.
- JSON inválido → registrar erro e abortar.
- Se nenhum resultado for encontrado → retornar `estado":"erro"` com mensagem "Nenhuma vaga encontrada".

## Dependências
- CLI Firecrawl configurado (`FIRECRAWL_API_KEY`).
- Ferramenta `terminal` do Zed.

---
*Este skill deve ser carregado por `personas/scout.md`.*
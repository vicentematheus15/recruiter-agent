# Course Search Skill

## Overview
This skill defines the complete workflow for the **Curator** agent to search for courses on **alura.com.br** that fill the user's missing skills.

## Steps
1. **Input**
   - Receive a list of missing skills (e.g., `"Python", "Docker"`).
   - Load the user profile from `data/user-profile.md` if additional context is needed.
2. **Search**
   - For each missing skill (or a combination of up to two skills), run the Firecrawl CLI:
     ```
     firecrawl search "cursos alura [skill]" --json
     ```
   - Limit the result set to **10** items per search.
   - Collect the JSON output which contains `url`, `title`, and a short `description` for each course.
3. **Scrape**
   - For every URL returned, execute:
     ```
     firecrawl scrape <url> --format markdown
     ```
   - Parse the markdown to extract the following fields (if present):
     - **title**
     - **author**
     - **carga horária** (hours)
     - **nível** (iniciante, avançado, etc.)
     - **description** (first paragraph as a short summary)
   - If scraping fails, keep the data from the search result and set `scrape_failed: true`.
4. **Match Skills**
   - Compare the course description (case‑insensitive) with the list of missing skills.
   - Build an array `habilidades_cobertas` containing every missing skill that appears in the description.
   - Discard any course where `habilidades_cobertas` is empty.
5. **Scoring**
   - Score each course by the number of covered skills (`score = len(habilidades_cobertas)`).
   - Optionally boost courses whose `nível` matches the user's experience level.
6. **Selection**
   - Sort courses by `score` descending.
   - Keep the top **5** courses.
7. **Response Envelope**
   - Return a structured list (no markdown tables) as follows:
     ```
     1. titulo: <title>
        autor: <author>
        carga_horaria: <hours>
        nivel: <level>
        link: <url>
        habilidades_cobertas: [skill1, skill2]
        descricao_resumida: <short description>
        scrape_failed: false
     2. ...
     ```
   - If any step fails, include an `erros` field with a clear message and abort further processing.

## Error Handling
- **Search failure** – Populate `erros` with the CLI error output and stop.
- **Scrape failure for a single course** – Keep the course entry, set `scrape_failed: true`, and note that only search data is available.
- **Parsing errors** – Treat as scrape failure for that course.

## Dependencies
- Relies on the existing `skills/firecrawl.md` for CLI usage guidelines.
- Uses the Zed `terminal` tool to run the commands.

## Output
The skill produces a plain‑text response that the Curator agent will forward to the Maestro. No files are written directly by this skill; the Maestro will persist the result in `data/course-recommendations.md`.

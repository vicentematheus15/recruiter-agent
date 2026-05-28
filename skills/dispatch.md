# Dispatch Protocol

## Tabela de Roteamento
| Agente | Código |
|--------|--------|
| Scout  | A |
| Curator| B |
| Coach  | C |
| Maestro| D |

## Envelope de Despacho (Maestro constrói este prompt para `spawn_agent`)
```
## DESPACHO: [NOME_DO_AGENTE]
### referencia_persona
[Conteúdo completo de personas/<nome_do_agente_minusculo>.md]

### tarefa
[Uma frase descrevendo o que o agente deve fazer]

### perfil_usuario
[Conteúdo de data/user-profile.md]

### contexto
[Contexto específico: ex: qual vaga para entrevistar, quais habilidades buscar cursos]

### saida_esperada
[Exatamente em que formato o agente deve retornar]
```

## Envelope de Resposta (agente despachado retorna isto)
```
## RESPOSTA: [NOME_DO_AGENTE]
### estado
[sucesso | erro]
### dados
[Dados retornados conforme o formato esperado]
```

## Especificações de Handoff
- Cada agente deve receber seu envelope via `spawn_agent`.
- O agente deve responder usando o Envelope de Resposta.
- Em caso de erro, preencher `estado` com `erro` e incluir mensagem.

## Despacho Sequencial do Coach (6 despachos)
O Coach será despachado seis vezes, cada uma para uma etapa da entrevista simulada.

## Regras de Tratamento de Erros
- Se `estado` for `erro`, o Maestro deve registrar o erro e tentar nova tentativa ou informar o usuário.

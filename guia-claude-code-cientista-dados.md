# Configurando o Claude Code para ciência de dados (LangChain/LangGraph)

Guia para o Claude Code do notebook do trabalho, focado em criação e sustentação de agentes com LangChain/LangGraph em Python (pip).

Três peças compõem essa configuração:

1. **`CLAUDE.md`** — contexto do projeto (gerado com `/init`)
2. **`~/.claude/settings.json`** — regras de permissão (o que pode rodar sem perguntar, o que sempre pergunta, o que é bloqueado)
3. **`~/.claude/agents/*.md`** — subagents globais, disponíveis em qualquer repositório

Comportamento final que essa configuração entrega:
- Criação de arquivo novo → **sempre pergunta**
- Edição de arquivo existente (ajuste de código) → **roda sem perguntar**
- `git push`, criação de branch, commit, merge, rebase e qualquer outra escrita em git/GitHub → **bloqueado**, você continua no controle total dessa parte
- `git status`, `diff`, `log` → seguem liberados (são leitura, não escrita)

---

## Passo 1 — CLAUDE.md do projeto (via `/init`)

Dentro de cada repositório, rode:

```
/init
```

Isso escaneia o projeto e gera um `CLAUDE.md` inicial na raiz. Ele é carregado automaticamente no começo de toda sessão naquele repo.

Duas coisas importantes sobre o `/init`:

- O arquivo gerado é só um ponto de partida — vale revisar e **cortar** o que não for regra que você reexplicaria pra um colega novo toda vez (comandos de build, convenções, arquitetura). Procedimento longo de várias etapas ou regra que só vale pra uma parte do código, não entra aqui — vira skill ou regra específica depois.
- Se quiser preferências pessoais que não vão pro git do time, crie um `CLAUDE.local.md` na raiz do projeto (o próprio `/init`, quando você escolhe a opção "pessoal", já adiciona ele ao `.gitignore`).

Checklist do que vale a pena garantir que fique no `CLAUDE.md` de um projeto de agentes:

```markdown
# <nome do projeto>

## Visão geral
O que esse agente faz, em 2-3 linhas.

## Setup
- Ambiente: pip + requirements.txt (ou pyproject.toml)
- Comando de instalação: pip install -r requirements.txt
- Como rodar localmente: <comando>

## Testes
- Comando: pytest (ou o que for usado)
- Onde ficam os testes: <pasta>

## Arquitetura do agente
- Onde fica o grafo LangGraph (StateGraph, nodes, edges)
- Onde fica o schema de state (TypedDict/Pydantic)
- Checkpointer usado (MemorySaver, SqliteSaver, etc.) e por quê
- Onde ficam os prompts/templates

## Convenções de código
- Padrão de nomenclatura de nodes
- Uso de async (quando é obrigatório, quando não)
- Tratamento de erro esperado em chamadas de LLM/tool

## Fluxo de trabalho
- Eu (Vini) cuido de commits, branches e push manualmente — não faça isso.
```

A última seção não substitui o bloqueio técnico do Passo 2 (o `CLAUDE.md` é contexto, não é enforcement — quem realmente bloqueia a ação é a configuração de permissões), mas ajuda o Claude a nem tentar sugerir esse tipo de ação.

---

## Passo 2 — Permissões granulares (o núcleo do que você pediu)

Isso é a parte que reforça de verdade: mesmo que uma instrução em prompt ou CLAUDE.md diga pra não fazer algo, quem decide o que roda ou não é o Claude Code, via essas regras — não o modelo.

Crie (ou edite) o arquivo **`~/.claude/settings.json`** — esse fica no seu usuário, então vale pra qualquer repositório que você abrir no notebook, sem precisar repetir por projeto:

```json
{
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Edit"
    ],
    "ask": [
      "Write"
    ],
    "deny": [
      "Bash(git push *)",
      "Bash(git commit *)",
      "Bash(git add *)",
      "Bash(git branch *)",
      "Bash(git checkout -b *)",
      "Bash(git merge *)",
      "Bash(git rebase *)",
      "Bash(git reset *)",
      "Bash(git tag *)",
      "Bash(git remote *)",
      "Bash(git stash push *)",
      "Bash(git cherry-pick *)",
      "Bash(git revert *)",
      "Bash(git pull *)",
      "Bash(gh *)"
    ]
  }
}
```

O que cada parte faz:

- **`"defaultMode": "default"`** — força o modo Manual como padrão da sessão. Isso importa porque em planos Pro/Max/Team o Claude Code às vezes já inicia em modo "auto" (um classificador aprova ações sozinho). As regras explícitas abaixo valem em qualquer modo, mas deixar o padrão em Manual garante que qualquer coisa **não coberta** por uma regra explícita também pare pra te perguntar, em vez de ser decidida por um classificador.
- **`"allow": ["Edit"]`** — libera a tool `Edit` (ajuste em arquivo já existente) sem pedir aprovação. Isso é o "autonomia nos ajustes de código" que você quer.
- **`"ask": ["Write"]`** — a tool `Write` (criação de arquivo novo) sempre pergunta. Isso é o "eu aprovo a criação de arquivos".
- **`"deny": [...]`** — qualquer comando de escrita em git (push, commit, add, branch, checkout -b, merge, rebase, reset, tag, remote, stash, cherry-pick, revert, pull) e qualquer uso do `gh` (CLI do GitHub) fica **bloqueado de verdade** — o Claude Code nem tenta, nem pergunta. `git status`, `git diff`, `git log` continuam liberados por serem só leitura (o Claude Code já reconhece esses como seguros automaticamente).

Um detalhe pra guardar: como o `deny` é avaliado antes de qualquer `allow`, essas regras de git valem mesmo que outra configuração (de projeto, por exemplo) tente liberar algo — regra de negação de qualquer nível sempre vence.

**Nuance a saber:** redirecionamento de shell (tipo `algum-comando > arquivo.txt`) é checado contra a regra de `Edit`, não de `Write` — então, na teoria, um comando Bash poderia criar um arquivo novo via `>` sem passar pela pergunta de `Write`. Isso é raro na prática (normalmente o Claude usa a tool `Write` mesmo), mas se quiser fechar esse buraco 100%, dá pra adicionar depois um hook `PreToolUse` que valida redirecionamentos — não é essencial pra começar.

### Opcional: reduzir perguntas repetitivas em comandos seguros

Se depois de um tempo de uso você perceber que fica aprovando sempre os mesmos comandos inofensivos, dá pra ir adicionando ao `allow`, por exemplo:

```json
"allow": [
  "Edit",
  "Bash(pytest *)",
  "Bash(python -m pytest *)",
  "Bash(pip list *)",
  "Bash(pip show *)"
]
```

Regra prática recomendada pela própria documentação: adiciona a regra na **segunda** vez que o prompt te incomodar, não na primeira — a primeira vez é só um sinal, a segunda já é um padrão.

---

## Passo 3 — Subagents globais (`~/.claude/agents/`)

Como você pediu subagents **globais** (não por repo), eles vão em `~/.claude/agents/` — ficam disponíveis em qualquer projeto do notebook automaticamente. Cada um é um arquivo `.md` com frontmatter YAML + prompt de sistema.

Você pode pedir pro próprio Claude Code escrever esses arquivos por você (ex: "crie um subagent pessoal em ~/.claude/agents/ que faça X"), ou colar o conteúdo abaixo diretamente. Note que o comando `/agents` não abre mais um wizard interativo (mudou em versões recentes) — a forma atual é pedir pro Claude criar/editar ou editar o arquivo você mesmo.

**Importante:** se a pasta `~/.claude/agents/` ainda não existir, depois de criar o primeiro arquivo nela você precisa **reiniciar a sessão do Claude Code** pra ele detectar a pasta nova.

### 3.1 `langgraph-reviewer.md` — revisor de arquitetura de grafos

```markdown
---
name: langgraph-reviewer
description: Revisa a estrutura de grafos LangGraph e cadeias LangChain — nodes, edges, state schema, checkpointer, tratamento de erro. Use proativamente depois de criar ou alterar um grafo de agente.
tools: Read, Grep, Glob, Bash
model: inherit
---

Você é um revisor sênior especializado em arquiteturas de agentes construídas com LangChain e LangGraph em Python.

Ao ser invocado:
1. Rode `git diff` (somente leitura) para ver o que mudou, se estiver dentro de um repositório git.
2. Localize a definição do grafo (StateGraph, nodes, edges, edges condicionais) e o schema de state (TypedDict / modelo Pydantic).
3. Revise, nesta ordem: corretude do schema de state, pureza/efeitos colaterais das funções de node, lógica de roteamento das edges (edges condicionais, limites de recursão), configuração de checkpointer/memória, tratamento de erro e retries em chamadas de LLM/tool, configuração de streaming (se usado).

Checklist específico de LangGraph:
- O schema de state tem tipos claros e campos obrigatórios vs. opcionais bem definidos
- Os nodes não engolem silenciosamente exceptions vindas de LLM ou de tool calls
- As edges condicionais cobrem todos os valores de roteamento possíveis (sem fallthrough silencioso)
- Limites de recursão/passos estão definidos em grafos com ciclos
- O checkpointer (MemorySaver, SqliteSaver, etc.) é adequado para o uso (dev vs. produção)
- Chamadas de tool têm timeout e retry/backoff onde faz sentido
- Nenhum segredo ou chave de API hardcoded no código dos nodes
- Prompts usados nos nodes estão externalizados/versionados, não espalhados como strings ad-hoc pelo código

Checklist geral de Python/LangChain:
- Type hints nas funções públicas
- Uso de async consistente com o resto do pipeline
- Nenhuma chamada bloqueante (I/O síncrono) dentro de nodes assíncronos
- Versões de dependências fixadas de forma apropriada

Apresente o feedback agrupado como:
- Crítico (quebra em produção ou causa comportamento incorreto)
- Atenção (deveria corrigir)
- Sugestão (melhoria opcional)

Seja específico: aponte o arquivo, o nome do node/edge, e mostre um trecho concreto de antes/depois para cada problema.
```

### 3.2 `python-debugger.md` — depuração de erros e stack traces

```markdown
---
name: python-debugger
description: Especialista em depuração de exceptions, stack traces e comportamento inesperado em pipelines Python de agentes (LangChain/LangGraph). Use proativamente sempre que houver um erro, teste falhando ou stack trace.
tools: Read, Edit, Bash, Grep, Glob
model: inherit
---

Você é um especialista em depuração de Python, focado em pipelines de agentes construídos com LangChain e LangGraph.

Ao ser invocado:
1. Capture a mensagem de erro completa e o stack trace.
2. Identifique os passos exatos de reprodução (qual node, qual input, qual tool call).
3. Isole a falha na menor unidade possível (um único node, uma única tool call, um único template de prompt).
4. Formule uma hipótese sobre a causa raiz antes de mexer em qualquer código.
5. Implemente a correção mínima que resolve a causa raiz, não só o sintoma.
6. Verifique a correção (rode novamente o teste que falhava ou reproduza o cenário original).

Preste atenção especial a modos de falha específicos de LangChain/LangGraph:
- Argumentos de tool call malformados (schema da tool não bate com o que o LLM retornou)
- Erros de chave no state (chave ausente, tipo errado) entre nodes
- Truncamento silencioso por limite de contexto
- Descompasso entre async/sync (chamada bloqueante dentro de um grafo assíncrono, ou o contrário)
- Loops de retry que mascaram o erro real

Para cada problema, reporte:
- Explicação da causa raiz
- Evidência que sustenta o diagnóstico (linhas de log, trecho do stack trace)
- A correção exata aplicada
- Como você verificou que funcionou
- Uma sugestão de uma linha para evitar recorrência (ex: uma validação, um teste, um type hint)

Você só roda comandos git de leitura (status, diff, log) para entender o contexto.
```

### 3.3 `test-runner.md` — roda testes e resume o que importa

```markdown
---
name: test-runner
description: Executa a suíte de testes pytest e reporta apenas falhas com mensagens de erro concisas, mantendo a saída verbosa fora da conversa principal. Use sempre que for necessário rodar testes.
tools: Bash, Read, Grep, Glob
model: haiku
---

Você roda a suíte de testes pytest do projeto e reporta só o que importa.

Ao ser invocado:
1. Detecte como os testes são rodados nesse projeto (pytest, `python -m pytest`, um alvo de Makefile, tox, etc.), checando arquivos de configuração (pytest.ini, pyproject.toml, tox.ini, Makefile) antes de assumir um comando.
2. Rode a suíte de testes (ou o subconjunto relevante ao que mudou, se pedido).
3. Analise a saída e reporte só:
   - Número de testes rodados / que passaram / que falharam / pulados
   - Para cada falha: nome do teste, arquivo:linha, a mensagem de assertion ou exception, e até 5 linhas do trecho mais relevante do traceback
4. Não cole a saída bruta completa do pytest na resposta — resuma.

Você nunca modifica código. Se uma falha parecer precisar de correção, descreva o que provavelmente está errado e sugira delegar ao subagent python-debugger.
```

### 3.4 `prompt-auditor.md` — revisão de prompts e templates

```markdown
---
name: prompt-auditor
description: Revisa prompts e templates usados em cadeias LangChain/LangGraph — clareza, risco de prompt injection, tamanho/custo de tokens, consistência de formato de saída. Use ao criar ou alterar prompts de sistema, templates ou few-shot examples.
tools: Read, Grep, Glob
model: inherit
---

Você é um revisor de prompt engineering para pipelines de agentes LLM em produção.

Ao ser invocado, localize os templates de prompt, prompts de sistema e exemplos few-shot relevantes ao pedido (procure por PromptTemplate, ChatPromptTemplate, SystemMessage, ou strings de prompt simples).

Revise cada prompt quanto a:
- Clareza e ambiguidade (instruções que podem ser mal-interpretadas pelo modelo)
- Risco de prompt injection via input do usuário (o prompt separa claramente instrução de dado do usuário?)
- Consistência de formato de saída (o prompt pede um formato — JSON, XML, etc. — de forma que o parser downstream realmente espera?)
- Tamanho e custo (prompts desnecessariamente longos, repetição, exemplos redundantes)
- Versionamento (o prompt está centralizado/reutilizável ou duplicado em vários lugares do código?)

Apresente os achados organizados por prompt/arquivo, cada um com:
- O problema
- Por que importa (impacto real: custo, confiabilidade, segurança)
- Uma versão sugerida do trecho problemático

Você é somente leitura: nunca edita arquivos. Se as mudanças forem aprovadas, entregue o texto exato para o usuário (ou outro subagent) aplicar.
```

### 3.5 `dependency-doctor.md` — auditoria de dependências pip

```markdown
---
name: dependency-doctor
description: Audita dependências pip do projeto (requirements.txt / pyproject.toml) em busca de versões desatualizadas, conflitos e vulnerabilidades conhecidas. Use periodicamente ou antes de atualizar dependências.
tools: Bash, Read, Grep, Glob
model: inherit
---

Você é um auditor de dependências para projetos Python gerenciados com pip.

Ao ser invocado:
1. Localize os arquivos de dependências (requirements.txt, requirements/*.txt, pyproject.toml, setup.cfg).
2. Use apenas comandos de inspeção somente leitura: `pip list --outdated`, `pip show <pacote>`, `pip check` e — se disponível no ambiente — `pip-audit` para CVEs conhecidas. Nunca rode `pip install`, `pip uninstall`, nem edite nenhum arquivo de dependências.
3. Cruze as versões fixadas com o que está instalado e com a versão mais recente disponível.

Reporte:
- Pacotes desatualizados (versão atual vs. mais recente), destacando os relevantes para o stack (langchain, langgraph, langchain-core, langsmith, providers de LLM)
- Vulnerabilidades conhecidas encontradas (se pip-audit estiver disponível)
- Conflitos de versão (`pip check`)
- Dependências fixadas (`==`) que poderiam ser mais flexíveis (`>=`), e vice-versa quando isso for um risco

Você nunca modifica arquivos de requirements nem instala/desinstala nada — só reporta. Entregue as atualizações de versão recomendadas para o usuário aplicar ou aprovar.
```

### 3.6 `docs-writer.md` — mantém documentação atualizada

```markdown
---
name: docs-writer
description: Mantém README, docstrings e documentação de arquitetura atualizados após mudanças no código. Restrito a arquivos de documentação — nunca toca em lógica de negócio. Use depois de finalizar uma feature ou alterar a arquitetura de um agente.
tools: Read, Edit, Grep, Glob, Bash
model: inherit
---

Você mantém a documentação de projetos Python de agentes (LangChain/LangGraph).

Escopo: você só edita README.md, arquivos em docs/**, CLAUDE.md e docstrings dentro de arquivos Python já existentes. Você nunca mexe em lógica de negócio, e nunca cria arquivos novos fora de documentação — sempre pergunte ao usuário antes se parecer necessário um novo arquivo de documentação.

Ao ser invocado:
1. Rode `git diff` (somente leitura) para ver o que mudou desde o último commit.
2. Atualize a documentação relevante: seções do README, notas de arquitetura, docstrings de funções/classes/nodes novos ou alterados.
3. Mantenha o tom consistente com a documentação existente.
4. Para agentes LangGraph, mantenha atualizada a descrição do grafo: nodes, o que cada um faz, schema de state, pontos de entrada/saída.

Nunca invente comportamento que não está no código. Se algo não estiver claro, pergunte em vez de supor.
```

### Como usar

Delegação automática acontece quando a descrição bate com o que você pediu, mas dá pra forçar:

```
Use o subagent langgraph-reviewer pra revisar esse grafo que acabei de criar
Use o test-runner e me mostra só o que falhou
```

---

## Passo 4 — Comandos úteis pra manter tudo afinado

- `/permissions` — abre o painel com todas as regras ativas e de onde vêm (dá pra adicionar/remover sem sair da sessão)
- `/memory` — lista todos os CLAUDE.md/CLAUDE.local.md carregados na sessão atual
- `Ctrl+E` num prompt de aprovação de Bash — mostra uma explicação do comando (o que faz, por que o Claude está rodando, risco Baixo/Médio/Alto) antes de você decidir
- Durante a conversa, começar uma linha com `#` pergunta em qual CLAUDE.md salvar aquela regra na hora (ex: `# sempre use type hints`)

## Checklist final

- [ ] `~/.claude/settings.json` criado com o bloco de `permissions` acima
- [ ] `~/.claude/agents/` criado com os 6 subagents
- [ ] Reiniciar a sessão do Claude Code (necessário se a pasta `agents/` era nova)
- [ ] Rodar `/init` no primeiro projeto e revisar/enxugar o `CLAUDE.md` gerado
- [ ] Testar: pedir um ajuste em código existente (deve rodar direto) e pedir a criação de um arquivo novo (deve perguntar)
- [ ] Testar: pedir pra fazer um `git commit` ou `git push` (deve ser bloqueado)

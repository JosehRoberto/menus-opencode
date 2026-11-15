# Planejamento estratégico — menu-reversa-opencode

> Documento de planejamento.
> Repositório alvo: `https://github.com/JosehRoberto/menu-reversa-opencode`
> Data: 2026-11-15
> Status: **Proposto** — aguardando aprovação.

## 1. Objetivo

Migrar o `menu-reversa-opencode` do modelo atual (scripts shell embutidos no README, interpretados pelo agente → resultado probabilístico) para um modelo **executável, multiplataforma e determinístico**, replicando o que foi feito em `menu-antares-opencode` e `menu-mira-opencode` v1.0.0: núcleo Python único + wrappers finos por sistema operacional, cobrindo Linux (Ubuntu e demais distros), macOS e Windows 10/11.

## 2. Diagnóstico do estado atual

| Item | Situação |
|---|---|
| Conteúdo do repo | Apenas `README.md` |
| Programa | Dois blocos Bash embutidos no README (instalação e sincronização) |
| Prefixos cobertos | `reversa-*` e `reversa` |
| Agente do frontmatter | `agent: general` |
| Leitura da `description` | `grep -m1 '^description:'` + `sed` — **não usa PyYAML** |
| `description: >-` (multi-linha) | Não ocorre hoje (0 de 77); o `grep`/`sed` **não** saberia tratar se passasse a ocorrer |
| Estilo das descrições | Escalares YAML simples ou com aspas simples/duplas (contendo `:`, `—`, `"`); PyYAML normaliza todos |
| `.gitignore` | **Não há gerenciamento** (não adiciona `.opencode/`) |
| Execução | Agente reinterpreta/traduz por SO |
| Interatividade | Sim (`read ... [y/N]`) → inviabiliza execução automatizada |
| Windows | Depende de Git Bash/WSL ou tradução livre |
| Determinismo | Nenhum (sem artefato fixo, sem pin de versão) |

## 3. Diferenças em relação a `menu-antares-opencode` e `menu-mira-opencode`

- Filtrar apenas `reversa-*` e `reversa`.
- **`agent: general`** no frontmatter (os outros usam `agent: build`).
- Corpo do comando: `Ative este comando para executar o agente \`X\` do Framework Reversa.` (texto diferente dos outros).
- Fallback da descrição: `Comando Reversa: <nome>` e header `# Comando \`X\` do Reversa`.
- Orfandade: apenas `reversa*.md` / `reversa.md`.
- Pré-requisito de origem: `npx reversa install` / `npx reversa update`.
- O README atual **não** declara pré-requisito de Python/PyYAML (não usa PyYAML).

## 4. Divergências a resolver (decisões)

1. **Adotar PyYAML + `json.dumps`** para a descrição (como Antares/Mira), substituindo `grep`/`sed`. Robusto para escalares com aspas e, se surgirem, `description: >-` multi-linha. **Validado:** para os 77 skills atuais a saída é **byte-idêntica** à atual (ver seção 5).
2. **Passar a gerenciar `.opencode/` no `.gitignore`** (idempotente), com flag `--keep-gitignore` para desativar — alinhando ao comportamento dos outros menus.
3. **Manter `agent: general`** no frontmatter gerado.

**Impacto:** nenhum churn nos **comandos** gerados (saída idêntica). A única mudança observável é a nova entrada `.opencode/` no `.gitignore`, que pode ser evitada com `--keep-gitignore`. O primeiro `sync` não deve propor atualização de comandos.

## 5. Arquitetura-alvo

| Arquivo | Papel |
|---|---|
| `menu-reversa.py` | Núcleo único, portátil, subcomandos `install` e `sync` |
| `bin/install.sh`, `bin/sync.sh` | Wrappers finos Linux/macOS (bash 3.2+) |
| `bin/install.ps1`, `bin/sync.ps1` | Wrappers finos Windows (PowerShell) |
| `tests/test_menu_reversa.py` | Testes (unittest) + determinismo |
| `README.md` | Documentação, prompts determinísticos, troubleshooting |
| `CHANGELOG.md`, `LICENSE`, `.gitignore` | Higiene e versionamento |

## 6. Reaproveitamento do trabalho dos outros menus

- Copiar `menu-mira.py` e ajustar: prefixos (`reversa-*`, `reversa`), `agent: general`, textos ("do Reversa"/"executar o agente ... do Framework Reversa"), fallback e glob de órfãos.
- Reutilizar a estrutura de wrappers e o padrão dos testes.
- **Fora de escopo:** núcleo compartilhado entre os repositórios `menu-*`.

## 7. Fases

### Fase 0 — Decisões e escopo
- Confirmar as decisões da seção 4.
- Confirmar flags: `--yes`, `--dry-run`, `--backup-dir`, `--keep-gitignore` (+ `--skills-root`/`--commands-dir`).

### Fase 1 — Núcleo `menu-reversa.py`
- Portar a lógica, filtrando `reversa-*`/`reversa`, com `agent: general` e textos do Reversa.
- Trocar `grep`/`sed` por **PyYAML + `json.dumps`**.
- Gerenciar `.opencode/` no `.gitignore` (idempotente) e backups; órfãos; não interativo.

### Fase 2 — Wrappers
- `bin/*.sh` (Linux/macOS) e `bin/*.ps1` (Windows), finos, com detecção de Python.

### Fase 3 — README
- Reescrita determinística por SO, com versão fixada; prompts de instalação/sincronização.
- Pré-requisitos de Python/PyYAML (novos) e `npx reversa install`/`npx reversa update`.
- Troubleshooting (PEP 668, macOS sem `python3`, ExecutionPolicy, encoding) e matriz de compatibilidade.
- Manter prompt legado apontando para o README histórico.

### Fase 4 — Testes
- Fixtures com `description: >-`, aspas, `+`, `&`; teste de determinismo (2x); órfãos, backups, `.gitignore`, `--dry-run`, `agent: general` no frontmatter.

### Fase 5 — Versionamento
- `CHANGELOG.md` (skill `changelog-convention`), tag `v1.0.0`, Release com `menu-reversa.py` anexado, pin de versão no README.

## 8. Matriz de compatibilidade (alvo)

| Plataforma | Wrapper | Requisitos | Observações |
|---|---|---|---|
| Ubuntu / Debian / Fedora / Mint | `bin/*.sh` | bash, python3, pyyaml | PEP 668 no pyyaml |
| macOS | `bin/*.sh` | python3 (CLT/Homebrew), pyyaml | bash 3.2; PEP 668 |
| Windows 10/11 | `bin/*.ps1` | Python 3, pyyaml | `-ExecutionPolicy Bypass`, encoding UTF-8 |

## 9. Checklist de determinismo

- [ ] Lógica só em arquivo versionado (não em prosa)
- [ ] Execução não interativa com `--yes`
- [ ] Detecção de interpretador e dependências com erro explícito
- [ ] Versão fixada (tag/commit) no prompt
- [ ] Saída reproduzível e ordenada
- [ ] Teste de reprodutibilidade rodando 2x

## 10. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| `description: >-` incorreta na versão antiga gerar churn | Backup automático + nota de migração |
| Mudança para PyYAML alterar bytes gerados | Testes de determinismo + comparação antes de sobrescrever |
| Passar a mexer no `.gitignore` de projetos existentes | Flag `--keep-gitignore` |
| Divergência entre os núcleos `menu-*` | Manter lógica idêntica; só variar prefixo/textos/agente |
| Shell do OpenCode no Windows sem Bash | Wrapper PowerShell explícito |
| Quebra de compatibilidade com usuários atuais | Manter prompt legado por 1 versão |

## 11. Critérios de conclusão (DoD)

- `menu-reversa.py install|sync` roda igual em Linux, macOS e Windows.
- Execução não interativa sem travar.
- README com passos por SO e versão fixada.
- Testes de determinismo passando; lint limpo.
- Release `v1.0.0` tagueada com script anexado.

## 12. Próximo passo sugerido

Aprovar este plano e iniciar pela **Fase 1** (núcleo `menu-reversa.py`), reaproveitando o código do `menu-mira-opencode` v1.0.0, com as decisões da seção 4.

## 13. Validação contra a instalação real (`teste-reversa-inicial`)

Análise feita sobre uma instalação real do Reversa em `/home/roberto/projetos/teste-reversa-inicial/.agents/skills`:

| Verificação | Resultado |
|---|---|
| Total de skills (`*/SKILL.md`) | 78 |
| Skills que casam com `reversa*` | **77** (`reversa` + 76 `reversa-*`) |
| Skills fora do prefixo (ignoradas) | 1 (`impeccable`) |
| Skills com `description: >-` / `\|` (multi-linha) | **0** |
| Descrições com `: ` no valor | 33 (protegidas por aspas simples/duplas no YAML) |
| Descrições com aspas duplas | 18 (7 iniciando com aspas) |
| Descrições com backslash | 0 |
| Skills sem `description` | 0 |
| `yaml.safe_load` do frontmatter | **77/77 OK** (sem erros) |
| Saída do núcleo proposto (PyYAML + `json.dumps`) vs. script atual (`grep`/`sed`) | **77/77 byte-idênticas** |

Conclusões:
- A migração para **PyYAML + `json.dumps` é segura** e **não altera** os comandos gerados para os skills atuais (nenhum churn).
- PyYAML é a escolha robusta: lida corretamente com escalares com aspas e com futuras descrições multi-linha.
- `impeccable` (sem prefixo `reversa`) é corretamente ignorado pelo filtro.


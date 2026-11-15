# Planejamento estratégico — menu-mira-opencode

> Documento de planejamento.
> Repositório alvo: `https://github.com/JosehRoberto/menu-mira-opencode`
> Data: 2026-11-15
> Status: **Proposto** — aguardando aprovação.

## 1. Objetivo

Migrar o `menu-mira-opencode` do modelo atual (scripts shell embutidos no README, interpretados pelo agente → resultado probabilístico) para um modelo **executável, multiplataforma e determinístico**, replicando o que foi feito no `menu-antares-opencode` v1.0.0: núcleo Python único + wrappers finos por sistema operacional, cobrindo Linux (Ubuntu e demais distros), macOS e Windows 10/11.

## 2. Diagnóstico do estado atual

| Item | Situação |
|---|---|
| Conteúdo do repo | Apenas `README.md` |
| Programa | Dois blocos Bash embutidos no README (instalação e sincronização) |
| Prefixos cobertos | Apenas `mira-*` e `mira` (sem Antares) |
| Geração da descrição | `desc.replace('"', '\\"')` — escaping frágil (não trata `\`, quebras, etc.) |
| Frontmatter gerado | Contém linha em branco entre `agent: build` e `---` (formato divergente do Antares) |
| Execução | Agente reinterpreta/traduz por SO |
| Interatividade | Sim (`read ... [y/N]`) → inviabiliza execução automatizada |
| Windows | Depende de Git Bash/WSL ou tradução livre |
| Determinismo | Nenhum (sem artefato fixo, sem pin de versão) |

## 3. Diferenças em relação ao `menu-antares-opencode`

O `menu-antares-opencode` v1.0.0 já cobre `mira-*` **junto** com `antares-*` (o Antares traz o Mira). O `menu-mira-opencode` é a variante **Mira-only**, para quem instala só o Mira. Por isso:

- Filtrar apenas `mira-*` e `mira`.
- Textos do núcleo: `# Comando \`X\` do Mira` e fallback `Comando Mira: <nome>` (sem "Antares").
- Orfandade: apenas comandos `mira*.md` / `mira.md`.
- Pré-requisito de origem do Mira: `npx mira install` / `npx mira update`.

## 4. Divergências de formato a resolver (decisão)

O gerador do Antares e o do Mira produzem `.md` **diferentes**. Ao portar, padronizar para o formato robusto do Antares:

- Serializar a descrição com **`json.dumps`** (substitui o `desc.replace` frágil).
- Remover a **linha em branco** entre `agent: build` e o `---` de fechamento.

**Impacto:** os arquivos já gerados mudam de bytes, então o primeiro `sync` vai propor atualização (com backup). Documentar e mitigar.

## 5. Arquitetura-alvo

Mesma do Antares:

| Arquivo | Papel |
|---|---|
| `menu-mira.py` | Núcleo único, portátil, subcomandos `install` e `sync` |
| `bin/install.sh`, `bin/sync.sh` | Wrappers finos Linux/macOS (bash 3.2+) |
| `bin/install.ps1`, `bin/sync.ps1` | Wrappers finos Windows (PowerShell) |
| `tests/test_menu_mira.py` | Testes (unittest) + determinismo |
| `README.md` | Documentação, prompts determinísticos, troubleshooting |
| `CHANGELOG.md`, `LICENSE`, `.gitignore` | Higiene e versionamento |

## 6. Reaproveitamento do trabalho do Antares

- Copiar `menu-antares.py` e ajustar: prefixos (`mira-*`, `mira`), textos ("do Mira", fallback) e glob de órfãos.
- Reutilizar a estrutura de wrappers e o padrão dos testes.
- **Decisão de escopo:** manter dois núcleos independentes (um por repo) nesta rodada. Extrair um núcleo compartilhado fica **fora de escopo** (avaliar depois, se os repos `menu-*` convergirem).

## 7. Fases

### Fase 0 — Decisões e escopo
- Confirmar as flags: `--yes`, `--dry-run`, `--backup-dir`, `--keep-gitignore` (+ `--skills-root`/`--commands-dir`).
- Confirmar a padronização de formato (seção 4).
- Congelar escopo: **fora** = núcleo compartilhado, GUI, pacotes de sistema.

### Fase 1 — Núcleo `menu-mira.py`
- Portar a lógica, filtrando `mira-*`/`mira`, textos do Mira, `.gitignore` idempotente, backups, órfãos.
- Serialização com `json.dumps`; frontmatter padronizado.
- Entrada robusta (PyYAML ausente, PEP 668) e execução não interativa.

### Fase 2 — Wrappers
- `bin/*.sh` (Linux/macOS) e `bin/*.ps1` (Windows), finos, com detecção de Python.

### Fase 3 — README
- Reescrita determinística por SO, com versão fixada; prompt de instalação/sincronização.
- Pré-requisito `npx mira install`/`npx mira update`.
- Troubleshooting (PEP 668, macOS sem `python3`, ExecutionPolicy, encoding) e matriz de compatibilidade.
- Manter prompt legado apontando para o README histórico.

### Fase 4 — Testes
- Fixtures com `description: >-`, aspas, `+`, `&`, emoji; teste de determinismo (rodar 2x); órfãos, backups, `.gitignore`, `--dry-run`.

### Fase 5 — Versionamento
- `CHANGELOG.md` (skill `changelog-convention`), tag `v1.0.0`, Release com `menu-mira.py` anexado, pin de versão no README.

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
| Mudança de formato do `.md` gerar churn | Backup automático + nota de migração no README/CHANGELOG |
| Divergência entre os núcleos `menu-*` | Manter lógica idêntica; só variar prefixo/textos |
| Shell do OpenCode no Windows sem Bash | Wrapper PowerShell explícito |
| PEP 668 bloqueando pyyaml | venv/`--user`/pacote do sistema |
| Quebra de compatibilidade com usuários atuais | Manter prompt legado por 1 versão |

## 11. Critérios de conclusão (DoD)

- `menu-mira.py install|sync` roda igual em Linux, macOS e Windows.
- Execução não interativa sem travar.
- README com passos por SO e versão fixada.
- Testes de determinismo passando; lint limpo.
- Release `v1.0.0` tagueada com script anexado.

## 12. Próximo passo sugerido

Aprovar este plano e iniciar pela **Fase 1** (núcleo `menu-mira.py`), reaproveitando o código do `menu-antares-opencode` v1.0.0.

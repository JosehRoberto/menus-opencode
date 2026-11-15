# Planejamento estratégico — menu-antares-opencode

> Documento de planejamento. Nenhuma implementação foi iniciada.
> Repositório alvo: `https://github.com/JosehRoberto/menu-antares-opencode`
> Data: 2026-11-15

## 1. Objetivo

Sair do modelo atual (prosa interpretada pelo agente → resultado probabilístico) para um modelo **executável, multiplataforma e determinístico**, cobrindo Linux (Ubuntu e demais), macOS e Windows 10/11.

## 2. Diagnóstico do estado atual

| Item | Situação |
|---|---|
| Conteúdo do repo | Apenas `README.md` |
| Programa | Embutido como blocos Bash no README |
| Execução | Agente reinterpreta/traduz por SO |
| Interatividade | Sim (`read ... [y/N]`) → inviabiliza execução automatizada |
| Windows | Depende de Git Bash/WSL ou tradução livre |
| Determinismo | Nenhum (sem artefato fixo, sem pin de versão) |

## 3. Arquitetura-alvo

**Núcleo Python único + wrappers finos por plataforma + README como documentação.**

| Arquivo | Papel |
|---|---|
| `menu-antares.py` | Fonte única da verdade. Subcomandos `install` e `sync`. Cobre `antares-*` e `mira-*` no mesmo passe |
| `bin/install.sh` / `bin/sync.sh` | Wrappers Bash (Linux + macOS) que só chamam o Python |
| `bin/install.ps1` / `bin/sync.ps1` | Wrappers PowerShell (Windows) |
| `README.md` | Uso, pré-requisitos por SO, prompts determinísticos, troubleshooting |
| `tests/` + `tests/fixtures/` | Provam saída estável (determinismo) |
| `CHANGELOG.md`, `LICENSE`, `.gitignore` | Higiene do repositório |

## 4. Fases

### Fase 0 — Decisões e escopo (sem código)
- Definir flags: `--yes` (não interativo), `--dry-run`, `--backup-dir`, `--keep-gitignore`.
- Definir comportamento não interativo padrão (backup automático; nunca remover sem `--yes` explícito).
- Congelar escopo: **fora** deste ciclo = GUI, publicação em pacotes (apt/npm), suporte a Alpine minimalist.

### Fase 1 — Núcleo `menu-antares.py`
- Extrair 100% da lógica do README para Python (PyYAML, `json.dumps`, `.gitignore` idempotente, backups com timestamp, detecção de órfãos).
- **Portabilidade:** `mktemp` com template, `os.path`/`pathlib`, UTF-8 explícito, separadores normalizados.
- **Entrada robusta:** verificar Python, verificar PyYAML com mensagem clara e fallback documentado (PEP 668 → venv / `--user` / pacote do sistema).
- **Determinismo:** sem prompts bloqueantes; saída ordenada; resumo final com contagens.

### Fase 2 — Wrappers por plataforma
- `bin/*.sh` (bash 3.2-compatible: sem recursos bash 4+) para Linux e macOS.
- `bin/*.ps1` para Windows, com nota de `-ExecutionPolicy Bypass`.
- Wrappers só detectam o launcher Python (`python3`/`python`/`py -3`) e chamam o núcleo.

### Fase 3 — Reescrita do README
- Estrutura por SO com passos **branch-e-verificação**, sem pedir tradução ao agente.
- Prompt de instalação passa a **clonar/baixar a versão fixada e executar** (`install`), não "seguir o procedimento".
- Seções: pré-requisitos por SO, **matriz de compatibilidade**, troubleshooting (PEP 668, macOS sem `python3`, execution policy, encoding).
- Reescrever a linha que hoje promete tradução automática.
- Manter compatibilidade: preservar o prompt antigo por uma versão, marcado como legado.

### Fase 4 — Testes e validação
- Fixtures de `.agents/skills` (com `description: >-`, aspas, `+`, `&`, emoji).
- **Teste de determinismo:** rodar 2x sobre a mesma fixture e comparar byte a byte.
- Testes de órfãos, backups, `.gitignore` idempotente, `--dry-run`.
- Lint do Python.

### Fase 5 — Distribuição e versionamento
- Tag/Release com o script anexado; **pin de versão** no README (evita alvo móvel).
- `CHANGELOG.md` versionado (seguir a skill `changelog-convention`).

## 5. Matriz de compatibilidade (alvo)

| Plataforma | Wrapper | Requisitos | Observações |
|---|---|---|---|
| Ubuntu/Debian/Fedora/Mint | `bin/*.sh` | bash, python3, pyyaml | PEP 668 no pyyaml |
| macOS | `bin/*.sh` | python3 (CLT/Homebrew), pyyaml | bash 3.2; PEP 668 |
| Windows 10/11 | `bin/*.ps1` | Python, pyyaml | ExecutionPolicy, encoding UTF-8 |

## 6. Checklist de determinismo

- [ ] Lógica só em arquivo versionado (não em prosa)
- [ ] Execução não interativa com `--yes`
- [ ] Detecção de interpretador e dependências com erro explícito
- [ ] Versão fixada (tag/commit) no prompt
- [ ] Saída reproduzível e ordenada
- [ ] Teste de reprodutibilidade rodando 2x

## 7. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Shell do OpenCode no Windows sem Bash | Wrapper PowerShell explícito |
| PEP 668 bloqueando pyyaml | venv/`--user`/pacote do sistema documentados |
| Divergência entre wrappers | Núcleo único em Python; wrappers triviais |
| Comportamento do LLM (camada 2) | Passos determinísticos + checkpoint de verificação |
| Quebra de compatibilidade com usuários atuais | Manter prompt legado por 1 versão |

## 8. Critérios de conclusão (DoD)

- `menu-antares.py install|sync` roda igual em Linux, macOS e Windows.
- Execução não interativa sem travar.
- README com passos por SO e versão fixada.
- Teste de determinismo passando; lint limpo.
- Release tagueada com script anexado.

## 9. Próximo passo sugerido

Aprovar este plano e iniciar pela **Fase 1** (núcleo Python), que concentra o ganho de determinismo.

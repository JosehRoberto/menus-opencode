# Planejamento estratégico — menu-reversa-openclaude

> Documento de planejamento.
> Repositório alvo: `https://github.com/JosehRoberto/menu-reversa-openclaude`
> Data: 2026-11-15
> Status: **Implementado** — v1.0.0 publicada e testada em 2026-11-15.

## 1. Objetivo

Migrar o `menu-reversa-openclaude` do modelo atual (scripts shell embutidos no README, interpretados pelo agente → resultado probabilístico) para um modelo **executável, multiplataforma e determinístico**, no mesmo padrão dos menus `menu-antares-opencode`, `menu-mira-opencode` e `menu-reversa-opencode` v1.0.0.

**Atenção:** este repositório é **conceitualmente diferente** dos outros — não gera arquivos `.md` de comando; ele cria **symlinks** para que o **OpenClaude** descubra os skills do Reversa. O núcleo Python precisa refletir essa operação.

## 2. Diagnóstico do estado atual

| Item | Situação |
|---|---|
| Conteúdo do repo | Apenas `README.md` |
| Programa | Dois blocos Bash embutidos no README (instalação e sincronização) |
| Alvo | **OpenClaude** (`@gitlawb/openclaude`), não OpenCode |
| Operação | Cria **symlinks** `.openclaude/skills/<name>` → `../../.agents/skills/<name>` |
| Geração de arquivos | **Nenhuma** (o `SKILL.md` original já serve; sem frontmatter sintético) |
| Dependência PyYAML | **Não usa** (não há parsing de frontmatter) |
| Prefixos cobertos | `reversa-*` e `reversa` |
| Backup | Copia `.openclaude/skills` para `skills.bak.<ts>` na 1ª execução |
| Órfãos | Links cuja fonte em `.agents/skills/` sumiu |
| Interatividade | Sim (`read -r -p`) → inviabiliza execução automatizada |
| Windows | `ln -s` não existe nativamente; README sugere fallback de cópia |
| Determinismo | Nenhum (sem artefato fixo, sem pin de versão) |

## 3. Diferenças em relação aos outros menus

| Aspecto | Menus OpenCode (`antares`/`mira`/`reversa`) | `reversa-openclaude` |
|---|---|---|
| Ferramenta-alvo | OpenCode | **OpenClaude** |
| Destino | `.opencode/commands/*.md` | `.openclaude/skills/*` (symlinks) |
| Operação | Gera `.md` com frontmatter | Cria/atualiza symlinks |
| PyYAML | Obrigatório | **Desnecessário** |
| `agent:` / `description` | Sim | Não |
| Windows | Wrappers `.ps1` | Symlink requer privilégio → fallback de cópia |
| Escopo do usuário | — | Variante `~/.openclaude/skills` |

## 4. Decisões a resolver

1. **Núcleo Python `menu-reversa-openclaude.py`** com subcomandos `install` e `sync`, reproduzindo a criação de symlinks, backup e detecção de órfãos.
2. **Fallback de cópia (`--copy`)**: em vez de symlink, copiar a pasta (`cp -a` / recursivo). Necessário no Windows sem *Developer Mode*/admin e em FS sem symlink. Comportamento padrão: tentar symlink e, se falhar, orientar/derivar para cópia (configurável).
3. **Variante global (`--global`)**: criar links em `~/.openclaude/skills` a partir de uma fonte central. Decidir se entra nesta versão (sugestão: incluir como flag opcional).
4. **Sem PyYAML** nos pré-requisitos (diferente dos outros menus).
5. **Nome do script**: `menu-reversa-openclaude.py`, mantendo consistência com o padrão `menu-<nome>.py`.

## 5. Arquitetura-alvo

| Arquivo | Papel |
|---|---|
| `menu-reversa-openclaude.py` | Núcleo único, portátil, subcomandos `install` e `sync` |
| `bin/install.sh`, `bin/sync.sh` | Wrappers finos Linux/macOS (bash 3.2+) |
| `bin/install.ps1`, `bin/sync.ps1` | Wrappers finos Windows (PowerShell) |
| `tests/test_menu_reversa_openclaude.py` | Testes (unittest) — symlink/cópia, órfãos, backup, determinismo |
| `README.md` | Documentação, prompts determinísticos, troubleshooting |
| `CHANGELOG.md`, `LICENSE`, `.gitignore` | Higiene e versionamento |

## 6. Fases

### Fase 0 — Decisões e escopo
- Confirmar as decisões da seção 4 (fallback `--copy`, `--global`).
- Confirmar flags: `--yes`, `--dry-run`, `--copy`, `--global`, `--skills-root`, `--oc-skills-dir`, `--backup-dir` (para o backup da 1ª execução).

### Fase 1 — Núcleo `menu-reversa-openclaude.py`
- `install`: criar `.openclaude/skills`, backup da pasta se não vazia, criar symlinks para cada `reversa*` (pular link válido; confirmar conflito).
- `sync`: criar/atualizar symlinks, detectar **órfãos** (links cujo alvo sumiu) e remover com confirmação.
- Portabilidade: `os.symlink` com tratamento de erro; fallback de cópia; caminhos relativos determinísticos (`../../.agents/skills/<name>`).
- Não interativo com `--yes`.

### Fase 2 — Wrappers
- `bin/*.sh` (Linux/macOS) e `bin/*.ps1` (Windows), finos, com detecção de Python.

### Fase 3 — README
- Reescrita determinística por SO, com versão fixada; preservar a seção "Por que funciona" (internos do OpenClaude).
- Pré-requisitos: Python 3 (**sem PyYAML**), OpenClaude, Reversa.
- Troubleshooting: symlink no Windows (Developer Mode / admin / `--copy`), FS sem symlink.
- Manter prompt legado apontando para o README histórico.

### Fase 4 — Testes
- Symlink criado/atualizado/ignorado; conflito com arquivo/dir real; órfão removido; backup da 1ª execução; modo `--copy`; `--dry-run`; não interativo; variante `--global` (usando `HOME`/`USERPROFILE` temporário); determinismo.

### Fase 5 — Versionamento
- `CHANGELOG.md` (skill `changelog-convention`), tag `v1.0.0`, Release com `menu-reversa-openclaude.py` anexado, pin de versão no README.

## 7. Matriz de compatibilidade (alvo)

| Plataforma | Wrapper | Requisitos | Symlink |
|---|---|---|---|
| Ubuntu / Debian / Fedora / Mint | `bin/*.sh` | bash, python3 | Nativo |
| macOS | `bin/*.sh` | python3 | Nativo |
| Windows 10/11 | `bin/*.ps1` | Python 3 | Requer Developer Mode/admin; senão `--copy` |

## 8. Checklist de determinismo

- [ ] Lógica só em arquivo versionado (não em prosa)
- [ ] Execução não interativa com `--yes`
- [ ] Detecção de Python com erro explícito
- [ ] Versão fixada (tag/commit) no prompt
- [ ] Saída reproduzível e ordenada
- [ ] Teste de reprodutibilidade rodando 2x

## 9. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Symlink indisponível no Windows | Flag `--copy` (fallback) + detecção e mensagem clara |
| `ln -s` relativo divergir entre ambientes | Usar caminho relativo fixo `../../.agents/skills/<name>` e validar alvo |
| Sobrescrever links/arquivos do usuário | Backup + confirmação (ou `--yes` explícito) |
| `npx reversa update` alterar skills | `sync` recalibra links e remove órfãos |
| Shell do OpenCode no Windows sem Bash | Wrapper PowerShell explícito |
| Quebra de compatibilidade com usuários atuais | Manter prompt legado por 1 versão |

## 10. Critérios de conclusão (DoD)

- `menu-reversa-openclaude.py install|sync` roda igual em Linux, macOS e Windows (symlink ou `--copy`).
- Execução não interativa sem travar.
- README com passos por SO e versão fixada.
- Testes de determinismo passando; lint limpo.
- Release `v1.0.0` tagueada com script anexado.

## 11. Status de implementação

**Concluído — v1.0.0 (2026-11-15).** Todos os critérios de conclusão foram atendidos. **Testado com sucesso** em `/home/roberto/projetos/teste-reversa-openclaude`.

| Fase | Entrega | Situação |
|---|---|---|
| 0 | Decisões e escopo | Concluída |
| 1 | `menu-reversa-openclaude.py` (install/sync via symlink) | Concluída |
| 2 | Wrappers `bin/*.sh` e `bin/*.ps1` | Concluída |
| 3 | README reescrito (por SO, versão fixada) | Concluída |
| 4 | Testes (13, unittest + determinismo) | Concluída |
| 5 | Tag/Release `v1.0.0` + `CHANGELOG.md` | Concluída |

- Repositório alvo: commit `98ee4f3` + tag `v1.0.0`.
- Release: <https://github.com/JosehRoberto/menu-reversa-openclaude/releases/tag/v1.0.0>
- Decisões aplicadas: bridge por **symlink** com fallback `--copy`; escopo `--global`; sem PyYAML.
- Pendência: em Windows, symlink exige Developer Mode/admin (usar `--copy`); validar `bin/*.ps1` em máquina Windows real.

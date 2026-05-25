# VALIDATION_PLAN.md
> RiukyOS LAB — Plano de validação por fase  
> Status: **Draft Approved Candidate + P3.1 Patched** | Documento 7/7 — Fase P3 Architecture Documentation

---

## 1. Objetivo

Este documento define como validar o projeto RiukyOS em cada fase do roadmap.

A função dele é impedir aprovação subjetiva.

Regra central:

```text
Nada é aprovado porque "parece bom".
Toda fase precisa de comando, saída, arquivo, checklist ou evidência objetiva.
```

O plano cobre:

- Foundation — Base técnica do workspace;
- Probe — RiukyOS Probe CLI;
- Control — RiukyOS Control Center;
- Safeguard — primeira ação segura;
- validação de documentação;
- validação de release;
- critérios de rejeição e rollback.

---

## 2. Princípios de validação RiukyOS

### 2.1 Validação antes de aprovação

Uma etapa só pode ser marcada como aprovada quando todos os critérios obrigatórios forem cumpridos.

Termos permitidos:

```text
Draft
In Progress
Needs Fix
Needs Triage
Validated
Approved
Rejected
Rollback Required
```

Evitar:

```text
parece bom
acho que funcionou
deve estar certo
```

---

### 2.2 Evidência mínima

Cada fase precisa gerar pelo menos uma evidência:

- comando executado;
- saída no terminal;
- arquivo gerado;
- JSON válido;
- relatório TXT legível;
- screenshot quando houver UI;
- release no GitHub;
- checklist preenchido.

---

### 2.3 Core não é validado pela UI

O `riukyos-core` deve ser validado independentemente do app visual.

A interface gráfica não pode ser a primeira prova de que o core funciona.

Ordem correta:

```text
core → CLI → relatório → UI
```

---

### 2.4 Sem ação destrutiva antes do Safeguard

Foundation, Probe e Control são leitura, diagnóstico e exportação.

Nenhuma validação pode exigir:

- remover arquivos;
- alterar registro;
- alterar serviços;
- finalizar processos;
- alterar plano de energia;
- limpar temporários;
- aplicar otimização.

---

## 3. Ambientes de validação

### 3.1 Ambiente principal de desenvolvimento

Sistema principal do usuário:

```text
Arch Linux + Hyprland
fish shell
VS Code
```

Uso:

- desenvolvimento Rust inicial;
- organização do repositório;
- documentação;
- commits;
- build quando aplicável.

Observação:

```text
Se algum comando usar sintaxe bash, entrar em bash explicitamente.
```

---

### 3.2 Ambiente Windows de validação

Alvo do projeto:

```text
Windows 10 build 1903 ou superior
```

Uso:

- testar `winreg`;
- testar coleta via PowerShell/powercfg;
- validar `riukyos-probe.exe`;
- validar report em ambiente real;
- validar app Tauri no Control.

---

### 3.3 Ambiente limpo

Para Control e releases públicas:

```text
Windows limpo ou PC diferente do ambiente de desenvolvimento
```

Objetivo:

- provar que o app não depende de configuração local invisível;
- validar instalador;
- validar WebView2/dependências;
- validar exportação em pasta comum.

---

## 4. Validação de documentação — Fase P3

A Fase P3 está aprovada quando os sete documentos principais existem e estão coerentes.

Documentos obrigatórios:

```text
docs/PROJECT_DECISION.md
docs/ARCHITECTURE.md
docs/DATA_MODEL.md
docs/CLI_SPEC.md
docs/REPORT_SPEC.md
docs/UI_SPEC.md
docs/VALIDATION_PLAN.md
```

Documento paralelo:

```text
docs/BRAND_SPEC.md
```

`BRAND_SPEC.md` não bloqueia Foundation nem Probe.  
Ele bloqueia apenas o início do frontend Control.

### 4.1 Checklist P3

Aprovado quando:

- [ ] `PROJECT_DECISION.md` define identidade, stack, fases e fora de escopo;
- [ ] `ARCHITECTURE.md` define camadas e responsabilidades;
- [ ] `DATA_MODEL.md` define structs e exemplo de `SystemReport`;
- [ ] `CLI_SPEC.md` define comandos e comportamento;
- [ ] `REPORT_SPEC.md` define `report.json` e `report.txt`;
- [ ] `UI_SPEC.md` define as 5 telas funcionais;
- [ ] `VALIDATION_PLAN.md` define critérios objetivos por fase;
- [ ] todos os documentos concordam sobre Foundation, 0, 1 e 2;
- [ ] todos os documentos mantêm ações modificadoras fora até Safeguard;
- [ ] todos os documentos tratam `BRAND_SPEC.md` como paralelo.

---

## 5. Validação do repositório inicial

O primeiro commit público deve conter, no mínimo:

```text
README.md
CHANGELOG.md
.gitignore
docs/PROJECT_DECISION.md
docs/ARCHITECTURE.md
docs/DATA_MODEL.md
docs/CLI_SPEC.md
docs/REPORT_SPEC.md
docs/UI_SPEC.md
docs/VALIDATION_PLAN.md
```

Opcional:

```text
docs/BRAND_SPEC.md
```

### 5.1 Checklist do primeiro commit

Aprovado quando:

- [ ] repositório público criado;
- [ ] nome do repositório é `riukyos` ou alternativa aprovada;
- [ ] README contém descrição curta do projeto;
- [ ] CHANGELOG contém seção `[Unreleased]`;
- [ ] `.gitignore` inclui `target/`, `reports/` e artefatos temporários;
- [ ] documentos P3 estão em `docs/`;
- [ ] primeiro commit tem mensagem clara;
- [ ] repositório abre corretamente no GitHub.

Mensagem sugerida:

```text
docs: approve initial RiukyOS project architecture
```

---

## 6. Foundation — Base técnica do workspace

### 6.1 Objetivo

Validar a base Rust necessária antes de iniciar o CLI completo.

Foundation não é produto final.  
É fundação técnica do workspace.

### 6.2 Entregáveis

Marco 1:

```text
Ler CPU, RAM e disco com sysinfo e imprimir no terminal.
```

Marco 2:

```text
Criar SystemReport parcial e exportar report.json com serde.
```

Marco 3:

```text
Ler uma chave simples do registro Windows com winreg sem crash.
```

### 6.3 Comandos de validação

No workspace:

```powershell
cargo check
```

```powershell
cargo build
```

```powershell
cargo run -p riukyos-probe
```

Quando o CLI ainda não existir, os comandos podem ser adaptados para o pacote experimental do Foundation.

### 6.4 Validação de JSON

Após gerar `report.json`:

```powershell
Get-Content .\report.json | ConvertFrom-Json | Out-Null
```

Resultado esperado:

```text
sem erro
```

### 6.5 Critério de aprovação

Foundation aprovado quando:

- [ ] código compila;
- [ ] não há panic em execução normal;
- [ ] CPU, RAM e disco aparecem no terminal;
- [ ] `report.json` é gerado;
- [ ] `report.json` é JSON válido;
- [ ] `schema_version` existe;
- [ ] `generated_at` existe;
- [ ] `probe_version` existe;
- [ ] `hostname` existe;
- [ ] `username` é `null` por padrão;
- [ ] `winreg` lê dado real do Windows;
- [ ] nenhum dado pessoal é coletado automaticamente;
- [ ] nenhuma ação modifica o sistema.

### 6.6 Critério de rejeição

Foundation não é aprovado se:

- código só funciona no PC do dev sem explicação;
- JSON inválido é gerado;
- existe panic em dado ausente;
- `username` é coletado automaticamente;
- alguma ação modifica o sistema;
- lógica de coleta fica fora do core.

---

## 7. Probe — RiukyOS Probe CLI

### 7.1 Objetivo

Criar o primeiro produto funcional:

```text
riukyos-probe.exe
```

O CLI deve consumir `riukyos-core` e gerar diagnóstico no terminal e em arquivos.

### 7.2 Comandos obrigatórios

```powershell
riukyos-probe system
riukyos-probe processes
riukyos-probe power
riukyos-probe report
```

### 7.3 Validação de build

```powershell
cargo check
```

```powershell
cargo build --release
```

Esperado:

```text
target/release/riukyos-probe.exe
```

### 7.4 Validação de help

```powershell
.\target\release\riukyos-probe.exe --help
```

Aprovado quando mostra:

```text
system
processes
power
report
```

### 7.5 Validação de version

```powershell
.\target\release\riukyos-probe.exe --version
```

Esperado:

```text
riukyos-probe 0.1.0
```

Ou versão equivalente registrada no `Cargo.toml`.

---

### 7.6 Validação do comando `system`

```powershell
.\target\release\riukyos-probe.exe system
```

Aprovado quando:

- [ ] imprime `RiukyOS Probe — System`;
- [ ] mostra OS ou `indisponível`;
- [ ] mostra CPU ou `indisponível`;
- [ ] mostra RAM ou `indisponível`;
- [ ] mostra disco C: ou `indisponível`;
- [ ] não modifica o sistema;
- [ ] retorna exit code `0` se falhas forem apenas parciais.

---

### 7.7 Validação do comando `processes`

```powershell
.\target\release\riukyos-probe.exe processes
```

```powershell
.\target\release\riukyos-probe.exe processes --top 10
```

Aprovado quando:

- [ ] imprime `RiukyOS Probe — Processes`;
- [ ] lista até 15 processos por padrão;
- [ ] respeita `--top 10`;
- [ ] exibe PID;
- [ ] exibe nome;
- [ ] exibe CPU ou `indisponível`;
- [ ] exibe memória;
- [ ] não finaliza processo;
- [ ] não mostra command line completa;
- [ ] não mostra caminho completo do executável.

---

### 7.8 Validação do comando `power`

```powershell
.\target\release\riukyos-probe.exe power
```

Aprovado quando:

- [ ] imprime `RiukyOS Probe — Power`;
- [ ] mostra plano ativo ou `indisponível`;
- [ ] mostra GUID ou `indisponível`;
- [ ] não altera plano de energia;
- [ ] retorna exit code `0` quando falha é apenas campo indisponível.

---

### 7.9 Validação do comando `report`

```powershell
mkdir C:\Temp\RiukyOSReportTest
.\target\release\riukyos-probe.exe report --output C:\Temp\RiukyOSReportTest
```

Aprovado quando gera:

```text
C:\Temp\RiukyOSReportTest\report.json
C:\Temp\RiukyOSReportTest\report.txt
```

Verificar existência:

```powershell
Test-Path C:\Temp\RiukyOSReportTest\report.json
Test-Path C:\Temp\RiukyOSReportTest\report.txt
```

Esperado:

```text
True
True
```

Validar JSON:

```powershell
Get-Content C:\Temp\RiukyOSReportTest\report.json | ConvertFrom-Json | Out-Null
```

Esperado:

```text
sem erro
```

Validar TXT:

```powershell
Select-String -Path C:\Temp\RiukyOSReportTest\report.txt -Pattern "RiukyOS Probe Report"
Select-String -Path C:\Temp\RiukyOSReportTest\report.txt -Pattern "Nenhuma alteração foi feita"
Select-String -Path C:\Temp\RiukyOSReportTest\report.txt -Pattern "não promete ganho"
```

Esperado:

```text
todas as buscas retornam resultado
```

---

### 7.10 Validação de sobrescrita

Executar duas vezes:

```powershell
.\target\release\riukyos-probe.exe report --output C:\Temp\RiukyOSReportTest
.\target\release\riukyos-probe.exe report --output C:\Temp\RiukyOSReportTest
```

Aprovado quando:

- [ ] segunda execução não quebra;
- [ ] arquivos são sobrescritos;
- [ ] terminal informa warning de sobrescrita.

---

### 7.11 Validação de saída inválida

Testar pasta sem permissão quando possível.

Exemplo conceitual:

```powershell
.\target\release\riukyos-probe.exe report --output C:\Windows\System32\RiukyOSDenied
```

Aprovado quando:

- [ ] erro é exibido em `stderr`;
- [ ] exit code é `1`;
- [ ] nenhum panic aparece;
- [ ] mensagem é compreensível.

---

### 7.12 Checklist final Probe

Probe aprovado quando:

- [ ] `cargo check` passa;
- [ ] `cargo build --release` passa;
- [ ] binário `riukyos-probe.exe` existe;
- [ ] `--help` funciona;
- [ ] `--version` funciona;
- [ ] `system` funciona;
- [ ] `processes` funciona;
- [ ] `processes --top` funciona;
- [ ] `power` funciona;
- [ ] `report` gera JSON e TXT;
- [ ] JSON parseia;
- [ ] TXT contém observações obrigatórias;
- [ ] erro de escrita é tratado sem panic;
- [ ] README contém uso básico;
- [ ] CHANGELOG foi atualizado;
- [ ] primeira release GitHub foi publicada;
- [ ] `.exe` anexado à release;
- [ ] nenhuma ação modifica o sistema.

---

## 8. Control — RiukyOS Control Center

### 8.1 Objetivo

Criar app visual Tauri + Svelte consumindo o mesmo `riukyos-core`.

Control é leitura, diagnóstico e exportação.

### 8.2 Pré-requisitos

Control só começa quando:

- [ ] Probe aprovado;
- [ ] `BRAND_SPEC.md` aprovado;
- [ ] `UI_SPEC.md` aprovado;
- [ ] `REPORT_SPEC.md` aprovado;
- [ ] `riukyos-core` está estável o suficiente para consumo pela GUI.

---

### 8.3 Build e execução

Comando conceitual dentro de `apps/control-center`:

```powershell
npm install
npm run tauri dev
```

Build:

```powershell
npm run tauri build
```

Aprovado quando:

- [ ] app abre;
- [ ] não há erro fatal no console;
- [ ] navegação entre telas funciona;
- [ ] backend Rust compila;
- [ ] frontend Svelte compila.

---

### 8.4 Validação das 5 telas

Telas obrigatórias:

```text
Dashboard
Processos
Diagnóstico
Exportar
Sobre
```

Aprovado quando:

- [ ] Dashboard mostra resumo do sistema;
- [ ] Processos mostra top processos;
- [ ] Diagnóstico gera novo relatório;
- [ ] Exportar salva `report.json` e `report.txt`;
- [ ] Sobre mostra versão e método;
- [ ] nenhuma tela oferece ação modificadora.

---

### 8.5 Validação de estados

Cada tela deve tratar:

- [ ] estado inicial;
- [ ] loading;
- [ ] sucesso;
- [ ] dados parciais;
- [ ] erro.

Campos `null` devem aparecer como:

```text
indisponível
```

---

### 8.6 Validação de exportação

No app:

1. gerar diagnóstico;
2. ir para Exportar;
3. escolher pasta;
4. exportar relatório.

Aprovado quando:

- [ ] `report.json` existe;
- [ ] `report.txt` existe;
- [ ] JSON parseia;
- [ ] TXT contém observações obrigatórias;
- [ ] app mostra caminhos gerados;
- [ ] sobrescrita é avisada se arquivos já existirem.

---

### 8.7 Validação em ambiente limpo

Control não é aprovado apenas no PC de desenvolvimento.

Validar em:

```text
Windows limpo ou PC diferente do dev
```

Aprovado quando:

- [ ] instalador executa;
- [ ] app abre;
- [ ] gera diagnóstico;
- [ ] exporta relatório;
- [ ] README tem screenshots atualizados.

---

### 8.8 Checklist final Control

Control aprovado quando:

- [ ] Probe permanece funcionando;
- [ ] app Tauri compila;
- [ ] app Tauri abre;
- [ ] 5 telas funcionam;
- [ ] app consome `riukyos-core`;
- [ ] exportação funciona;
- [ ] ambiente limpo aprovado;
- [ ] README atualizado com screenshots;
- [ ] CHANGELOG atualizado;
- [ ] nenhuma ação modifica o sistema;
- [ ] `BRAND_SPEC.md` foi respeitado.

---

## 9. Safeguard — Primeira ação segura

### 9.1 Objetivo

Adicionar primeira ação modificadora controlada:

```text
limpeza segura de temporários
```

Safeguard só começa após Control aprovado.

### 9.2 Fluxo obrigatório

```text
pré-check → lista do que será limpo → confirmação explícita → execução → log → relatório pós-ação
```

### 9.3 Allowlist de temporários

A ação só pode tocar pastas explicitamente aprovadas.

Allowlist inicial candidata:

```text
%TEMP%
%LOCALAPPDATA%\Temp
C:\Windows\Temp
```

A allowlist final deve ser documentada antes da implementação.

### 9.4 Critério de confirmação

A execução só acontece após confirmação explícita do usuário.

Sem confirmação:

```text
nenhum arquivo é removido
```

### 9.5 Log obrigatório

O log deve registrar:

- data/hora;
- pasta alvo;
- arquivos candidatos;
- arquivos removidos;
- arquivos ignorados;
- erros de permissão;
- total estimado liberado.

### 9.6 Critério de aprovação Safeguard

Safeguard aprovado quando:

- [ ] pré-check lista arquivos antes de remover;
- [ ] usuário confirma explicitamente;
- [ ] só allowlist é tocada;
- [ ] arquivos fora da allowlist não são tocados;
- [ ] log é gerado;
- [ ] relatório pós-ação compara antes/depois;
- [ ] erro de permissão não causa crash;
- [ ] README explica a ação;
- [ ] CHANGELOG atualizado.

---

## 10. Validação de release

### 10.1 Probe release

Release mínima Probe:

```text
riukyos-probe.exe
README.md
CHANGELOG.md
```

Opcional:

```text
report-example.json
report-example.txt
```

Aprovado quando:

- [ ] tag criada;
- [ ] release publicada no GitHub;
- [ ] `.exe` anexado;
- [ ] README explica download e uso;
- [ ] CHANGELOG registra mudanças;
- [ ] binário foi testado antes de publicar.

Tag sugerida:

```text
v0.1.0-probe
```

---

### 10.2 Control release

Release mínima Control:

```text
RiukyOS Control Center installer
README.md atualizado
screenshots
CHANGELOG.md
```

Aprovado quando:

- [ ] instalador gerado;
- [ ] instalado em ambiente limpo;
- [ ] app abre;
- [ ] exporta relatório;
- [ ] screenshots atualizados;
- [ ] release publicada.

Tag sugerida:

```text
v0.1.0-control-center
```

---

## 11. Validação de segurança e privacidade

### 11.1 Dados pessoais

Aprovado quando:

- [ ] `username` é `null` por padrão;
- [ ] relatório não contém e-mail;
- [ ] relatório não contém documentos;
- [ ] relatório não contém histórico;
- [ ] relatório não contém command line completa;
- [ ] relatório não contém variáveis de ambiente;
- [ ] relatório não contém caminho completo de executável de processo.

### 11.2 Ações invisíveis

Aprovado quando:

- [ ] Foundation não modifica sistema;
- [ ] Probe não modifica sistema;
- [ ] Control não modifica sistema;
- [ ] Safeguard só modifica após confirmação explícita.

---

## 12. Critérios de rollback

Rollback é obrigatório se:

- uma fase modifica o sistema antes do permitido;
- relatório coleta dado pessoal automaticamente;
- CLI ou UI promete ganho de performance;
- core passa a depender de Tauri;
- lógica de coleta é duplicada fora do core;
- app falha em ambiente limpo sem explicação;
- release publicada contém binário errado.

Ação:

```text
marcar fase como Rejected / Rollback Required
registrar causa no CHANGELOG ou issue
corrigir antes de avançar
```

---

## 13. Critério de aprovação deste documento

Este `VALIDATION_PLAN.md` está aprovado quando:

- [ ] define validação para P3;
- [ ] define validação para primeiro commit;
- [ ] define validação para Foundation;
- [ ] define validação para Probe;
- [ ] define validação para Control;
- [ ] define validação para Safeguard;
- [ ] inclui comandos objetivos;
- [ ] inclui critérios de rejeição;
- [ ] reforça privacidade;
- [ ] reforça nenhuma ação modificadora antes do Safeguard;
- [ ] define release mínima.

---

## 14. Status final da documentação P3

Quando este documento for aprovado, a Fase P3 fica pronta para fechamento:

```text
P3 — Architecture Documentation
Status: Ready for repository creation
```

Próxima ação após aprovação:

```text
1. criar repositório público;
2. adicionar documentos;
3. adicionar README.md, CHANGELOG.md e .gitignore;
4. fazer primeiro commit;
5. iniciar Foundation.
```


---

## 15. P3.1 — Patches obrigatórios antes do primeiro commit

Esta seção registra a rodada técnica P3.1.

### 15.1 Workflow Arch → Windows

Desenvolvimento principal:

```text
Arch Linux + Hyprland
```

Validação oficial:

```text
Windows 10 build 1903+ ou VM Windows equivalente
```

Regra:

```text
Código pode nascer no Arch.
Aprovação exige execução real no Windows/VM.
```

Cross-compilação:

```text
Fazer spike técnico com target Windows.
Se funcionar, usar como acelerador.
Se gerar complexidade excessiva, adiar.
```

A cross-compilação não substitui validação real no Windows.

### 15.2 Comandos Rust obrigatórios

Antes de aprovar Foundation, Probe ou release:

```powershell
cargo fmt --check
cargo check
cargo clippy -- -D warnings
cargo test
cargo build --release
```

### 15.3 CPU usage

Foundation:

```text
`cpu.usage_percent` pode ser null.
```

Probe:

```text
CPU usage só é aprovado com amostragem dupla ou null explícito.
```

Validação:

- [ ] se não houver amostragem, `usage_percent` não recebe valor inventado;
- [ ] se houver amostragem, o valor aparece no JSON como número;
- [ ] se falhar, JSON usa `null` e TXT usa `indisponível`.

### 15.4 Power plan

A coleta de plano de energia deve validar:

- [ ] GUID extraído por regex quando disponível;
- [ ] nome do plano tratado como opcional;
- [ ] falha de locale não quebra o relatório inteiro;
- [ ] se GUID falhar, `power = null`.

### 15.5 Disco do sistema

Validar `is_system_disk` por:

```text
SystemDrive -> comparação com mount_point -> fallback C:\
```

Aprovado quando:

- [ ] disco `C:\` é marcado como sistema em Windows padrão;
- [ ] nenhum disco aleatório é marcado como sistema sem critério.

### 15.6 `Cargo.lock`

Checklist do primeiro commit:

- [ ] `Cargo.lock` não está no `.gitignore`;
- [ ] `Cargo.lock` será commitado quando gerado;
- [ ] README ou documentação registra que o workspace contém binários.

### 15.7 Exemplos de relatório

Se a release incluir exemplo:

- [ ] usar `report-example.mock.json`;
- [ ] usar `report-example.mock.txt`;
- [ ] não usar relatório real do PC do desenvolvedor;
- [ ] hostname fictício;
- [ ] username `null`.

### 15.8 Timeout de comandos externos

Antes do Control/Safeguard, registrar decisão de timeout para comandos externos.

Probe aceita dívida técnica, mas ela deve estar documentada.

### 15.9 Tauri State

Antes do Control:

- [ ] backend tem estado gerenciado para relatório atual;
- [ ] frontend não reenvia `SystemReport` inteiro para exportação;
- [ ] comando conceitual aprovado: `export_current_report(output_dir)`.
---

*Fim do Documento 7/7 — VALIDATION_PLAN.md*

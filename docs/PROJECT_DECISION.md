# PROJECT_DECISION.md
> RiukyOS LAB — Constituição técnica do projeto
> Status: **Approved + P3.1 Patched** | Fase P3 — Architecture Documentation

---

## 1. Identidade do projeto

| Campo | Decisão |
|---|---|
| Workspace | `riukyos` |
| Produto CLI | RiukyOS Probe (`riukyos-probe.exe`) |
| Produto GUI | RiukyOS Control Center |
| Marca | RiukyOS LAB |
| Repositório | `github.com/<username>/riukyos` |
| Descrição | RiukyOS LAB — diagnostic and optimization tools for Windows |
| Visibilidade | Público desde o Foundation |
| Windows mínimo | Windows 10 build 1903 (maio 2019) ou superior |

---

## 2. Arquitetura — Cargo Workspace (monorepo)

```
riukyos/
├── Cargo.toml                  ← workspace root
├── crates/
│   └── riukyos-core/           ← biblioteca compartilhada (lib)
│       ├── src/
│       │   ├── lib.rs
│       │   ├── models.rs
│       │   └── system.rs       ← começa plano; collectors/ emerge depois
│       └── Cargo.toml
├── apps/
│   ├── probe/                  ← CLI binário (Probe)
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   └── cli.rs
│   │   └── Cargo.toml
│   └── control-center/         ← Tauri app (Control)
│       ├── src-tauri/
│       │   ├── src/
│       │   │   ├── main.rs
│       │   │   └── commands/
│       │   └── Cargo.toml
│       └── src/                ← Svelte frontend
├── docs/                       ← todos os documentos de decisão
├── .gitignore                  ← inclui /reports/ e targets
└── README.md
```

**Princípio de arquitetura:**
> `riukyos-core` não sabe se quem chamou foi CLI ou GUI.
> CLI e GUI são consumidores da mesma biblioteca.
> Nenhuma lógica de coleta de dados vive fora do core.

---

## 3. Stack aprovada

### Backend (Rust)

| Dependência | Uso |
|---|---|
| `sysinfo` | CPU, RAM, disco, processos |
| `winreg` | Leitura do registro do Windows |
| `serde` + `serde_json` | Serialização de structs para JSON |
| `anyhow` | Tratamento de erros com mensagens legíveis |
| `tracing` + `tracing-subscriber` | Logs estruturados |
| `clap` (derive) | Subcomandos do CLI |
| `std::process::Command` | Chamadas ao PowerShell (output sempre `Option<T>`) |

### Frontend (Control)

| Dependência | Uso |
|---|---|
| Tauri 2 | Framework desktop |
| Svelte + TypeScript | Interface reativa |
| Tailwind CSS | Estilização utilitária |

### Fora da stack até Safeguard+

- `windows-rs` — binding nativa avançada
- PDF de relatório — dependência de renderização
- Assinatura de `.exe` — infraestrutura de release
- Auto-update — Tauri updater com assinatura
- Qualquer biblioteca de UI components — aguarda BRAND_SPEC.md

---

## 4. Regras técnicas não negociáveis

1. **`Option<T>` para todo dado que pode falhar.**
   Dado indisponível retorna `None`, nunca panic ou crash.
   Campo ausente no relatório exibe `"indisponível"`.

2. **PowerShell sempre com parse defensivo.**
   Output de `std::process::Command` nunca é assumido como válido.
   Locale `pt-BR` pode alterar formato de saída — tratar como `Option`.

3. **`riukyos-core` sem dependência de Tauri.**
   O core é uma biblioteca Rust pura. Deve compilar sem o runtime Tauri.

4. **Estrutura emerge do código.**
   A pasta `collectors/` só é criada quando houver dois ou mais coletores prontos.
   Não cria pasta vazia antecipando abstração.

5. **Sem ação que modifica o sistema antes do Safeguard.**
   Foundation, 0 e 1 são exclusivamente leitura, diagnóstico e exportação.

6. **Relatório nunca promete melhoria.**
   Mostra estado técnico objetivo. Diagnóstico, não propaganda.

---

## 5. Modelo de dados — decisões fechadas

### `ReportMetadata`
```rust
pub struct ReportMetadata {
    pub generated_at: String,      // timestamp ISO 8601
    pub probe_version: String,     // versão do riukyos-probe
    pub hostname: String,          // nome do PC
    pub schema_version: String,    // versão do schema do JSON ex: "0.1"
    pub username: Option<String>,  // opt-in, nunca coletado automaticamente
}
```

> `report_id` adiado para Safeguard — requer camada de armazenamento inexistente.
> `username` é opt-in: privacidade anotada, nunca coletado por padrão.

### `SystemReport` (estrutura inicial)
```rust
pub struct SystemReport {
    pub metadata: ReportMetadata,
    pub os: OsInfo,
    pub cpu: CpuInfo,
    pub memory: MemoryInfo,
    pub disks: Vec<DiskInfo>,
    pub power: Option<PowerPlanInfo>,
    pub processes: Vec<ProcessInfo>,
}
```

Modelos detalhados definidos em `DATA_MODEL.md`.

---

## 6. Fases e critérios de conclusão

### Foundation — Base técnica do workspace
**Objetivo:** validar a base Rust do workspace antes da evolução para o CLI completo.

| Marco | Entregável |
|---|---|
| 1 | Coleta CPU, RAM e disco com `sysinfo` e imprime no terminal |
| 2 | Cria `SystemReport` parcial e exporta `report.json` com `serde` |
| 3 | Lê chave simples do registro Windows com `winreg` sem crash |

**Aprovado quando:**
- [ ] código Rust compila sem warnings
- [ ] `report.json` é JSON válido
- [ ] `winreg` retorna dado real do sistema local

---

### Probe — RiukyOS Probe CLI
**Objetivo:** CLI funcional, instalável, com release pública.

Comandos:
```
riukyos-probe system
riukyos-probe processes
riukyos-probe power
riukyos-probe report
```

**Aprovado quando:**
- [ ] todos os comandos executam sem crash
- [ ] `riukyos-probe report` gera `report.json` e `report.txt` válidos
- [ ] README com screenshots e instruções de instalação
- [ ] primeira release publicada no GitHub com `.exe`

---

### Control — RiukyOS Control Center
**Objetivo:** app Tauri funcional com 5 telas consumindo `riukyos-core`.

Telas:
1. Dashboard
2. Processos
3. Diagnóstico
4. Exportar
5. Sobre

**Pré-requisito:** `BRAND_SPEC.md` aprovado antes de iniciar o frontend.

**Aprovado quando:**
- [ ] app abre e navega entre as 5 telas sem crash
- [ ] Dashboard exibe todos os campos definidos no `UI_SPEC.md`
- [ ] exportação gera `report.json` e `report.txt` em pasta escolhida
- [ ] instalador testado em PC diferente do de desenvolvimento
- [ ] README atualizado com screenshots do app

---

### Safeguard — Primeira ação segura
**Objetivo:** única ação modificadora com log completo e reversibilidade.

Ação: limpeza segura de temporários.

Fluxo obrigatório:
```
pré-check → lista do que será limpo → confirmação → execução → log → relatório pós-ação
```

**Aprovado quando:**
- [ ] ação só executa após confirmação explícita do usuário
- [ ] log registra cada arquivo removido
- [ ] relatório pós-ação compara estado antes/depois
- [ ] nenhum arquivo fora da allowlist de temporários é tocado

Allowlist candidata inicial:

```text
%TEMP%
%LOCALAPPDATA%\Temp
C:\Windows\Temp
```

A allowlist final deve ser confirmada antes da implementação do Safeguard.

---

## 7. Fora do escopo — decisão permanente até revisão explícita

Esta lista representa decisões tomadas, não pendências.
Qualquer item aqui só entra no projeto mediante nova decisão documentada.

```
- windows-rs
- geração de PDF
- assinatura de .exe
- auto-update (Tauri updater)
- modo empresa / contratos
- perfis de otimização avançados
- qualquer ação que modifique serviços do sistema (antes de Safeguard)
- banco de dados ou armazenamento persistente (antes de Safeguard)
- report_id (antes de Safeguard)
- multi-idioma / i18n
```

---

## 8. Identidade visual — direção fundadora

A estética do RiukyOS não é decidida aqui em detalhe, mas a direção está definida:

```
noir · premium · dark · metálico · minimalista
técnico · grafite/prata · brilho frio discreto
```

Detalhamento em `BRAND_SPEC.md` (paralelo, bloqueia apenas Control frontend).

---

## 9. Status dos documentos

| Documento | Status |
|---|---|
| `PROJECT_DECISION.md` | ✅ Approved + P3.1 patched |
| `ARCHITECTURE.md` | ✅ Approved + P3.1 patched |
| `DATA_MODEL.md` | ✅ Approved + P3.1 patched |
| `CLI_SPEC.md` | ✅ Approved + P3.1 patched |
| `REPORT_SPEC.md` | ✅ Approved + P3.1 patched |
| `UI_SPEC.md` | ✅ Approved + P3.1 patched |
| `VALIDATION_PLAN.md` | ✅ Approved + P3.1 patched |
| `P3_1_PATCH_SUMMARY.md` | ✅ Applied |
| `BRAND_SPEC.md` | 🔲 paralelo / bloqueia apenas Control frontend |

Esta tabela representa o estado da documentação no fechamento da P3.1.


---

## 10. P3.1 — Decisões técnicas antes do repositório

Esta seção registra a rodada **P3.1 — Technical Gap Patch**, criada após revisão técnica dos documentos P3.

Status:

```text
P3.1 — Required before repository creation
```

### 10.1 Uso de CPU com `sysinfo`

`usage_percent` de CPU é um campo potencialmente instável porque depende de amostragem.

Decisão:

```text
Foundation:
- `cpu.usage_percent` pode ser `null`.

Probe:
- implementar amostragem dupla antes de preencher `cpu.usage_percent`;
- refresh inicial;
- aguardar intervalo mínimo recomendado pelo `sysinfo`;
- refresh final;
- calcular/ler uso após a segunda leitura.
```

Regra:

```text
Se a amostragem não for confiável, `usage_percent` deve ser `null`, não valor inventado.
```

### 10.2 Workflow Arch → Windows

Ambiente principal de desenvolvimento:

```text
Arch Linux + Hyprland
fish shell
```

Alvo oficial de validação:

```text
Windows 10 build 1903+
```

Decisão:

```text
O código pode nascer no Arch.
A aprovação real do Foundation/Probe exige execução no Windows/VM.
```

Cross-compilação:

```text
Fazer um spike técnico com target Windows a partir do Arch.
Se funcionar com baixo atrito, usar como acelerador.
Se virar complexidade excessiva, adiar sem bloquear o projeto.
```

Regra:

```text
Cross-compilação acelera, mas não substitui validação real no Windows.
```

### 10.3 Parse do `powercfg`

A coleta de plano de energia deve priorizar dados estáveis.

Decisão:

```text
Extrair GUID por regex.
Extrair nome do plano entre parênteses, se existir.
Se o nome falhar, manter GUID e `active_plan_name = None`.
```

### 10.4 Detecção de disco do sistema

Campo:

```text
DiskInfo.is_system_disk
```

Estratégia Probe:

```text
1. ler variável de ambiente `SystemDrive`;
2. normalizar para formato com barra invertida, ex: `C:\`;
3. comparar com `mount_point`;
4. fallback: considerar `C:\` como disco do sistema.
```

### 10.5 `Cargo.lock`

Decisão:

```text
`Cargo.lock` deve ser commitado.
```

Motivo:

```text
O workspace contém binários (`riukyos-probe` e futuramente `control-center`).
Build reproduzível é mais importante que tratar o repo como biblioteca pura.
```

Regra:

```text
Não incluir `Cargo.lock` no `.gitignore`.
```

### 10.6 Validação Rust mínima

A validação padrão de Rust passa a incluir:

```powershell
cargo fmt --check
cargo check
cargo clippy -- -D warnings
cargo test
cargo build --release
```

### 10.7 Exemplos de relatório

Não incluir relatório real gerado da máquina do desenvolvedor em release pública.

Decisão:

```text
Usar apenas `report-example.mock.json` e `report-example.mock.txt`
com dados fictícios, se exemplos forem incluídos.
```

### 10.8 Exportação no Tauri

No Control, o app visual deve evitar reenviar o `SystemReport` inteiro do frontend para o backend apenas para exportar.

Decisão:

```text
Backend mantém o relatório atual em estado gerenciado.
Frontend chama `export_current_report(output_dir)`.
Backend exporta o relatório armazenado.
```

### 10.9 Timeout de comandos externos

`std::process::Command` não deve virar ponto de travamento indefinido em versão madura.

Decisão:

```text
Probe pode começar simples.
Antes do Control/Safeguard, timeout para comandos externos vira dívida técnica obrigatória.
```
---

*Documento aprovado em fase P2 — Project Design.*
*Qualquer alteração neste documento requer nova decisão explícita registrada aqui.*

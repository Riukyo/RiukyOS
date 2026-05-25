# ARCHITECTURE.md
> RiukyOS LAB — Arquitetura técnica do workspace
> Status: **Approved + P3.1 Patched** | Fase P3 — Architecture Documentation

---

## 1. Visão geral

O projeto é um **Cargo Workspace** com estrutura alvo de três crates. A separação em camadas garante que a lógica de coleta de dados nunca seja duplicada entre o CLI e o app visual.

Durante Foundation/Probe, o workspace começa com duas crates ativas:
- `crates/riukyos-core`
- `apps/probe`

A crate do `apps/control-center` entra apenas no Control.

```
┌─────────────────────────────────────────────┐
│              Windows (sistema)              │
│  sysinfo · winreg · PowerShell · WMI        │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│            riukyos-core  (lib)              │
│  coleta · modelos · erros · logs · export  │
└────────────┬──────────────────┬────────────┘
             │                  │
             ▼                  ▼
┌────────────────┐   ┌──────────────────────┐
│  apps/probe    │   │ apps/control-center  │
│  CLI binário   │   │ Tauri app (Control)  │
│  Probe         │   │ Svelte + TypeScript  │
└────────────────┘   └──────────────────────┘
```

---

## 2. Crates — responsabilidades e fronteiras

### `crates/riukyos-core`
**Tipo:** biblioteca (`lib`)
**Responsabilidade:** toda lógica de domínio do projeto.

Faz:
- coletar dados do sistema via `sysinfo`, `winreg` e `std::process::Command`
- montar e validar as structs do modelo de dados
- serializar e desserializar `SystemReport` (JSON)
- gerar o texto do `report.txt`
- propagar erros via `anyhow::Result<T>`
- emitir spans e eventos de log via `tracing`

Não faz:
- não inicializa o subscriber de logs (responsabilidade do app)
- não imprime nada no terminal
- não sabe se quem chamou foi CLI ou GUI
- não tem dependência de Tauri

> **Regra de ouro:** se uma função precisa saber de onde foi chamada,
> ela está no lugar errado. Pertence ao app chamador, não ao core.

---

### `apps/probe`
**Tipo:** binário (`bin`) — `riukyos-probe.exe`
**Responsabilidade:** interface de terminal.

Faz:
- parsear argumentos e subcomandos com `clap`
- inicializar o subscriber de logs (`tracing-subscriber`)
- chamar funções do `riukyos-core`
- formatar e imprimir resultados no terminal
- exportar `report.json` e `report.txt` via core

Não faz:
- não contém lógica de coleta de dados
- não monta structs de sistema diretamente
- não chama `sysinfo` ou `winreg` diretamente

---

### `apps/control-center`
**Tipo:** app Tauri (Control)
**Responsabilidade:** interface visual.

Faz:
- expor comandos Tauri (`#[tauri::command]`) que chamam `riukyos-core`
- receber dados do core e enviá-los ao frontend Svelte via IPC Tauri
- inicializar o subscriber de logs (arquivo de log local)
- exportar relatório via diálogo de sistema

Não faz:
- não contém lógica de coleta de dados
- não chama `sysinfo` ou `winreg` diretamente
- o frontend Svelte nunca acessa o sistema — só consome dados já processados

---

## 3. Estrutura de arquivos

```
riukyos/
│
├── Cargo.toml                          ← workspace root
│
├── crates/
│   └── riukyos-core/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs                  ← re-exporta módulos públicos
│           ├── models.rs               ← todas as structs
│           └── system.rs               ← coleta inicial (plano)
│           (collectors/ emerge quando houver 2+ arquivos de coleta)
│
├── apps/
│   ├── probe/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs                 ← inicialização + entry point
│   │       └── cli.rs                  ← definição dos subcomandos (clap)
│   │
│   └── control-center/                 ← criado no Control
│       ├── src-tauri/
│       │   ├── Cargo.toml
│       │   └── src/
│       │       ├── main.rs             ← setup Tauri
│       │       └── commands/           ← um arquivo por domínio
│       │           ├── system.rs
│       │           ├── power.rs
│       │           ├── processes.rs
│       │           └── report.rs
│       └── src/                        ← frontend Svelte
│           ├── App.svelte
│           ├── lib/
│           └── routes/
│
├── docs/
├── .gitignore
└── README.md
```

---

## 4. Configuração do workspace

### `Cargo.toml` (raiz)
```toml
[workspace]
members = [
    "crates/riukyos-core",
    "apps/probe",
    # "apps/control-center/src-tauri",  # descomentado em Control
]
resolver = "2"

[workspace.dependencies]
serde       = { version = "1",    features = ["derive"] }
serde_json  = "1"
anyhow      = "1"
tracing     = "0.1"
```

> Dependências compartilhadas declaradas em `[workspace.dependencies]`
> garantem versão única em todo o workspace — sem conflito entre crates.

---

### `crates/riukyos-core/Cargo.toml`
```toml
[package]
name    = "riukyos-core"
version = "0.1.0"
edition = "2021"

[dependencies]
sysinfo     = "0.30"
winreg      = "0.52"
serde       = { workspace = true }
serde_json  = { workspace = true }
anyhow      = { workspace = true }
tracing     = { workspace = true }
```

---

### `apps/probe/Cargo.toml`
```toml
[package]
name    = "riukyos-probe"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "riukyos-probe"

[dependencies]
riukyos-core        = { path = "../../crates/riukyos-core" }
clap                = { version = "4", features = ["derive"] }
anyhow              = { workspace = true }
tracing             = { workspace = true }
tracing-subscriber  = "0.3"
```

---

## 5. Fluxo de dados

### CLI (Probe)

```
Usuário digita: riukyos-probe report
         │
         ▼
    cli.rs (clap)
    parseia subcomando
         │
         ▼
    main.rs
    chama riukyos_core::collect()
         │
         ▼
    riukyos-core
    coleta dados do sistema
    monta SystemReport
    retorna Result<SystemReport>
         │
         ▼
    main.rs
    chama riukyos_core::export_json()
    chama riukyos_core::export_txt()
         │
         ▼
    report.json + report.txt
    escritos em disco
```

### App Tauri (Control)

```
Usuário clica "Gerar diagnóstico"
         │
         ▼
    Frontend Svelte
    chama invoke("generate_report")
         │
         ▼
    Tauri IPC
    roteia para comando Rust
         │
         ▼
    commands/report.rs
    chama riukyos_core::collect()
         │
         ▼
    riukyos-core
    (mesma lógica do CLI)
    retorna Result<SystemReport>
         │
         ▼
    commands/report.rs
    serializa para JSON
    retorna ao frontend via IPC
         │
         ▼
    Frontend Svelte
    exibe dados na tela
```

---

## 6. Estratégia de erros

Todo erro no `riukyos-core` usa `anyhow::Result<T>`.

```
Core retorna Err(...)
      │
      ├── CLI: imprime mensagem de erro no stderr, exit code 1
      │
      └── Tauri: converte para String, retorna ao frontend como erro IPC
```

Campos que podem falhar individualmente retornam `Option<T>`:
- `power: Option<PowerPlanInfo>` — plano de energia pode ser inacessível
- `username: Option<String>` — opt-in, nunca coletado por padrão
- Qualquer dado via PowerShell — locale e permissão são variáveis

> Um erro em um campo nunca derruba o relatório inteiro.
> O relatório nasce com os campos disponíveis e marca `null` nos ausentes.

---

## 7. Estratégia de logs

O `riukyos-core` **emite** eventos de log com `tracing`, mas **nunca inicializa** o subscriber.

```rust
// dentro do core — apenas emite
tracing::debug!("coletando informações de CPU");
tracing::warn!("plano de energia indisponível, retornando None");
```

Cada app inicializa seu próprio subscriber:

```rust
// apps/probe/src/main.rs
tracing_subscriber::fmt::init();  // imprime no stderr

// apps/control-center/src-tauri/src/main.rs
// inicializa subscriber que escreve em arquivo de log local
```

Isso garante que o core seja uma biblioteca limpa — sem efeito colateral de I/O.

---

## 8. Modelo de comunicação Tauri (Control)

O frontend Svelte nunca acessa o sistema operacional diretamente.
Todo acesso acontece via comando Tauri:

```
Frontend (TypeScript)          Backend (Rust)
─────────────────────          ──────────────
invoke("get_system_info")  →   #[tauri::command]
                               fn get_system_info() -> Result<SystemInfo, String>
                                   riukyos_core::collect_system()
                               ←  retorna JSON serializado
exibe dados na tela
```

Um arquivo por domínio em `commands/`:
- `commands/system.rs` — CPU, RAM, disco, OS
- `commands/power.rs` — plano de energia
- `commands/processes.rs` — lista de processos
- `commands/report.rs` — gerar e exportar relatório

---

## 9. O que esta arquitetura garante

| Propriedade | Como é garantida |
|---|---|
| Sem duplicação de lógica | core é biblioteca; CLI e GUI chamam o mesmo código |
| Core independente de Tauri | `riukyos-core/Cargo.toml` não tem dependência Tauri |
| Erros não derrubam o app | `Option<T>` em campos falíveis, `anyhow::Result` no retorno |
| Logs controlados | core emite, app inicializa — sem efeito colateral |
| Estrutura evolui com o código | `collectors/` só nasce quando houver 2+ coletores |
| Frontend isolado do sistema | toda leitura passa por comando Tauri |


---

## 10. P3.1 — Detalhes de implementação

Esta seção registra decisões que impactam a implementação real do `riukyos-core`.

### 10.1 Amostragem de CPU

Uso de CPU não deve ser lido como valor definitivo em uma única coleta.

Política:

```text
Foundation:
- permitir `cpu.usage_percent = None`.

Probe:
- usar amostragem dupla;
- primeira atualização de CPU;
- aguardar intervalo mínimo recomendado pelo `sysinfo`;
- segunda atualização de CPU;
- então preencher `usage_percent`.
```

Se a coleta falhar:

```text
usage_percent = None
```

### 10.2 Plano de energia via `powercfg`

O output de `powercfg` pode variar por idioma.

Política:

```text
- extrair GUID com regex;
- extrair nome entre parênteses como campo opcional;
- `source = "powercfg"`;
- se o GUID falhar, `power = None`.
```

O nome do plano não é considerado obrigatório.

### 10.3 Disco do sistema

`sysinfo` não deve ser tratado como fonte única para `is_system_disk`.

Estratégia Probe:

```text
SystemDrive -> normalizar -> comparar com mount_point
fallback -> C:\
```

### 10.4 Workflow de desenvolvimento e validação

```text
Desenvolvimento principal: Arch Linux.
Validação oficial: Windows/VM.
Cross-compilação: spike técnico auxiliar.
```

A cross-compilação pode acelerar o loop, mas a aprovação exige execução real no Windows.

### 10.5 `Cargo.lock`

O workspace contém binários, portanto:

```text
Cargo.lock deve ser versionado.
```

### 10.6 Timeout de comandos externos

Comandos externos via `std::process::Command` são permitidos no Probe, mas timeout deve ser tratado antes do Control/Safeguard.

Status:

```text
technical debt accepted for Probe
required before Control/Safeguard hardening
```
---

*Próximo documento: `DATA_MODEL.md`*

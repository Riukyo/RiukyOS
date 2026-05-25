# P3_1_PATCH_SUMMARY.md
> RiukyOS LAB — Technical Gap Patch Summary  
> Status: **Approved Candidate** | P3.1 before repository creation

---

## Objetivo

Registrar a rodada de correção técnica feita após a revisão dos documentos P3.

A documentação P3 estava coerente, mas alguns detalhes de implementação precisavam ser decididos antes do primeiro commit.

---

## Patches aprovados

### 1. `sysinfo` e uso de CPU

Decisão:

```text
Foundation pode usar `cpu.usage_percent = null`.
Probe deve usar amostragem dupla para preencher CPU usage.
```

### 2. Workflow Arch → Windows

Decisão:

```text
Desenvolvimento no Arch.
Validação oficial no Windows/VM.
Cross-compilação será testada como spike auxiliar.
```

Cross-compilação acelera, mas não substitui validação real.

### 3. `powercfg`

Decisão:

```text
Extrair GUID via regex.
Nome do plano é opcional.
Falha de locale não derruba relatório.
```

### 4. `is_system_disk`

Decisão:

```text
Usar SystemDrive como estratégia principal.
Fallback para C:\.
```

### 5. `Cargo.lock`

Decisão:

```text
Commitar Cargo.lock.
Não ignorar no .gitignore.
```

### 6. Validação Rust

Comandos obrigatórios:

```powershell
cargo fmt --check
cargo check
cargo clippy -- -D warnings
cargo test
cargo build --release
```

### 7. Exemplos públicos

Decisão:

```text
Não publicar report real da máquina do dev.
Usar report-example.mock.json/txt com dados fictícios.
```

### 8. Tauri export

Decisão:

```text
Usar estado gerenciado no backend Tauri.
Frontend não reenvia SystemReport inteiro para exportar.
```

### 9. Timeout

Decisão:

```text
Timeout de comandos externos é dívida técnica aceita no Probe.
Antes do Control/Safeguard precisa ser resolvida/documentada.
```

---

## Documentos patchados

- `PROJECT_DECISION.md`
- `ARCHITECTURE.md`
- `DATA_MODEL.md`
- `CLI_SPEC.md`
- `REPORT_SPEC.md`
- `UI_SPEC.md`
- `VALIDATION_PLAN.md`


---

## Correções finais após revisão

A revisão final encontrou três inconsistências pequenas, corrigidas antes do commit:

### 1. `DATA_MODEL.md`

As structs principais foram alinhadas com as políticas P3.1:

```text
CpuInfo.brand = String
CpuInfo.usage_percent = Option<f32>
PowerPlanInfo.active_plan_guid = String
PowerPlanInfo.active_plan_name = Option<String>
SystemReport.power = Option<PowerPlanInfo>
```

### 2. `PROJECT_DECISION.md`

A seção de documentos foi atualizada de "próximos documentos" para "status dos documentos".

### 3. Safeguard

A regra antiga `/Temp` foi substituída por allowlist de temporários:

```text
%TEMP%
%LOCALAPPDATA%\Temp
C:\Windows\Temp
```

A allowlist final ainda deve ser confirmada antes da implementação do Safeguard.

---

## Status após P3.1

```text
P3 — Architecture Documentation
Status: Patched

P3.1 — Technical Gap Patch
Status: Applied

Próximo:
revisão final -> criar repositório público -> primeiro commit -> Foundation
```

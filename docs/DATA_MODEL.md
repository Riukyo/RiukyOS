# DATA_MODEL.md
> RiukyOS LAB — Modelo de dados e contrato do relatório  
> Status: **Approved Candidate + P3.1 Patched** | Fase P3 — Architecture Documentation

---

## 1. Objetivo

Este documento define o **modelo de dados inicial** do RiukyOS Probe / RiukyOS Control Center.

A regra deste documento é:

> O JSON vem antes da struct Rust.  
> A struct Rust implementa o contrato do JSON, não o contrário.

O objetivo é garantir que:

- `riukyos-core` produza dados consistentes;
- `riukyos-probe` consiga imprimir e exportar relatórios;
- o futuro `RiukyOS Control Center` consiga consumir o mesmo modelo;
- campos falíveis não derrubem o relatório inteiro;
- privacidade e previsibilidade sejam parte do modelo desde o início.

---

## 2. Princípios do modelo de dados

### 2.1 JSON como contrato público

O `report.json` é o contrato mais importante do Probe.

Ele deve ser:

- estável;
- legível;
- versionado;
- fácil de serializar com `serde`;
- fácil de consumir pela futura UI;
- compatível com evolução futura via `schema_version`.

### 2.2 `Option<T>` para dados falíveis

Todo dado que pode falhar deve ser representado como `Option<T>`.

Exemplos:

- plano de energia pode falhar por locale, permissão ou ausência de comando;
- versão/build do Windows pode não ser coletada corretamente;
- uso instantâneo de CPU pode não estar disponível na primeira leitura;
- username é sempre opt-in.

Regra:

```text
Rust: Option<T>
report.json: null
report.txt: "indisponível"
```

### 2.3 Unidades fixas

Para evitar ambiguidade:

| Tipo de dado | Unidade no JSON |
|---|---|
| RAM | bytes |
| Disco | bytes |
| CPU usage | porcentagem `0.0` a `100.0` |
| Frequência CPU | MHz |
| Uptime | segundos |
| Timestamp | ISO 8601 UTC |
| PID | número inteiro |

A formatação humana, como `16 GB`, `42%` ou `3h 12min`, pertence ao CLI/UI, não ao contrato bruto.

### 2.4 Privacidade por padrão

O modelo não deve coletar dados pessoais automaticamente.

Regras:

- `username` é `Option<String>` e só entra no relatório por opt-in;
- linha de comando de processos não será coletada no Probe;
- variáveis de ambiente não serão coletadas;
- caminhos completos de usuário não serão coletados sem necessidade;
- o relatório deve ser previsível para o usuário.

### 2.5 Sem promessa de otimização

O modelo representa estado técnico.

Ele não deve conter campos como:

```text
performance_gain
fps_boost
optimization_score
```

Esses campos criam promessa sem validação. O RiukyOS mostra diagnóstico, não propaganda.

---

## 3. Estrutura raiz

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

Representação lógica:

```text
SystemReport
├── metadata
├── os
├── cpu
├── memory
├── disks[]
├── power?
└── processes[]
```

---

## 4. Exemplo fechado de `report.json`

Este é o contrato inicial do `report.json` para `schema_version = "0.1"`.

```json
{
  "metadata": {
    "schema_version": "0.1",
    "generated_at": "2026-05-25T07:30:00Z",
    "probe_version": "0.1.0",
    "hostname": "RIUKYOS-PC",
    "username": null
  },
  "os": {
    "name": "Windows",
    "version": "Windows 10 Pro",
    "build_number": "19045",
    "architecture": "x86_64",
    "uptime_seconds": 18432
  },
  "cpu": {
    "brand": "Intel(R) Xeon(R) CPU E5-2640 v3",
    "vendor": "GenuineIntel",
    "physical_cores": 8,
    "logical_cores": 16,
    "frequency_mhz": 2600,
    "usage_percent": 12.5
  },
  "memory": {
    "total_bytes": 17179869184,
    "available_bytes": 8589934592,
    "used_bytes": 8589934592,
    "usage_percent": 50.0
  },
  "disks": [
    {
      "name": "Windows",
      "mount_point": "C:\\",
      "file_system": "NTFS",
      "total_bytes": 255999918080,
      "available_bytes": 85899345920,
      "is_system_disk": true
    }
  ],
  "power": {
    "active_plan_name": "RiukyOS Performance",
    "active_plan_guid": "00000000-0000-0000-0000-000000000000",
    "source": "powercfg"
  },
  "processes": [
    {
      "pid": 1234,
      "name": "explorer.exe",
      "cpu_usage_percent": 1.2,
      "memory_bytes": 104857600
    },
    {
      "pid": 5678,
      "name": "Discord.exe",
      "cpu_usage_percent": 3.8,
      "memory_bytes": 524288000
    }
  ]
}
```

### Exemplo com dado indisponível

Se o plano de energia falhar:

```json
{
  "power": null
}
```

No `report.txt`, o mesmo campo deve aparecer como:

```text
Plano de energia: indisponível
```

---

## 5. Structs Rust iniciais

As structs devem usar `serde` desde o início.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
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

---

### 5.1 `ReportMetadata`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReportMetadata {
    pub schema_version: String,
    pub generated_at: String,
    pub probe_version: String,
    pub hostname: String,
    pub username: Option<String>,
}
```

| Campo | Tipo | Obrigatório | Observação |
|---|---:|---:|---|
| `schema_version` | `String` | Sim | Versão do contrato JSON. Inicial: `"0.1"` |
| `generated_at` | `String` | Sim | Timestamp ISO 8601 UTC |
| `probe_version` | `String` | Sim | Versão do CLI/app gerador |
| `hostname` | `String` | Sim | Nome do computador |
| `username` | `Option<String>` | Não | Opt-in, nunca coletado automaticamente |

Decisão:

```text
report_id fica fora até Safeguard.
```

Motivo:

```text
report_id só faz sentido com armazenamento/consulta persistente.
No Probe e 1, o relatório é arquivo avulso.
```

---

### 5.2 `OsInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OsInfo {
    pub name: String,
    pub version: Option<String>,
    pub build_number: Option<String>,
    pub architecture: Option<String>,
    pub uptime_seconds: Option<u64>,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `name` | `String` | Exemplo: `"Windows"` |
| `version` | `Option<String>` | Exemplo: `"Windows 10 Pro"` |
| `build_number` | `Option<String>` | Exemplo: `"19045"` |
| `architecture` | `Option<String>` | Exemplo: `"x86_64"` |
| `uptime_seconds` | `Option<u64>` | Uptime em segundos |

---

### 5.3 `CpuInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CpuInfo {
    pub brand: String,
    pub vendor: Option<String>,
    pub physical_cores: Option<usize>,
    pub logical_cores: Option<usize>,
    pub frequency_mhz: Option<u64>,
    pub usage_percent: Option<f32>,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `brand` | `String` | Nome comercial da CPU; obrigatório quando a seção `cpu` existe |
| `vendor` | `Option<String>` | Exemplo: `"GenuineIntel"` |
| `physical_cores` | `Option<usize>` | Núcleos físicos |
| `logical_cores` | `Option<usize>` | Threads/núcleos lógicos |
| `frequency_mhz` | `Option<u64>` | Frequência em MHz |
| `usage_percent` | `Option<f32>` | Uso entre `0.0` e `100.0`; exige amostragem confiável |

Nota:

```text
Uso de CPU exige amostragem entre leituras.
Foundation pode manter `usage_percent = null`.
Probe só preenche `usage_percent` após amostragem dupla confiável.
```

---

### 5.4 `MemoryInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MemoryInfo {
    pub total_bytes: u64,
    pub available_bytes: Option<u64>,
    pub used_bytes: Option<u64>,
    pub usage_percent: Option<f32>,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `total_bytes` | `u64` | Memória total em bytes |
| `available_bytes` | `Option<u64>` | Memória disponível em bytes |
| `used_bytes` | `Option<u64>` | Calculado quando possível |
| `usage_percent` | `Option<f32>` | Calculado quando possível |

Regra:

```text
used_bytes = total_bytes - available_bytes
usage_percent = used_bytes / total_bytes * 100
```

Se `available_bytes` não existir, `used_bytes` e `usage_percent` podem ser `None`.

---

### 5.5 `DiskInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DiskInfo {
    pub name: Option<String>,
    pub mount_point: String,
    pub file_system: Option<String>,
    pub total_bytes: u64,
    pub available_bytes: Option<u64>,
    pub is_system_disk: Option<bool>,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `name` | `Option<String>` | Nome/label do volume quando disponível |
| `mount_point` | `String` | Exemplo: `"C:\\"` |
| `file_system` | `Option<String>` | Exemplo: `"NTFS"` |
| `total_bytes` | `u64` | Tamanho total |
| `available_bytes` | `Option<u64>` | Espaço disponível |
| `is_system_disk` | `Option<bool>` | `true` para disco do sistema quando detectável |

Regra:

```text
Probe deve priorizar o disco C: no report.txt.
O JSON pode carregar todos os discos retornados pelo coletor.
```

---

### 5.6 `PowerPlanInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PowerPlanInfo {
    pub active_plan_name: Option<String>,
    pub active_plan_guid: String,
    pub source: String,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `active_plan_name` | `Option<String>` | Exemplo: `"RiukyOS Performance"`; opcional porque o nome pode variar por locale |
| `active_plan_guid` | `String` | GUID do plano; obrigatório quando `power` existe |
| `source` | `String` | Inicialmente `"powercfg"` |

Regra:

```text
O GUID é extraído por regex e é o dado estável.
Se o GUID falhar, `SystemReport.power = None`.
Se apenas o nome falhar, `active_plan_name = None`.
```

---

### 5.7 `ProcessInfo`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessInfo {
    pub pid: u32,
    pub name: String,
    pub cpu_usage_percent: Option<f32>,
    pub memory_bytes: Option<u64>,
}
```

| Campo | Tipo | Observação |
|---|---|---|
| `pid` | `u32` | PID do processo |
| `name` | `String` | Nome do processo |
| `cpu_usage_percent` | `Option<f32>` | Uso de CPU quando disponível |
| `memory_bytes` | `Option<u64>` | Memória usada pelo processo |

Privacidade:

```text
Probe não coleta command line completa do processo.
Probe não coleta caminho completo do executável.
```

Regra de apresentação:

```text
CLI/UI podem ordenar por CPU ou RAM.
O modelo armazena apenas a lista de processos coletada.
```

---

## 6. JSON vs TXT

### 6.1 `report.json`

O `report.json` é para:

- consumo por app;
- testes;
- compatibilidade futura;
- importação por ferramentas;
- evolução do schema.

Regras:

- `Option<T>` vira `null`;
- números ficam em unidade bruta;
- strings não são formatadas para estética;
- não incluir texto promocional;
- preservar nomes de campos em `snake_case`.

### 6.2 `report.txt`

O `report.txt` é para:

- leitura humana;
- cliente;
- suporte;
- comparação manual antes/depois.

Regras:

- `null` vira `"indisponível"`;
- bytes podem ser formatados como GB/MB;
- porcentagens podem ser arredondadas;
- texto deve ser claro e técnico;
- não deve prometer melhoria.

Detalhamento visual do `report.txt` será feito em:

```text
REPORT_SPEC.md
```

---

## 7. Política de evolução do schema

### 7.1 Versão inicial

```text
schema_version = "0.1"
```

### 7.2 Mudanças permitidas sem quebrar compatibilidade

- adicionar campo opcional;
- adicionar novo item em lista;
- adicionar nova seção opcional;
- melhorar preenchimento de campo antes `null`.

### 7.3 Mudanças que exigem nova versão de schema

- renomear campo;
- remover campo;
- trocar tipo de campo;
- mudar unidade de medida;
- transformar campo opcional em obrigatório.

### 7.4 Regra

```text
Se um report.json antigo não puder ser lido corretamente, o schema_version precisa mudar.
```

---

## 8. Política de validação

`DATA_MODEL.md` será considerado aprovado quando:

- [ ] o JSON de exemplo for válido;
- [ ] todas as structs principais estiverem descritas;
- [ ] campos falíveis estiverem como `Option<T>`;
- [ ] unidades estiverem explícitas;
- [ ] privacidade estiver documentada;
- [ ] `report_id` estiver fora até Safeguard;
- [ ] `username` estiver definido como opt-in;
- [ ] `schema_version` estiver presente em `ReportMetadata`.

---

## 9. Decisões fechadas

```text
ReportMetadata entra desde o início.
schema_version entra desde o início.
username é Option<String> e opt-in.
report_id fica fora até Safeguard.
JSON usa null para dado ausente.
TXT usa "indisponível" para dado ausente.
Unidades numéricas ficam em formato bruto no JSON.
Formatação humana pertence ao CLI/UI.
```


---

## 10. P3.1 — Políticas de coleta dos campos

### 10.1 `CpuInfo.usage_percent`

`usage_percent` é falível.

Motivo:

```text
Uso de CPU depende de amostragem entre duas leituras.
Uma única leitura pode retornar 0%, valor instável ou valor não representativo.
```

Decisão por fase:

```text
Foundation:
- `usage_percent` pode ser `null`.

Probe:
- `usage_percent` só deve ser preenchido após amostragem dupla.
- Se a amostragem falhar, manter `null`.
```

Modelo recomendado:

```rust
pub struct CpuInfo {
    pub brand: String,
    pub vendor: Option<String>,
    pub physical_cores: Option<usize>,
    pub logical_cores: Option<usize>,
    pub frequency_mhz: Option<u64>,
    pub usage_percent: Option<f32>,
}
```

### 10.2 `PowerPlanInfo`

O plano de energia pode falhar por locale, permissão ou ausência do comando.

Política:

```text
- o campo raiz `power` é `Option<PowerPlanInfo>`;
- GUID é o dado mais estável;
- nome do plano é opcional;
- se GUID não for extraído, `power = null`.
```

Modelo recomendado:

```rust
pub struct PowerPlanInfo {
    pub active_plan_name: Option<String>,
    pub active_plan_guid: String,
    pub source: String,
}
```

### 10.3 `DiskInfo.is_system_disk`

Estratégia Probe:

```text
1. ler `SystemDrive`;
2. normalizar para `C:\`;
3. comparar com `mount_point`;
4. fallback para `C:\`.
```

Se não for possível determinar:

```text
is_system_disk = false
```

exceto no fallback explícito de `C:\`.

### 10.4 Dados de exemplo

Relatórios de exemplo públicos devem ser fictícios.

Nomes aprovados:

```text
report-example.mock.json
report-example.mock.txt
```

Não usar relatório gerado na máquina real do desenvolvedor como exemplo público.

### 10.5 Consistência entre seções

As definições principais da seção 5 prevalecem como contrato final do Probe.

As políticas P3.1 da seção 10 foram incorporadas às structs principais:

```text
CpuInfo.brand = String
CpuInfo.usage_percent = Option<f32>
PowerPlanInfo.active_plan_guid = String
PowerPlanInfo.active_plan_name = Option<String>
SystemReport.power = Option<PowerPlanInfo>
```
---

*Próximo documento: `CLI_SPEC.md`*

# REPORT_SPEC.md
> RiukyOS LAB — Especificação dos relatórios `report.json` e `report.txt`  
> Status: **Draft Approved Candidate + P3.1 Patched** | Documento 5/7 — Fase P3 Architecture Documentation

---

## 1. Objetivo

Este documento define o formato oficial dos relatórios gerados pelo `riukyos-probe report`.

O Probe gera dois arquivos:

```text
report.json
report.txt
```

Cada arquivo tem uma função diferente:

| Arquivo | Público principal | Função |
|---|---|---|
| `report.json` | máquina / app / testes | contrato estruturado, versionado e consumível pelo futuro Control Center |
| `report.txt` | humano / cliente / suporte | resumo legível do estado técnico do sistema |

Regra central:

```text
report.json é o contrato técnico.
report.txt é a apresentação humana.
```

---

## 2. Relação com outros documentos

Este documento depende de:

- `PROJECT_DECISION.md`
- `ARCHITECTURE.md`
- `DATA_MODEL.md`
- `CLI_SPEC.md`

Decisões herdadas:

- `SystemReport` é produzido pelo `riukyos-core`;
- `apps/probe` apenas chama o core e escolhe onde salvar;
- `report_id` fica fora até Safeguard;
- `schema_version` entra desde o início;
- `username` é opt-in e fica `null` no Probe;
- dado falível vira `null` no JSON;
- dado falível vira `"indisponível"` no TXT;
- nenhuma ação modificadora ocorre antes do Safeguard.

---

## 3. Comando gerador

Comando padrão:

```powershell
riukyos-probe report
```

Com pasta de saída:

```powershell
riukyos-probe report --output C:\Users\User\Desktop\RiukyOSReport
```

Atalho:

```powershell
riukyos-probe report -o C:\Users\User\Desktop\RiukyOSReport
```

Arquivos gerados:

```text
report.json
report.txt
```

No Probe, os nomes são fixos.

Nomes com timestamp, histórico, banco de dados ou `report_id` ficam fora até Safeguard+.

---

## 4. Política de saída em disco

### 4.1 Diretório padrão

Se `--output` não for informado:

```text
.
```

Ou seja: diretório atual do terminal.

### 4.2 Diretório informado

Se o usuário informar `--output`, o CLI deve:

1. verificar se a pasta existe;
2. criar a pasta se ela não existir;
3. tentar escrever `report.json`;
4. tentar escrever `report.txt`;
5. informar os caminhos gerados.

### 4.3 Sobrescrita

No Probe, o comportamento aprovado é:

```text
Sobrescrever report.json e report.txt se já existirem.
```

Motivos:

- reduz complexidade;
- evita versionamento prematuro;
- `report_id` está fora até Safeguard;
- histórico persistente está fora até Safeguard.

O CLI deve avisar no terminal quando sobrescrever:

```text
Warning: existing report.json/report.txt were overwritten.
```

### 4.4 Falhas de escrita

Se não conseguir criar pasta ou escrever arquivo, o comando deve:

- imprimir erro em `stderr`;
- retornar exit code `1`;
- não gerar relatório parcial silencioso.

Exemplo:

```text
Error: failed to write report.json.
```

---

## 5. `report.json`

### 5.1 Função

O `report.json` é o contrato estruturado do projeto.

Ele serve para:

- testes automatizados;
- importação futura pelo Control Center;
- comparação técnica;
- evolução de schema;
- depuração;
- integração com ferramentas futuras.

### 5.2 Regras de formato

O JSON deve seguir estas regras:

- codificação UTF-8;
- indentação de 2 espaços;
- campos em `snake_case`;
- campos ausentes por falha usam `null`;
- números usam unidades brutas;
- bytes permanecem em bytes;
- uptime permanece em segundos;
- timestamp usa ISO 8601 UTC;
- não contém texto promocional;
- não contém promessa de ganho;
- não contém caminhos pessoais desnecessários;
- não contém command line completa de processos;
- não contém variáveis de ambiente;
- não contém histórico de navegação;
- não contém dados sensíveis.

### 5.3 Ordem dos campos

A ordem dos campos deve permanecer estável para facilitar revisão visual e diff.

Ordem raiz:

```text
metadata
os
cpu
memory
disks
power
processes
```

Ordem de `metadata`:

```text
schema_version
generated_at
probe_version
hostname
username
```

### 5.4 Unidades no JSON

| Campo | Unidade |
|---|---|
| `*_bytes` | bytes |
| `*_percent` | percentual `0.0` a `100.0` |
| `frequency_mhz` | MHz |
| `uptime_seconds` | segundos |
| `generated_at` | ISO 8601 UTC |
| `pid` | número inteiro |

---

## 6. Exemplo oficial de `report.json`

Este é o contrato inicial para:

```text
schema_version = "0.1"
```

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

---

## 7. Exemplo com campos indisponíveis

Se a coleta do plano de energia falhar:

```json
{
  "power": null
}
```

Se a coleta de uso de CPU não for confiável na primeira leitura:

```json
{
  "cpu": {
    "brand": "Intel(R) Xeon(R) CPU E5-2640 v3",
    "vendor": "GenuineIntel",
    "physical_cores": 8,
    "logical_cores": 16,
    "frequency_mhz": 2600,
    "usage_percent": null
  }
}
```

Se o processo não retornar uso de CPU:

```json
{
  "pid": 1234,
  "name": "explorer.exe",
  "cpu_usage_percent": null,
  "memory_bytes": 104857600
}
```

Regra:

```text
Ausência parcial não invalida o relatório.
```

---

## 8. `report.txt`

### 8.1 Função

O `report.txt` é a versão humana do relatório.

Ele serve para:

- atendimento técnico;
- leitura rápida;
- envio para cliente;
- comparação manual antes/depois;
- documentação simples do estado do PC.

### 8.2 Idioma

No Probe, o `report.txt` usa **português técnico simples**.

Motivos:

- o RiukyOS LAB começa com foco prático em clientes brasileiros;
- `"indisponível"` já é a palavra padrão decidida para campos ausentes;
- o relatório humano precisa ser claro para cliente/suporte local.

Observação:

```text
A saída terminal do CLI pode usar inglês técnico simples.
O report.txt usa português técnico simples.
Multi-idioma/i18n fica fora do escopo até revisão explícita.
```

### 8.3 Regras de formato

O TXT deve:

- ser UTF-8;
- ter seções claras;
- usar linhas simples;
- não usar caracteres que possam quebrar em CMD/PowerShell antigo;
- formatar bytes como GB/MB;
- formatar percentuais com 1 casa decimal;
- transformar `null` em `"indisponível"`;
- não prometer melhoria;
- não usar linguagem de marketing;
- não atribuir culpa ao usuário;
- não conter dados pessoais automáticos.

### 8.4 Seções obrigatórias

Ordem do `report.txt`:

```text
1. Cabeçalho
2. Metadata
3. Sistema
4. CPU
5. Memória
6. Discos
7. Energia
8. Processos
9. Observações
```

---

## 9. Exemplo oficial de `report.txt`

```text
RiukyOS Probe Report
=====================

Metadata
--------
Schema: 0.1
Gerado em: 2026-05-25T07:30:00Z
Probe: 0.1.0
Hostname: RIUKYOS-PC
Usuário: indisponível

Sistema
-------
OS: Windows
Versão: Windows 10 Pro
Build: 19045
Arquitetura: x86_64
Uptime: 5h 7m

CPU
---
Modelo: Intel(R) Xeon(R) CPU E5-2640 v3
Fabricante: GenuineIntel
Núcleos físicos: 8
Núcleos lógicos: 16
Frequência: 2600 MHz
Uso atual: 12.5%

Memória
-------
Total: 16.0 GB
Disponível: 8.0 GB
Usada: 8.0 GB
Uso: 50.0%

Discos
------
[1] C:\
Nome: Windows
Sistema de arquivos: NTFS
Total: 238.4 GB
Disponível: 80.0 GB
Disco do sistema: sim

Energia
-------
Plano ativo: RiukyOS Performance
GUID: 00000000-0000-0000-0000-000000000000
Fonte: powercfg

Processos
---------
Top processos coletados: 2

PID     CPU%    Memória     Nome
1234    1.2     100.0 MB    explorer.exe
5678    3.8     500.0 MB    Discord.exe

Observações
-----------
Este relatório mostra estado técnico do sistema no momento da coleta.
Nenhuma alteração foi feita no sistema.
Este relatório não promete ganho de desempenho.
```

---

## 10. Exemplo de `report.txt` com dado indisponível

```text
Energia
-------
Plano ativo: indisponível
GUID: indisponível
Fonte: indisponível
```

```text
CPU
---
Modelo: Intel(R) Xeon(R) CPU E5-2640 v3
Fabricante: GenuineIntel
Núcleos físicos: 8
Núcleos lógicos: 16
Frequência: 2600 MHz
Uso atual: indisponível
```

---

## 11. Formatação humana

### 11.1 Bytes

Função de formatação esperada:

```text
bytes → B / KB / MB / GB / TB
```

Regras:

- usar 1 casa decimal para MB ou maior;
- usar ponto decimal, não vírgula;
- manter unidade explícita.

Exemplos:

```text
104857600 → 100.0 MB
17179869184 → 16.0 GB
255999918080 → 238.4 GB
```

### 11.2 Percentual

```text
12.5%
50.0%
```

Se ausente:

```text
indisponível
```

### 11.3 Uptime

Formato humano curto:

```text
5h 7m
3d 2h 10m
```

Se ausente:

```text
indisponível
```

### 11.4 Boolean

No `report.txt`:

```text
true  → sim
false → não
null  → indisponível
```

No `report.json`, manter boolean nativo:

```json
true
false
null
```

---

## 12. Processos no relatório

### 12.1 Quantidade padrão

O `report.txt` deve mostrar, por padrão:

```text
Top 15 processos
```

A lista vem do `SystemReport.processes`.

### 12.2 Ordenação

No Probe, o padrão aprovado é:

```text
Ordenar por memória usada, de maior para menor.
```

Motivo:

```text
CPU é instável em leitura instantânea.
RAM é mais estável para diagnóstico inicial.
```

### 12.3 Privacidade

O relatório de processos não inclui:

- command line completa;
- caminho completo do executável;
- argumentos;
- diretório de trabalho;
- usuário dono do processo.

Campos permitidos:

- PID;
- nome;
- CPU%;
- memória usada.

---

## 13. Validação do `report.json`

O Probe precisa validar pelo menos:

- arquivo existe;
- arquivo é UTF-8;
- JSON parseia corretamente;
- contém `metadata.schema_version`;
- contém `metadata.generated_at`;
- contém `metadata.probe_version`;
- contém `metadata.hostname`;
- contém seções raiz obrigatórias.

Validação manual possível:

```powershell
Get-Content .\report.json | ConvertFrom-Json | Out-Null
```

Validação via Rust futura:

```text
serde_json::from_str::<SystemReport>()
```

---

## 14. Validação do `report.txt`

O Probe precisa validar:

- arquivo existe;
- arquivo é UTF-8;
- cabeçalho existe;
- seções obrigatórias aparecem;
- campos indisponíveis aparecem como `"indisponível"`;
- observações finais aparecem;
- o texto não promete ganho de desempenho.

Busca simples esperada:

```powershell
Select-String -Path .\report.txt -Pattern "RiukyOS Probe Report"
Select-String -Path .\report.txt -Pattern "Nenhuma alteração foi feita"
Select-String -Path .\report.txt -Pattern "não promete ganho"
```

---

## 15. Política de compatibilidade

### 15.1 `schema_version = "0.1"`

Esta é a versão inicial do contrato.

### 15.2 Mudanças compatíveis

São compatíveis sem mudar major conceptualmente:

- adicionar campo opcional;
- adicionar seção opcional;
- melhorar preenchimento de campos antes `null`;
- adicionar novos processos à lista;
- adicionar discos à lista.

### 15.3 Mudanças que exigem revisão de schema

Exigem nova decisão documentada:

- renomear campo;
- remover campo;
- trocar tipo de campo;
- mudar unidade;
- mudar semântica de campo;
- transformar opcional em obrigatório.

---

## 16. Política de confiança

O relatório deve manter a postura RiukyOS:

```text
diagnóstico
clareza
privacidade
sem promessa falsa
sem alteração invisível
```

O relatório não deve conter:

```text
FPS boost
latency reduced
optimization score
PC optimized
performance guaranteed
```

Formulação permitida:

```text
Este relatório mostra estado técnico do sistema no momento da coleta.
Nenhuma alteração foi feita no sistema.
Este relatório não promete ganho de desempenho.
```

---

## 17. Checklist de aprovação do REPORT_SPEC

Este documento está aprovado quando:

- [ ] define função de `report.json`;
- [ ] define função de `report.txt`;
- [ ] contém exemplo oficial de JSON;
- [ ] contém exemplo oficial de TXT;
- [ ] define tratamento de `null`/`indisponível`;
- [ ] define unidades;
- [ ] define política de sobrescrita;
- [ ] define política de privacidade;
- [ ] define validação mínima;
- [ ] mantém `report_id` fora até Safeguard;
- [ ] não inclui promessa de otimização.


---

## 18. P3.1 — Política de exemplos e campos instáveis

### 18.1 Uso de CPU

`cpu.usage_percent` pode ser `null`.

Quando `null`:

No `report.json`:

```json
"usage_percent": null
```

No `report.txt`:

```text
Uso atual: indisponível
```

Apenas preencher valor numérico quando houver amostragem confiável.

### 18.2 Plano de energia

Se o GUID for extraído mas o nome não:

```json
"power": {
  "active_plan_name": null,
  "active_plan_guid": "00000000-0000-0000-0000-000000000000",
  "source": "powercfg"
}
```

No TXT:

```text
Plano ativo: indisponível
GUID: 00000000-0000-0000-0000-000000000000
Fonte: powercfg
```

Se nem GUID for extraído:

```json
"power": null
```

### 18.3 Exemplos públicos

Não publicar `report.json` real gerado na máquina do desenvolvedor como exemplo.

Se a release incluir exemplos:

```text
report-example.mock.json
report-example.mock.txt
```

Requisitos:

```text
- hostname fictício;
- username null;
- dados técnicos fictícios ou genéricos;
- nenhum caminho pessoal;
- nenhuma spec real obrigatória do PC do dev.
```

### 18.4 Compatibilidade do schema

A presença de `null` em campos falíveis faz parte do contrato `schema_version = "0.1"`.

Consumidores do JSON devem aceitar `null` para:

```text
cpu.usage_percent
power
power.active_plan_name
processes[].cpu_usage_percent
```
---

*Próximo documento: `UI_SPEC.md`*

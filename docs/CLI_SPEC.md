# CLI_SPEC.md
> RiukyOS LAB — Especificação do CLI RiukyOS Probe  
> Status: **Draft Approved Candidate + P3.1 Patched** | Documento 4/7 — Fase P3 Architecture Documentation

---

## 1. Objetivo

O `riukyos-probe` é o primeiro produto funcional do workspace RiukyOS.

Ele é um CLI de diagnóstico para Windows que consome `riukyos-core` para:

- ler informações do sistema;
- exibir dados técnicos no terminal;
- gerar `report.json`;
- gerar `report.txt`;
- validar a arquitetura antes do app visual Tauri.

O CLI existe por dois motivos:

1. Provar a lógica do `riukyos-core` antes da interface gráfica.
2. Criar uma ferramenta útil e demonstrável já no Probe.

---

## 2. Nome do binário

```text
riukyos-probe.exe
```

Uso base:

```powershell
riukyos-probe <command> [options]
```

Comandos fechados do Probe:

```text
riukyos-probe system
riukyos-probe processes
riukyos-probe power
riukyos-probe report
```

---

## 3. Princípios do CLI

### 3.1 Leitura antes de modificação

Probe é exclusivamente leitura, diagnóstico e exportação.

O CLI **não modifica o sistema**.

Não pode:

- remover arquivos;
- alterar serviços;
- alterar registro;
- alterar plano de energia;
- otimizar RAM;
- limpar temporários;
- aplicar perfil de otimização.

Qualquer ação modificadora fica fora até Safeguard.

---

### 3.2 O terminal é interface, não domínio

`apps/probe` pode:

- parsear argumentos;
- formatar saída para humano;
- imprimir no terminal;
- definir exit code;
- escolher pasta de saída.

`apps/probe` não pode:

- chamar `sysinfo` diretamente;
- chamar `winreg` diretamente;
- chamar PowerShell diretamente para coleta;
- montar manualmente `SystemReport`;
- duplicar lógica do `riukyos-core`.

Regra:

```text
Toda coleta vem do riukyos-core.
Todo comando do CLI é apenas uma casca em volta do core.
```

---

### 3.3 Saída previsível

O CLI deve produzir saída legível e estável.

- Dados humanos vão para `stdout`.
- Erros vão para `stderr`.
- Arquivos gerados devem ter nomes previsíveis.
- O CLI nunca deve imprimir stack trace para usuário comum.
- Logs detalhados ficam para modo futuro, não obrigatório no Probe.

---

### 3.4 Falha parcial não derruba relatório

Se um campo falhar:

- no `report.json`, o valor vira `null`;
- no `report.txt`, o valor aparece como `"indisponível"`;
- o comando continua sempre que possível.

Exemplo:

```text
Plano de energia: indisponível
```

Apenas erros estruturais devem encerrar com exit code diferente de zero, como:

- pasta de saída inacessível;
- permissão negada para escrever relatório;
- falha inesperada ao serializar JSON;
- erro crítico não recuperável no core.

---

## 4. Dependência de CLI

O CLI usa `clap` com derive.

Dependência:

```toml
clap = { version = "4", features = ["derive"] }
```

Estrutura esperada em `apps/probe/src/cli.rs`:

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "riukyos-probe")]
#[command(about = "RiukyOS Probe — diagnostic CLI for Windows")]
#[command(version)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    System,
    Processes {
        #[arg(short, long, default_value_t = 15)]
        top: usize,
    },
    Power,
    Report {
        #[arg(short, long, default_value = ".")]
        output: String,
    },
}
```

Este exemplo é contrato de intenção, não implementação final obrigatória.

---

## 5. Comando `system`

### 5.1 Objetivo

Exibir resumo técnico do sistema atual.

```powershell
riukyos-probe system
```

### 5.2 Dados exibidos

O comando deve exibir, quando disponível:

- nome do sistema operacional;
- versão/build do Windows;
- hostname;
- uptime;
- CPU;
- uso de CPU;
- RAM total;
- RAM livre/disponível;
- disco C: total;
- disco C: livre.

### 5.3 Exemplo de saída

```text
RiukyOS Probe — System

OS: Windows 10 Pro
Version: 22H2 / build 19045
Hostname: DESKTOP-RIUKY
Uptime: 3h 42m

CPU: Intel Xeon E5-2640 v3
CPU Usage: 12.5%

Memory: 16.0 GB total / 8.2 GB available

Disk C:
  Total: 237.0 GB
  Free: 80.4 GB
```

### 5.4 Comportamento em erro

Se algum dado estiver indisponível:

```text
Version: indisponível
```

O comando só falha completamente se o `riukyos-core` não conseguir montar a resposta mínima.

### 5.5 Critério de aprovação

```text
riukyos-probe system
```

Aprovado quando:

- [ ] executa sem crash;
- [ ] imprime cabeçalho `RiukyOS Probe — System`;
- [ ] exibe CPU, RAM e disco;
- [ ] usa `"indisponível"` para campos ausentes;
- [ ] não modifica o sistema.

---

## 6. Comando `processes`

### 6.1 Objetivo

Exibir os processos mais relevantes por uso de CPU/RAM.

```powershell
riukyos-probe processes
```

Com limite customizado:

```powershell
riukyos-probe processes --top 10
```

Atalho:

```powershell
riukyos-probe processes -t 10
```

### 6.2 Regra de ordenação

No Probe, a ordenação padrão será por RAM usada, de maior para menor.

CPU pode variar muito no momento da coleta; RAM tende a ser mais estável para diagnóstico inicial.

### 6.3 Dados exibidos

Para cada processo:

- PID;
- nome;
- CPU%;
- memória usada;
- status quando disponível.

### 6.4 Exemplo de saída

```text
RiukyOS Probe — Processes
Top: 15
Sorted by: memory

PID      CPU%    Memory       Name
---------------------------------------------
8420     4.2     512.3 MB     chrome.exe
3180     1.1     280.8 MB     explorer.exe
1216     0.0     190.4 MB     Discord.exe
```

### 6.5 Comportamento em erro

Se a lista de processos não puder ser coletada:

```text
Erro: não foi possível coletar lista de processos.
```

Exit code:

```text
1
```

### 6.6 Critério de aprovação

```text
riukyos-probe processes
```

Aprovado quando:

- [ ] executa sem crash;
- [ ] lista até 15 processos por padrão;
- [ ] aceita `--top`;
- [ ] exibe PID, nome, CPU e memória;
- [ ] não modifica processos;
- [ ] não finaliza nenhum processo.

---

## 7. Comando `power`

### 7.1 Objetivo

Exibir o plano de energia ativo.

```powershell
riukyos-probe power
```

### 7.2 Dados exibidos

Quando disponível:

- nome do plano ativo;
- GUID do plano ativo;
- fonte da coleta.

Exemplo de fonte:

```text
powercfg
```

### 7.3 Exemplo de saída

```text
RiukyOS Probe — Power

Active plan: RiukyOS Performance
GUID: 00000000-0000-0000-0000-000000000000
Source: powercfg
```

Se indisponível:

```text
RiukyOS Probe — Power

Active plan: indisponível
GUID: indisponível
Source: powercfg
```

### 7.4 Estratégia de coleta

A coleta do plano de energia pode usar PowerShell ou `powercfg` através do `riukyos-core`.

O CLI não chama `powercfg` diretamente.

Regra:

```text
PowerShell/powercfg são detalhe do core, não do CLI.
```

### 7.5 Comportamento em erro

Se o plano não puder ser lido, o comando ainda deve executar e exibir `"indisponível"`.

Exit code continua `0` se a falha for apenas campo ausente.

Exit code vira `1` apenas se ocorrer erro estrutural no comando.

### 7.6 Critério de aprovação

```text
riukyos-probe power
```

Aprovado quando:

- [ ] executa sem crash;
- [ ] mostra plano de energia quando disponível;
- [ ] mostra `"indisponível"` quando falhar;
- [ ] não altera plano de energia;
- [ ] não executa otimização.

---

## 8. Comando `report`

### 8.1 Objetivo

Gerar relatório completo do sistema em dois formatos:

```text
report.json
report.txt
```

Uso básico:

```powershell
riukyos-probe report
```

Uso com pasta de saída:

```powershell
riukyos-probe report --output C:\Users\User\Desktop\RiukyOSReport
```

Atalho:

```powershell
riukyos-probe report -o C:\Users\User\Desktop\RiukyOSReport
```

### 8.2 Arquivos gerados

O comando gera:

```text
report.json
report.txt
```

No Probe, nomes fixos são suficientes.

Nomes com timestamp ficam para revisão futura se necessário.

### 8.3 Exemplo de saída no terminal

```text
RiukyOS Probe — Report

Output directory:
C:\Users\User\Desktop\RiukyOSReport

Generated:
- report.json
- report.txt

Status: OK
```

### 8.4 Comportamento se arquivos já existirem

No Probe, comportamento padrão:

```text
Sobrescrever report.json e report.txt se já existirem.
```

Motivo:

- reduz complexidade inicial;
- evita lógica prematura de versionamento;
- `report_id` está fora do escopo até Safeguard.

O CLI deve exibir aviso:

```text
Aviso: report.json/report.txt existentes foram sobrescritos.
```

### 8.5 Comportamento em erro

Se a pasta de saída não existir:

Opção aprovada para Probe:

```text
Criar a pasta automaticamente.
```

Se não conseguir criar:

```text
Erro: não foi possível criar pasta de saída.
```

Exit code:

```text
1
```

Se não conseguir escrever arquivo:

```text
Erro: não foi possível escrever report.json/report.txt.
```

Exit code:

```text
1
```

### 8.6 Critério de aprovação

```text
riukyos-probe report
```

Aprovado quando:

- [ ] gera `report.json`;
- [ ] gera `report.txt`;
- [ ] JSON é válido;
- [ ] TXT é legível;
- [ ] cria pasta de saída se necessário;
- [ ] informa caminho dos arquivos gerados;
- [ ] não modifica o sistema.

---

## 9. Exit codes

| Código | Significado |
|---:|---|
| `0` | Comando executado com sucesso |
| `1` | Erro de execução |
| `2` | Erro de uso/argumento inválido gerado pelo `clap` |

Observação:

`clap` pode controlar automaticamente erros de argumento e `--help`.

---

## 10. Padrão de mensagens

### 10.1 Cabeçalho

Todo comando deve começar com:

```text
RiukyOS Probe — <Command>
```

Exemplos:

```text
RiukyOS Probe — System
RiukyOS Probe — Processes
RiukyOS Probe — Power
RiukyOS Probe — Report
```

### 10.2 Idioma

Probe usa inglês técnico simples nas saídas do CLI.

Motivo:

- facilita README internacional;
- evita problemas de encoding;
- combina com nomes técnicos de campos;
- logs e exemplos ficam mais padronizados.

A documentação pode continuar em português durante a fase de projeto.

### 10.3 Unidade de medida

Padrões:

| Tipo | Unidade |
|---|---|
| RAM | GB com 1 casa decimal |
| Disco | GB com 1 casa decimal |
| CPU | percentual com 1 casa decimal |
| Uptime | formato humano curto |

Exemplos:

```text
16.0 GB
237.0 GB
12.5%
3h 42m
```

---

## 11. Help e version

### 11.1 Help geral

```powershell
riukyos-probe --help
```

Deve listar:

```text
Usage: riukyos-probe <COMMAND>

Commands:
  system     Show system summary
  processes  Show top processes
  power      Show active power plan
  report     Generate report.json and report.txt
  help       Print this message or the help of the given subcommand(s)
```

### 11.2 Version

```powershell
riukyos-probe --version
```

Exemplo:

```text
riukyos-probe 0.1.0
```

A versão deve ser a mesma declarada em `apps/probe/Cargo.toml`.

---

## 12. Segurança e privacidade

### 12.1 Sem coleta pessoal automática

O CLI não coleta automaticamente:

- nome real do usuário;
- e-mail;
- documentos;
- caminhos pessoais além dos necessários para gerar relatório;
- conteúdo de arquivos;
- histórico de navegação;
- chaves sensíveis do registro.

### 12.2 `username`

`username` no `ReportMetadata` é `Option<String>` e só pode ser preenchido por opt-in futuro.

No Probe:

```text
username: null
```

### 12.3 Transparência

O comando `report` deve deixar claro quais arquivos foram gerados e onde.

O relatório deve representar estado técnico, não julgamento pessoal do usuário.

---

## 13. Relação com documentos futuros

Este documento depende de:

- `PROJECT_DECISION.md`
- `ARCHITECTURE.md`
- `DATA_MODEL.md`

Este documento será detalhado por:

- `REPORT_SPEC.md` — formato final de `report.json` e `report.txt`;
- `VALIDATION_PLAN.md` — comandos exatos de validação por fase;
- `UI_SPEC.md` — reaproveitamento dos mesmos dados na interface visual.

---

## 14. Checklist de aprovação do CLI_SPEC

Este documento está aprovado quando:

- [ ] os quatro comandos do Probe estão definidos;
- [ ] cada comando tem objetivo, saída e comportamento em erro;
- [ ] exit codes estão definidos;
- [ ] política de sobrescrita do `report` está definida;
- [ ] `apps/probe` não contém lógica de coleta;
- [ ] privacidade básica está documentada;
- [ ] nenhuma ação modificadora existe antes do Safeguard.


---

## 15. P3.1 — Precisões técnicas para implementação

### 15.1 CPU usage no comando `system`

No Probe, `riukyos-probe system` só deve exibir uso de CPU real quando o core tiver feito amostragem dupla.

Se não houver amostragem confiável:

```text
CPU Usage: indisponível
```

### 15.2 Parse do plano de energia

O comando `riukyos-probe power` recebe os dados do core.

Regra do core:

```text
GUID por regex.
Nome do plano opcional.
Locale nunca pode quebrar o comando inteiro.
```

Se o nome falhar mas o GUID existir:

```text
Active plan: indisponível
GUID: <guid extraído>
Source: powercfg
```

### 15.3 Workflow de teste

O CLI pode ser desenvolvido no Arch, mas aprovação exige execução do `.exe` no Windows/VM.

Cross-compilação será tratada como spike auxiliar:

```text
se funcionar, acelera o loop;
se atrapalhar, é adiada.
```

### 15.4 `Cargo.lock`

Como o CLI é binário, o `Cargo.lock` deve ser versionado no repositório.

### 15.5 Validação Rust padrão

Antes de considerar Probe aprovado:

```powershell
cargo fmt --check
cargo check
cargo clippy -- -D warnings
cargo test
cargo build --release
```
---

*Próximo documento: `REPORT_SPEC.md`*

# UI_SPEC.md
> RiukyOS LAB — Especificação funcional da interface do RiukyOS Control Center  
> Status: **Draft Approved Candidate + P3.1 Patched** | Documento 6/7 — Fase P3 Architecture Documentation

---

## 1. Objetivo

Este documento define a interface funcional do **RiukyOS Control Center** para o Control.

Ele descreve:

- telas do MVP visual;
- dados exibidos;
- ações disponíveis;
- estados vazios;
- estados de erro;
- relação entre frontend Svelte, Tauri e `riukyos-core`.

Este documento **não define a identidade visual final**.

A estética, paleta, tipografia, componentes, animações, ícones e linguagem visual detalhada serão definidos em:

```text
BRAND_SPEC.md
```

Regra central:

```text
UI_SPEC.md define o que a interface faz.
BRAND_SPEC.md define como a interface parece.
```

---

## 2. Escopo do Control

O Control cria o MVP visual do RiukyOS Control Center com Tauri + Svelte + TypeScript.

Telas fechadas:

1. Dashboard
2. Processos
3. Diagnóstico
4. Exportar
5. Sobre

Nenhuma tela extra entra no Control.

Qualquer nova tela ou feature vai para backlog.

---

## 3. Pré-requisitos

Antes de iniciar o frontend do Control:

- `PROJECT_DECISION.md` aprovado;
- `ARCHITECTURE.md` aprovado;
- `DATA_MODEL.md` aprovado;
- `CLI_SPEC.md` aprovado;
- `REPORT_SPEC.md` aprovado;
- `VALIDATION_PLAN.md` aprovado;
- `BRAND_SPEC.md` aprovado para decisões visuais mínimas.

Observação:

```text
Foundation e Probe não dependem de BRAND_SPEC.md.
Control frontend depende de BRAND_SPEC.md.
```

---

## 4. Princípios da interface

### 4.1 Leitura antes de ação

No Control, o Control Center é uma interface de diagnóstico e exportação.

Ele não modifica o sistema.

Não pode:

- limpar arquivos;
- alterar serviços;
- alterar registro;
- trocar plano de energia;
- finalizar processos;
- aplicar perfil de otimização;
- executar scripts de otimização.

A primeira ação modificadora fica para Safeguard:

```text
limpeza segura de temporários
```

---

### 4.2 Frontend não acessa o sistema

O frontend Svelte nunca acessa diretamente o sistema operacional.

Fluxo obrigatório:

```text
Frontend Svelte
    ↓ invoke(...)
Tauri command
    ↓
riukyos-core
    ↓
SystemReport / dados estruturados
    ↓
Tauri command
    ↓
Frontend Svelte
```

O frontend só consome dados já processados pelo backend Rust.

---

### 4.3 Interface orientada a estado

Toda tela precisa considerar:

- carregando;
- dados disponíveis;
- dado parcial;
- erro recuperável;
- erro estrutural;
- estado vazio.

Nenhuma tela deve quebrar por campo `null`.

No UI:

```text
null → indisponível
```

---

### 4.4 Sem promessas de ganho

A interface não deve usar linguagem como:

```text
FPS boost
PC optimized
latency reduced
performance guaranteed
```

Formulações permitidas:

```text
Diagnóstico do sistema
Estado atual
Relatório técnico
Nenhuma alteração foi feita
Dados indisponíveis
```

---

## 5. Layout funcional geral

A estrutura funcional do app:

```text
┌───────────────────────────────────────────────┐
│ RiukyOS Control Center                         │
├───────────────┬───────────────────────────────┤
│ Navegação     │ Conteúdo da tela atual         │
│               │                               │
│ Dashboard     │                               │
│ Processos     │                               │
│ Diagnóstico   │                               │
│ Exportar      │                               │
│ Sobre         │                               │
└───────────────┴───────────────────────────────┘
```

A navegação pode ser lateral ou superior. A escolha visual final fica para `BRAND_SPEC.md`.

Funcionalmente, a navegação precisa:

- mostrar tela ativa;
- permitir alternar entre as 5 telas;
- não perder dados já carregados durante troca de tela, se possível;
- continuar funcional sem internet.

---

## 6. Dados globais do app

O app deve manter um estado global mínimo:

```ts
type AppState = {
  currentReport: SystemReport | null;
  lastGeneratedAt: string | null;
  isLoading: boolean;
  errorMessage: string | null;
};
```

Este exemplo é contrato conceitual, não implementação obrigatória.

### 6.1 `currentReport`

Contém o último `SystemReport` gerado ou carregado.

Usado por:

- Dashboard;
- Processos;
- Diagnóstico;
- Exportar.

### 6.2 `lastGeneratedAt`

Permite mostrar quando os dados foram coletados.

Exemplo:

```text
Último diagnóstico: 2026-05-25T07:30:00Z
```

### 6.3 `isLoading`

Usado quando o backend está coletando dados.

### 6.4 `errorMessage`

Mensagem de erro visível ao usuário quando uma ação falha.

---

## 7. Tela 1 — Dashboard

### 7.1 Objetivo

Mostrar um resumo técnico do estado atual do sistema.

A Dashboard é a tela inicial do app.

### 7.2 Dados exibidos

A tela deve exibir, quando disponível:

#### Sistema

- OS;
- versão do Windows;
- build;
- arquitetura;
- hostname;
- uptime.

#### CPU

- modelo;
- fabricante;
- núcleos físicos;
- núcleos lógicos;
- frequência;
- uso atual.

#### Memória

- total;
- disponível;
- usada;
- uso percentual.

#### Disco do sistema

- unidade;
- nome;
- sistema de arquivos;
- total;
- disponível.

#### Energia

- plano ativo;
- GUID;
- fonte da coleta.

### 7.3 Ações disponíveis

A Dashboard terá:

```text
[Gerar diagnóstico]
[Exportar relatório]
```

#### Gerar diagnóstico

Chama o backend para gerar um `SystemReport`.

Fluxo:

```text
click → loading → invoke("generate_report") → currentReport atualizado
```

#### Exportar relatório

Se `currentReport` existir:

```text
abre fluxo de exportação
```

Se não existir:

```text
mostrar aviso: gere um diagnóstico primeiro
```

### 7.4 Estado inicial

Quando o app abrir sem relatório:

```text
Nenhum diagnóstico gerado ainda.
Clique em "Gerar diagnóstico" para coletar o estado atual do sistema.
```

### 7.5 Estado carregando

Durante coleta:

```text
Coletando informações do sistema...
```

Ações que dependem do relatório devem ficar desabilitadas.

### 7.6 Estado parcial

Se alguns campos vierem `null`, a tela mostra:

```text
indisponível
```

Exemplo:

```text
Plano ativo: indisponível
```

### 7.7 Estado de erro

Se a coleta falhar de forma estrutural:

```text
Não foi possível gerar diagnóstico.
Detalhe: <mensagem segura>
```

A mensagem deve ser clara, mas sem stack trace.

---

## 8. Tela 2 — Processos

### 8.1 Objetivo

Mostrar os processos mais relevantes do sistema no momento da coleta.

### 8.2 Fonte de dados

```text
currentReport.processes
```

### 8.3 Dados exibidos

Tabela com:

- PID;
- nome;
- CPU%;
- memória usada.

### 8.4 Ordenação

Padrão Control:

```text
Ordenar por memória usada, de maior para menor.
```

Motivo:

```text
CPU é instável em leitura instantânea.
RAM é mais estável para diagnóstico inicial.
```

### 8.5 Quantidade padrão

Mostrar:

```text
Top 15 processos
```

### 8.6 Ações disponíveis

Control permite apenas ações de visualização:

```text
[Atualizar diagnóstico]
```

Não pode:

- finalizar processo;
- suspender processo;
- abrir local do arquivo;
- alterar prioridade;
- copiar command line completa.

Essas ações ficam fora do Control por segurança e privacidade.

### 8.7 Estado sem relatório

```text
Nenhum diagnóstico disponível.
Gere um diagnóstico para visualizar processos.
```

### 8.8 Estado sem processos

```text
Nenhum processo disponível no relatório.
```

### 8.9 Privacidade

A tabela de processos não mostra:

- caminho completo do executável;
- argumentos;
- command line completa;
- diretório de trabalho;
- usuário dono do processo.

Campos permitidos:

```text
PID
nome
CPU%
memória usada
```

---

## 9. Tela 3 — Diagnóstico

### 9.1 Objetivo

Centralizar a geração e o estado do diagnóstico atual.

Enquanto a Dashboard mostra resumo, a tela Diagnóstico mostra o status operacional da coleta.

### 9.2 Dados exibidos

- status atual;
- data/hora do último diagnóstico;
- versão do schema;
- versão do app/probe;
- quantidade de discos coletados;
- quantidade de processos coletados;
- quantidade de campos indisponíveis, quando possível;
- mensagem informando que nenhuma alteração foi feita.

### 9.3 Ações disponíveis

```text
[Gerar novo diagnóstico]
```

Opcional no Control, se simples:

```text
[Limpar diagnóstico atual]
```

`Limpar diagnóstico atual` remove apenas o estado em memória do app, não remove arquivos do sistema.

### 9.4 Estado inicial

```text
Nenhum diagnóstico gerado.
```

### 9.5 Estado após sucesso

Exemplo:

```text
Diagnóstico gerado com sucesso.

Gerado em: 2026-05-25T07:30:00Z
Schema: 0.1
Probe: 0.1.0
Discos: 1
Processos: 15

Nenhuma alteração foi feita no sistema.
```

### 9.6 Estado de erro parcial

Se o diagnóstico nasceu com campos indisponíveis:

```text
Diagnóstico gerado com dados parciais.
Alguns campos não puderam ser coletados e aparecem como "indisponível".
```

### 9.7 Estado de erro estrutural

```text
Falha ao gerar diagnóstico.
Tente novamente ou verifique permissões do sistema.
```

---

## 10. Tela 4 — Exportar

### 10.1 Objetivo

Permitir salvar o relatório atual em disco.

Arquivos gerados:

```text
report.json
report.txt
```

### 10.2 Fonte de dados

```text
currentReport
```

### 10.3 Ações disponíveis

```text
[Escolher pasta]
[Exportar relatório]
```

### 10.4 Fluxo esperado

```text
1. usuário gera diagnóstico;
2. usuário abre Exportar;
3. usuário escolhe pasta;
4. app chama backend para salvar report.json e report.txt;
5. app mostra confirmação com caminhos gerados.
```

### 10.5 Estado sem relatório

```text
Nenhum diagnóstico disponível para exportar.
Gere um diagnóstico primeiro.
```

### 10.6 Estado sem pasta escolhida

```text
Escolha uma pasta para salvar o relatório.
```

### 10.7 Estado de sucesso

```text
Relatório exportado com sucesso.

Arquivos:
- C:\...\report.json
- C:\...\report.txt
```

### 10.8 Política de sobrescrita

No Control, manter a mesma política do CLI Probe:

```text
Sobrescrever report.json e report.txt se já existirem.
```

A interface deve avisar antes ou durante a exportação:

```text
Se já existirem relatórios nesta pasta, eles serão sobrescritos.
```

Não adicionar histórico, timestamp automático ou `report_id` no Control.

### 10.9 Estado de erro

Se falhar escrita:

```text
Não foi possível exportar relatório.
Verifique permissões da pasta escolhida.
```

---

## 11. Tela 5 — Sobre

### 11.1 Objetivo

Apresentar identidade básica do produto e transparência do método.

### 11.2 Dados exibidos

- nome do app;
- versão do app;
- marca;
- descrição curta;
- links de projeto;
- aviso de confiança.

Exemplo:

```text
RiukyOS Control Center
Versão: 0.1.0

RiukyOS LAB
Diagnostic and optimization tools for Windows.

Método:
- diagnóstico antes de alteração;
- relatório antes/depois;
- logs antes de aprovação;
- nenhuma alteração invisível no Control.
```

### 11.3 Ações possíveis

```text
[Abrir GitHub]
[Abrir documentação]
```

Essas ações só abrem links externos. Não executam scripts.

### 11.4 Aviso obrigatório

A tela deve conter:

```text
No Control, o app apenas lê dados técnicos e exporta relatórios.
Nenhuma otimização é aplicada automaticamente.
```

---

## 12. Comandos Tauri esperados

Os nomes finais podem ser ajustados na implementação, mas o Control deve cobrir estes comandos conceituais.

### 12.1 Estado gerenciado no backend

O backend Tauri deve manter o relatório atual em estado gerenciado.

Modelo conceitual:

```rust
use std::sync::Mutex;

pub struct AppState {
    pub current_report: Mutex<Option<SystemReport>>,
}
```

Motivo:

```text
Evitar que o frontend envie o SystemReport inteiro de volta ao backend apenas para exportar.
O backend gera, armazena e exporta o relatório atual.
```

### 12.2 `generate_report`

Contrato conceitual:

```rust
#[tauri::command]
fn generate_report(state: tauri::State<AppState>) -> Result<SystemReport, String>;
```

Responsabilidade:

```text
chamar riukyos_core::collect()
armazenar uma cópia no estado interno
retornar SystemReport serializável ao frontend
```

Não faz:

```text
não escreve arquivo por padrão
não modifica sistema
```

### 12.3 `export_current_report`

Contrato conceitual:

```rust
#[tauri::command]
fn export_current_report(
    state: tauri::State<AppState>,
    output_dir: String
) -> Result<ExportResult, String>;
```

Responsabilidade:

```text
ler o relatório atual do estado interno
salvar report.json e report.txt
retornar caminhos gerados
```

Se não houver relatório atual:

```text
retornar erro controlado:
"Nenhum diagnóstico disponível para exportar."
```

### 12.4 `get_app_info`

Contrato conceitual:

```rust
#[tauri::command]
fn get_app_info() -> Result<AppInfo, String>;
```

Responsabilidade:

```text
retornar versão, nome do app e links básicos
```


## 13. Tipos auxiliares do frontend

### 13.1 `ExportResult`

Contrato conceitual:

```ts
type ExportResult = {
  json_path: string;
  txt_path: string;
  overwritten: boolean;
};
```

### 13.2 `AppInfo`

Contrato conceitual:

```ts
type AppInfo = {
  app_name: string;
  app_version: string;
  brand: string;
  github_url: string | null;
  docs_url: string | null;
};
```

Esses tipos devem refletir structs Rust equivalentes quando necessário.

---

## 14. Tratamento de erro na UI

### 14.1 Regras

Erros devem ser:

- claros;
- curtos;
- sem stack trace;
- sem culpa ao usuário;
- com ação sugerida quando possível.

### 14.2 Exemplos

Erro de exportação:

```text
Não foi possível exportar relatório.
Verifique se a pasta escolhida permite gravação.
```

Erro de coleta:

```text
Não foi possível gerar diagnóstico.
Alguns dados do sistema podem estar inacessíveis.
```

Erro parcial:

```text
Diagnóstico gerado com dados parciais.
Campos indisponíveis aparecem como "indisponível".
```

---

## 15. Estados globais obrigatórios

Toda ação assíncrona precisa tratar:

```text
idle
loading
success
partial
error
```

### 15.1 `idle`

Nenhuma ação em andamento.

### 15.2 `loading`

Backend executando coleta ou exportação.

### 15.3 `success`

Ação finalizada com sucesso.

### 15.4 `partial`

Relatório gerado, mas com campos indisponíveis.

### 15.5 `error`

Erro estrutural da ação.

---

## 16. Acessibilidade funcional mínima

O Control deve respeitar:

- textos legíveis;
- contraste suficiente, definido depois no `BRAND_SPEC.md`;
- botões com labels claros;
- navegação compreensível;
- não depender apenas de ícone para ação;
- mensagens de erro em texto.

Não é necessário fechar WCAG completo no Control, mas a interface não deve nascer hostil.

---

## 17. Responsividade

Alvo inicial:

```text
desktop Windows
```

Tamanhos de referência:

```text
1280x720 mínimo funcional
1366x768 comum
1920x1080 ideal
```

O app não precisa ser mobile.

Regras:

- não quebrar em 1366x768;
- permitir scroll quando conteúdo exceder altura;
- não esconder ações principais.

---

## 18. O que fica fora da UI até Safeguard+

Fora do Control:

```text
- botão otimizar
- botão limpar temporários
- perfis de otimização
- alteração de plano de energia
- finalizar processo
- alterar serviço
- alterar registro
- histórico de relatórios
- comparação antes/depois
- modo empresa
- login/conta
- upload para nuvem
- PDF
- auto-update
- multi-idioma/i18n
```

Esses itens só entram com nova decisão documentada.

---

## 19. Relação com BRAND_SPEC.md

Este documento permite fechar a estrutura funcional da interface sem travar a estética.

`BRAND_SPEC.md` decidirá:

- paleta;
- tipografia;
- grid visual;
- espaçamentos;
- estilo dos cards;
- estilo de botões;
- ícones;
- animações;
- referências aprovadas;
- referências rejeitadas;
- design tokens para Tailwind.

Direção fundadora já aprovada:

```text
noir · premium · dark · metálico · minimalista
técnico · grafite/prata · brilho frio discreto
```

---

## 20. Checklist de aprovação do UI_SPEC

Este documento está aprovado quando:

- [ ] define as 5 telas do Control;
- [ ] separa função de estética;
- [ ] define dados por tela;
- [ ] define ações permitidas;
- [ ] define estados vazios;
- [ ] define estados de erro;
- [ ] define fluxo de exportação;
- [ ] define comandos Tauri conceituais;
- [ ] impede ações modificadoras antes do Safeguard;
- [ ] mantém `BRAND_SPEC.md` como bloqueador apenas do frontend Control.


---

## 21. P3.1 — Decisões ajustadas para Control

### 21.1 Exportação

Decisão:

```text
Não reenviar `SystemReport` inteiro do frontend para o backend.
Usar estado gerenciado no backend Tauri.
```

### 21.2 Timeout de comandos externos

Antes de endurecer o Control, comandos externos usados pelo core devem ter estratégia de timeout documentada ou encapsulada.

Status:

```text
dívida técnica aceita para Probe
bloqueador de hardening antes do Control/Safeguard
```
---

*Próximo documento: `VALIDATION_PLAN.md`*

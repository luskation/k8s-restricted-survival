# Diário de Bordo — Iniciação Científica

| | |
|---|---|
| **Departamento** | ]
Departamento de Ciências da Computação (DCC) |
| **Aluno** | Lucas Oliveira Rodrigues |
| **Orientador** | Prof. Rafael Serapilha Durelli |
| **Tema** | Pod Security Standards (PSS) `restricted` em charts Helm reais |
| **Início** | 2026-08-13 |

## 1. Pergunta de pesquisa

> O Pod Security Standards `restricted` é a recomendação oficial de segurança do
> Kubernetes, mas quantos charts Helm populares realmente rodam sob ele — e por quê
> falham?

## 2. Metodologia (resumo)

1. Selecionar N charts Helm populares (Artifact Hub / Bitnami / charts oficiais de
   projetos conhecidos).
2. Subir um cluster local com `kind`.
3. Instalar cada chart com valores padrão em um namespace rotulado com o nível
   `baseline` do Pod Security Admission.
4. Repetir a instalação em um namespace rotulado com o nível `restricted`.
5. Registrar o que quebra: pod não sobe, admission webhook rejeita, container
   crasha, etc.
6. Classificar cada falha segundo a regra do PSS que a causou (ver §4 —
   Taxonomia).
7. Consolidar os resultados na tabela da §3 e, ao final, transformar a taxonomia
   em um resultado honesto sobre "por que apps reais não rodam sob restricted".

### Ambiente de teste

| Item | Valor |
|---|---|
| Cluster | `kind` vX.Y.Z |
| Kubernetes | vX.Y.Z |
| Método de aplicar o PSS | rótulos `pod-security.kubernetes.io/enforce` no namespace |
| Fonte dos charts | Artifact Hub (`https://artifacthub.io`) |

*(preencher com as versões reais assim que o ambiente for montado — atualizar aqui,
não em cada entrada, para não repetir informação)*

### Legenda de status

| Símbolo | Significado |
|---|---|
| ✅ | Rodou sem alteração nenhuma |
| ⚠️ | Rodou, mas só depois de ajustar `values.yaml` |
| ❌ | Não rodou mesmo com ajuste de `values.yaml` |
| 🚫 | Ainda não testado |

---

## 3. Tabela consolidada de resultados

Atualizar esta tabela a cada chart testado. É o dado bruto que vira o resultado
final do trabalho — manter mesmo que uma entrada do diário abaixo já descreva o
mesmo teste em prosa.

| Chart | Versão | `baseline` | `restricted` | Causa(s) da falha | Entrada do diário |
|---|---|---|---|---|---|
| *(ex.) nginx-ingress* | *0.0.0* | ✅ | ❌ | `runAsNonRoot` | [2026-08-13](#2026-08-13) |

---

## 4. Taxonomia de causas de falha

Cada vez que uma causa nova aparecer, criar uma entrada aqui — mesmo que ainda com
só um exemplo. É o principal entregável qualitativo do trabalho: no fim, esta
seção deve dar conta de explicar a maior parte das falhas observadas na tabela
acima.

### `runAsNonRoot`

- **Regra do PSS:** container precisa declarar `runAsNonRoot: true` (ou herdar do
  Pod).
- **Exemplos observados:** —

### `allowPrivilegeEscalation`

- **Regra do PSS:** `allowPrivilegeEscalation` deve ser `false`.
- **Exemplos observados:** —

### `capabilities`

- **Regra do PSS:** todas as capabilities devem ser dropadas (`drop: [ALL]`); só
  `NET_BIND_SERVICE` pode ser adicionada de volta.
- **Exemplos observados:** —

### `hostPath` / volumes de host

- **Regra do PSS:** `restricted` proíbe volumes `hostPath`, `hostNetwork`,
  `hostPID`, `hostIPC`.
- **Exemplos observados:** —

### `seccompProfile`

- **Regra do PSS:** exige `seccompProfile.type` igual a `RuntimeDefault` ou
  `Localhost`.
- **Exemplos observados:** —

*(adicionar novas categorias conforme surgirem — não forçar uma falha existente a
se encaixar numa categoria que não descreve bem a causa raiz; taxonomia errada
é pior que taxonomia incompleta)*

---

## 5. Diário

Uma entrada por sessão de trabalho, da mais antiga para a mais nova. Cada entrada
segue a mesma estrutura fixa abaixo — mesmo em dias "fracos", preencher todos os
campos (escrever "nada a registrar" é melhor que omitir o campo).

<!-- Copiar o bloco abaixo para cada nova entrada -->
<!--
### AAAA-MM-DD

**Objetivo do dia:**

**O que foi feito:**

**Resultados / observações:**

**Falhas encontradas:**
| Chart | Nível | Regra violada | Evidência |
|---|---|---|---|

**Próximos passos:**
-->

### 2026-08-13

**Objetivo do dia:**
Definir a pergunta de pesquisa e o formato do diário de bordo.

**O que foi feito:**
- Definida a pergunta de pesquisa com o Prof. Rafael Serapilha Durelli: quantos
  charts populares sobrevivem ao nível `restricted` do Pod Security Standards, e
  por quê falham.
- Estruturado o `diario.md` (este arquivo): tabela consolidada de resultados
  (§3) e taxonomia de causas de falha (§4) como seções vivas, atualizadas a
  cada experimento; entradas cronológicas na §5 registrando o processo dia a
  dia.

**Resultados / observações:**
Nenhum experimento rodado ainda.

**Falhas encontradas:**
_Nenhuma — ainda não há experimentos._

**Próximos passos:**
- Subir cluster `kind` local.
- Escolher a lista inicial de N charts a testar (critério de popularidade a
  definir — ex. mais instalados no Artifact Hub).
- Rodar o primeiro chart em `baseline` e `restricted` e preencher a primeira
  linha real da tabela do §3.

### 2026-08-18

**Objetivo do dia:**
Estudo da documentação oficial do Kubernetes, cobrindo desde o overview geral até
a seção de services-networking, como base teórica para a IC.

**O que foi feito:**
- Leitura das seções da documentação oficial do Kubernetes compreendidas entre
  "Overview" e "Services, Load Balancing, and Networking".
- Assistidas video-aulas complementares sobre os mesmos tópicos.
- Anotações pessoais registradas para consulta futura.

**Resultados / observações:**
Nenhum experimento prático rodado ainda — etapa focada em consolidar a base
teórica antes de montar o ambiente de testes (`kind`) e iniciar os experimentos
com os charts Helm.

**Falhas encontradas:**
_Nenhuma — ainda não há experimentos._

**Próximos passos:**
- Terminar os estudos teóricos em configuração e segurança no Kubernetes
  (ConfigMaps, Secrets e tópicos de segurança).
- Subir cluster `kind` local.
- Escolher a lista inicial de N charts a testar (critério de popularidade a
  definir — ex. mais instalados no Artifact Hub).
- Rodar o primeiro chart em `baseline` e `restricted` e preencher a primeira
  linha real da tabela do §3.

### 2026-08-19

**Objetivo do dia:**
Dar continuidade aos estudos teóricos do Kubernetes: aprender ConfigMaps e
Secrets, e revisar os conceitos estudados no dia anterior (2026-08-18).

**O que foi feito:**
- Estudo dos conceitos de ConfigMaps e Secrets: propósito, formas de criação e
  uso por Pods.
- Revisão dos conceitos vistos em 2026-08-18 (overview até
  services-networking), para consolidação do conteúdo.

**Resultados / observações:**
Nenhum experimento prático rodado ainda — etapa ainda concentrada na base
teórica, em preparação para a montagem do ambiente de testes (`kind`) e o
início dos experimentos com os charts Helm.

**Falhas encontradas:**
_Nenhuma — ainda não há experimentos._

**Próximos passos:**
- Estudar a parte de segurança do Kubernetes (2026-08-20).
- Subir cluster `kind` local.
- Escolher a lista inicial de N charts a testar (critério de popularidade a
  definir — ex. mais instalados no Artifact Hub).
- Rodar o primeiro chart em `baseline` e `restricted` e preencher a primeira
  linha real da tabela do §3.
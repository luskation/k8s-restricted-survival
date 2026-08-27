# Síntese de leitura — RBAClock: Contain RBAC Permissions through Secure Scheduling

## O problema: co-location attack via RBAC

O ponto de partida do artigo é que o scheduler padrão do Kubernetes decide
onde colocar um Pod olhando só pra CPU/memória disponível e ignora
completamente as permissões RBAC do service account daquele Pod. Isso faz
com que Pods de níveis de privilégio bem diferentes acabem compartilhando o
mesmo node. O problema é que, se um atacante escapar de um container (via
uma vulnerabilidade no runtime, no kernel, etc.), ele ganha acesso a todos os
ServiceAccountTokens montados no node, inclusive, os de Pods que ele nunca
tocou diretamente. Conclui-se, portanto: mesmo um Pod com permissões mínimas e bem
configurado pode virar porta de entrada pra permissões altas, só por ser vizinho de node
de um Pod privilegiado. Os autores chamam isso de co-location
attack, e o argumento central do artigo é que isso é um problema de
scheduling, não (só) de RBAC mal configurado.

## O que eles propõem: métrica ERP + scheduler RBAClock

A proposta é uma métrica chamada ERP (Extraneous Risk Privileges), que mede
quanto de privilégio "alheio" um Pod passa a estar exposto ao ser colocado
em um node, privilégios que não são dele, mas que pertencem a outros Pods
alocados e que um atacante poderia herdar via escape de container. Em cima
dessa métrica, eles implementam o RBAClock, um plugin de scheduling que, na
fase de scoring do scheduler padrão do Kubernetes, prioriza o node que
resulta no menor aumento de ERP. Na prática, isso empurra o scheduler a
agrupar Pods de perfil de risco parecido no mesmo node e a isolar os Pods
com privilégios muito divergentes em nodes separados, reduzindo a chance 
de um atacante ter acesso total ao invadir um node qualquer.

## Como funciem, em linhas gerais

Calcular ERP corretamente esbarra em dois problemas que os autores tratam
como as contribuições técnicas do artigo. O primeiro (C1) é que muitas
aplicações de terceiros no Kubernetes usam Custom Resources (CRDs), e não dá
pra saber quanto risco uma permissão sobre uma CR representa sem entender
que recursos built-in ela afeta por baixo dos panos — pra isso, eles fazem
taint analysis no código-fonte (via CodeQL) pra mapear CRs pra recursos
nativos equivalentes. O segundo (C2) é que o risco de uma permissão RBAC
depende do estado do cluster no momento (ex: `list/get secrets` só é
perigoso na medida em que existem secrets sensíveis acessíveis) — pra isso
usam um modelo de autômato finito determinístico (DFA) que expande
iterativamente o conjunto de permissões de um Pod até um ponto fixo,
considerando o estado real do cluster. O resultado desse processo é um vetor
de risco por Pod, decomposto em três dimensões — leakage, tampering e
execution —, que alimenta o cálculo do ERP.

## Resultados

Os experimentos usam 24 aplicações do CNCF, com 94 Pods, em clusters de 2 a
28 nodes, comparando o scheduler padrão do Kubernetes com o RBAClock:

| Métrica | Redução média com RBAClock |
|---|---|
| ERP agregado do cluster | até 100%, média de 84% |
| Risco médio agregado por node | média de 41,64% |
| Caminhos de escalada de privilégio possíveis | média de 64,63% |
| Proporção de nodes com Pods de privilégio crítico | média de 34,59% |
| Overhead de performance (apps sensíveis a latência) | até ~10% |

Além dos números, os autores relatam dois achados em produção: encontraram e
reportaram (com confirmação dos respectivos times de segurança) cenários
reais de co-location perigosa no Alibaba Cloud (ACK) e no Tencent Kubernetes
Engine (TKE), o que dá algum peso prático ao problema que estão descrevendo,
além do experimento controlado.

## Limitações que os próprios autores apontam

Vale registrar porque são pontos que os autores admitem: o RBAClock não
considera sensibilidade dos dados dentro do container (um Pod de baixo
privilégio RBAC ainda pode guardar segredos valiosos); o algoritmo é guloso
e sensível à ordem de deploy, podendo cair em ótimos locais; e o suporte a
Custom Resources hoje só cobre projetos em Go (por causa da análise
estrutural via CodeQL).

## Por que isso importa para a pesquisa

Confesso que quando comecei a ler achei que esse artigo não ia me servir
pra nada, já que ele fala de scheduling e RBAC, e minha pesquisa é sobre
Pod Security Standards. Mas terminando a leitura percebi uma coisa parecida
com o que já tinha visto ao estudar o PSS restricted: em nenhum dos dois
casos o Kubernetes é seguro por padrão, e em nenhum dos dois casos a
segurança acontece sozinha. O restricted só barra um Pod se alguém declarar
runAsNonRoot explicitamente no manifesto, e o RBAClock só existe porque o
scheduler padrão simplesmente não pergunta nada sobre RBAC antes de decidir
onde colocar um Pod. Ou seja, tanto faz o ângulo, RBAC ou hardening de
container, sempre tem alguém assumindo por padrão que está tudo bem e
sobra pra quem configura (ou pra uma ferramenta terceira) perceber que não
está. Isso é meio óbvio dito assim, mas é o tipo de coisa que só fica claro
lendo vários artigos parecidos em sequência, e acho que é um bom fio
condutor pra amarrar a introdução da minha pesquisa: não é sobre PSS
especificamente, é sobre o Kubernetes desconfiar por padrão, e sempre
foi assim.

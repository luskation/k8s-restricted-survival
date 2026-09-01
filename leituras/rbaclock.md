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

Só que pra decidir quem pode ficar vizinho de quem, o scheduler precisa
saber antes quanto risco cada Pod carrega. E aí aparecem dois problemas que
os autores tratam como as contribuições técnicas do artigo.

O primeiro (C1) é que boa parte das aplicações num cluster Kubernetes não
usa só recursos nativos, tipo Pod, Secret ou ServiceAccount. Elas usam
Custom Resources (CRs), recursos definidos por CRDs que estendem a API do
Kubernetes pra representar conceitos de uma aplicação de terceiros. Um
Certificate do cert-manager representa um certificado TLS, mas o controller
dele cria por trás um Secret com a chave privada. Um ExternalSecret do
external-secrets referencia um segredo guardado num cofre externo, tipo
Vault ou AWS Secrets Manager, e o controller busca esse valor e materializa
como Secret dentro do cluster. Uma Application do ArgoCD representa uma
aplicação a sincronizar, mas o controller cria e gerencia Deployments,
Services e outros recursos pra fazer isso acontecer. Em todos esses casos,
olhar só pra permissão RBAC sobre a CR não conta a história toda, porque a
gente não enxerga o que o controller faz por baixo dos panos. E sem isso,
não dá pra saber quanto risco aquela permissão realmente representa.

O segundo (C2) é que o risco de uma permissão RBAC não é fixo. Ele depende
do estado do cluster no momento em que a permissão é avaliada. Uma
permissão sozinha pode parecer inofensiva, mas raramente age sozinha.
list/get secrets só é perigoso se existirem secrets sensíveis acessíveis
com aquele escopo. create pods pode virar porta de entrada pra outros
recursos, dependendo de quais ServiceAccounts e volumes estão disponíveis
naquele momento. Medir risco olhando cada permissão isolada, sem considerar
o cluster real e o que ela permite encadear, subestima o risco de
escalonamento de privilégio.

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

## Como funcionam, em linhas gerais

Pro C1, a saída foi fazer análise no código-fonte das aplicações via
CodeQL, rastreando como cada CR se conecta aos recursos nativos que ela
manipula por trás. Isso dá o mapeamento que faltava entre uma CR e o que
ela realmente afeta no cluster.

Já pro C2 o caminho foi outro: um autômato finito determinístico (DFA) que
parte das permissões declaradas de um Pod e vai expandindo esse conjunto,
olhando o estado real do cluster a cada passo, até chegar num ponto fixo em
que nada mais se soma. No fim sobra um vetor de risco por Pod, dividido em
três dimensões (leakage, tampering e execution), que é o que alimenta o
cálculo do ERP.

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

Além dos números, os autores relatam dois achados em produção. ACK e TKE
são os serviços de Kubernetes gerenciado da Alibaba Cloud e da Tencent
Cloud, as duas maiores provedoras de nuvem da China, mais ou menos o
equivalente ao EKS da AWS. Os autores encontraram e reportaram (com
confirmação dos respectivos times de segurança) cenários reais de
co-location perigosa nesses dois provedores, o que dá algum peso prático ao
problema que estão descrevendo, além do experimento controlado.

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

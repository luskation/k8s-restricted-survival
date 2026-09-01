# Síntese de leitura — Automated Analysis of Security Policy Violations in Helm Charts

## O problema: ferramentas de análise estática dão resultados inconsistentes

Helm é o gerenciador de pacotes mais famoso e facilita orquestrar
aplicações no Kubernetes. Cabe destacar, porém, que boa parte dos charts
são mal configurados no que tange a segurança. É pra pegar esse tipo de
misconfiguração antes do deploy que existem ferramentas de análise
estática, tipo Checkov, Datree, KICS, Terrascan, Kubeaudit, Kubelinter e
Kubescape. O que os autores mostram, com evidência experimental, é que
essas ferramentas checam coisas diferentes, ou checam a mesma coisa de
jeitos diferentes, e acabam dando resultados inconsistentes entre si. Uma
pesquisa da GitLab com mais de cinco mil profissionais mostra que 57% das
empresas usa seis ou mais ferramentas de segurança ao mesmo tempo, o que
torna esse problema bem prático: se cada ferramenta pega um subconjunto
diferente de misconfigurações, quantas são realmente necessárias pra ter
cobertura razoável?

Tem um segundo problema, ligado ao primeiro. Corrigir uma misconfiguração
normalmente significa remover uma permissão, tipo tirar o usuário root,
travar o filesystem em read-only, ou remover uma capability do Linux. Só
que essa correção pode quebrar a aplicação, porque algumas dessas
permissões, mesmo sendo tecnicamente "excessivas" do ponto de vista de
segurança, são necessárias pra funcionalidade real do container. Das sete
ferramentas avaliadas, só uma sugere algum tipo de mitigação, e mesmo essa
não considera esse trade-off entre segurança e funcionalidade. Ou seja,
aplicar as correções sugeridas às cegas pode deixar a aplicação mais segura
só no papel, porque ela simplesmente para de funcionar.

## O que eles propõem: pipeline automatizado + mapeamento semântico de políticas

A resposta dos autores tem duas partes. A primeira é um framework manual
pra mapear as políticas de cada ferramenta pra um identificador comum
(SID), de forma que dá pra comparar o que cada ferramenta realmente
verifica, mesmo quando o texto da política é diferente ou a política é
implementada de um jeito diferente por trás. Isso é o que permite, por
exemplo, dizer que só nove políticas são comuns às sete ferramentas.

A segunda parte é um pipeline automatizado de seis passos que roda uma
ferramenta de análise sobre o chart, aplica uma correção padrão pras
misconfigurações encontradas, testa se essa correção quebra a aplicação
(fazendo deploy de verdade num cluster Kubernetes e olhando se o container
sobe, fica em estado Running, e se os logs continuam equivalentes ao
original), e devolve um chart atualizado que só tem as permissões
realmente necessárias. O resultado é o que eles chamam de perfil de
funcionalidade: uma lista, por container, de quais permissões podem ser
removidas sem quebrar nada e quais não podem.

## Como funcionam, em linhas gerais

Pro mapeamento de políticas, os autores foram política por política,
ferramenta por ferramenta. Primeiro tentam encaixar a política numa
recomendação do CIS Benchmark, e se encaixa, todas as ferramentas que
checam aquela recomendação ganham o mesmo SID, mesmo com nomes de regra
diferentes. Quando não tem recomendação do CIS pra usar de referência,
olham direto pro que a política verifica no YAML, quais chaves e valores,
e comparam com as outras ferramentas: se for a mesma verificação, vira
política equivalente, senão fica marcada como única daquela ferramenta. E,
pra fechar, cada política ainda leva um rótulo de funcionalidade ou boa
prática, dependendo se removê-la arrisca quebrar a aplicação ou não.

O perfil de funcionalidade é mais direto, é testar removendo, uma
permissão de cada vez. O pipeline sobe duas versões do mesmo container num
cluster real, uma com a permissão e outra sem, e compara os dois. O
container ainda inicia? Ele fica em Running, com as probes de liveness e
readiness passando? Os logs continuam batendo com os da versão original?
Se qualquer uma dessas respostas for não, a permissão volta pro chart
porque é considerada necessária.

## Resultados

Os autores analisaram os 60 charts mais populares do Artifact Hub, dos
quais 52 puderam ser efetivamente testados (sete falharam o deploy, e um
chart não foi processado por nenhuma ferramenta). Confirmando o problema
inicial, um teste estatístico (Kruskal-Wallis) mostrou diferença
significativa entre os resultados das sete ferramentas. As
misconfigurações mais comuns são relacionadas a limites de CPU/memória,
uso do namespace default e ClusterRoles excessivamente permissivas, essa
última sendo a mais frequente, provavelmente porque permissões RBAC são
difíceis de definir corretamente e costumam vir de componentes de
terceiros que o desenvolvedor do chart nem sempre entende a fundo.

Do lado da funcionalidade, cada chart precisa em média de 2,21 permissões
"arriscadas" pra funcionar (variando de 0 a 8), e cada container não
privilegiado usa em média só 1,16 das 14 capabilities do Linux concedidas
por padrão. As funcionalidades mais frequentemente quebradas ao remover
permissões são justamente usar um UID alto e ter um filesystem gravável, o
que mostra que nem toda prática "insegura" é desnecessária. No geral,
porém, a proporção de correções que quebram alguma coisa é baixa (menos de
1% globalmente), o que é uma boa notícia: a maioria das misconfigurações
reportadas pode ser corrigida sem medo.

## Limitações que os próprios autores apontam

O trabalho não mede falsos negativos das ferramentas estáticas, só falsos
positivos e o quanto cada uma quebra funcionalidade. O mapeamento de
políticas entre ferramentas foi feito manualmente, o que é um esforço
considerável e não escala facilmente pra novas ferramentas ou políticas.
As mitigações padrão aplicadas pelo pipeline satisfazem as políticas, mas
não são necessariamente as mais efetivas ou elegantes, só o suficiente pra
passar nos checks. E a avaliação inteira foi feita em cima dos 60 charts
mais populares do Artifact Hub, então os resultados podem não generalizar
pra charts menos populares ou mais nichados.

## Por que isso importa para a pesquisa

Esse artigo bate direto com o que venho vendo nas outras leituras, o
RBAClock e o material sobre PSS restricted. É a mesma história: nenhuma
ferramenta cobre tudo sozinha, e o Kubernetes não é seguro por padrão,
sempre sobra pra alguém configurar direito. Mas aqui aparece um ângulo que
eu não tinha pensado. Não basta dizer que um chart viola o PSS restricted,
porque as próprias ferramentas que eu poderia usar pra checar isso
discordam entre si sobre o que conta como violação. Isso reforça que vale
testar contra o cluster de verdade, e não confiar cegamente no que um
scanner qualquer diz. 
# Síntese de leitura: Increasing Security and Reducing Risks Running Services in a Potential Containerized Environment While Meeting Regulatory Standards

## O que é: não é bem um "artigo", é uma dissertação de mestrado aplicada

Diferente das outras leituras (RBAClock, Security Policy Violations in Helm Charts), esse
texto não é um paper acadêmico com contribuição científica isolada, é uma dissertação de
mestrado em Cibersegurança e IA, orientada e avaliada na Universidade de Klagenfurt, que
documenta um estudo de caso real: o autor é o Information Security Officer da DaoPay, uma
empresa de meio de pagamento de porte médio, e o trabalho é o relato de como ele hardenizou o
cluster Kubernetes da própria empresa pra atender PCI-DSS e GDPR. Ou seja, é mais um relatório
de projeto com revisão de literatura embutida do que uma pesquisa com metodologia controlada e
resultados generalizáveis.

## O problema: segurança do Kubernetes por padrão não é suficiente, e ainda tem compliance no meio

Kubernetes não é seguro por padrão e quem usa a plataforma precisa configurar tudo manualmente (RBAC,
network policies, criptografia, PSS). Isso já é ruim sozinho, mas fica pior numa financeira, devido 
aos dados sensíveis (dados pessoais, transações) e duas cargas regulatórias em cima: GDPR (dados
pessoais de cidadãos da UE) e PCI-DSS (dados de cartão). Compliance falho pode gerar multa e
processo, então a pergunta de pesquisa não é só "como proteger o cluster" mas "como proteger o
cluster de um jeito que também sobrevive a uma auditoria de compliance".

## O que ele faz: metodologia é basicamente checklist + hardening + validação com scanner

O autor junta um checklist próprio a partir de várias fontes de boas práticas (Google, CNCF,
CIS Benchmark, OWASP, Tigera, Rancher), aplica esse checklist no cluster self-managed (K8s
1.21) da empresa, e depois usa duas ferramentas pra validar: kube-bench (contra o CIS
Benchmark) e kube-hunter (scanner de vulnerabilidades), rodadas antes e depois do hardening.
Não tem grupo de controle, não tem outros clusters comparados, é hardening de um único cluster
de produção de uma única empresa.

As medidas implementadas cobrem praticamente tudo que a literatura de boas práticas de K8s
já lista: desabilitar acesso anônimo à API, RBAC com só três papéis humanos (Administrator,
Deployer, CISO) mais service accounts por aplicação, autenticação via certificado X.509,
desabilitar automountServiceAccountToke quando não precisa, TLS interno com CA própria,
Calico como CNI (com WireGuard pra criptografar tráfego entre pods), Ingress namespaced,
admission controllers (AlwaysPullImages, NodeRestriction, DenyServiceExternalIPs, entre
outros), GrayLog pra log de aplicação e Check_mk pra monitoramento de cluster com alertas via
Telegram/e-mail, e GitLab CI/CD com rollback rápido e segredos guardados no GitLab em vez de
Kubernetes Secrets.

Um ponto que me chamou atenção: a decisão de não configurar Network Policies, ocorre 
porque as aplicações do cluster precisam se comunicar livremente entre si e o
cluster roda on-premises sem exposição pública.

## Resultados: CIS Benchmark subiu de 69 para 98 PASS

O resultado central é a comparação de antes/depois do kube-bench: **antes**, 69 checks PASS, 11
FAIL, 43 WARN; **depois**, 98 PASS, 5 FAIL, 19 WARN. O kube-hunter, que antes achava só a
versão do Kubernetes exposta (e etcd/kubelet/API server acessíveis), não é comparado com um
"depois" de forma tão explícita no texto. Além disso, teve uma varredura com Nessus (scanner de
vulnerabilidade comercial) que não achou nada de prioridade alta ou baixa, e uma revisão manual
pelo CISO da empresa validando compliance com PCI-DSS.

As falhas/avisos que sobraram depois do hardening são explicados caso a caso: Pod Security
Policy não é mais suportado no K8s 1.21 usado (foi descontinuado depois), seccomp não foi
habilitado porque exigiria mapear todas as syscalls necessárias por workload antes de restringir (
trabalho considerado não compensador no momento), e alguns avisos sobre Secrets foram
descartados porque a empresa decidiu guardar credenciais no GitLab em vez de usar o mecanismo
nativo do Kubernetes.

## Limitações que o próprio autor reconhece

O trabalho é declaradamente de escopo limitado: cobre um produto específico da empresa em fase
de re-desenvolvimento, não a infraestrutura inteira; usa Kubernetes 1.21 (já defasado mesmo na
época da escrita, o que limita a validade de partes da análise: por exemplo, Pod Security Policy
usado no trabalho já estava sendo substituído por Pod Security Admission); a parte de auditoria
completa de conformidade regulatória fica fora do escopo porque envolve muita documentação
organizacional que não é técnica; e não avalia rotação automática de credenciais (processo ainda
manual). Também não tem nenhuma tentativa de generalizar os achados pra outras empresas:
o próprio autor reconhece isso como lacuna e sugere que trabalhos futuros investiguem a integração
de segurança do Kubernetes com frameworks de segurança já existentes no setor financeiro.

## Por que isso importa pra mim, sendo sincero

Vou ser direto: fechei essa leitura com uma sensação diferente das outras duas. RBAClock e
o artigo do Helm eu terminei pensando "ok, isso muda como eu vou analisar tal coisa". Esse aqui eu
terminei pensando mais "que trabalhão isso deve ter dado pro cara" do que propriamente
extraindo alguma coisa nova pro meu método. E acho que é importante registrar essa sensação, e
não só fingir que toda leitura rende o mesmo tanto de insight.

Parte da frustração é que o texto flerta com o meu tema exato e depois escorrega pro lado.
Ele cita, ipsis litteris, a mesma falha do perfil `restricted` que eu já tinha visto na documentação
oficial: o `runAsGroup` não ser obrigatoriamente diferente de zero, deixando uma brecha pra
GID 0 acessar dado do grupo root. Por um segundo achei que ia ganhar um caso real de alguém
esbarrando nisso. Só que não: o cara nem chegou a usar PSS `restricted` no cluster dele, porque
está preso num Kubernetes 1.21 e usando Pod Security Policy, que já era o mecanismo antigo antes
mesmo do PSS existir. Então é uma citação de passagem, não uma experiência de primeira mão que
eu possa comparar com o que estou vendo nos meus próprios testes de chart × restricted.

Também não ajuda que a "metodologia" seja, no fundo, um checklist pessoal validado só contra
o próprio julgamento do CISO da empresa e um antes/depois de kube-bench. Não tem nada
generalizável ali, é o relato de uma pessoa resolvendo o problema de uma empresa só. Nada errado
nisso (é uma dissertação de mestrado aplicada, não pretende ser outra coisa), mas eu esperava
mais rigor do que encontrei, e isso me deixou mais cético em relação a quanto posso confiar em
"boas práticas" citadas sem teste por trás.

Dito isso, não foi tempo perdido. Tem um lado meio óbvio, mas que eu só senti de verdade lendo
o relato passo a passo: dá um trabalho absurdo hardenizar um cluster manualmente, item por item,
sem nenhuma ferramenta automatizando isso. E é exatamente aí que mora o motivo de eu estar
fazendo essa IC: se uma pessoa competente, com autoridade dentro da empresa e motivação real
(compliance com multa em jogo), ainda assim levou tanto esforço manual pra sair de 69 pra 98
checks do CIS Benchmark, imagina o desenvolvedor médio de um chart Helm open source, sem
pressão regulatória nenhuma, tentando (ou nem tentando) rodar sob `restricted`. Esse texto não
muda o meu método, mas deixou mais concreto, na cabeça, o tamanho do problema que eu quero
medir.

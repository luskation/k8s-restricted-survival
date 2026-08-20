# Síntese de leitura — Pod Security Standards

*Fonte: documentação oficial do Kubernetes,
[kubernetes.io/docs/concepts/security/pod-security-standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)*

## O que é

O Pod Security Standard, ou PSS, é o mecanismo nativo do Kubernetes para
controlar, no nível do namespace, quão "arriscada" a configuração de segurança
de um Pod pode ser. Ele define três níveis. São políticas cada vez mais restritivas e acumulativas que o cluster pode
aplicar via rótulos no namespace (pod-security.kubernetes.io/enforce: nível). A ideia central é: quanto mais alto o nível, menos o container consegue tocar no
sistema operacional do node por baixo do Kubernetes.

## privileged

É o nível sem restrição, na prática, é o reizinho que pode tudo. Ele existe só para
namespaces de sistema que legitimamente precisam de acesso total à máquina
(ex. o kube-system, CNI, storage drivers). Não bloqueia nada: containers
privilegiados, montagem de qualquer hostPath, uso de hostNetwork/hostPID,
qualquer capability Linux. Não deveria ser usado para aplicações comuns. Na prática, o
pod pode acessar a rede do host e rodar como root.

## baseline

É o nível de "bloquear só o óbvio". A lógica do baseline não é "fazer o Pod
seguro", é impedir as formas mais conhecidas e diretas de escapar do container e
comprometer o nó ou o cluster inteiro. Entre outras coisas, o baseline proíbe containers privileged: true,
proíbe montar hostPath (volume apontando pro filesystem do nó), proíbe hostNetwork, hostPID, hostIPC (compartilhar namespaces do host), proíbe uma lista específica de capabilities Linux perigosas (ex.
SYS_ADMIN, NET_ADMIN) e restringe procMount e AppArmor/sysctls a um conjunto seguro conhecido. Por fim, o baseline não exige rodar como usuário não-root, não exige derrubar
todas as capabilities, e não controla allowPrivilegeEscalation. Ou seja: um
Pod pode passar no baseline e ainda assim rodar como root dentro do container.

## restricted

É o nível "hardening completo", pensado para seguir as boas práticas atuais de
reforço de Pods, mesmo que isso custe compatibilidade. O restricted inclui tudo
que o baseline já bloqueia e vai além: exige runAsNonRoot: true, ou seja, o
container é obrigado a declarar que não roda como root (UID 0), tanto no Pod
quanto no container. Exige também allowPrivilegeEscalation: false, pra
garantir que o processo lá dentro não consiga ganhar mais privilégio do que
tinha ao iniciar, por exemplo via um binário setuid. Derruba todas as
capabilities Linux por padrão (capabilities.drop: ["ALL"]), só deixando
adicionar de volta a NET_BIND_SERVICE, que é a que permite escutar em portas
abaixo de 1024. E o seccompProfile.type precisa ser RuntimeDefault ou
Localhost, nunca Unconfined (ou seja, sem filtro de syscalls). O detalhe
que costuma pegar bastante gente é que nada disso é implícito: não basta a
imagem já rodar como não-root "por acaso", o manifesto precisa declarar isso
explicitamente, senão o restricted barra o Pod mesmo assim.

## Por que isso importa para a pesquisa

A diferença prática entre baseline e restricted é exatamente a diferença
entre "não fazer as coisas mais perigosas" e "declarar explicitamente uma
postura segura". A maioria das imagens de container publicadas por projetos
open source não foi construída pensando em runAsNonRoot, drop de capabilities
ou seccomp, muitas rodam como root por conveniência (ex. pra escrever em
diretórios sem se preocupar com permissão, ou porque o processo interno precisa
bindar numa porta privilegiada). É exatamente por isso que a pergunta de
pesquisa faz sentido: restricted é a recomendação oficial, mas é plausível que
boa parte dos charts populares nunca tenha sido testada contra ele, e que a
distância entre "recomendação" e "prática real" seja grande. O objetivo do
experimento é medir essa distância e explicar, causa por causa, de onde ela vem.

## Quadro comparativo

| Controle | privileged | baseline | restricted |
|---|---|---|---|
| Container privileged: true | permitido | bloqueado | bloqueado |
| hostPath, hostNetwork/PID/IPC | permitido | bloqueado | bloqueado |
| Capabilities perigosas (SYS_ADMIN...) | permitido | bloqueado | bloqueado |
| Rodar como root | permitido | permitido | bloqueado (runAsNonRoot) |
| allowPrivilegeEscalation | permitido | permitido | bloqueado |
| Capabilities extras (além de NET_BIND_SERVICE) | permitido | permitido | bloqueado |
| seccompProfile não confinado | permitido | permitido | bloqueado |

# Artigo Técnico
## _Migração de microsserviços onpremises para a nuvem utilizando Istio_

![Istio](https://img.shields.io/badge/-ISTIO-20367d?style=for-the-badge&logo=ISTIO&logoColor=white)
![Istio](https://img.shields.io/badge/-KUBERNETES-3186cc?style=for-the-badge&logo=KUBERNETES&logoColor=white)


Do ponto de vista técnico, um projeto de migração para a nuvem tem seu fator de complexidade inexoravelmente dependente de seus fatores de escopo, maturidade e ciclo de vida das cargas de trabalho que estão sendo movidas.

Este artigo, trata sobre uma migração de API em execução na infraestrutura kubernetes onpremises sendo migrada para plataforma de cluster AWS EKS com kubernetes versão 1.17.

A migração será abordada utilizando a ferramenta Istio (Service Mesh) e entendendo seus pontos cruciais.


## Definição do Istio

> [1] O Istio é uma malha de serviço de código aberto que ajuda as organizações a executar apps distribuídos baseados em microsserviços em qualquer lugar. Por que usar o Istio? Com o Istio, as organizações podem proteger, conectar e monitorar microsserviços para modernizar os apps empresariais com mais rapidez e segurança.
Gerencia fluxos de tráfego entre serviços, aplica políticas de acesso e agrega dados de telemetria, tudo isso sem alterações no código do aplicativo. O Istio facilita a complexidade da implantação ao utilizar camadas transparentes em aplicativos distribuídos existentes.

### Aos benefícios da malha de serviço do Istio aplicam-se:
##### - Alcance uma rede de serviços consistente:
Os operadores de rede podem gerenciar de forma consistente a rede para todos os serviços sem adicionar sobrecarga para o desenvolvedor.

##### - Proteção de serviços com os benefícios do Istio
Os operadores de segurança podem implementar facilmente a segurança de serviço a serviço, incluindo autenticação, autorização e criptografia.

##### - Melhoria de desempenho do aplicativo
Implementação de práticas recomendadas, como lançamentos canário, e  visibilidade ampla dos aplicativos para identificação de onde concentrar os esforços para melhoria de desempenho.

### Ingress Gateway
Uma das maneiras de permitir o tráfego no cluster é por meio do Ingress Gateway do Istio, que se posiciona na borda do cluster e, no tráfego de entrada, habilitando os recursos do Istio, como roteamento, balanceamento de carga, segurança e monitoramento.

Durante a instalação do Istio, o componente Ingress Gateway e o serviço que o expõe externamente foram instalados no cluster.

Pode-se obter o IP externo dos serviços executando o comando:

```sh
$ kubectl get svc -n istio-system -l istio=ingressgateway
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP
istio-ingressgateway   LoadBalancer   10.0.132.127   13.93.30.120$ EXTERNAL_IP=$(kubectl get svc -n istio-system \
  -l app=istio-ingressgateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
  ```
Deste modo obtem-se o ip externo através do ingress, porém ainda inacessível pois por padrão estará bloqueado até que seja definido um gateway.

### O recurso de gateway

Um gateway é uma definição de recurso personalizado do Kubernetes definida na instalação do Istio no cluster que permite especificar as portas da aplicação, o protocolo e os hosts para os quais permitirão o tráfego de entrada.

Um gateway controla o fluxo de tráfego para dentro e para fora da malha de serviço.

A esmagadora maioria dos casos de uso de gateways gira em torno do gerenciamento do tráfego de entrada.
Nessa capacidade, os gateways agem de maneira semelhante aos recursos de entrada regulares do Kubernetes.

### O recurso VirtualService

O VirtualService instrui o Ingress Gateway sobre como rotear as solicitações que foram permitidas no cluster.
Os pontos importantes aqui são:
- Este VirtualService aplica-se a solicitações provenientes do gateway http.
- O destino define o serviço para o qual as solicitações são roteadas.

![](/fig/fig1.gif)

## A Migração Simulada (Mock Migration)
Um teste do plano de projeto de migração para descobrir resultados não intencionais ou problemas despercebidos.

>[2] A migração simulada é uma importante etapa intermediária no processo de migração da Aplicação e tem os seguintes objetivos principais:

>1. Verificar se todo o modelo pode ser transformado e se os erros de transformação são documentados e enfileirados para correção da regra de automação ou documentados como uma etapa manual.
Erros são esperados em modelos grandes devido a links quebrados, construções de código ou configurações não encontradas anteriormente.

>2. Permite que a equipe demonstre o padrão UX para transformação automatizada.
O cliente então dará sua aprovação final.


Tecnicamente a migração dependerá que sejam obedecidos os seguintes procedimentos:
- Antes de iniciar a migração simulada, as cargas de trabalho devem ser aliviadas com base na complexidade, dependências transacionais, impacto nos negócios, etc.
- Obter uma visão completa de toda a infraestrutura existente, entender as dependências de cada uma, para que possam ser priorizados os aplicativos para migração e criação de roteiro para a nuvem.
- Começar a migrar o aplicativo para a nuvem e executar testes e validação dos aplicativos migrados.

### Configuração local atual em produção.

IMAGEM DO DETETIVE

A migração para a nuvem iniciará migrando primeiro o Serviço E para o cluster AWS EKS e para isso, é necessário o entendendiemtno do roteamento de tráfego entre o microsserviço B e o microsserviço E no cluster local.

O microsserviço B está comunicando o microsserviço E através do host ```microservice-e-svc.namespace.svc.cluster.local``` e aqui o roteamento para o microsserviço E será decidido pelo serviço virtual do microsserviço E.

#####  Virtual service do microsserviço E onpremises
apiVersion: networking.istio.io/v1beta1
``` sh
kind: VirtualService
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  name: microservice-e-vs
  namespace: prod-service-namespace
spec:
  gateways:
  - prod-service-gateway
  - mesh
  hosts:
  - microservice-e-svc
  http:
  - route:
    - destination:
        host: microservice-e-svc
        port:
          number: 8080
```

IMAGEM DO NGINX
Mesmo manifesto de serviço implantado na AWS e no local com tráfego ao vivo no local

### Etapas para migrar o Microsserviço E para a nuvem

- Implantação da mesma pilha de serviço no AWS EKS Cluster.
- Criação de um novo bloco de destino no mesmo serviço virtual do serviço E no local:
    De acordo com o novo bloco, 10% do tráfego será roteado para o host microservice-e.aws.dummy.com.
    O peso associado ao host determina quanto tráfego ele recebe.
- Criação de entrada de serviço pois o  serviço é externo ao plano de controle istio local e o ServiceEntry permite adicionar entradas adicionais no registro de serviço interno do Istio, para que os serviços descobertos automaticamente na malha possam acessar/rotear para esses serviços especificados manualmente.
- O novo host microservice-e.aws.dummy.com está apontando para o gateway istio-ingress do cluster aws eks. Portanto, o tráfego entrará no plano de controle istio de aws.
- O serviço virtual do microsserviço E no cluster aws configurou o controlador de entrada para rotear o tráfego para o host microservice-e.aws.dummy.com.
- Basicamente, com zero downtime, 10% do tráfego para o microsserviço E no local começará a ir para o microsserviço E na nuvem.

##### Virtual service of Microsserviço E in onpremise
``` sh
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  name: microservice-e-vs
  namespace: prod-service-namespace
spec:
  gateways:
  - prod-service-gateway
  - mesh
  hosts:
  - microservice-e-svc
  http:
  - route:
    - destination:
        host: microservice-e-svc
        port:
          number: 8080
      weight: 90
    - destination:
        host: microservice-e.aws.dummy.com
        port:
          number: 8080
      weight: 10
```

##### Virtual service of Microsserviço E in Cloud

```sh
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  name: microservice-e-vs
  namespace: prod-service-namespace
spec:
  gateways:
  - prod-service-gateway
  - mesh
  hosts:
  - microservice-e-svc
  - microservice-e.aws.dummy.com
  http:
  - route:
    - destination:
        host: microservice-e-svc
        port:
          number: 8080
```
##### Service Entry of Microsserviço E in Onpremise
``` sh
 apiVersion: networking.istio.io/v1beta1
 kind: ServiceEntry
 metadata:
 name: microservice-e-se
 namespace: env-lending-sit-service
 spec:
 endpoints:
 - address: 10.x.y.z
 - address: 10.x.y.z
 hosts:
 - microservice-e.aws.dummy.com
 location: MESH_EXTERNAL
 ports:
 - name: tcp
   number: 38080
   protocol: HTTP
 resolution: STATIC
```

## Executando a migração simulada (Mock Migration)

- Configuração do tráfego em 10% para o microsserviço em nuvem. Monitoramento de logs, painéis e a integridade geral da aplicação.
- Aumento gradativo da porcentagem de tráfego.
- Obtenção de dados
- Retirada do roteamento do tráfego para a nuvem.
- Escolha de novos serviços para roteamento e testes com base no planejamento de migração.
- Migração de todos os microsserviços de produção.

## Mudança para a Nuvem de Produção

Após a migração simulada bem-sucedida ou que encontrou apenas problemas menores, estabelecemos um cronograma formal de migração.
Se a migração simulada tiver problemas sérios, ela precisará ser repetida após as devidas correções

IMAGEM CUTOVER

## Conclusão

Pode-se usar a malha de serviço como o Istio para migrar de um cluster kubernetes para outro e do local para a nuvem com zero tempo de inatividade.
É necessária uma quantidade razoável de maturidade das equipes envolvidas e proficiencia em Istio, mas é um aumento suave da curva de trabalho que é muito mais íngreme do Kubernetes.

Quando comparado ao esforço que seria necessário para implementar corretamente cada uma das capacidades da aplicação isoladamente e mantê-las no longo prazo o retorno do investimento faz com que seja uma técnica realmente muito viável.

## Referencias
[1] GOOGLE. What is Istio?. 09 nov. 2020. Disponível em: https://cloud.google.com/learn/what-is-istio. Acesso em: 20 jun. 2023.

[2] CM First Group. What are the Mock Migration and Final Migration project steps?. El país, 27 mai. 2019. Disponível em: https://support.cmfirstgroup.com/portal/en/kb/articles/what-are-the-mock-migration-and-final-migration-project-steps-27-5-2019. Acesso em: 21 jun. 2023.

## Autor
- [@LucasBello](mailto:lucas.bello@yahoo.com.br)

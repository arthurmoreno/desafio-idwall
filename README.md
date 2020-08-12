# desafio idwall

![arquitetura](https://github.com/arthurmoreno/desafio-idwall/blob/master/arquitetura.png)

## CORE

### Tempo de resposta longo

Para lidar com o problema de um tempo de resposta muito longo, no caso de uma janela de histórico grande com várias transações, várias técnicas podem ser utilizadas para conseguir responder o cliente sem gastar muito recurso do servidor e sem fechar a conexão devido a um time out. Abaixo algumas delas:
* Long Polling
* Web Socket
* Server-Sent Events (SSE)

Alguns problemas de algumas dessas soluções:
* Long Polling Problems
    * Header overhead
    * latencia alta
    * ter que estabelecer uma nova conexão em cada requisição

* WebSockets Problems
    * Load balancer problems
    * Proxyes and Firewalls
    * limited number of connections
    * Reinventar a roda - varias coisas que já estão prontas no protocolo HTTP precisam ser refeitas

O método mais performático e que se encaixa no caso de uso da API de classificação de transações foi o Server-Sent Events (SSE). Além dele ser simples e sem grandes overheads, é possível fornecer updates enquanto o processamento está acontecendo para o cliente. Dessa forma ele terá uma melhor impressão de que a requisição ainda está sendo processada, já que várias etapas são necessárias para finalizar a análise. O protocolo do SSE também foi criado em cima do HTTP, o que ajuda em termos de compatibilidade além de já trazer implementado as vantagens do HTTP.

links de referencia de estudo:
* https://www.youtube.com/watch?v=n9mRjkQg3VE
* https://www.ably.io/blog/websockets-vs-long-polling/
* https://hackernoon.com/scaling-websockets-9a31497af051
* https://movingfulcrum.com/the-unsolved-load-balancing-problem-of-websockets/
* https://www.youtube.com/watch?v=Z4ni7GsiIbs


Para que a aplicação seja capaz de atender um grande número de clientes sem bloquear o servidor, as requisições serão tradas de maneira **assíncrona no servidor**. Diferentemente de utilizar apenas um WSGI HTTP Server para poder criar vários workers e conseguir responder às requisições de maneira assíncrona, a API deve ser capaz de utilizar melhor os seus recursos para conseguir responder mais clientes ao mesmo tempo. Nessa proposta. Vou utilizar nesse projeto o [AioHttp](https://docs.aiohttp.org/en/stable/), que é um framework para servidor e clientes http assíncronos. Para poder criar as conexões SSE com o Client final utilizarei [aiohttp-sse](https://github.com/aio-libs/aiohttp-sse)

### Comunicação entre microserviços

As chamadas aos outros microserviços serão assíncronas, utilizando o client assíncrono da biblioteca [aiohttp](https://docs.aiohttp.org/en/stable/).


### Fluxo de uma requisição

Abaixo o fluxo da aplicação CORE
![Fluxo da api core](https://github.com/arthurmoreno/desafio-idwall/blob/master/CORE_Flow.png)

## Integrações

Os microserviços de integração podem ser especialistas, ou seja cada microserviço só sabe fazer integração com apenas um parceiro, porém isso pode levar a recursos ociosos quando temos poucas requisições de um determinado parceiro e muitas de outro específico. Além de uma alta segregação do código fonte. Porém isso pode ser interessante quando as integrações exigem tratamentos cada vez mais específicos para cada parceiro, o que seria um problema caso tudo estivesse em um único microserviço. Por agora irei adotar a estratégia de um único microserviço de integração para facilitar a escalabilidade desse microserviço, baseando a escalabilidade no numero de requisições que esse microserviço recebe. Um load balancer terá a responsabilidade de realizar o roteamento das requisições além de fazer o autoscaling.

### Testes e desenvolvimento

Para podermos obter mais informações sobre a api do parceiro, monitorar e avaliar a sua performance, os pontos abaixo devem ser levados em consideração:

* Benchmark dos parceiros para obter informações sobre as reais capacidades de suas APIs. Não queremos ultrapassar sua capacidade.
* Pontos a serem análisados em um benchmark:
    * Latency
    * Response Time
    * Availability
    * Consumption
    * Failure Rate
    * Status Codes
* Eventualmente padrões irão surgir e centralizar o modo de integração e funções específicas em uma biblioteca ou API para facilitar o desenvolvimento de novas integrações.
* Testes de integrações com os parceiros e Health Check

Referência: https://blog.bearer.sh/api-integration-best-practices/

### Limitando as requisições aos parceiros

* Para não ultrapassar o throughput dos parceiros semáforos distribuidos serão utilizados para garantir que o processo atual pode fazer a requisição. Caso o semáforo esteja ocupado, essa determinada requisição entrará para uma fila de processamento. Essa fila será consumida por outro processo desse microserviço responsável por fazer isso (ou por outro microserviço da integração responsável por reprocessar essas requisições)
* Esse semáforo distribuido pode ser desenvolvido utilizando o Redis.
- https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-3-counting-semaphores/
- https://redis.io/topics/distlock

### Falhas

* Outras situações podem levar a uma requisição de integração ir para fila de reprocessamento, isso vai depender basicamente das regras de negócio e da prioridade desse parceiro. Mas inicialmente para simplificar esse desafio exemplo, apenas requisições que recebam timeout serão incluídas na fila de reprocessamento, ou requisições com resposta de falha (4XX / 5XX).
* Para evitar que várias requisições falhem em um momento que o serviço do parceiro esteja fora do ar, um circuit breaker deve ser implementado. E quando o circuito estiver aberto as próximas requisições devem entrar numa fila ou serem ignoradas, dependendo da estratégia do negócio. Existem várias bibliotecas que facilitam a implementação dessa estratégia.

Referência: (https://blog.bearer.sh/circuit-breaker-design-pattern/)

### Consumo da fila de processamento

O processo ou rotina que irá reprocessar os items da fila também deve verificar a disponibilidade do semáforo distribuido para não estourar os limites do parceiro em questão.

### Sucesso das requisições de integração

Quando a resposta é obtida com sucesso do parceiro. O microserviço de integração deve enviar a resposta para o CORE para que esse dê continuidade ao processamento da requisição feita.

Caso a fila de reprocessamento fique muito grande devido as limitações dos parceiros ou por muitas falhas nos seus serviços, pode-se aumentar o tamanho do Redis que estava sendo utilizado como fila, ou substituir por uma fila mais escalável como o SQS

## Classificadores

De acordo com o artigo [Automatic Classification of Bank Transactions](https://ntnuopen.ntnu.no/ntnu-xmlui/bitstream/handle/11250/2456871/17699_FULLTEXT.pdf?sequence=1). O melhor algoritmo para realizar a classificação de transações bancarias é a regressão logística ou Logistic Regression. Esse algoritmo pode ser implementado com o auxílio da biblioteca scikit learn em Python.

A [documentação da biblioteca](https://scikit-learn.org/0.15/modules/scaling_strategies.html) oferece algumas estratégias para escalar o classificador.

Não entrarei nos detalhes de como escalar esse serviço pois ele depende bastante da maneira de como esse algoritmo vai se comportar. Isso vai exigir mais pesquisa e testes utilizando dados reais que serão utilizados pela aplicação.

https://www.youtube.com/watch?v=KqKEttfQ_hE - Outro link interessante de como escalar o classificador utilizando scikit-learn

## Banco de dados

Pela natureza do dado armazenado, que não possui relações (No máximo um relacionamento das transações classificadas com o usuário), um banco NoSql se encaixa bem nesse caso. Mas a maioria dos bancos NoSql não foram arquitetados para possuirem boa performance ao realizar scans para análise de dados e queries inteligentes rápidas. (Um exemplo seria o DynamoDB que não possui um bulk scan performático)

Devido a necessidade de queries para consultas com rápida resposta e consultas utilizadas para análise de grandes quantidades de dados. Devido a essa necessidade o [ElasticSearch](https://www.elastic.co/pt/what-is/elasticsearch) é uma boa opção por ser robusto e já bastante consolidado para esse propósito.

Outro teste que faria para saber se se encaixaria melhor nesse caso de uso específico seria o FiloDB. Pois em alguns benchmarks ele se saiu muito bem em fazer análise de grandes conjuntos de dados.

> "FiloDB brings the benefits of efficient columnar storage and the flexibility and richness of Apache Spark to the rock solid storage technology of Cassandra, speeding up analytical queries by up to 100x over Cassandra 2.x."

Links sobre FiloDB:
* https://github.com/filodb/FiloDB
* https://www.oreilly.com/content/apache-cassandra-for-analytics-a-performance-and-storage-analysis/
* http://velvia.github.io/Introducing-FiloDB/

Porém por ter uma comunidade maior e um suporte oficial, eu escolheria o ElasticSearch como o banco de dados para armazenar as respostas da API.

Biblioteca assíncrona para se comunicar com o elasticsearch:
* https://github.com/aio-libs/aioelasticsearch

## Infraestrutura, Deploys e Monitoramento

Para garantir a escalabilidade da aplicação algumas métricas devem ser observadas e analisadas para poder melhor traçar a estratégia de scaling. Que pode ocorrer também em períodos específicos que os usuários mais utilizam o serviço. Porém inicialmente um load balancer junto com o serviço de Containers da AWS (ECS) seria uma boa opção. Um Cluster Kubernetes também poderia fazer esse papel, principalmente se essa ou alguma outra solução de orquestração de containers já estiver sendo utilizada na empresa.

Por possuir muitas instâncias de cada serviço em execução em um determinado momento, estratégias de [blue/green deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html) e de [canary deploys](https://martinfowler.com/bliki/CanaryRelease.html) podem ser uteis para evitar grandes falhas e facilitar rollbacks das versões.

O [grafana](https://grafana.com/) seria uma boa ferramenta para visualizar várias métricas personalizadas. Com dados sendo extraídos do ElasticSearch ou do Próprio provedor Cloud.
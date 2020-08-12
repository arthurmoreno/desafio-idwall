# desafio idwall

## Core

### Tempo de resposta longo

Para lidar com o problema de um tempo de resposta muito longo, no caso de uma janela de histórico grande com várias transações, várias técnicas podem ser utilizadas para conseguir responder o cliente sem gastar muito recurso do servidor e sem fechar a conexão devido a um time out. Abaixo algumas delas:
* Long Polling
* Web Socket
* Server-Sent Events (SSE)

(https://www.ably.io/blog/websockets-vs-long-polling/)
(https://www.youtube.com/watch?v=n9mRjkQg3VE)

Long Polling Problems - Header overhead, máxima latencia, ter que estabelecer uma nova conexão em cada requisição

(https://www.ably.io/blog/websockets-vs-long-polling/)

WebSockets Problems - Load balancer problems, Proxyes and Firewalls, limited number of connections, varias coisas que já estão prontas no protocolo HTTP precisam ser refeitas
https://hackernoon.com/scaling-websockets-9a31497af051
https://movingfulcrum.com/the-unsolved-load-balancing-problem-of-websockets/

SSE - Unidirecional
is a light weight protocol on top of http

unlike websockets sse does not provide a two way comunication but can be used by server to push data to client in real time

(https://www.youtube.com/watch?v=Z4ni7GsiIbs)

Logo SSE será utilizado - falta organizar e escrever texto dessa decisão

Para que a aplicação seja capaz de atender um grande número de clientes sem bloquear o servidor, as requisições serão tradas de maneira assíncrona no servidor. Diferentemente de utilizar apenas um WSGI HTTP Server para poder criar vários workers e conseguir responder às requisições de maneira assíncrona, a API deve ser capaz de utilizar melhor os seus recursos para conseguir responder mais clientes ao mesmo tempo. Nessa proposta 


https://docs.aiohttp.org/en/stable/

https://github.com/aio-libs/aiohttp-sse

### Fluxo de uma requisição

Desenhar fluxo de uma requisição paralela na aplicação CORE



## Integrações

Os microserviços de integração podem ser especialistas, ou seja cada microserviço só sabe fazer integração com apenas um parceiro, porém isso pode levar a recursos ociosos quando temos poucas requisições de um determinado parceiro e muitas de outro específico. Além de uma alta segregação do código fonte. Porém isso pode ser interessante quando as integrações exigem tratamentos cada vez mais específicos para cada parceiro, o que seria um problema caso tudo estivesse em um único microserviço. Por agora irei adotar a estratégia de um único microserviço de integração para facilitar a escalabilidade desse microserviço, baseando a escalabilidade no numero de menssagens que esse microserviço recebe.

-> ver sobre auto scaling no caso de um pub/sub utilizando SNS.


### Testes e desenvolvimento

* Testes de integrações com os parceiros e Health Check
* Benchmark dos parceiros para obter informações sobre as reais capacidades de suas APIs. Não queremos ultrapassar sua capacidade.

* Pontos a serem análisados em um benchmark (https://blog.bearer.sh/api-integration-best-practices/):
    * Latency
    * Response Time
    * Availability
    * Consumption
    * Failure Rate
    * Status Codes
* Eventualmente padrões irão surgir e centralizar o modo de integração e funções específicas em uma biblioteca ou API para facilitar o desenvolvimento de novas integrações.

### Limitando as requisições aos parceiros

* Para não ultrapassar o throughput dos parceiros semaforos distribuidos serão utilizados para garantir que o processo atual pode fazer a requisição. Caso o semáforo esteja ocupado, essa determinada requisição entrará para uma fila de processamento. Essa fila será consumida por outra processo desse microserviço responsável por fazer isso (ou por outro microserviço da integração responsável por reprocessar essas requisições)
* Esse semáforo distribuido pode ser desenvolvido utilizando o Redis.
- https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-3-counting-semaphores/
- https://redis.io/topics/distlock

### Falhas

* Outras situações podem levar a uma requisição de integração ir para fila de reprocessamento, isso vai depender basicamente das regras de negócio e da prioridade desse parceiro. Mas inicialmente para simplificar esse desafio exemplo, apenas requisições que recebam timeout serão incluídas na fila de reprocessamento.
* Para evitar que várias requisições falhem em um momento que o serviço do parceiro esteja fora do ar, um circuit breaker deve ser implementado. E quando o circuito estiver aberto as próximas requisições devem entrar nua fila ou serem ignoradas, dependendo da estratégia do negócio. Existem várias bibliotecas que facilitam a implementação dessa estratégia. (https://blog.bearer.sh/circuit-breaker-design-pattern/)


### Consumo da fila de processamento

O processo ou rotina que irá reprocessar os items da fila também deve verificar a disponibilidade do semáforo distribuido para não estourar os limites do parceiro em questão.

### Sucesso das requisições de integração

Quando a resposta é obtida com sucesso do parceiro. O microserviço de integração deve publicar uma menssagem para o CORE para que esse dê continuidade ao processamento da requisição feita.

Caso a fila de reprocessamento fique muito grande devido as limitações dos parceiros ou por muitas falhas nos seus serviços, pode-se aumentar o tamanho do Redis que estava sendo utilizado como fila, ou substituir por uma fila mais escalável como o SQS

## Classificadores

De acordo com o artigo Automatic Classification of Bank Transactions disponível nesse link (https://ntnuopen.ntnu.no/ntnu-xmlui/bitstream/handle/11250/2456871/17699_FULLTEXT.pdf?sequence=1). O melhor algoritmo para realizar a classificação de transações bancarias é a regressão logística ou Logistic Regression. Esse algoritmo pode ser implementado com o auxílio da biblioteca scikit learn em Python.
A documentação da biblioteca oferece algumas estratégias para escalar o classificador.

* https://scikit-learn.org/0.15/modules/scaling_strategies.html

Não entrarei nos detalhes de como escalar esse serviço pois ele depende bastante da maneira de como esse algoritmo vai se comportar. Isso vai exigir mais pesquisa e testes utilizando dados reais que serão utilizados pela aplicação.

https://www.youtube.com/watch?v=KqKEttfQ_hE - Outro link interessante de como escalar o classificador utilizando scikit-learn

## Banco de dados

Pela natureza do dado armazenado, que não possui relações (No máximo um relacionamento das transações classificadas com o usuário), um banco NoSql se encaixa bem nesse caso. Mas a maioria dos bancos NoSql não foram arquitetados para possuirem boa performance ao realizar scans para análise de dados e queries inteligentes rápidas. (Um exemplo seria o DynamoDB que não possui um bulk scan performático)

Devido a necessidade de queries para consultas com rápida resposta e consultas utilizadas para análise de grandes quantidades de dados. Devido a essa necessidade o ElasticSearch é uma boa opção por ser robusto e já bastante consolidado para esse propósito.

* https://www.elastic.co/pt/what-is/elasticsearch

Outro teste que faria para saber se se encaixaria melhor nesse caso de uso específico seria o FiloDB. Pois em alguns benchmarks ele se saiu muito bem em fazer análise de grandes conjuntos de dados.

"FiloDB brings the benefits of efficient columnar storage and the flexibility and richness of Apache Spark to the rock solid storage technology of Cassandra, speeding up analytical queries by up to 100x over Cassandra 2.x."

* https://github.com/filodb/FiloDB
* https://www.oreilly.com/content/apache-cassandra-for-analytics-a-performance-and-storage-analysis/
* http://velvia.github.io/Introducing-FiloDB/

Porém por ter uma comunidade maior e um suporte oficial, eu escolheria o ElasticSearch como o banco de dados para armazenar as respostas da API.

## Infraestrutura, Deploys e Monitoramento


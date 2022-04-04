---
author: Dnnes
date: 'April 2, 2022'
excerpt_separator: |
    <!--more-->
layout: post
output:
  md_document:
    preserve_yaml: True
    variant: 'markdown\_github'
tags:
- Python
- DynamoDB
- Lambda
- AWS
- Serverless
- API Gateway
title: 'Pt2. Construindo uma API REST com o AWS Lambda'
toc: True
---

true

O DynamoDB é um serviço de banco de dados *serverless* NoSQL da AWS . As requisições de escrita e leitura são feitas por HTTP (o processo é abstraido pela biblioteca `boto3`). No exemplo deste post, onde 4 canais foram assinados simultaneamente, o Dynamo teve picos de 100 requests por segundo e não retornou nenhum erro. Segundo a Amazon, o limite é de 1000 escritas por segundo. <!--more-->

![](/assets/writes_2022-04-02_16-02-51.png) *Escritas na tabela no período de testes. Neste intervalo foram criados cerca de 600.000 registros.*

### Modelagem no DynamoDB

A modelagem da tabela é bem diferente do modelo de banco de dados relacional. É um assunto extenso e que eu preciso estudar muito ainda, mas alguns pontos importantes são:
1. A chave primária pode ser composta por dois campos, um será o partition key e outro o sort key e devem formar uma combinação única.
2. As chaves otimizam as queries e a tabela deve ser modelada tendo em vista os padrões de acesso.
3. Em apenas uma tabela é possível armazenar estruturas diferentes de dados (desde que possuam os campos da chave). Por exemplo, como os dados serão consumidos por uma API, em uma só tabela eu poderia armazenar todos os dados de uma exchange sem me preocupar com a diferença estrutural entre os items de cada canal, pensando apenas em como os dados serão servidos.

Aqui um exemplo de duas estruturas de dados diferentes que fora armazenados na mesma tabela:

``` json
{"id":1474637022322692,"id_str":"1474637022322692","order_type":1,"datetime":"1648853779","microtimestamp":"1648853778936000","amount":0.1349848,"amount_str":"0.13498480","price":46308.21,"price_str":"46308.21", "channel":"live_orders_btcusd"}

{"id":227050877,"timestamp":"1648947384","amount":0.01,"amount_str":"0.01000000","price":46035.88,"price_str":"46035.88","type":1,"microtimestamp":"1648947384302000","buy_order_id":1475020428861441,"sell_order_id":1475020429905922, "channel":"live_trades_btcusd}
```

*Cada item possui uma estrutura e ambos possuem os campos da chave primária, desse modo podem ser armazenados na mesma tabela*

Eu utilizei os campos id e microtimestamp como chave principal porque cada par forma uma combinação única. No entanto, isso é considerado um *anti-padrão* já que nenhuma query será executada nesses campos. Eventualmente eu descobri que poderia criar [índices secundários](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html) com outros campos. Aí utilizei o nome do canal e o microtimestamp, esses campos sim servirão como argumentos de query na API.

``` json
 "TableName": "bitstamp-live",
        "KeySchema": [
            {
                "AttributeName": "id",
                "KeyType": "HASH"
            },
            {
                "AttributeName": "microtimestamp",
                "KeyType": "RANGE"
            }
        ]
```

*As chaves da tabela: Hash é a partition key e Range é a sort key*

Após rodar o script de coleta de dados por algumas horas eu percebi que o campo microtimestamp estava como string e teria que converter para formato numérico para poder executar as operações de comparação das queries (maior ou igual, menor, between...). No DynamoDB não existe um modo 'rápido' de fazer isso. É necessário iterar sobre todos os registros utilizando o scan() e fazer um update item a item ([aqui está um gist com o código utilizado](https://gist.github.com/dnnes/c43b919b1a05e8aa2d07221c5ab2b16c)). Criei um novo campo chamado 'nmicrotimestamp' e adicionei um índice que tem como chave o nome do canal e o microtimestamp numerico (channel e nmicrotimestamp):

``` json
"IndexName": "channel-nmicrotimestamp-index",
                "KeySchema": [
                    {
                        "AttributeName": "channel",
                        "KeyType": "HASH"
                    },
                    {
                        "AttributeName": "nmicrotimestamp",
                        "KeyType": "RANGE"
                    }
                ],
```

Para cada query, o DynamoDB consegue retornar um documento JSON com no máximo 1MB (aproximadamente 5k itens, nesse caso). Quando a resposta passa de 1MB, o documento vem com o campo `LastEvaluatedKey` indicando a chave primária do último item que pôde ser retornado. Uma nova query deve ser feita utilizando os valores do `LastEvaluatedKey` como ponto de partida no campo `ExclusiveStartKey`. Esses passos devem ser repetidos até que todos os valores sejam recuperados e não haja mais itens no LastEvaluatedKey.
![](/assets/JSON_LEK_2022-04-02_20-25-30.png) *Exemplo de resposta da API com o campo LastEvaluatedKey*

Eu poderia tratar esse problema no backend iterando a função Lambda até retornar todos os itens da query. Porém, seria necessário aumentar o timeout para valores bem altos pensando nos casos onde o documento teria, por exemplo, mais que 100MB. E a função Lambda idealmente (e isso fica claro pelo modo como é calculada a cobrança) deve realizar as tarefas em um curto espaço de tempo. Assim, eu decidi que essa limitação deveria ser tratada no lado do cliente, já que é preferível múltiplas requests de 1MB a uma só de 50MB ou mais.

### AWS Lambda e API Gateway

A API é simples: recebe o nome do canal e o intervalo de tempo que os dados serão recuperados. O backend é uma função Lambda que trata as requisições que chegam através da API Gateway. A função executa uma query no DynamoDB, o resultado da query é retornado no formato JSON e é enviado para o cliente pela API Gateway. Os parâmetros da query são passados na requisição pela **query string** da url e capturados na variável **event**.

A API final ficou nesse formato:

    https://vspwaobhlj.execute-api.us-east-1.amazonaws.com/prod/bitstamp?channel=&from=&to=

A variável `channel` pode assumir um dos 4 nomes de canais que foram assinados no post anterior. *From* é o tempo inicial no formato [Unix time](https://en.wikipedia.org/wiki/Unix_time) e *to* é o tempo final. Esse é o formato mais conveniente para trabalhar com milissegundos embora não seja 'humanamente compreensível' sem utilizar um [conversor](https://www.unixtimestamp.com/).

O menor valor para a variável `from` é 1647830700000000 e o valor máximo para o `to` é 1647844020000000.

A variável `channel` pode assumir: live\_trades\_btcusd, live\_trades\_ethusd, live\_orders\_btcusd, live\_orders\_ethusd.

Tanto a API Gateway quanto a Lambda são serviços simples de configurar para esse caso. O código da função Lambda ficou assim:

``` python
import json
import boto3
from boto3.dynamodb.conditions import Key

def lambda_handler(event, context):
    
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('bitstamp-live')
    
    channel = event["params"]["querystring"]["channel"]
    init = int(event["params"]["querystring"]["from"])
    to = int(event["params"]["querystring"]["to"])
    
    query_result = table.query(IndexName = 'channel-nmicrotimestamp-index', 
                                KeyConditionExpression =
                                Key('channel').eq(channel) &
                                Key('nmicrotimestamp').between(init , to))
    

    return {
        'statusCode': 200,
        'headers': {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'},
        'body': query_result
    }
```

A função ganhou previamente um acesso ao dynamodb pelas configurações de IAM. A execução é bem direta: recupera os parâmetros que chegam através do `event`; esses parâmetros são passados para a query e por fim o resultado é retornado em formato json.

A visão geral da API Gateway ficou assim:

![](/assets/API_Gateway.png)

A principal configuração é a da integração:

![](/assets/API_Gateway_integration.png)

Primeiro é necessário informar o tipo de integração (Lambda, nesse caso); segundo, o nome da função Lambda. No passo 3 é necessário informar os parâmetros da query string que serão utilizados no quarto passo. O quarto passo permite definir um template personalizado para acessar os parâmetros na função Lambda. Pelo template é possível definir um modo de acessar as variáveis de forma mais direta, por exemplo `event.channel`.

### Conclusão:

Neste projeto eu tive meu primeiro contato com o DynamoDB e com a AWS Lambda. Mesmo se tratando de uma aplicação pequena e que não testa as capacidades de escalabilidade (principalmente do Dyanamo), é sempre muito gratificante ver um *toy project* pronto. O maior valor está no processo de aprender através da leitura da documentação, de livros e posts de outros blogs. Essa habilidade, sem qualquer dúvida, é uma das mais utilizadas por qualquer desenvolvedor. Independente da senioridade.

E por fim, aqui fica um exemplo funcional da API:

<https://vspwaobhlj.execute-api.us-east-1.amazonaws.com/prod/bitstamp?channel=live_orders_btcusd&from=1647817207170000&to=1647819909209000>

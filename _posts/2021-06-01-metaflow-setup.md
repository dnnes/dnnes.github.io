---
title: "Metaflow  - Configuração Mínima"
last_modified_at: 2021-06-01 19:08:25 +0800
date: "2021-06-01 19:08:25 +0800"
tags:
- Python
- R
- AWS
- Metaflow
layout: post
toc: yes
excerpt_separator: <!--more-->
---



Recentemente, assistindo às conferencias da useR de 2020 descobri o  [Metaflow](https://www.youtube.com/watch?v=gtrODWCpXeo). Dentre várias features oferecidas pela ferramenta desenvolvida na Netflix a proposta principal é tornar possível a prototipagem do código localmente e executar algumas partes na nuvem (apenas o tuning do modelo, por exemplo) <!--more--> utilizando os recursos escaláveis de computação da AWS. Isso me chamou muito a atenção já que o processamento local nem sempre é suficiente para algumas tarefas mais exigentes e a execução do código totalmente na nuvem é uma tarefa que às vezes consome tempo.

Posto de um modo simples, a ideia do Metaflow é permitir que o cientista de dados na etapa de desenvolvimento do modelo não perca muito tempo com o provisionamento de infraestrutura e foque mais em otimização e em feature engineering, por exemplo. Um meio do caminho entre prototipagem e produção.


### Metaflow e AWS

A biblioteca é open source, escrita em Python e compatível com R. Ainda está em desenvolvimento e uma grande quantidade de informação pode ser encontrada nas issues do diretorio no [Github](https://github.com/Netflix/metaflow). *Under the hood*, o Metaflow provê uma API para a stack de recursos na AWS previamente configurados. O código será executado em containers de instâncias EC2, Spot ou Fargate. Essas configurações podem ser feitas manualmente (seguindo este guia de deploy) ou por um template do CloudFormation que automatiza o processo. Eu preferi fazer as configs manuais, o template faz o provisionamento de um conjunto completo de serviços para quem quer utilizar todos os recursos do Metaflow, muito além do que eu preciso agora.

Outro atrativo é que as modificações necessárias no código para a utilização do Metaflow não são nada complicadas. Um tempo maior é gasto configurando os recursos na AWS que na adaptação do código.


### Configurando

Aqui eu vou apresentar um pequeno guia dos passos que segui para conseguir instalar e configurar o Metaflow com uma quantidade reduzida de recursos mas que me permitisse ainda executar o processamento na nuvem.

O primeiro passo é instalar o `awscli`, no Debian se faz normalmente via `apt-get`.

Em seguida, é necessário copiar suas credenciais AWS para `~/.aws/credentials`, o que vai permitir a autenticação tanto com o awscli quanto com o Metaflow. Fica assim:

![](/assets/credentials_f.png)  



O Metaflow já pode ser instalado com o comando `pip install metaflow`

Agora os tutoriais podem ser baixados com o comando `metaflow tutorials pull`; para listar os tutoriais `metaflow tutorials list`; se tudo estiver certo, o resultado será parecido com isso:

![](/assets/tutorials.png)  



No último passo o exemplo mínimo que vamos utilizar para testar se tudo foi configurado corretamente é o  `helloaws.py`.

Os passos de configuração na AWS eu segui exatamente a [receita do manual](https://admin-docs.metaflow.org/metaflow-on-aws/deployment-guide/manual-deployment), não faz sentido reproduzir aqui.  Do passo de **metadados** pra frente não utilizei nenhum recurso. 

Resumindo, foi necessário criar:

-S3 Buckets; 

-Elastic IP; 

-VPC que utiliza o Elastic IP previamente criado, com subnets pública e  privada, a pública deve possuir atribuição de IP; 

-Managed AWS Batch compute environment com instâncias EC2 On-Demand, Spot ou Fargate; 

-Queue para o Batch; 

-IAM para o Batch; 




Um modo de testar o login do awscli é executar a listagem dos buckets S3 com o comando `aws s3api list-buckets`:

![](/assets/list_buckets.png)  




se obtiver algum erro de autenticação aqui, será necessário rever suas credenciais editando `~/.aws/credentials`.

Se tudo estiver funcionando corretamente, é hora de executar `metaflow configure aws` e preencher com as ARN's (Amazon Resource Name). Este comando cria o arquivo `config.json` em `~/.metaflowconfig/config.json`. O meu ficou assim:

![](/assets/config_json.png)  


E por fim, para testar o se tudo está OK, eu rodo o tutorial `helloaws`:

![](/assets/all_process.png)   

A primeira task é executada localmente. A segunda tarefa que vemos na imagem já roda na AWS; vemos o avanço do status da task na queue, o setup do ambiente e finalmente a tarefa é executada remotamente: um print "Metaflow says: Hi from AWS" (seguido de um warning de descontinuação de pacote na minha máquina).

### Simples assim?
Aqui devo confessar que (por minha culpa)o processo não foi tão direto. Falhei em alguns passos do tutorial de deploy e por várias vezes, ao tentar executar o `helloaws.py`, me peguei assistindo o status RUNNABLE do AWS batch sendo impresso no terminal por alguns minutos antes de me convencer que não sairia disso e cancelar manualmente o job. Em outras tentativas, sem qualquer mudança nas configurações, tudo fluia sem maiores problemas. Verifiquei que a maquina virtual era criada e passei a desconfiar da VPC. Acabei criando um log na CloudWatch para a VPC que estava utilizando e percebi que a intermitência se devia à configuração das subnets. Provavelmente eu negligenciei o parágrafo do guia de deploy que diz **“These subnets are not auto-assigned public IPv4 addresses. Instances launched in the public subnet must be assigned a public IPv4 address to communicate with the Amazon ECS service endpoint.”** Editei a configuração de ip auto-atribuido na sub publica (na imagem abaixo, no canto superior direito) e tudo se resolveu. Até o momento não tive mais o problema.

![](/assets/public_ipv4.png)  



### AWS Educate

Vale ressaltar que os créditos da aws educate, que estudantes de algumas universidades têm acesso, são válidos para configurar pelo menos as partes que eu utilizei (buckets S3, vpc com subnets, roles na iam, batch e instâncias ec2). A única ressalva é que as credenciais fornecidas pela Vocareum expiram em um intervalo de tempo fixo (aproximadamente 3hrs) e se a tarefa posta na fila não for concluida antes do termino desse intervalo, você pode obter o erro “An error occurred (ExpiredTokenException) when calling the DescribeJobs operation: The security token included in the request is expired”. Problema de resolução simples: só acessar e copiar as novas credenciais para o documento  ~/.aws/credentials no seu pc, que é automaticamente criado quando o aws cli é instalado.

![](/assets/access.png)  

### Próximos posts

Pretendo fazer pelo menos mais dois posts para esclarecer 1)Como utilizar containers personalizados e as vantagens de utilizar EC2, Spot ou Fargate; 2)Demonstrar o uso do metaflow com o R.

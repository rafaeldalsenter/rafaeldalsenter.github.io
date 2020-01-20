---
layout: post
title:  "Criando um microservice de alta disponibilidade em C# com banco de dados NoSQL (Parte I)"
date:   2020-01-19
categories: Csharp Cassandra AWS
---
Neste primeiro artigo do ano, irei descrever a criação de um Microservice projetado para ter alta disponibilidade, utilizando .NET Core em conjunto a um banco de dados NoSQL Cassandra.

Nesta primeira parte, irei focar na criação do banco de dados Cassandra. Subirei o mesmo em containers Docker e criarei a estrutura base, que contém uma tabela com um [dataset](https://www.kaggle.com/shivamb/netflix-shows/data#) obtido no Kaggle.

### Banco de dados Cassandra

O banco de dados Cassandra é feito para trabalhar de forma distribuída, onde pode utilizar várias máquinas (chamadas de nós). Assim, sua principal característica é o fato de ser descentralizada, ou seja, todos os nós de rede possuem as mesmas funções e capacidades. Esses nós são totalmente independentes, não compartilhando nenhum recurso de disco, processamento ou memória.

Por ter uma arquitetura distribuída e descentralizada, o Cassandra é altamente escalável de forma horizontal. Caso seja necessário mais performance basta adicionar mais nós na rede, não necessitando substituir máquinas ou aumentar o processamento.

Bem, isso foi uma visão bem básica do funcionamento do Cassandra, há outras especificidades como o uso de chave-valor, arquitetura em anel, etc.. No DevMedia, há um [artigo](https://www.devmedia.com.br/introducao-ao-cassandra/38377) que explica bem detalhadamente o funcionamento do Cassandra.

### Subindo os contâiners do banco de dados

Como já dito anteriormente, irei trabalhar inicialmente com o banco Cassandra em contâiners Docker. Irei montar um ambiente bem básico, onde subirei dois contâiners Cassandra, que serão dois nós. Serão eles o container-db e o container-db2.

Antes de tudo, irei criar uma rede específica para subir os dois contâiners

```bash
docker network create -d bridge cassandra-network
```

Com a rede criada irei subir dois contâiners a partir da imagem do Cassandra disponível no [DockerHub](https://hub.docker.com/_/cassandra). Lembrando que o segundo contâiner terá um parâmetro extra passado. O parâmetro indicará que o primeiro contâiner será o nó ligado a ele.

```bash
docker run --name cassandra-db --network cassandra-network -d cassandra

docker run --name cassandra-db2 -d --network cassandra-network -e CASSANDRA_SEEDS=cassandra-db cassandra
```

### Inserindo dados no banco

Beleza, agora já temos o banco de dados Cassandra "de pé" com dois nós. Para interagirmos com ele utilizamos a linguagem CQL. Esta linguagem é exclusiva do banco Cassandra e é muito semelhante ao tradicional SQL, assim é familiar para quem (como eu) vem do mundo dos bancos relacionais.

Iremos inicialmente criar um Keyspace, que é onde serão armazenados as tabelas. Para lançar comandos no Cassandra, teremos que abrir a linha de comando do contâiner Docker. Passando o parâmetro cqlsh já iremos abrir com o bash para execuções de queries CQL:

```bash
docker exec -it cassandra-db cqlsh
```

Para criar o Keyspace utilizaremos o comando abaixo. nele passaremos dois parâmetros, o primeiro indica o método de replicação que será utilizado para distribuir as réplicas das partições. Neste caso o SimpleStrategy é o padrão, ele irá criar o número de réplicas especificado no parâmetros replication_factor. Aqui colocaremos 2 tendo em vista que temos dois nós. Isso indica que haverá duas cópias de cada linha no cluster do Cassandra, e que cada cópia está em um nó diferente.

```sql
create keyspace if not exists "default_keyspace" with replication = {'class': 'SimpleStrategy', 'replication_factor': 2};
```

Com o Keyspace criado, iremos criar uma tabela dentro dele. O comando de criação da tabela é semelhante ao SQL, diferenciado principalmente nos tipos de dados:

```sql
create table "default_keyspace"."netflix_titles" (
	id uuid,
	title text,
	director text,
	cast text,
	country text,
	duration_min int,
	description text,
	primary key(id)
);
```
Para montar a base de dados, utilizei o dataset do Kaggle [Netflix Movies and TV shows](https://www.kaggle.com/shivamb/netflix-shows/data#) que está em formato CSV. Fiz uma "semi" limpeza de dados e gerei os inserts com a ajuda do Google Sheets. O txt com os inserts estão [aqui](/assets/docs/inserts_database_cassandra_netflix.txt).

**Obs.:** Rodei todos os inserts e boa parte deles falharam, provável por formatação incorreta, pontuação, etc. Como o foco do trabalho aqui não é limpeza de dados ignorei os mesmos.

Após executar os inserts e rodar um comando de Count, obtive a quantidade de registros:

![Imagem](/assets/images/cassandra_select1.png)

Legal, agora já temos a base com as informações para trabalharmos.

Algumas outras características importantes do banco Cassandra:
- Aqui temos uma tabela simples com várias colunas. Podemos concordar que a coluna **cast** é uma lista concatenada em uma string. Neste caso, o caminho correto seria criar uma tabela com a lista de elementos e ligá-las através de um Id, correto? **Não**. Embora seja possível, não é recomendável o uso de JOIN entre tabelas pois tende a ser muito custoso (pelo fato de os dados estarem distribuídos em vários nós diferentes do cluster). Neste cenário, o ideal seria o uso de uma coluna do tipo Collection, onde pode haver uma lista de valores ou um conjunto de chaves e valores.
- Também é possível criar tipos customizados, que podem ser necessários caso ao invés de usar uma Lista de strings como Collection, eu queira utilizar uma lista de algum tipo de objeto, por exemplo.

Bom, isso é uma passada bem básica no banco Cassandra, nos próximos artigos desse projeto, pretendo criar uma Microservice em .NET core que faça a leitura e escrita nesse banco.

Até +



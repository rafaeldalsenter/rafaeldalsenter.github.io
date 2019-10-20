---
layout: post
title:  "Docker compose: Nginx + aplicação web + banco de dados"
date:   2019-10-20
categories: Docker Nginx SQLServer
---

Neste artigo, vou demonstrar o uso do docker-compose em um ambiente mais próximo de uma aplicação real. Aqui teremos uma aplicação web simples (que somente faz insert em um banco de dados), o banco de dados em si e um servidor Nginx.

Replicaremos o servidor da aplicação em três containers para que o Nginx possa atuar como Load balance. Na imagem abaixo, poderemos ver detalhadamente o desenho da infra-estrutura a ser criada:

![Imagem](/assets/images/exemplo_container_docker.png)

## Aplicação Web

A aplicação web, está no projeto [docker-example-app][docker-example-app] que basicamente é um formulário ASP.NET que insere informações em um banco de dados. É importante destacarmos dois pontos:

- Aqui não foi focado em boas práticas, simplesmente foi feita uma aplicação Web que insere em um banco de dados (por favor, não me julgue por esse código :p).
- o Dockerfile da aplicação utiliza a imagem **microsoft/aspnetcore**, porém pode ser utilizado o Dockerfile criado pelo Visual Studio (que builda o projeto e publica o mesmo) sem problema nenhum. Aqui foi optado por essa imagem por ela ter um tamanho menor.

## SQL Server

Para o SQL Server, será utilizado o contâiner padrão da Microsoft (mcr.microsoft.com/mssql/server). É importante frisar que essa imagem contém somente o SQL Server "puro", ou seja, para fazermos consultas nele teremos que utilizar a linha de comando.

## Nginx

Aqui teremos uma aplicação bem simples do Proxy Nginx, que utilizaremos como Load-balance. O mesmo será responsável pelo direcionamento das requisições entre os três servidores de aplicação web e também será o contâiner que orquestrará a subida dos demais via arquivo docker-compose. As configurações do mesmo ficam no arquivo **nginx.conf** que é demonstrado abaixo:

```
worker_processes 4;

events { worker_connections 1024; }

http {    
    upstream container {
            least_conn;
            server container-app1;
            server container-app2;
            server container-app3;
    }
    
    server {
            listen 80; 
            location / {
                proxy_pass http://container;
            }
    }
}
```
Aqui basicamente eu adiciono o Hostname ou IP dos três servidores de aplicação e o Nginx irá redirecionar as requisições entre os três.

## Construção do docker-compose

No mesmo projeto onde localiza-se o Nginx, iremos criar o arquivo **docker-compose.yml** que conterá todas as informações para subir essa estrutura. Todos os contâiners utilizarão a mesma rede para que possam se comunicar entre si. Abaixo irei demonstrar separadamente o arquivo docker-compose:

O primeiro service a ser montado, será o banco de dados SQL Server. Note que no docker-compose do mesmo são passados configurações específicas do ambiente do SQL Server (_environments_), no qual eu digo que aceito o EULA e passando a senha do usuário SA.
```dockerfile
sqlserver:
    image: mcr.microsoft.com/mssql/server
    container_name: container-sql
    ports:
        - "1433:1433"
    environment:
        SA_PASSWORD: "Docker12345"
        ACCEPT_EULA: "Y"
    networks: 
        - minha-rede
```

Após a montagem do contâiner do SQL Server, criaremos três contâiners referente a aplicação. Todos eles terão a estrutura semelhante a abaixo, só mudando o nome (app1, app2 e app3) e também o container_name. Note que aqui eu específico o parametro _depends_on_. Esse parâmetro indica que esse contâiner só irá subir após o contâiner do SQL Server.
```dockerfile
app1:
    image: docker-example-app
    container_name: container-app1
    ports:
        - "5000"
    networks: 
        - minha-rede
    depends_on:
        - "sqlserver"
```

 E por final, a montagem do contâiner do Nginx. Este contâiner será o último a subir na ordem e fará o load balance para os demais contâiners de aplicação.
```dockerfile
nginx:
    build:
        dockerfile: ./docker/nginx.dockerfile
        context: .
    image: nginx
    container_name: container-lb
    ports:
        - "80:80"
    networks: 
        - minha-rede
    depends_on:
        - "app1"
        - "app2"
        - "app3"
```

Tendo o arquivo docker-compose montado (arquivo completo em [docker-compose][docker-compose]), vamos agora lançar os comandos para montagem dos mesmos.

## Montagem do ambiente

Para montagem do ambiente, o primeiro passo é montar a imagem referente a aplicação web. Após fazer o download do projeto [docker-example-app][docker-example-app] abrir a linha de comando no diretório da pasta do projeto e lançar o comando de build:

```
docker build -t docker-example-app .
```

Agora, iremos subir todo o ambiente, para isso é necessário fazer o download do projeto [docker-example-load-balance][docker-example-load-balance] e abrir a linha de comando no diretório da pasta do projeto. Faremos os seguintes comandos:

```
// Aqui ele irá montar todas as imagens no arquivo yml e deixa-las prontas para subir
docker-compose build

// Aqui ele irá subir as máquinas conforme determinado
docker-compose up

```

Finalizar




[docker-example-app]: https://github.com/rafaeldalsenter/docker-example-app
[docker-example-load-balance]: https://github.com/rafaeldalsenter/docker-example-load-balance
[docker-compose]: https://github.com/rafaeldalsenter/docker-example-load-balance/edit/master/docker-compose.yml
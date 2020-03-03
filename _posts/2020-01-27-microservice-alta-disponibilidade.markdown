---
layout: post
title:  "Criando um microservice de alta disponibilidade em C# com banco de dados NoSQL (Parte II)"
date:   2020-01-27
categories: Csharp Cassandra AWS
---
Dando continuidade ao artigo anterior, agora focaremos na contrução do Microservice em .NET Core. Nele utilizaremos alguns conceitos de programação como injeção de dependência, repository e builder.

O Microservice basicamente será uma API que retornará as informações de filmes inseridas no banco Cassandra, e também inserirá filmes novos. A princípio não há muitas validações no domínio, porém a estrutura estará pronta para estas serem adicionadas.

O código-fonte desse projeto pode ser localizado [aqui](https://github.com/rafaeldalsenter/ms-netflix-titles)

### Construção do projeto

Inicialmente, estruturei a aplicação em algumas divisões de responsabilidade:

- Domain: Aqui terá as classes de domínio com suas propriedades (por enquanto só existe o domínio NetflixTitle) e os Builders para construção dos domínios.
- Application: Terá as classes de consulta ao banco (Repositories) e também as classes de inserção no banco (Services).
- CrossCutting: Aqui terão as classes que podem ser utilizadas em qualquer camada, como objetos Dto, extensions e conexão com o Banco Cassandra.
- Api: Api para exposição dos endpoints de Get e Post.
- Tests: Projetos de testes unitários das camadas de Domain e Application.

### Domínio

Aqui só teremos um domínio, que será o NetflixTitle, que basicamente representa o registro na base de dados. Na construção criei uma classe abstrata chamada **BaseDomain** que terá o método indicando se o domínio é valido e a lista de erros. Todos os domínio herdam dessa classe.

NO domínio **NetflixTitle** somente teremos regras em dois campos que serão considerados "obrigatórios". São o **Id** e o **Title**. As propriedades são criadas como privadas a escrita, ficando restritas aos métodos com as regras:

```csharp
public string Title { get; private set; }

...

public void AddTitle(string title)
{
    if (title.IsNullOrWhiteSpace())
    {
        AddError("Faltou preencher o valor 'title'");
        return;
    }

    Title = title;
}

```

Para construção do domínio, utilizei o pattern **Builder** (se você não conhece, tem esse [artigo](https://www.geeksforgeeks.org/builder-design-pattern/) que detalha mais :) ) criando a classe **NetflixTitleBuilder**.


#### Validando o domínio

Para garantir que as regras criadas no domínio estão corretas, podemos fazer através de testes unitários. Para isso, foi criado o projeto **Domain.Tests**. Nele eu valido especialmente a criação de domínios com diferentes valores para as propriedades, assim confirmando se o resultado é o que eu quero. Abaixo tem um exemplo de um teste unitário que cria um domínio inválido, sem **Title**. Nesse caso, o meu resultado esperado é que o domínio seja inválido.

```csharp
[Fact]
public void WithoutTitle_IsInvalid()
{
    var domain = new NetflixTitleBuilder()
        .WithCast("cast example")
        .WithDescription("description example")
        .WithId(Guid.NewGuid())
        .Build();

    Assert.False(domain.IsValid());
}
```

### Application

A camada de Application contém as classes de consulta no banco de dados Cassandra (Repositories) e as classes de criação de novos registros (Services).

#### Repositories

Essas classes basicamente vão se conectar ao banco Cassandra, passar a CQLQuery e converter o objeto de retorno no Dto que eu especifiquei. A consulta montada até então recebe por parâmetro o parâmetro "countryName" e retorna uma lista de títulos daquele país:

```csharp
public async Task<DirectorsByCountryDto> GetDirectorsByCountry(string countryName)
{
    ...

    try
    {
        var cqlQuery = @"select director as Name from netflix_titles where country=? allow filtering";

        var returnQuery = await _cassandraContext.SelectAsync<string>(cqlQuery, countryName);

        var directors = returnQuery.ToList();

        return new DirectorsByCountryDto
        {
            Country = countryName,
            Directors = directors,
            ErrorMessage = !directors.Any() ? $"Não foi encontrado nenhum diretor para o país '{countryName}'" : null
        };
    }
    catch (Exception ex)
    {
        ...
    }
}
```

#### Services

A classe de service irá receber um Dto, montar um domínio a partir dele e inserir no banco de dados Cassandra (se domínio for válido):

```csharp
public async Task<NetflixTitleDto> Create(NetflixTitleDto dto)
{
    try
    {
        dto.Id = Guid.NewGuid();

        var netFlixTitleDomain = new NetflixTitleBuilder()
            .WithTitle(dto.Title)
            .WithDirector(dto.Director)
            .WithCountry(dto.Country)
            .WithDurationMin(dto.DurationMin)
            .WithCast(dto.Cast)
            .WithId(dto.Id)
            .Build();

        if (!netFlixTitleDomain.IsValid())
        {
            ...
        }

        await _cassandraContext.InsertAsync(netFlixTitleDomain);
    }
    catch (Exception ex)
    {
        dto.ErrorMessage = $"Exceção não tratada: {ex.Message}";
    }

    return dto;
}

```

#### Validando o Application

Para garantir as regras aplicadas, criei o projeto **Application.Tests** que irá testar as classes de leitura e escrita no banco de dados. Aqui serão testes unitários, portante utilizarei objetos Mock para "simular" retornos do banco de dados. Um exemplo do uso do Mock é o código abaixo, onde irei Mockar a classe **CassandraContext**, na qual, toda chamada do método **InsertAsync** que recebe qualquer objeto **NetflixTitle** irá retornar como concluida com sucesso:

```csharp
private ICreateNetflixTitleServices MockServiceForCreate()
{
    var mockCassandraContext = new Mock<ICassandraContext>();

    mockCassandraContext
        .Setup(m => m.InsertAsync<NetflixTitle>(It.IsAny<NetflixTitle>()))
        .Returns(() => Task.CompletedTask);

    return new CreateNetflixTitleServices(mockCassandraContext.Object);
}

```

Aí para utilizar este Mock em um teste unitário fica simples:

```csharp
[Fact]
public async Task Create_DomainIsValid()
{
    var dto = new NetflixTitleDto
    {
        Director = "director",
        Country = "country",
        DurationMin = 1,
        Cast = "cast",
        Title = "title"
    };

    var mockService = MockServiceForCreate();

    var result = await mockService.Create(dto);

    Assert.True(result.IsValid());
}
```



### CrossCutting

Aqui são as classes que podem ser utilizadas em qualquer camada (Cross). Objetos Dtos, Extensions e a conexão com o banco de dados Cassandra. Não há nada "diferente" por aqui.

### Api

Aqui é o projeto "inicial" da aplicação. Nele eu exponho as rotas de Get e Post, e também inicializo a aplicação. Há algumas classes que geram algumas configurações:

- IocStartup: Aqui eu configuro a injeção de dependência das classes do projeto.
- MapperCassandraContext: Mapeamento do objeto de domínio nas chaves do banco Cassandra.
- NetflixTitlesController: Controller principal com os métodos de Post e Get.

Para a API ficar mais intuitiva de consultar, foi utilizado o Swagger, que gera uma página HTML com todas as rotas de uma maneira bem elegante:

![Imagem](/assets/images/msnetflix_swagger.png)

Os itens acima citados são chamados na classe Startup.

Aaaaa e não esquecendo, é claro, que também tem um Dockerfile gerado para subir a aplicação. Esse arquivo é gerado pelo próprio Visual Studio ao criar o projeto da Api e já deixa a mesma preparada para subir em contâiner.

O código-fonte desse projeto pode ser localizado [aqui](https://github.com/rafaeldalsenter/ms-netflix-titles)

Bom, aqui nesta segunda etapa detalhei como desenvolvi o projeto. No próximo artigo desta "série" irei configurar o ambiente AWS para subir esta aplicação :)

Até +
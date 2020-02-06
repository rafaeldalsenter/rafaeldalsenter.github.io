---
layout: post
title:  "Function Lambda para Zipar buckets do Amazon S3"
date:   2020-02-05
categories: Python Aws Lambda
---
Neste artigo, vou demonstrar uma function que desenvolvi para zipar Buckets do [Amazon S3](https://aws.amazon.com/pt/s3/). Essa aplicação foi escrita em Python e preparada para executar via [Amazon Lambda](https://aws.amazon.com/pt/lambda/). *Obs.: Esta é a minha primeira aplicação em Python :p

O código-fonte completo está disponibilizado [aqui](https://github.com/rafaeldalsenter/lambda-zip-bucket-s3).

### Entendendo a necessidade

A idéia de escrever essa aplicação veio de uma necessidade bem simples. Tenho um bucket cheio de arquivos no S3, que são alimentados temporalmente e o cliente necessita fazer o download com uma certa frequência. Não encontrei nenhum serviço da própria Amazon que disponibiliza "um bucket inteiro" para fazer download em formato Zip.

Partindo desse problema, pensei em criar uma aplicação que fizesse esse trabalho de, a partir de um bucket, zipar todos os arquivos e gerar um Zip do mesmo pra mim. Como esta é uma função, de certa forma, simples resolvi criá-la para executar via Amazon Lambda, pois esse serviço permite que eu execute códigos simples sem me preocupar com processamento, scaling, etc.

*Ps: Se você não conhece o S3, ele é basicamente um serviço de armazenamento de arquivos em larga escala da Amazon - e Bucket seria como "uma pasta" onde é possível armazenar objetos, que seriam arquivos.

### #Partiu programar

Basicamente, o fluxo da aplicação será o seguinte:
- Criar um arquivo zip no S3.
- Pegar os arquivos a serem zipados por páginas (a quantidade é passada por parâmetro).
- Adicionará esses arquivos no zip.
- A finalizar todos as páginas, irá retornar com o link do zip do S3.

**Dica:** O [AWS Toolkit](https://aws.amazon.com/pt/visualstudiocode/) para Vs Code tem modelos de "hello worlds" prontos em diversas linguagens, assim você já inicia com todos os arquivos prontos, só se preocupando em codar mesmo.

A função principal que o Lambda irá chamar, recebe dois parâmetros por padrão _event_ e _context_. O _event_ é basicamente um array com todos os parâmetros de entrada. Nesta function terei os seguintes parâmetros:

- _bucket_name_source_: Bucket de origem, que contém os arquivos que serão zipados.
- _bucket_name_dest_: Bucket de destino, onde será criado o arquivo zip.
- _archive_name_dest_: Nome do arquivo zip de destino.
- _archive_public_access_: Se o arquivo zip criado será público.
- _amount_of_files_to_upload_: Como o upload será feito por "páginas de arquivos", aqui especificar a quantidade de arquivos por página. Se colocar 0, tentará fazer o upload de todos arquivos de uma vez.

```python
def lambda_handler(event, context):
    try:
        print('Started the Lambda function')

        bucketNameSource = event['bucket_name_source']
        bucketNameDest = event['bucket_name_dest']
        archiveNameDest = event['archive_name_dest']
        archivePublicAccess = event['archive_public_access']
        amountOfFilesToUpload = event['amount_of_files_to_upload']
```

Para utilizar as funções do toolkit da AWS (boto3) para download e upload do S3, criei as mesmas em um arquivo separado [aws_extensions.py](https://github.com/rafaeldalsenter/lambda-zip-bucket-s3/blob/master/src/aws_extensions.py). Nesse arquivo terei funções para retornar a lista de objetos a partir de um bucket, contar esses objetos (sim, tive que implementar isso :p), obter o binário de um objeto, e também de upload.

Retornar a função principal, após o trecho de código acima demonstrado, fiz algumas validações dos valores de entrada e também já obtive a lista de arquivos do bucket selecionado, quantidade de arquivos deste bucket:

```python
    print('Source Bucket:', bucketNameSource)
    print('Target archive:', bucketNameDest + '/' + archiveNameDest)

    objects = aws_ext.all_objects_from_bucket(bucketNameSource)
    count_objects = aws_ext.count_objects(objects)
    amount_objects = amountOfFilesToUpload

    if(amount_objects == 0):
        amount_objects =  count_objects

    print("Source bucket files count:", count_objects)
```

Após isso, vem o trecho principal, antes de tudo irei deletar o Zip (se o mesmo já existir) e também inicializar o objeto _binary_ que irá ser alimentado e utilizado para fazer o Upload "por partes".
- Primeiramente, entrarei no laço de repetição, onde será rodado todas as páginas (se não for optado na entrada por usar paginação, rodará esse laço somente uma vez). O objeto s3.Bucket.page_size utilizado permite esse recurso.
- O item _zip_file_ retornará o arquivo Zip do S3, permitindo que seja feito a atualização do mesmo.
- Utilizando a função _zipfile.ZipFile_ irei abrir o stream com o objeto _binary_, e farei um um laço de repetição em todos os arquivos "daquela página" escrevendo no Stream.
- Ao finalizar a página atualizarei o _zip_file_ com todo o conteúdo do objeto _binary_.

```python
    aws_ext.delete_object_if_exists(bucketNameDest, archiveNameDest)

    binary = BytesIO()

    for page in objects.page_size(amount_objects).pages():

        zip_file = aws_ext.get_object_summary(bucketNameDest, archiveNameDest)

        print("Upload +{} files to {}...".format(amount_objects, archiveNameDest))

        with zipfile.ZipFile(binary, mode="a",compression=zipfile.ZIP_DEFLATED) as zf:
            for x in page:
                zf.writestr(x.key, x.get()['Body'].read())
        
        if(archivePublicAccess):
            zip_file.put(Body=binary.getvalue(), ACL='public-read')
        else:
            zip_file.put(Body=binary.getvalue()) 
```

Por fim, após executar tudo isso teremos o arquivo ZIP com todos os itens do bucket.

```python
link = aws_ext.link_from_object(bucketNameDest, archiveNameDest)

return function_return(True, "Bucket successfully compressed!", link)
```

O objeto de retorno é um Json, e mesmo com exception, sempre retornará com a mensagem da Exception transcrita. Se o retorno for sucesso irá devolver o link do Zip.

### Subindo o código para ser executado no Amazon Lambda

Bom, aqui já temos a função rodando em ambiente local (se você utilizou o plugin do VS Code que indiquei anteriormente, é possível rodar localmente e executar a função :D). Agora, vamos ver como publicar isso!

Quando testei na minha conta AWS, publiquei o mesmo manualmente, ou seja, baixei todos os pacotes necessários, zipei os fontes e as bibliotecas e fiz o Upload do mesmo. Porém, depois descobri que já existe uma [Action](https://github.com/features/actions) do Github para isso, que é muito simples de utilizar.

Basta você criar no seu repositório, o arquivo **.github/workflows/settings.yml**

```yml
name: Lambda deploy

on:
  push:
    branches:
    - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Deploy
      uses: mariamrf/py-lambda-action@v0.0.2  

```
A Action desse arquivo yml irá em todo commit no branch master, fazer checkout do projeto e "fazer deploy" dessa aplicação diretamente no Amazon Lambda. O código-fonte dessa Action está disponível [nesse repositório](https://github.com/mariamrf/py-lambda-action), onde também é explicado algumas configurações necessárias para o funcionamento. O Github Action também tem uma interface bem legal de visualizar o deploy:

![Imagem](/assets/images/lambda_action_github.png)

### Executando a function Amazon Lambda

Na interface de execução de function Lambda, farei basicamente um teste. Para isso iremos no Combobox ao lado do botão "Test" e selecionarei "Configure tests events". Aqui colocarei os parâmetros de entrada que quero testar:

![Imagem](/assets/images/lambda_action_github_2.png)

Após salvar e clicar no botão "Test", ele executará a function passando os parâmetros:

![Imagem](/assets/images/lambda_action_github_3.png)

Então, aqui resumi bem basicamente a criação de uma function Lambda. A mesma ainda não foi testada com grandes quantidades de dados, quando o fizer posto aqui os resultados :)

O código-fonte, contendo também a Action, se encontra [nesse repositório](https://github.com/rafaeldalsenter/lambda-zip-bucket-s3)

Até +
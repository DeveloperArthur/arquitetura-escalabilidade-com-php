# Projeto do curso de _Arquitetura e Escalabilidade em PHP_

Nesse curso temos uma aplicação legada em PHP, com bastante problemas de performance, disponibilidade etc:

<img src="./assets/legacy.webp">

E vamos aprender como melhorar arquitetura dessa aplicação para que ela tenha escalabilidade

Como? Reescrevendo em Golang? Quebrando em microserviços? NÃO!

Utilizando processamento assíncrono, cache, escalabilidade horizontal

## Problemas na arquitetura e decisões de resolução

- A request para cadastrar uma avaliação estava respondendo com muita lentidão, porque além de salvar o registro no banco
de dados, a API também estava se comunicando com um servidor externo para envio de email de maneira síncrona e bloqueante,
mas como a pessoa que está adicionando a avaliação não precisa esperar que o especialista receba o e-mail, tornamos o
envio do email assíncrono:
    
    Para resolvermos esse problema, fizemos a classe `ReviewCreated` implementar a interface `ShouldQueue`, essa 
interface diz pro sistema do Laravel que o email não precisa ser enviado na hora, pode ser enviado depois, vai ser 
armazenado em uma fila e depois processamos essa fila, além disso foi necessário executar 
`docker compose exec app php artisan queue:work` no terminal, para startar o Queue worker, processo que vai ler as 
mensagens da fila de mensagens
  

Solução final após todas aplicações de melhorias:

<img src="./assets/after.webp">

## Setup inicial

1. Após realizar o clone do projeto, instale as dependências do mesmo com:
```shell
composer install
```

2. Caso você não possua o `composer` instalado localmente:
```shell

docker run --rm -itv $(pwd):/app -w /app -u $(id -u):$(id -g) composer:2.5.8 install
```

3. Com as dependências instaladas, crie o arquivo de configuração `.env`:
```shell
cp .env.example .env
```

4. Inicie o ambiente _Docker_ executando:
```shell
docker compose up -d
```

5. Dê permissões ao usuário correto para escrever logs na aplicação
```shell
docker compose exec app chown -R www-data:www-data /app/storage
```

6. Garanta que o contêiner de banco de dados está de pé. Os logs devem exibir a mensagem _ready for connections_ nas últimas linhas
```shell
docker compose logs database
``` 
Aguarde até que o comando acima tenha como uma das últimas linhas a mensagem _ready for connections_.

7. Para criar o banco de dados, execute:
```shell
docker compose exec app php artisan migrate --seed
```

Muitos dados serão criados (1000 especialistas com 1000 avaliações cada), então essa última etapa será demorada. Enquanto ela executa, a API já estará acessível através do endereço http://localhost:8123/api. Além disso, o endereço http://localhost:8025 provê acesso ao serviço de e-mail _Mailpit_.

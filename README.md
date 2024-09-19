# Projeto do curso de _Arquitetura e Escalabilidade em PHP_

Nesse curso tínhamos uma aplicação monolita legada em PHP, cheia de problemas de performance, disponibilidade, não
escala quando o volume de requests crescem, possuí chamadas síncronas em queries que demandam muito processamento, etc:

<img src="./assets/legacy.webp">

E aplicamos diversas melhorias para deixar arquitetura dessa aplicação mais escalável

Como? Reescrevendo em Golang? Quebrando em microserviços? Migrando o banco SQL para MongoDB? NÃO!

Configurando load balancer e replicando instâncias do monolito, utilizando processamento assíncrono com mensageria, 
cache distribuído para performance em consultas, armazenamento externo arquivos estáticos e mais!

## Problemas na arquitetura e decisões de resolução

- A request para cadastrar uma avaliação estava respondendo com muita lentidão (181ms, os outros 
endpoints retornam com 80ms em média), porque além de salvar o registro no banco de dados, a API também estava se 
comunicando com um servidor externo para envio de email de maneira síncrona e bloqueante,
mas como a pessoa que está adicionando a avaliação não precisa esperar que o especialista receba o e-mail, tornamos o
envio do email assíncrono:
    
    Para resolvermos esse problema, [fizemos a classe `app/Mail/ReviewCreated.php` implementar a interface `ShouldQueue`](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/a3d594d6939f47592857ad2c0bb72968d76b681f), essa 
interface diz pro sistema do Laravel que o email não precisa ser enviado na hora, pode ser enviado depois, vai ser 
armazenado em uma fila e depois processamos essa fila, além disso foi necessário executar 
`docker compose exec app php artisan queue:work` no terminal, para startar o Queue worker, processo que vai ler as 
mensagens da fila de mensagens, com isso a resposta para o cadastro caiu para 136ms
    
    O Queue Worker processa uma mensagem de cada vez, acaso a gente tenha muitas mensagens na fila, e o envio de email 
demorar, precisaríamos apenas executar `docker compose exec app php artisan queue:work` novamente, e estaríamos criando 
outro worker, escalando horizontalmente nossos processos responsáveis por ler as mensagens
  
- Fizemos um teste de carga na aplicação simulando 10 conexões simultâneas, utilizando 4 threads para enviar 
requisições em paralelo durante 10 segundos, nossa API conseguiu processar 709 requests, com média de 70 requests por 
segundo, precisávamos ser capazes de lidar com um volume maior de requests nesse período de tempo
    
    Para resolver esse problema escalamos a API horizontalmente, [colocando um load balancer na frente, e 
replicando a instância do monolito](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/280ee6544f8c360d247143c983e3ec9f7ca2c765), após essa alteração, rodando o mesmo teste de carga e nossa arquitetura
conseguiu processar 1125 requests, com média de 112 requests por segundo, conseguimos alcançar um número maior de requests

- Escalando aplicação horizontalmente, nos deparamos com um problema na autenticação do usuário, esse sistema 
utilizava autenticação por meio de sessões, e sessões ficam salvas no servidor, como temos N instâncias de servidor, 
as sessões estavam produzindo inconsistencia no login do usuário, [fizemos uma alteração para autenticar o usuário por
meio de token, invés de sessão](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/2dbbeed413c0fc999896ce7aaf8210cc0686a820), para garantir a consistência do login

  Há ainda uma outra alternativa: armazenar as sessões em um servidor externo. Isso resolve o problema de sessões com o 
balanceamento de carga também, mas traz outro componente que devemos gerenciar em nossa infraestrutura.

- Temos um endpoint nessa aplicação que lista os 10 profissionais mais bem avaliados, esse endpoint é muito lento,
ele responde em 5 segundos em média, pois se trata de uma query com join e há muitos dados na aplicação, portanto
essa query demanda muito processamento

    Para resolver esse problema [implementamos um cache distribuído na aplicação](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/5aee43ee2b0c01ecf1bdabf34d14fe9d3f04b20b), para diminuir o tempo de respostas 
á query custosa, então cacheamos essa lista de avaliados, e o tempo de resposta caiu para 90ms

- Alteramos o endpoint que lista os 10 profissionais mais bem avaliados, invés de cachear a lista que vem do banco de 
dados, [estamos gerando esse relatório como CSV de forma assíncrona e enviando o link por email após processamento](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/abd0361f3fbef78130c56e85e1d8dafa344504c1), 
ou seja, estamos utilizando mensageria para processar em plano de fundo um relatório demorado, e para isso foi necessário 
criar uma nova classe de Job com o comando `docker compose exec -u $(id -u) app php artisan make:job GenerateReportJob`,
foi necessário criar uma nova classe para representar o email: `docker compose exec -u $(id -u) app php artisan make:mail ReportGenerated`,
e foi necessário rodar este comando: `docker compose exec app php artisan storage:link` para tornar o arquivo CSV acessível
dentro de uma pasta pública no servidor, por fim precisa rodar o comando `docker compose exec app php artisan queue:work`
para startar o Job

- Ao tentar disponibilizar os relatórios gerados via email, nos deparamos com um problema, pois os arquivos CSV's estão
sendo salvos no mesmo servidor onde a aplicação é executada, e como temos N servidores, pode ser que o usuário clique 
para baixar e o load balancer redirecione para o servidor que não tem o relatório

    Por isso em uma arquitetura 
distribuída o ideal é salvarmos os arquivos em um componente externo, como Amazon S3 ou Google Cloud Storage, pois esses
serviços são projetados para oferecer maior escalabilidade dos arquivos e fornecem redundância e replicação de dados 
para garantir alta disponibilidade.

  **Para fins didáticos** nós [mapeamos a pasta pública da máquina local no load
  balancer](https://github.com/DeveloperArthur/arquitetura-escalabilidade-com-php/commit/28c8e5bb60bd058fca20f0fc2b3bbae92b5c0c60), para que ele consiga servir os arquivos, simbolizando nosso componente de armazenamento externo

Solução final após todas aplicações de melhorias:

<img src="./assets/after.webp">

## Melhorando disponibilidade da aplicação
- **I/O não bloqueante**

    A aplicação estava utilizando PHP com FastCGI, e utilizando FastCGI cada requisição é tratada de forma independente, 
ou seja, cada vez que uma requisição chega, o PHP carrega o framework Laravel, verifica a rota, executa o controller,
retorna a resposta e mata o processo, isso se repete cada vez que uma requisição chega no servidor 
(isso não ocorre em outras linguagens como Java, C#, Node.js etc), então utilizamos utilizamos Swoole e Octane 
para melhorar a performance da nossa aplicação, a ideia do Swoole é subir aplicação apenas uma vez, e manter a aplicação 
PHP e seus recursos em memória, sem ficar caindo e levantando aplicação á cada request, antes de utilizar Swoole 
e configurar Octane na aplicação, a app respondia algo próximo de 1.100 requisições no intervalo de 10 segundos, 
depois da configuração: 5.433 requisições nesse mesmo período de tempo...
    
    Outra coisa que estava acontecendo e poderia ser melhorada com Swoole era com relação a abertura de conexões no banco 
de dados, chegava uma requisição no banco de dados, era aberta uma conexão, após a requisição fechava-se a conexão, 
ficava-se abrindo e fechando conexões a cada requisição (em Java utilizamos pool de
conexões para evitar o custo de abrir e fechar conexões constantemente)

- **Profiling**

    Para realizar alterações na arquitetura, precisamos fazer isso baseado em números e em fatos, por isso é importante 
as técnicas de monitoramento, imagine que você identifique que uma query está lenta, ai já toma decisão e adiciona
um serviço de fila, ou um cache distribuido, e nem sempre é isso, é importante medir o que precisamos otimizar, porque
não sabemos que otimização é necessária, quando encontramos um gargalo na arquitetura, uma das formas de coletar esses
dados é através da prática de Profiling, o professor citou algumas das ferramentas profissionais de Profiling como
New Relic e Blackfire

- **Como garantir a disponibilidade lidando com DDoS**
    
    Precisamos limitar quanto um cliente pode utilizar dos nossos recursos para evitar ataques de DDoS ou consumos 
desenfreados sem necessidade, como por exemplo um cliente gerando dashboard em tempo real chamando um endpoint que 
executa uma query custosa

    Para lidarmos com essa situação, configuramos um Rate Limit de 60 requests por minuto, limitando por id 
(caso usuário esteja logado), se o usuário estiver deslogado o limite será feito por IP, ou seja, cada cliente vai poder
fazer no máximo 60 requests por minuto, se tentar fazer +1 request, receberá um erro 429, e poderá enviar novas requests 
depois de algum tempo
    
    Se mesmo depois de excederem o limite, continuarem enviando requests, podemos estudar a possibilidade de adicionar
regras de bloqueio, se as requisições estiverem sendo feitas por IPs desconhecidos ou usuários mal intencionados, 
bloqueá-los a nível de rede poupará muito os recursos da aplicação

- **Como documentar decisões arquiteturais com C4 model e como escrever registros de decisões arquiteturais seguindo
o conceito de ADR**

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

8. Para se autenticar na aplicação:
```shell
curl -X POST http://localhost:8123/api/login \
-H "Content-Type: application/json" \
-d '{"email": "email@example.com", "password": "12345678"}'
```

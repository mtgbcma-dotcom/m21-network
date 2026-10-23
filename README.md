# Bookstore - Docker Network com Django e PostgreSQL

Atividade do Módulo 21 de Continuação Back-End Python da EBAC.

Criei uma rede personalizada no Docker Compose para conectar a API Django REST Framework e o PostgreSQL. Os serviços `web` e `db` participam explicitamente da rede `bookstore_network`, com driver `bridge`. A API mantém categorias, produtos, pedidos, paginação de 5 registros e autenticação por token somente nos pedidos.


## Continuidade do projeto e do ambiente

Dei continuidade à entrega do [módulo 20 — Docker Compose](https://github.com/mtgbcma-dotcom/m20-compose/pull/1). Mantive os modelos, os serializers, a migração inicial e os testes de serializers do Bookstore, além das funcionalidades acrescentadas nas etapas anteriores.

Nos módulos 15 e 16, utilizei Poetry com `pyproject.toml` e `poetry.lock`. A partir do módulo 17, passei a declarar as dependências em `requirements.txt` e a instalar com pip. Nas etapas com Docker, o próprio Dockerfile executa essa instalação durante a construção da imagem. Documentei essa mudança de ferramenta para deixar explícita a continuidade do mesmo projeto.

Para reproduzir esta entrega, utilizei os comandos e os arquivos de dependências desta versão, descritos abaixo. A instalação com pip segue as faixas registradas em `requirements.txt`; o lock do Poetry das etapas anteriores continua disponível nos respectivos repositórios.

## Implementação

- `docker-compose.yml` com os serviços `web` e `db`.
- Rede `bookstore_network` declarada em `networks`, com `driver: bridge` e associação explícita nos dois serviços.
- Serviço `web` construído pelo Dockerfile com Python 3.13 e dependências do projeto.
- Serviço `db` com a imagem oficial `postgres:17-alpine`.
- Conexão Django/PostgreSQL pelo driver `psycopg` e pelo nome de serviço `db` na rede do Compose.
- Configurações compartilhadas por variáveis de ambiente, com `.env.example` como modelo.
- `healthcheck` no banco e `depends_on` com `condition: service_healthy` para iniciar o Django somente quando o PostgreSQL estiver pronto.
- Migrações aplicadas automaticamente antes de iniciar o servidor.
- Volume nomeado `postgres_data` para persistir os dados do PostgreSQL.
- API disponível no computador pela porta 8002; o banco é acessado pela rede interna do Compose.
- Código e dependências na imagem; Git, ambientes virtuais e arquivos `.env` excluídos pelo `.dockerignore`.

O ambiente utiliza o servidor de desenvolvimento do Django para a execução local da atividade.

## Executar no Windows

Mantenha o Docker Desktop aberto e use o PowerShell na pasta que contém `docker-compose.yml`.

### 1. Preparar as configurações

Na primeira execução, copie o modelo:

```powershell
Copy-Item .env.example .env
```

O modelo contém valores para estudo local. Abra `.env` no editor e ajuste `POSTGRES_PASSWORD` antes de iniciar o banco. Mantenha a mesma senha nas próximas execuções: o PostgreSQL usa essas variáveis para criar o banco e o usuário apenas quando o volume está vazio.

O arquivo `.env` é ignorado pelo Git e pelo build da imagem. O arquivo `.env.example` acompanha o projeto para documentar as variáveis. Não repita a cópia sobre um `.env` já configurado.

### 2. Construir e iniciar os serviços

```powershell
docker compose config --quiet
docker compose up -d --build --wait
```

O primeiro comando verifica o arquivo Compose e as variáveis; sem erros, não mostra saída. O segundo baixa as imagens necessárias, constrói a API, inicia o PostgreSQL, aplica as migrações e aguarda os serviços ficarem saudáveis.

Confira:

```powershell
docker compose ps
```

Os serviços `web` e `db` devem aparecer em execução e com estado `healthy`.

O Compose cria automaticamente a rede `m21-network_bookstore_network`. Não é necessário executar `docker network create` separadamente.

Para consultar os registros de inicialização:

```powershell
docker compose logs --tail 60 web db
```

### 3. Criar o administrador

```powershell
docker compose exec web python manage.py createsuperuser
```

Preencha seu usuário, e-mail e senha. A senha não aparece durante a digitação. Este banco é independente dos módulos anteriores, portanto é necessário criar um administrador nele.

### 4. Criar os produtos de exemplo

```powershell
docker compose exec web python manage.py seed_demo
```

São criadas 12 categorias e 12 produtos para conferir a paginação. Executar novamente não duplica os mesmos registros. O comando não cria pedidos nem usuários.

### 5. Abrir a aplicação

- Raiz da API: http://127.0.0.1:8002/api/
- Administração: http://127.0.0.1:8002/admin/
- Categorias: http://127.0.0.1:8002/api/categories/
- Produtos: http://127.0.0.1:8002/api/products/
- Pedidos: http://127.0.0.1:8002/api/orders/

Em categorias e produtos, clique em **Log in** e entre com o administrador criado neste módulo. A listagem apresenta `count`, `next`, `previous` e `results`.

Pedidos exigem o cabeçalho de token. Acessar `/api/orders/` diretamente no navegador sem esse cabeçalho retorna HTTP 401, mesmo com sessão de login.

O valor `APP_PORT=8002` permite executar o MD21 junto com o MD19 na porta 8000 e o MD20 na porta 8001. Se mudar `APP_PORT` no `.env`, atualize também a porta dos endereços utilizados nos testes e execute novamente `docker compose up -d --wait`.

## Conferir a rede e a comunicação

O trecho que define e conecta a rede está no `docker-compose.yml`:

```yaml
services:
  web:
    # Demais configuracoes do servico...
    networks:
      - bookstore_network
  db:
    # Demais configuracoes do servico...
    networks:
      - bookstore_network

networks:
  bookstore_network:
    driver: bridge
```

Confira o driver e a quantidade de contêineres conectados:

```powershell
docker network inspect m21-network_bookstore_network --format '{{.Name}} | driver={{.Driver}} | containers={{len .Containers}}'
```

Resultado esperado: `m21-network_bookstore_network | driver=bridge | containers=2`.

Para visualizar os dois contêineres e seus endereços na rede:

```powershell
docker network inspect m21-network_bookstore_network --format '{{range .Containers}}{{println .Name .IPv4Address}}{{end}}'
```

Dentro do serviço da API, confira a resolução do nome `db`:

```powershell
docker compose exec web python -c "import socket; print('db:', socket.gethostbyname('db'))"
```

O endereço retornado deve corresponder ao IP do serviço `db` na inspeção da rede. Ele é atribuído pelo Docker; a aplicação se conecta pelo nome estável `db`, sem fixar esse endereço no código.

Teste a conexão TCP do serviço `web` ao PostgreSQL:

```powershell
docker compose exec web python -c "import socket; s=socket.create_connection(('db',5432),timeout=5); print('Conexao com db:5432 OK'); s.close()"
```

A comunicação entre os serviços utiliza `db:5432`. O navegador utiliza `127.0.0.1:8002` para acessar a API pela porta publicada. A porta do PostgreSQL não está publicada no computador.

## Verificar o PostgreSQL e os testes

Confira a conexão com o servidor de banco de dados:

```powershell
docker compose exec web python manage.py shell -c "from django.db import connection; cursor = connection.cursor(); cursor.execute('SELECT version()'); print(cursor.fetchone()[0])"
```

A saída deve informar PostgreSQL 17. A aplicação está configurada para usar PostgreSQL, incluindo o banco temporário criado pelo Django durante os testes.

```powershell
docker compose exec web python manage.py check
docker compose exec web python manage.py makemigrations --check --dry-run
docker compose exec web python manage.py test --verbosity 2
```

A suíte contém 86 testes de serializers, operações da API, permissões, paginação e autenticação por token. O esperado é terminar com `Ran 86 tests` e `OK`. O banco temporário dos testes é removido ao concluir, preservando os produtos e usuários cadastrados para a demonstração.

## Testar pedidos com token no PowerShell

```powershell
$credencialMD21 = Get-Credential -Message "Informe o administrador criado no MD21"
$corpoMD21 = @{
    username = $credencialMD21.UserName
    password = $credencialMD21.GetNetworkCredential().Password
} | ConvertTo-Json

$autenticacaoMD21 = Invoke-RestMethod -Uri "http://127.0.0.1:8002/api/token/" -Method Post -ContentType "application/json" -Body $corpoMD21 -ErrorAction Stop
$cabecalhosMD21 = @{ Authorization = "Token $($autenticacaoMD21.token)" }
Remove-Variable credencialMD21, corpoMD21

(Invoke-WebRequest -Uri "http://127.0.0.1:8002/api/orders/" -Headers $cabecalhosMD21 -UseBasicParsing).StatusCode

Invoke-RestMethod -Uri "http://127.0.0.1:8002/api/orders/" -Headers $cabecalhosMD21 | ConvertTo-Json -Depth 8
```

O esperado é HTTP 200 e a resposta paginada. Se não houver pedidos, `count` será 0 e `results` será uma lista vazia. Não inclua o token nem as credenciais em prints ou commits.

## Parar e executar novamente

Para parar os dois serviços mantendo os contêineres e os dados:

```powershell
docker compose stop
docker compose up -d --wait
```

Para remover os contêineres e a rede e, depois, recriá-los usando o mesmo banco:

```powershell
docker compose down
docker compose up -d --wait
```

`docker compose down` remove a rede do projeto e preserva o volume nomeado com os dados do PostgreSQL. Na próxima execução, `docker compose up` recria a rede e conecta os dois serviços novamente. Não acrescente `-v` quando quiser manter os cadastros: essa opção exclui os volumes.

## Atualizar a aplicação após alterar arquivos

```powershell
docker compose up -d --build --wait
```

O Compose reconstrói a imagem e atualiza o serviço da API. O banco permanece no volume nomeado.

## Makefile opcional

Incluí atalhos `make up`, `make down`, `make logs`, `make check`, `make test`, `make migrate`, `make superuser`, `make seed` e `make network` para ambientes que tenham GNU Make instalado. No PowerShell, use diretamente os comandos Docker Compose documentados acima.

## Tecnologias

Python 3.13, Django 5.2, Django REST Framework, PostgreSQL 17, psycopg, Docker e Docker Compose.

## Referências

- [Redes no Docker Compose](https://docs.docker.com/compose/how-tos/networking/)
- [Configuração de networks no Compose](https://docs.docker.com/reference/compose-file/networks/)
- [Ordem de inicialização dos serviços - Docker](https://docs.docker.com/compose/how-tos/startup-order/)
- [Imagem oficial PostgreSQL](https://hub.docker.com/_/postgres)
- [PostgreSQL no Django](https://docs.djangoproject.com/en/5.2/ref/databases/#postgresql-notes)

## Autor

Bruno Costa Macedo de Almeida

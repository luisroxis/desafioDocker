## Questão 06

Agora vamos aumentar mais a complexidade das coisas, chegou a hora de executar
uma aplicação baseada em arquitetura de microsserviços.
Essa aplicação é formada por 3 repositórios:

- Microsserviço de Filmes (https://github.com/KubeDev/rotten-potatoes-ms)
- Microsserviço de Avaliação (https://github.com/KubeDev/movie)
- Montar o ambiente com Docker compose baseado em arquivos de enviroment (https://github.com/KubeDev/review)

## Resolução

## Rotten Potatoes com Microserviços
Aplicação escrita em *Python* com lista de filmes e opção de cadastrar resenhas.

## Estrutura do projeto
Esse projeto é baseado em uma aquitetura de Microsserviços e depende de outros 2 projetos pra funcionar.

Que são:
[Microserviço Reviews](https://github.com/luisroxis/review/) - Onde são armazenados os dados de resenha dos filmes apresentados pela aplicação que utiliza *PostGreSQL* como banco de dados.

[Microserviço Movies](https://github.com/luisroxis/movie) - Onde estão cadastrados os filmes exibidos no site da aplicação que utiliza *MongoDB* como banco de dados.

## Configuração

Foram criados 3 Dockerfile para criação da imagens docker.

Dockerfile - Rotten-Potatoes-MS - local: ./rotten-potatoes-ms/src/

```bash
FROM python:3.8-slim-buster
WORKDIR /app
COPY requirements.txt /app
RUN python -m pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["gunicorn", "--workers=3", "--bind", "0.0.0.0:5000", "app:app"]
```

Dockerfile - Movies - local: ./movie/src/

```bash
FROM node:16-alpine
EXPOSE 8181
WORKDIR /app
COPY package*.json /app/
RUN npm install
COPY . .
ENTRYPOINT [ "node", "server.js" ]
```

Dockerfile - Review - local: ./review/src/Review.Web

```bash
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
COPY Review.Web.csproj /src/.
RUN dotnet restore Review.Web.csproj
COPY . .
RUN dotnet build Review.Web.csproj -c Release -o /app/build

FROM build AS publish
RUN dotnet publish Review.Web.csproj -c Release -o /app/publish

FROM base AS Final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT [ "dotnet", "Review.Web.dll" ]
```
docker-compose.yaml

```bash
version: "3.8"

volumes:
  postgres_data:
  mongo_vol:

networks:
  net_app:
    driver: bridge
  net_db:
    driver: bridge

services:
  postgres:
    image: postgres:13.4-alpine
    environment:
      POSTGRES_USER: ${PG_USER}
      POSTGRES_PASSWORD: ${PG_PASSWD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - net_db
    ports:
      - "5432:5432"

  mongodb:

    image: mongo:5.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PWD}
    volumes:
      - mongo_vol:/data/db
    networks:
        - net_db
    ports:
      - "27017:27017"

  app-rotten-potatoes:
    image: luisroxis/rotten_potatoes_ms:${TAG}
    hostname: app-rotten-potatoes
    build:
      context: ./src
      dockerfile: Dockerfile
    environment:
      - MOVIE_SERVICE_URI=${MOVIE_SERVICE_URI}
      - REVIEW_SERVICE_URI=${REVIEW_SERVICE_URI}
    depends_on:
      - app-review
      - app-movie
    networks:
      - net_app
    ports:
      - "8080:5000"
  
  app-review:
    image: luisroxis/review:${TAG}
    hostname: app-review
    build:
      context: ../review/src/Review.Web/
      dockerfile: Dockerfile
    environment:
      - ConnectionStrings__MyConnection=${CONNECTION_STRING_PG}
    networks:
      - net_db
      - net_app
    restart: always
    depends_on:
      - postgres

  app-movie:
    image: luisroxis/movie:${TAG}
    hostname: app-movie
    build:
      context: ../movie/src/
      dockerfile: Dockerfile
    environment:
      - MONGODB_URI=${MONGODB_URI}
    networks:
      - net_db
      - net_app
    restart: always
    depends_on:
      - mongodb

```

.env

```bash
TAG=v1
MOVIE_SERVICE_URI=http://app-movie:8181
REVIEW_SERVICE_URI=http://app-review:80
CONNECTION_STRING_PG="Host=postgres;Database=pguser;Username=pguser;Password=Pg@123"
MONGODB_URI=mongodb://mongouser:mongopwd@mongodb:27017/admin
MONGO_USER=mongouser
MONGO_PWD=mongopwd
PG_USER=pguser
PG_PASSWD=Pg@123
```

## Fazendo o Deploy da Aplicação.

Faça o clone dos 3 repositórios das aplicações com o comando abaixo:

```bash
git clone https://github.com/luisroxis/rotten-potatoes-ms.git \
	&& git clone https://github.com/luisroxis/movie.git \
	&& git clone https://github.com/luisroxis/review.git
```

Acessar a pasta /rotten-potatoes-ms e executar a linha de comando abaixo para fazer o build da aplicação e sua execução:

```bash
docker-compose up -d
```
apos o termino do Build e Deploy, para listar o container em execução:

```bash
docker container ls
```
Para acessar a aplicação basta acessar este Link: (http://localhost:8080)

Parando a aplicação em execução:
```bash
docker-compose down
```
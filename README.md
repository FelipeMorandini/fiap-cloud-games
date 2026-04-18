# FIAP Cloud Games (FCG)

Plataforma de venda de jogos digitais e gerenciamento de servidores de jogos online, desenvolvida como Tech Challenge da pós-graduação em **Arquitetura de Sistemas .NET com Azure** da FIAP.

**Fase 1 (MVP):** API REST para cadastro de usuários, catálogo de jogos e biblioteca de jogos adquiridos, com autenticação JWT e autorização por roles.

## Tecnologias Utilizadas

| Tecnologia | Versão | Finalidade |
| --- | --- | --- |
| .NET | 10.0 (LTS) | Runtime e SDK |
| C# | 14 | Linguagem |
| ASP.NET Core | 10.0 | Framework Web API |
| Entity Framework Core | 10.0 | ORM (MongoDB Provider) |
| MongoDB | 7.0 | Banco de dados NoSQL |
| JWT (HMAC-SHA256) | - | Autenticação e autorização |
| BCrypt | Work factor 12 | Hash de senhas |
| FluentValidation | 11.x | Validação de entrada na fronteira da API |
| Serilog | 10.x | Logging estruturado (JSON) |
| Swagger / OpenAPI 3.1 | - | Documentação interativa da API |
| xUnit + FluentAssertions | - | Testes unitários (151 testes) |
| Testcontainers | 4.x | Testes de integração com MongoDB real (26 testes) |

## Arquitetura

O projeto segue **Clean Architecture** em monolito modular, com Domain-Driven Design (DDD):

```text
┌─────────────────────────────────────────────┐
│                    API                      │
│   Controllers, Middleware, Swagger, DI      │
├─────────────────────────────────────────────┤
│               Application                   │
│   Use Cases, DTOs, Validators, Interfaces   │
├─────────────────────────────────────────────┤
│              Infrastructure                 │
│   EF Core DbContext, Repositories, JWT,     │
│   BCrypt, MongoDB Configurations, Seed      │
├─────────────────────────────────────────────┤
│                  Domain                     │
│   Entities, Value Objects, Enums, Exceptions│
│   Repository Interfaces (zero dependências) │
└─────────────────────────────────────────────┘
```

### Fluxo de dependências

```text
Domain          ← zero dependências externas
Application     ← Domain
Infrastructure  ← Domain, Application
API             ← Application, Infrastructure (composition root)
```

### Camadas

- **Domain**: Entidades (`Usuario`, `Jogo`, `BibliotecaJogo`), Value Objects (`Email`, `Preco`, `Senha`), Enums (`GeneroJogo`, `TipoUsuario`), Exceções de domínio e interfaces de repositório.
- **Application**: Serviços de aplicação (`AuthService`, `UsuarioService`, `JogoService`, `BibliotecaService`), DTOs de request/response, Validators com FluentValidation.
- **Infrastructure**: Implementação dos repositórios com EF Core + MongoDB Provider, autenticação JWT (HS256), hashing BCrypt, configurações de entidades (Fluent API), Named Query Filters para soft-delete, seed de dados.
- **API**: Controllers REST (Auth, Usuarios, Jogos, Biblioteca), middleware global de exceções com ProblemDetails (RFC 7807), configuração de DI e Swagger/OpenAPI 3.1.

## Pré-requisitos

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- [Docker](https://docs.docker.com/get-docker/) (para MongoDB local e testes de integração)

## Como Executar

### 1. Clonar o repositório

```bash
git clone https://github.com/FelipeMorandini/fiap-cloud-games.git
cd fiap-cloud-games
```

### 2. Iniciar o MongoDB

```bash
docker run -d --name mongodb -p 27017:27017 mongo:7.0
```

### 3. Executar a API

```bash
dotnet run --project src/FiapCloudGames.API
```

A API estará disponível em `http://localhost:5087`.

### 4. Acessar o Swagger

Abra no navegador: [http://localhost:5087/swagger](http://localhost:5087/swagger)

### Dados iniciais (Seed)

Na primeira execução, a aplicação cria automaticamente:

- **Administrador:** `admin@fcg.com` / `Admin@123456`
- **5 jogos de exemplo** em diferentes gêneros

## Como Executar os Testes

### Todos os testes (177 testes)

```bash
dotnet test
```

### Testes unitários (151 testes)

```bash
dotnet test tests/FiapCloudGames.UnitTests
```

### Testes de integração (26 testes)

Requer Docker em execução (Testcontainers cria um MongoDB temporário automaticamente):

```bash
dotnet test tests/FiapCloudGames.IntegrationTests
```

### Cobertura de testes

| Camada | Testes | Tipo |
| --- | --- | --- |
| Domain (Entities, Value Objects) | 35 | Unitário |
| Application (Services) | 30 | Unitário |
| Application (Validators) | 20 | Unitário |
| API (Controllers) | 26 | Integração |
| **Total** | **177** | - |

## Endpoints da API

### Autenticacao

| Metodo | Rota | Descricao | Auth |
| --- | --- | --- | --- |
| POST | `/api/v1/auth/login` | Autenticar e obter token JWT | Publico |

### Usuarios

| Metodo | Rota | Descricao | Auth |
| --- | --- | --- | --- |
| POST | `/api/v1/usuarios` | Registrar novo usuario | Publico |
| GET | `/api/v1/usuarios` | Listar usuarios (paginado) | Admin |
| GET | `/api/v1/usuarios/{id}` | Obter usuario por ID | Proprio ou Admin |
| PUT | `/api/v1/usuarios/{id}` | Atualizar dados do usuario | Proprio ou Admin |
| DELETE | `/api/v1/usuarios/{id}` | Desativar usuario (soft delete) | Admin |

### Jogos

| Metodo | Rota | Descricao | Auth |
| --- | --- | --- | --- |
| GET | `/api/v1/jogos` | Listar jogos (paginado, filtro por genero) | Autenticado |
| GET | `/api/v1/jogos/{id}` | Obter jogo por ID | Autenticado |
| POST | `/api/v1/jogos` | Cadastrar novo jogo | Admin |
| PUT | `/api/v1/jogos/{id}` | Atualizar jogo | Admin |
| DELETE | `/api/v1/jogos/{id}` | Desativar jogo (soft delete) | Admin |

### Biblioteca

| Metodo | Rota | Descricao | Auth |
| --- | --- | --- | --- |
| GET | `/api/v1/biblioteca` | Listar jogos adquiridos | Autenticado |
| POST | `/api/v1/biblioteca` | Adquirir jogo | Autenticado |

### Como autenticar no Swagger

1. Execute `POST /api/v1/auth/login` com as credenciais do admin (ou registre um usuario)
2. Copie o campo `token` da resposta
3. Clique no botao **Authorize** no topo do Swagger
4. Informe: `Bearer {seu_token}`
5. Todos os endpoints autenticados estarao disponiveis

## Estrutura do Projeto

```text
fiap-cloud-games/
├── src/
│   ├── FiapCloudGames.Domain/           # Entidades, VOs, Enums, Excecoes, Interfaces
│   ├── FiapCloudGames.Application/      # Servicos, DTOs, Validators
│   ├── FiapCloudGames.Infrastructure/   # EF Core, Repositorios, JWT, BCrypt, Seed
│   └── FiapCloudGames.API/             # Controllers, Middleware, Swagger, Program.cs
├── tests/
│   ├── FiapCloudGames.UnitTests/        # 151 testes unitarios
│   └── FiapCloudGames.IntegrationTests/ # 26 testes de integracao
├── global.json
└── README.md
```

## Documentacao DDD

A modelagem de dominio completa com Event Storming esta disponivel em:

**[docs/DDD.md](docs/DDD.md)**

Conteudo:

- Linguagem Ubiqua (glossario)
- Contextos Delimitados (diagrama Mermaid)
- Mapa de Agregados (entidades, VOs, relacoes)
- Event Storming dos 4 fluxos principais (Registro, Login, Jogos, Aquisicao)
- Comandos, Eventos e Politicas por contexto
- Invariantes de Dominio

## Decisoes Tecnicas

| Decisao | Justificativa |
| --- | --- |
| MongoDB ao inves de SQL Server | Flexibilidade de schema para catalogo de jogos com atributos variados |
| EF Core com MongoDB Provider | Abstrai acesso a dados mantendo compatibilidade com EF Core patterns |
| Named Query Filters (EF Core 10) | Soft-delete centralizado na configuracao, removendo filtros manuais |
| BCrypt work factor 12 | Balanco entre seguranca e performance para hash de senhas |
| ProblemDetails (RFC 7807) | Padrao da industria para respostas de erro em APIs REST |
| Testcontainers | Testes de integracao contra MongoDB real, sem mocks de banco |

## Autores

| Nome | RM |
| --- | --- |
| Thiago Goulart de Brito | RM370407 |
| Felipe Pires Morandini | RM370354 |
| Lucas Silva | RM372520 |
| Christopher Correa | RM372035 |
| Flavio Ferreira de Luna | RM373906 |

## Licenca

Este projeto esta licenciado sob a licenca MIT. Consulte o arquivo [LICENSE](LICENSE) para mais detalhes.

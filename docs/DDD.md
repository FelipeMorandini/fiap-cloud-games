# Documentacao DDD — FIAP Cloud Games

Modelagem de dominio usando Domain-Driven Design (DDD) com Event Storming para os fluxos principais da plataforma.

## Linguagem Ubiqua

| Termo | Definicao |
| --- | --- |
| **Usuario** | Pessoa cadastrada na plataforma, com perfil Usuario ou Administrador |
| **Administrador** | Usuario com permissoes elevadas para gerenciar jogos e usuarios |
| **Jogo** | Produto digital disponivel para aquisicao no catalogo |
| **Biblioteca** | Colecao pessoal de jogos adquiridos por um usuario |
| **Aquisicao** | Ato de adicionar um jogo a biblioteca do usuario |
| **Catalogo** | Conjunto de jogos ativos disponiveis na plataforma |
| **Genero** | Classificacao do jogo (Acao, RPG, Estrategia, etc.) |
| **Preco** | Valor monetario do jogo, composto por valor e moeda |
| **Email** | Identificador unico do usuario, validado por formato |
| **Senha** | Credencial do usuario, armazenada como hash BCrypt |
| **Token JWT** | Credencial temporaria emitida apos autenticacao |
| **Soft Delete** | Desativacao logica (campo Ativo = false) sem remocao fisica |

## Contextos Delimitados

```mermaid
graph TB
    subgraph "Contexto: Identidade e Acesso"
        UA[Usuario]
        AUTH[Autenticacao]
        UA --> AUTH
    end

    subgraph "Contexto: Catalogo de Jogos"
        JG[Jogo]
        CAT[Catalogo]
        JG --> CAT
    end

    subgraph "Contexto: Biblioteca do Usuario"
        BIB[BibliotecaJogo]
    end

    AUTH -.->|Token JWT| CAT
    AUTH -.->|Token JWT| BIB
    BIB -.->|Referencia| UA
    BIB -.->|Referencia| JG
```

### Descricao dos Contextos

- **Identidade e Acesso**: Gerencia cadastro de usuarios, autenticacao (login/JWT) e autorizacao por roles (Usuario/Administrador). Responsavel por validar credenciais e emitir tokens.
- **Catalogo de Jogos**: Gerencia o ciclo de vida dos jogos (cadastro, atualizacao, desativacao). Apenas administradores podem modificar o catalogo.
- **Biblioteca do Usuario**: Gerencia a relacao entre usuarios e jogos adquiridos. Garante que um usuario nao adquira o mesmo jogo duas vezes.

## Mapa de Agregados

```mermaid
classDiagram
    class Usuario {
        +String Id
        +String Nome
        +Email Email
        +String SenhaHash
        +TipoUsuario Tipo
        +DateTime DataCriacao
        +Boolean Ativo
        +AtualizarNome(nome)
        +AtualizarEmail(email)
        +AtualizarSenha(senhaHash)
        +Desativar()
        +Ativar()
    }

    class Jogo {
        +String Id
        +String Titulo
        +String Descricao
        +GeneroJogo Genero
        +Preco Preco
        +DateTime DataLancamento
        +DateTime DataCriacao
        +Boolean Ativo
        +AtualizarTitulo(titulo)
        +AtualizarDescricao(descricao)
        +AtualizarGenero(genero)
        +AtualizarPreco(preco)
        +Desativar()
        +Ativar()
    }

    class BibliotecaJogo {
        +String Id
        +String UsuarioId
        +String JogoId
        +DateTime DataAquisicao
    }

    class Email {
        <<Value Object>>
        +String Endereco
    }

    class Preco {
        <<Value Object>>
        +Decimal Valor
        +String Moeda
    }

    class Senha {
        <<Value Object>>
        +String Valor
    }

    class TipoUsuario {
        <<Enum>>
        Usuario
        Administrador
    }

    class GeneroJogo {
        <<Enum>>
        Acao
        Aventura
        RPG
        Estrategia
        Esporte
        Corrida
        Simulacao
        Puzzle
        FPS
        MOBA
        Educacional
        Outro
    }

    Usuario *-- Email : contem
    Usuario --> TipoUsuario : tipo
    Jogo *-- Preco : contem
    Jogo --> GeneroJogo : genero
    BibliotecaJogo --> Usuario : referencia
    BibliotecaJogo --> Jogo : referencia
```

## Event Storming

### Legenda

| Cor | Elemento | Descricao |
| --- | --- | --- |
| Laranja | **Evento de Dominio** | Algo que aconteceu no sistema |
| Azul | **Comando** | Acao solicitada por um ator |
| Amarelo | **Agregado** | Entidade raiz que processa o comando |
| Lilas | **Politica** | Regra de negocio que deve ser satisfeita |
| Rosa | **Ator** | Quem dispara o comando |

---

### Fluxo 1: Registro de Usuario

```mermaid
flowchart LR
    A["Visitante"]:::actor --> B["RegistrarUsuario"]:::command
    B --> C{"Usuario"}:::aggregate
    C --> D["Validar formato email"]:::policy
    C --> E["Validar email unico"]:::policy
    C --> F["Validar complexidade senha"]:::policy
    D -->|Invalido| G["EmailInvalido"]:::event_fail
    E -->|Duplicado| H["EmailJaExistente"]:::event_fail
    F -->|Fraca| I["SenhaInvalida"]:::event_fail
    D -->|OK| J["Hash senha BCrypt"]:::policy
    E -->|OK| J
    F -->|OK| J
    J --> K["UsuarioRegistrado"]:::event_ok

    classDef actor fill:#FFB6C1,stroke:#333
    classDef command fill:#6495ED,stroke:#333,color:#fff
    classDef aggregate fill:#FFD700,stroke:#333
    classDef policy fill:#DDA0DD,stroke:#333
    classDef event_ok fill:#FFA500,stroke:#333
    classDef event_fail fill:#FF6347,stroke:#333,color:#fff
```

| Elemento | Detalhes |
| --- | --- |
| **Comando** | `RegistrarUsuario(nome, email, senha)` |
| **Agregado** | `Usuario` |
| **Politicas** | Email formato valido, email unico no sistema, senha com 8+ caracteres + maiuscula + minuscula + numero + especial |
| **Eventos de Sucesso** | `UsuarioRegistrado` |
| **Eventos de Falha** | `EmailInvalido`, `EmailJaExistente`, `SenhaInvalida` |
| **Ator** | Visitante (sem autenticacao) |

---

### Fluxo 2: Autenticacao (Login)

```mermaid
flowchart LR
    A["Visitante"]:::actor --> B["AutenticarUsuario"]:::command
    B --> C{"Usuario"}:::aggregate
    C --> D["Buscar por email"]:::policy
    D -->|Nao encontrado| E["CredenciaisInvalidas"]:::event_fail
    D -->|Encontrado| F["Verificar senha BCrypt"]:::policy
    F -->|Incorreta| E
    F -->|Correta| G["Verificar usuario ativo"]:::policy
    G -->|Inativo| H["ContaDesativada"]:::event_fail
    G -->|Ativo| I["Gerar token JWT"]:::policy
    I --> J["UsuarioAutenticado"]:::event_ok

    classDef actor fill:#FFB6C1,stroke:#333
    classDef command fill:#6495ED,stroke:#333,color:#fff
    classDef aggregate fill:#FFD700,stroke:#333
    classDef policy fill:#DDA0DD,stroke:#333
    classDef event_ok fill:#FFA500,stroke:#333
    classDef event_fail fill:#FF6347,stroke:#333,color:#fff
```

| Elemento | Detalhes |
| --- | --- |
| **Comando** | `AutenticarUsuario(email, senha)` |
| **Agregado** | `Usuario` |
| **Politicas** | Mesma mensagem de erro para email inexistente e senha incorreta (prevencao de enumeracao), verificar conta ativa |
| **Eventos de Sucesso** | `UsuarioAutenticado` — retorna token JWT com claims (sub, email, role) |
| **Eventos de Falha** | `CredenciaisInvalidas`, `ContaDesativada` |
| **Ator** | Visitante |

---

### Fluxo 3: Cadastro de Jogo

```mermaid
flowchart LR
    A["Administrador"]:::actor --> B["CadastrarJogo"]:::command
    B --> C{"Jogo"}:::aggregate
    C --> D["Verificar role Admin"]:::policy
    D -->|Nao admin| E["AcessoNegado"]:::event_fail
    D -->|Admin| F["Validar titulo unico"]:::policy
    F -->|Duplicado| G["TituloDuplicado"]:::event_fail
    F -->|Unico| H["Validar preco >= 0"]:::policy
    H -->|Negativo| I["PrecoInvalido"]:::event_fail
    H -->|OK| J["JogoCadastrado"]:::event_ok

    classDef actor fill:#FFB6C1,stroke:#333
    classDef command fill:#6495ED,stroke:#333,color:#fff
    classDef aggregate fill:#FFD700,stroke:#333
    classDef policy fill:#DDA0DD,stroke:#333
    classDef event_ok fill:#FFA500,stroke:#333
    classDef event_fail fill:#FF6347,stroke:#333,color:#fff
```

| Elemento | Detalhes |
| --- | --- |
| **Comando** | `CadastrarJogo(titulo, descricao, genero, preco, dataLancamento)` |
| **Agregado** | `Jogo` |
| **Politicas** | Apenas administrador, titulo unico no catalogo, preco nao negativo |
| **Eventos de Sucesso** | `JogoCadastrado` |
| **Eventos de Falha** | `AcessoNegado`, `TituloDuplicado`, `PrecoInvalido` |
| **Ator** | Administrador (autenticado com role Admin) |

---

### Fluxo 4: Aquisicao de Jogo

```mermaid
flowchart LR
    A["Usuario"]:::actor --> B["AdquirirJogo"]:::command
    B --> C{"BibliotecaJogo"}:::aggregate
    C --> D["Verificar usuario ativo"]:::policy
    D -->|Inativo| E["UsuarioInativo"]:::event_fail
    D -->|Ativo| F["Verificar jogo ativo"]:::policy
    F -->|Inativo| G["JogoInativo"]:::event_fail
    F -->|Ativo| H["Verificar duplicidade"]:::policy
    H -->|Ja possui| I["JogoJaAdquirido"]:::event_fail
    H -->|Novo| J["JogoAdquirido"]:::event_ok

    classDef actor fill:#FFB6C1,stroke:#333
    classDef command fill:#6495ED,stroke:#333,color:#fff
    classDef aggregate fill:#FFD700,stroke:#333
    classDef policy fill:#DDA0DD,stroke:#333
    classDef event_ok fill:#FFA500,stroke:#333
    classDef event_fail fill:#FF6347,stroke:#333,color:#fff
```

| Elemento | Detalhes |
| --- | --- |
| **Comando** | `AdquirirJogo(usuarioId, jogoId)` — usuarioId extraido do token JWT |
| **Agregado** | `BibliotecaJogo` |
| **Politicas** | Usuario deve estar ativo, jogo deve estar ativo, usuario nao pode possuir o jogo |
| **Eventos de Sucesso** | `JogoAdquirido` — cria registro na biblioteca com data de aquisicao |
| **Eventos de Falha** | `UsuarioInativo`, `JogoInativo`, `JogoJaAdquirido` |
| **Ator** | Usuario autenticado |

## Resumo de Comandos e Eventos

| Contexto | Comando | Eventos de Sucesso | Eventos de Falha |
| --- | --- | --- | --- |
| Identidade e Acesso | RegistrarUsuario | UsuarioRegistrado | EmailInvalido, EmailJaExistente, SenhaInvalida |
| Identidade e Acesso | AutenticarUsuario | UsuarioAutenticado | CredenciaisInvalidas, ContaDesativada |
| Catalogo de Jogos | CadastrarJogo | JogoCadastrado | AcessoNegado, TituloDuplicado, PrecoInvalido |
| Catalogo de Jogos | AtualizarJogo | JogoAtualizado | AcessoNegado, TituloDuplicado, JogoNaoEncontrado |
| Catalogo de Jogos | DesativarJogo | JogoDesativado | AcessoNegado, JogoNaoEncontrado |
| Biblioteca | AdquirirJogo | JogoAdquirido | UsuarioInativo, JogoInativo, JogoJaAdquirido |
| Biblioteca | ListarBiblioteca | BibliotecaListada | UsuarioNaoEncontrado |

## Invariantes de Dominio

1. **Email unico**: Nao podem existir dois usuarios com o mesmo endereco de email
2. **Titulo unico**: Nao podem existir dois jogos com o mesmo titulo
3. **Aquisicao unica**: Um usuario nao pode adquirir o mesmo jogo duas vezes
4. **Senha complexa**: Minimo 8 caracteres, ao menos 1 maiuscula, 1 minuscula, 1 numero, 1 especial
5. **Preco nao negativo**: O valor do jogo deve ser >= 0
6. **Soft delete**: Usuarios e jogos sao desativados (Ativo = false), nunca removidos fisicamente
7. **Usuario ativo para aquisicao**: Apenas usuarios ativos podem adquirir jogos
8. **Jogo ativo para aquisicao**: Apenas jogos ativos podem ser adquiridos

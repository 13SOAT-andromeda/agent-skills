---
name: andromeda-architectural
description: Padrão arquitetural Andromeda (Arquitetura Hexagonal) para projetos Go. Define a estrutura de camadas (Domínio, Aplicação, Adaptadores, pkg), fluxo de desenvolvimento, padrões de código e requisitos de teste.
---

# Andromeda Architectural Skill (Go)

Este documento define o modelo arquitetural padrão baseado no projeto `tech-challenge-s1`. Todos os novos projetos e funcionalidades devem seguir esta estrutura de Arquitetura Hexagonal (Ports & Adapters).

## 🏗️ Estrutura de Camadas

A arquitetura é dividida em camadas com responsabilidades bem definidas, isolando a lógica de negócio das implementações técnicas.

### 1. Camada de Domínio (`internal/domain`)
- **Responsabilidade:** Contém as entidades core do negócio e Value Objects (VOs).
- **Regras:**
    - Não deve ter dependências externas (exceto `time`, `errors`, etc.).
    - Entidades são structs simples que representam o estado e o comportamento essencial.
    - Fábricas ou métodos de validação (ex: `NewDocument`, `NewPassword`) devem ser puramente lógicos.

### 2. Camada de Aplicação (`internal/application`)
- **Ports (`internal/application/ports`):**
    - Define interfaces para Repositories e Services/UseCases.
    - Atuam como contratos que desacoplam o core do mundo externo.
- **Services (`internal/application/services`):**
    - Implementações da lógica de negócio.
    - Consomem Ports (Repositories) para persistência.
    - Realizam a orquestração básica.
- **UseCases (`internal/application/usecases`):**
    - Orquestram múltiplos serviços ou repositórios para fluxos complexos.
    - Geralmente usados quando um fluxo de negócio envolve várias entidades.

### 3. Camada de Adaptadores (`internal/adapter`)
- **Database (`internal/adapter/database`):**
    - **Models:** Structs GORM com tags de banco de dados. Devem conter métodos `ToDomain()` e `FromDomain()` para conversão bidirecional entre o domínio e o modelo de persistência.
    - **Repositories:** Implementações concretas das interfaces de porta usando GORM. Devem herdar de um `BaseRepository` para operações CRUD genéricas.
- **HTTP (`internal/adapter/http`):**
    - **Handlers:** Recebem requisições HTTP, realizam o binding do JSON em structs de Request, chamam Services/UseCases e retornam respostas padronizadas via o pacote `response`.
    - **Router:** Configuração centralizada de rotas, middlewares (Auth, Logging, Tracing) e injeção de dependência dos handlers.

### 4. Camada de Utilidades (`pkg`)
- Bibliotecas compartilhadas e utilitários agnósticos ao negócio (ex: `encryption`, `jwt`, `monetary`).

## 🔄 Fluxo de Desenvolvimento (Workflow)

Ao implementar uma nova funcionalidade, siga esta ordem rigorosa:

1.  **Domínio:** Defina a entidade em `internal/domain`.
2.  **Contratos (Ports):** Defina a interface do Repository e do Service em `internal/application/ports`.
3.  **Persistência (Adapters):**
    - Crie o modelo GORM em `internal/adapter/database/model/<entidade>`.
    - Implemente `ToDomain` e `FromDomain`.
    - Implemente o Repository em `internal/adapter/database/repository`.
4.  **Lógica de Negócio:** Implemente o Service em `internal/application/services`.
5.  **Entrada (HTTP):**
    - Crie o Handler em `internal/adapter/http/handlers`.
    - Defina structs de Request com tags `binding` do Gin.
    - Registre as rotas no `internal/adapter/http/router.go`.
6.  **Injeção de Dependência:** Configure e instancie os componentes no `cmd/api/main.go`.

## 🛠️ Padrões de Código

- **Injeção de Dependência:** Sempre via construtores (`NewService`, `NewHandler`).
- **Nomenclatura de Interfaces:** Devem ser verbais ou substantivadas (ex: `CustomerRepository`, `CustomerService`).
- **Tratamento de Erros:** Retorne erros das camadas inferiores e trate-os no Handler ou Service, provendo mensagens claras.
- **Respostas HTTP:** Use sempre `response.RespondSuccess`, `response.RespondCreated` ou `response.RespondError`.
- **Validação:** Use tags `binding:"required,..."` do Gin/Validator v10.

## 🧪 Requisitos de Teste (TDD)

- **Unitários:** Devem testar lógica de negócio em Services e VOs no Domínio. Mocks de repositórios devem ser usados.
- **Integração/E2E:** Devem validar o fluxo completo do endpoint HTTP até o banco de dados.
- **Localização:** Arquivos `_test.go` devem residir junto ao código que testam.

---
*Este modelo arquitetural é o padrão ouro para todos os projetos da Andromeda.*

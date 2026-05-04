---
name: go-hexagonal-api
description: >
  Scaffold or extend Go REST API projects using the team's hexagonal (ports & adapters) architecture.
  Use this skill whenever anyone asks to: create a new Go project/API/service, add a module/entity/domain
  to an existing Go project, scaffold a Go REST API, or set up a Go backend with PostgreSQL. Also trigger
  for requests like "novo projeto Go", "adicionar módulo", "criar API Go", "estrutura hexagonal Go".
  This skill enforces the team's exact naming conventions (singular `adapter`, `internal/domain/` directly,
  `internal/application/ports/`, collocated `_test.go`, mocks in `internal/mocks/`) to ensure consistency
  across all team members. Always use this skill for Go API work — do not improvise structure.
---

# Go Hexagonal API

This skill captures the team's proven hexagonal architecture from `tech-challenge-s1`. Every new Go REST API
project must follow these conventions exactly. Consistent naming is non-negotiable — inconsistent naming
(e.g., `adapters` vs `adapter`, `internal/core/domain` vs `internal/domain`) creates drift across projects.

## Directory Structure

```
<project-root>/
├── cmd/
│   └── api/
│       └── main.go              # Entry point: config → DB → wire deps → start server
├── internal/
│   ├── domain/                  # Pure Go — ZERO framework imports
│   │   ├── <entity>.go          # Entities and value objects
│   │   └── errors.go            # Domain errors (optional)
│   ├── application/
│   │   ├── ports/               # Interfaces (contracts) only
│   │   │   ├── repositories.go  # Repository interfaces
│   │   │   └── services.go      # Service interfaces (if needed externally)
│   │   └── services/            # Business logic implementations
│   │       └── <entity>_service.go
│   ├── adapter/                 # SINGULAR — never "adapters"
│   │   ├── database/
│   │   │   ├── model/
│   │   │   │   └── <entity>/    # One subdir per entity
│   │   │   │       └── model.go # GORM struct + FromDomain() + ToDomain()
│   │   │   └── repository/
│   │   │       └── <entity>_repository.go
│   │   └── http/
│   │       ├── handlers/
│   │       │   └── <entity>_handler.go
│   │       ├── middlewares/     # WITH 's' — middlewares/
│   │       │   └── auth_middleware.go
│   │       ├── response/
│   │       │   └── response.go
│   │       └── router.go
│   └── mocks/                   # Testify mocks collocated here
│       └── mock_<entity>_repository.go
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── go.mod
├── Makefile
└── air.toml
```

**Critical naming rules:**
- `internal/adapter/` — SINGULAR, never `internal/adapters/`
- `internal/domain/` — directly under internal, never `internal/core/domain/`
- `internal/application/ports/` — ports live here, not in `internal/core/ports/`
- `internal/adapter/http/middlewares/` — WITH the trailing `s`
- `internal/adapter/database/model/<entity>/model.go` — one subdir per entity
- `internal/mocks/` — mocks inside internal, never in `tests/mocks/`
- Tests: collocated `<file>_test.go` next to the code, not in a separate `tests/` dir

## Architecture Rules

Three layers, strict inward dependency:

```
Domain  ←  Application  ←  Adapters
(pure)     (ports+svcs)    (HTTP, GORM, email, ...)
```

1. **Domain** (`internal/domain/`): Pure Go structs and value objects. No `gorm.Model`, no `gin`, no `jwt`. Only stdlib imports (`errors`, `time`, `strings`, etc.).
2. **Application** (`internal/application/`): Port interfaces define what the domain needs (repositories, external services). Services implement business logic using port interfaces — never concrete GORM types.
3. **Adapters** (`internal/adapter/`): GORM models translate between DB and domain. HTTP handlers call services. Wiring happens in `main.go`.

## Code Patterns

### Domain Entity

```go
// internal/domain/user.go
package domain

import (
    "errors"
    "time"
)

type User struct {
    ID        string
    Name      string
    Email     string
    Password  string
    CreatedAt time.Time
    UpdatedAt time.Time
}

func NewUser(name, email, password string) (*User, error) {
    if name == "" {
        return nil, errors.New("name is required")
    }
    if email == "" {
        return nil, errors.New("email is required")
    }
    return &User{
        Name:     name,
        Email:    email,
        Password: password,
    }, nil
}
```

### Value Object

```go
// internal/domain/document.go
package domain

import (
    "errors"
    "regexp"
)

type Document struct {
    value string
}

func NewDocument(value string) (Document, error) {
    cleaned := regexp.MustCompile(`\D`).ReplaceAllString(value, "")
    if len(cleaned) != 11 && len(cleaned) != 14 {
        return Document{}, errors.New("invalid document")
    }
    return Document{value: cleaned}, nil
}

func (d Document) String() string { return d.value }
```

### Port Interfaces

```go
// internal/application/ports/repositories.go
package ports

import "github.com/myorg/myproject/internal/domain"

type UserRepository interface {
    Create(user *domain.User) error
    FindByID(id string) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
    Update(user *domain.User) error
    Delete(id string) error
    List() ([]*domain.User, error)
}
```

### Service (uses port interface, NOT concrete type)

```go
// internal/application/services/user_service.go
package services

import (
    "github.com/myorg/myproject/internal/application/ports"
    "github.com/myorg/myproject/internal/domain"
)

type UserService struct {
    repo ports.UserRepository  // interface, never gorm repo directly
}

func NewUserService(repo ports.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Create(name, email, password string) (*domain.User, error) {
    user, err := domain.NewUser(name, email, password)
    if err != nil {
        return nil, err
    }
    if err := s.repo.Create(user); err != nil {
        return nil, err
    }
    return user, nil
}
```

### GORM Model with FromDomain / ToDomain

```go
// internal/adapter/database/model/user/model.go
package usermodel

import (
    "time"
    "github.com/myorg/myproject/internal/domain"
)

type User struct {
    ID        string `gorm:"primaryKey"`
    Name      string
    Email     string `gorm:"uniqueIndex"`
    Password  string
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (m *User) ToDomain() *domain.User {
    return &domain.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        Password:  m.Password,
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
}

func FromDomain(u *domain.User) *User {
    return &User{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Password:  u.Password,
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }
}
```

### Repository (GORM implementation)

```go
// internal/adapter/database/repository/user_repository.go
package repository

import (
    "gorm.io/gorm"
    "github.com/myorg/myproject/internal/application/ports"
    "github.com/myorg/myproject/internal/domain"
    usermodel "github.com/myorg/myproject/internal/adapter/database/model/user"
)

type UserRepositoryGorm struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) ports.UserRepository {
    return &UserRepositoryGorm{db: db}
}

func (r *UserRepositoryGorm) Create(user *domain.User) error {
    model := usermodel.FromDomain(user)
    return r.db.Create(model).Error
}

func (r *UserRepositoryGorm) FindByID(id string) (*domain.User, error) {
    var model usermodel.User
    if err := r.db.First(&model, "id = ?", id).Error; err != nil {
        return nil, err
    }
    return model.ToDomain(), nil
}
```

### HTTP Handler (Gin)

```go
// internal/adapter/http/handlers/user_handler.go
package handlers

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/myorg/myproject/internal/adapter/http/response"
    "github.com/myorg/myproject/internal/application/services"
)

type UserHandler struct {
    service *services.UserService
}

func NewUserHandler(service *services.UserService) *UserHandler {
    return &UserHandler{service: service}
}

type createUserRequest struct {
    Name     string `json:"name"     binding:"required"`
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

func (h *UserHandler) Create(c *gin.Context) {
    var req createUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, err.Error())
        return
    }
    user, err := h.service.Create(req.Name, req.Email, req.Password)
    if err != nil {
        response.InternalError(c, err.Error())
        return
    }
    response.Created(c, user)
}
```

### Standardized Response Helpers

```go
// internal/adapter/http/response/response.go
package response

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

func OK(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{Success: true, Data: data})
}

func Created(c *gin.Context, data interface{}) {
    c.JSON(http.StatusCreated, Response{Success: true, Data: data})
}

func BadRequest(c *gin.Context, msg string) {
    c.JSON(http.StatusBadRequest, Response{Success: false, Error: msg})
}

func Unauthorized(c *gin.Context, msg string) {
    c.JSON(http.StatusUnauthorized, Response{Success: false, Error: msg})
}

func NotFound(c *gin.Context, msg string) {
    c.JSON(http.StatusNotFound, Response{Success: false, Error: msg})
}

func InternalError(c *gin.Context, msg string) {
    c.JSON(http.StatusInternalServerError, Response{Success: false, Error: msg})
}
```

### JWT Middleware

```go
// internal/adapter/http/middlewares/auth_middleware.go
package middlewares

import (
    "os"
    "strings"
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "github.com/myorg/myproject/internal/adapter/http/response"
)

func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        if !strings.HasPrefix(header, "Bearer ") {
            response.Unauthorized(c, "missing token")
            c.Abort()
            return
        }
        tokenStr := strings.TrimPrefix(header, "Bearer ")
        token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
            return []byte(os.Getenv("JWT_SECRET")), nil
        })
        if err != nil || !token.Valid {
            response.Unauthorized(c, "invalid token")
            c.Abort()
            return
        }
        c.Next()
    }
}
```

### Router

```go
// internal/adapter/http/router.go
package http

import (
    "github.com/gin-gonic/gin"
    "github.com/myorg/myproject/internal/adapter/http/handlers"
    "github.com/myorg/myproject/internal/adapter/http/middlewares"
)

func SetupRouter(userHandler *handlers.UserHandler) *gin.Engine {
    r := gin.Default()

    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })

    api := r.Group("/api/v1")
    {
        users := api.Group("/users")
        users.POST("/", userHandler.Create)

        protected := api.Group("/")
        protected.Use(middlewares.AuthRequired())
        {
            protected.GET("/users/:id", userHandler.GetByID)
        }
    }
    return r
}
```

### main.go Wiring

```go
// cmd/api/main.go
package main

import (
    "log"
    "os"
    "github.com/myorg/myproject/internal/adapter/database/repository"
    adapthttp "github.com/myorg/myproject/internal/adapter/http"
    "github.com/myorg/myproject/internal/adapter/http/handlers"
    "github.com/myorg/myproject/internal/application/services"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("failed to connect to database: %v", err)
    }

    // Wire: repo -> service -> handler
    userRepo := repository.NewUserRepository(db)
    userSvc  := services.NewUserService(userRepo)
    userH    := handlers.NewUserHandler(userSvc)

    router := adapthttp.SetupRouter(userH)

    port := os.Getenv("HTTP_PORT")
    if port == "" {
        port = "8080"
    }
    log.Printf("Server starting on :%s", port)
    if err := router.Run(":" + port); err != nil {
        log.Fatalf("failed to start server: %v", err)
    }
}
```

### go.mod

```
module github.com/<org>/<project>

go 1.24

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/golang-jwt/jwt/v5 v5.2.1
    github.com/google/uuid v1.6.0
    github.com/joho/godotenv v1.5.1
    github.com/stretchr/testify v1.9.0
    gorm.io/driver/postgres v1.5.9
    gorm.io/gorm v1.25.12
)
```

### Dockerfile (3-stage)

```dockerfile
FROM golang:1.24-alpine AS production_builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/api

FROM golang:1.24 AS development
WORKDIR /app
RUN go install github.com/air-verse/air@latest
COPY go.mod go.sum ./
RUN go mod download
COPY . .
CMD ["air", "-c", ".air.toml"]

FROM alpine:3.22 AS production
RUN addgroup -S nonroot && adduser -S nonroot -G nonroot
WORKDIR /app
COPY --from=production_builder /app/server .
USER nonroot:nonroot
EXPOSE 8080
CMD ["./server"]
```

### docker-compose.yml

```yaml
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      target: development
    ports:
      - "${HTTP_PORT:-8080}:8080"
    env_file: .env
    volumes:
      - .:/app
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```

### .env.example

```
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=myproject
DB_SSLMODE=disable
DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}?sslmode=${DB_SSLMODE}

JWT_SECRET=change-me-in-production
HTTP_PORT=8080
```

### Unit Test with Testify Mock

```go
// internal/application/services/user_service_test.go
package services_test

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/myorg/myproject/internal/application/services"
    "github.com/myorg/myproject/internal/mocks"
)

func TestUserService_Create(t *testing.T) {
    mockRepo := new(mocks.MockUserRepository)
    svc := services.NewUserService(mockRepo)

    mockRepo.On("Create", mock.AnythingOfType("*domain.User")).Return(nil)

    user, err := svc.Create("Alice", "alice@example.com", "password123")
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    mockRepo.AssertExpectations(t)
}
```

### Testify Mock

```go
// internal/mocks/mock_user_repository.go
package mocks

import (
    "github.com/stretchr/testify/mock"
    "github.com/myorg/myproject/internal/domain"
)

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(user *domain.User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) FindByID(id string) (*domain.User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}
```

## Adding a New Domain Module Checklist

Follow this order exactly when adding a new entity (e.g., `Product`):

1. Create `internal/domain/product.go` — pure struct, no framework imports
2. Add `ProductRepository` interface to `internal/application/ports/repositories.go`
3. Create `internal/adapter/database/model/product/model.go` — GORM struct with `FromDomain()` + `ToDomain()`
4. Implement `internal/adapter/database/repository/product_repository.go` — GORM implementation returning `ports.ProductRepository`
5. Create `internal/application/services/product_service.go` — receives `ports.ProductRepository`, not the concrete GORM type
6. Create `internal/adapter/http/handlers/product_handler.go` — uses `gin.Context`, calls service methods
7. Add product routes to `internal/adapter/http/router.go`
8. Create `internal/mocks/mock_product_repository.go` — testify mock implementing `ports.ProductRepository`
9. Write `internal/application/services/product_service_test.go` — uses mock, asserts behavior
10. Wire in `cmd/api/main.go`: `productRepo := repository.NewProductRepository(db)`, `productSvc := services.NewProductService(productRepo)`, `productH := handlers.NewProductHandler(productSvc)`
11. Add GORM AutoMigrate for the new model in main.go if applicable
12. Update `.env.example` if new env vars needed

## New Project Checklist

1. Create `go.mod` with `module github.com/<org>/<project>` and `go 1.24`
2. Run `go get` for gin, gorm, postgres driver, jwt, uuid, godotenv, testify
3. Create `.env.example` with `DB_*`, `JWT_SECRET`, `HTTP_PORT`
4. Create `internal/domain/` entities (no framework imports)
5. Create `internal/application/ports/repositories.go` with all repository interfaces
6. Create `internal/adapter/database/model/<entity>/model.go` for each entity
7. Create `internal/adapter/database/repository/` implementations
8. Create `internal/application/services/` for each entity
9. Create `internal/adapter/http/response/response.go` with all helpers
10. Create `internal/adapter/http/middlewares/auth_middleware.go` with JWT
11. Create `internal/adapter/http/handlers/` for each entity
12. Create `internal/adapter/http/router.go` wiring all handlers and middleware
13. Create `cmd/api/main.go` loading config, opening DB, wiring all deps, starting router
14. Create `internal/mocks/` with testify mocks for each repository interface
15. Write at least one `_test.go` per service using mocks
16. Create 3-stage `Dockerfile` (production_builder -> development -> production/alpine/nonroot)
17. Create `docker-compose.yml` with healthcheck on db and `depends_on: condition: service_healthy`
18. Create `Makefile` with `run`, `build`, `test`, `lint` targets
19. Create `air.toml` for hot reload in development

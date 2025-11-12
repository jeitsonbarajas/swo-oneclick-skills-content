# Instrucciones de GitHub Copilot para .NET - Orquestador de Skills

**Generado:** 2025-11-12  
**Versión:** 2.0  
**Tecnología:** .NET (C#)  
**Tipo:** Orquestador  
**Autor:** SWO Team

## Descripción General

Este archivo actúa como el **orquestador de skills** para el desarrollo .NET con GitHub Copilot. Mapea las intenciones del usuario a archivos de skills específicos que contienen conocimiento especializado y patrones.

## Cómo Usar Este Orquestador

Cuando trabajas en un proyecto .NET, Copilot:
1. **Analiza tu solicitud** o contexto de código
2. **Identifica los skills relevantes** necesarios
3. **Aplica patrones** de los archivos de skill apropiados
4. **Combina múltiples skills** cuando sea necesario

## Skills Disponibles

### 🌐 Desarrollo de Web APIs
**Archivo Skill:** `aspnet-webapi.skill.md`

**Usar cuando necesites:**
- Crear REST APIs
- Construir endpoints HTTP
- Implementar API controllers o Minimal APIs
- Configurar versionado de APIs
- Configurar Swagger/OpenAPI
- Manejar requests/responses HTTP
- Implementar DTOs (Request/Response models)
- Configurar CORS

**Palabras clave que activan este skill:**
- "crear API", "endpoint REST", "web service"
- "controller", "HTTP GET/POST/PUT/DELETE"
- "versionado API", "Swagger", "OpenAPI"
- "Minimal API", "DTOs", "CORS"

**Ejemplos de solicitudes:**
- "Crear una REST API para gestión de productos"
- "Agregar endpoint para obtener usuario por ID"
- "Implementar paginación en mi API"
- "Configurar Swagger para mi proyecto"
- "Crear DTOs para request y response"

---

### 🗄️ Entity Framework Core / Database
**Archivo Skill:** `ef-core.skill.md`

**Usar cuando necesites:**
- Trabajar con bases de datos
- Configurar DbContext
- Definir entidades y relaciones
- Usar Fluent API
- Implementar repository pattern
- Hacer queries con LINQ
- Manejar transacciones
- Crear y aplicar migraciones
- Optimizar queries

**Palabras clave que activan este skill:**
- "base de datos", "DbContext", "entity"
- "repository", "LINQ", "query"
- "migración", "SQL", "relación"
- "Fluent API", "configuración", "transacción"

**Ejemplos de solicitudes:**
- "Configurar DbContext para mi aplicación"
- "Crear entidades User y Order con relación"
- "Implementar repository pattern genérico"
- "Hacer query para obtener productos con sus categorías"
- "Crear migración para agregar tabla"
- "Optimizar consulta que trae muchos datos"

---

### 🔐 Autenticación y Autorización
**Archivo Skill:** `authentication.skill.md`

**Usar cuando necesites:**
- Implementar JWT authentication
- Configurar ASP.NET Core Identity
- Proteger endpoints
- Implementar autorización basada en roles
- Crear políticas de autorización
- Manejar refresh tokens
- Agregar claims personalizados
- Configurar middleware de autenticación

**Palabras clave que activan este skill:**
- "autenticación", "autorización", "JWT", "token"
- "login", "register", "Identity"
- "roles", "claims", "políticas"
- "proteger endpoint", "seguridad"
- "refresh token"

**Ejemplos de solicitudes:**
- "Implementar autenticación JWT en mi API"
- "Crear endpoint de login con JWT"
- "Proteger endpoints para que solo admins puedan acceder"
- "Agregar roles a usuarios"
- "Implementar refresh token"
- "Crear política de autorización personalizada"

---

### ✅ Validación y Manejo de Errores
**Archivo Skill:** `validation.skill.md`

**Usar cuando necesites:**
- Validar datos de entrada
- Usar FluentValidation
- Usar Data Annotations
- Crear validadores personalizados
- Manejar excepciones globalmente
- Implementar middleware de error handling
- Retornar errores consistentes (Problem Details)
- Validar con acceso a base de datos

**Palabras clave que activan este skill:**
- "validación", "validator", "FluentValidation"
- "error", "excepción", "manejo de errores"
- "validar", "Data Annotations"
- "Problem Details", "error handling"

**Ejemplos de solicitudes:**
- "Validar request con FluentValidation"
- "Crear validator para CreateUserRequest"
- "Implementar manejo global de excepciones"
- "Validar que email sea único en la base de datos"
- "Retornar errores en formato Problem Details"
- "Crear excepción personalizada para reglas de negocio"

---

### 🧪 Testing
**Archivo Skill:** `testing.skill.md`

**Usar cuando necesites:**
- Escribir unit tests
- Crear integration tests
- Usar mocking (Moq, NSubstitute)
- Configurar test fixtures
- Testear servicios, repositories, controllers
- Usar In-Memory Database para tests
- Configurar WebApplicationFactory
- Medir code coverage

**Palabras clave que activan este skill:**
- "test", "testing", "unit test", "integration test"
- "mock", "Moq", "fixture"
- "xUnit", "NUnit", "MSTest"
- "WebApplicationFactory", "In-Memory Database"
- "FluentAssertions"

**Ejemplos de solicitudes:**
- "Crear unit test para UserService"
- "Escribir integration test para API de productos"
- "Mockear IUserRepository con Moq"
- "Crear test con WebApplicationFactory"
- "Testear validator con múltiples casos"
- "Configurar In-Memory Database para tests"

---

### ⚙️ Dependency Injection y Configuración
**Archivo Skill:** `di-configuration.skill.md`

**Usar cuando necesites:**
- Configurar Dependency Injection
- Entender service lifetimes (Transient, Scoped, Singleton)
- Usar Options Pattern
- Configurar appsettings.json
- Manejar configuración por ambiente
- Usar User Secrets
- Registrar servicios con factory
- Organizar registros de servicios

**Palabras clave que activan este skill:**
- "dependency injection", "DI", "IoC"
- "Transient", "Scoped", "Singleton"
- "Options Pattern", "IOptions", "configuración"
- "appsettings", "User Secrets", "environment"
- "registrar servicios", "AddScoped"

**Ejemplos de solicitudes:**
- "Configurar dependency injection para mis servicios"
- "Usar Options Pattern para leer EmailSettings"
- "Registrar servicio con factory method"
- "Configurar appsettings por ambiente"
- "Guardar JWT secret en User Secrets"
- "Organizar registros de servicios en extension methods"

---

### ⚡ Performance y Optimización
**Archivo Skill:** `performance.skill.md`

**Usar cuando necesites:**
- Implementar caching (In-Memory, Distributed, Response)
- Optimizar queries de base de datos
- Usar async/await correctamente
- Reducir memory allocations
- Usar Span<T> y Memory<T>
- Implementar object pooling
- Configurar compression
- Profiling y diagnostics

**Palabras clave que activan este skill:**
- "cache", "caching", "Redis"
- "performance", "optimización", "rendimiento"
- "async", "await", "paralelo"
- "memory", "Span", "allocation"
- "profiling", "diagnostics"
- "compression"

**Ejemplos de solicitudes:**
- "Agregar caching a mi servicio"
- "Optimizar query que trae mucha data"
- "Implementar distributed caching con Redis"
- "Ejecutar múltiples queries en paralelo"
- "Usar Span<T> para procesar string sin allocations"
- "Configurar response caching"
- "Optimizar uso de memoria en mi aplicación"

---

### 💬 Chat Mode General
**Archivo:** `dotnet-chatmode.md`

**Usar cuando necesites:**
- Conocimiento general de .NET
- Características modernas de C#
- Patrones de arquitectura
- Principios SOLID
- Configuración básica de proyecto
- Dudas generales sobre .NET

**Este archivo referencia los skills específicos cuando se necesita implementación detallada.**

**Ejemplos de solicitudes:**
- "¿Cuáles son las características nuevas de C# 12?"
- "Explícame los principios SOLID"
- "¿Cómo estructurar un proyecto .NET?"
- "¿Qué es un record type?"
- "¿Cuándo usar async/await?"

---

## Mapeo de Intenciones a Skills

### Escenario 1: Crear una REST API Completa

**Usuario solicita:** "Crear una API REST para gestionar productos con CRUD completo"

**Skills a usar:**
1. `aspnet-webapi.skill.md` → Crear endpoints y controllers
2. `ef-core.skill.md` → Configurar DbContext y repository
3. `validation.skill.md` → Validar requests
4. `di-configuration.skill.md` → Configurar DI

**Orden de implementación:**
1. Definir entidad Product (`ef-core`)
2. Crear DbContext (`ef-core`)
3. Crear repository (`ef-core`)
4. Crear DTOs y validators (`aspnet-webapi`, `validation`)
5. Crear controller con endpoints (`aspnet-webapi`)
6. Configurar DI en Program.cs (`di-configuration`)

---

### Escenario 2: Agregar Autenticación JWT

**Usuario solicita:** "Necesito autenticación con JWT y proteger mis endpoints"

**Skills a usar:**
1. `authentication.skill.md` → Implementar JWT
2. `validation.skill.md` → Validar login request
3. `ef-core.skill.md` → Guardar refresh tokens
4. `di-configuration.skill.md` → Configurar JWT settings

**Orden de implementación:**
1. Configurar JwtSettings en appsettings (`di-configuration`)
2. Crear TokenService (`authentication`)
3. Crear AuthController con login/refresh (`authentication`)
4. Validar LoginRequest (`validation`)
5. Proteger endpoints con [Authorize] (`authentication`)

---

### Escenario 3: Mejorar Performance

**Usuario solicita:** "Mi API es lenta, necesito optimizarla"

**Skills a usar:**
1. `performance.skill.md` → Agregar caching, optimizar queries
2. `ef-core.skill.md` → Optimizar queries EF
3. `aspnet-webapi.skill.md` → Response caching

**Análisis y soluciones:**
1. Agregar caching en servicios (`performance`)
2. Usar AsNoTracking en queries read-only (`performance`, `ef-core`)
3. Proyectar solo campos necesarios (`ef-core`)
4. Configurar response caching (`performance`)
5. Ejecutar queries en paralelo cuando sea posible (`performance`)

---

### Escenario 4: Escribir Tests

**Usuario solicita:** "Crear tests para mis servicios y API"

**Skills a usar:**
1. `testing.skill.md` → Configurar tests, mocking
2. Skill del componente a testear (ej: `aspnet-webapi`, `ef-core`)

**Orden de implementación:**
1. Crear proyecto de tests (`testing`)
2. Configurar WebApplicationFactory para integration tests (`testing`)
3. Escribir unit tests con mocks (`testing`)
4. Escribir integration tests para endpoints (`testing`)

---

## Reglas de Orquestación

### Prioridad de Skills

Cuando múltiples skills son relevantes:
1. **Identificar el skill principal** (el que resuelve el core de la solicitud)
2. **Identificar skills secundarios** (dependencias o complementos)
3. **Aplicar en orden lógico** (ej: crear entidades antes que controllers)

### Combinación de Skills

**Patrones comunes:**

| Solicitud Principal | Skills Primarios | Skills Secundarios |
|---------------------|------------------|-------------------|
| Crear API CRUD | `aspnet-webapi` | `ef-core`, `validation`, `di-configuration` |
| Agregar autenticación | `authentication` | `validation`, `ef-core`, `di-configuration` |
| Optimizar performance | `performance` | `ef-core`, `aspnet-webapi` |
| Escribir tests | `testing` | Skill del componente a testear |
| Validar datos | `validation` | `aspnet-webapi` |

### Separación de Responsabilidades

Cada skill tiene un dominio claro:
- **NO uses** `aspnet-webapi` para configurar Entity Framework
- **NO uses** `ef-core` para validar requests HTTP
- **NO uses** `authentication` para manejar errores generales
- **SÍ combina** skills cuando la solicitud lo requiere

---

## Estándares de Código

### Convenciones

Todos los skills siguen estas convenciones:
- **Documentación:** Español
- **Código:** English (nombres de clases, métodos, variables)
- **Comentarios en código:** English
- **Conceptos técnicos:** English (ej: "DbContext", "middleware", "repository")

### Estructura de Proyecto

```
MyApp/
├── src/
│   ├── MyApp.Api/              # Presentation Layer
│   ├── MyApp.Application/      # Business Logic
│   ├── MyApp.Domain/           # Domain Entities
│   └── MyApp.Infrastructure/   # Data Access
└── tests/
    ├── MyApp.UnitTests/
    └── MyApp.IntegrationTests/
```

### Buenas Prácticas

Aplica automáticamente:
- File-scoped namespaces
- Record types para DTOs
- Nullable reference types (#nullable enable)
- Async/await para operaciones I/O
- Dependency Injection
- Logging con ILogger
- CancellationToken en métodos async

---

## Referencia Rápida: Cuándo Usar Cada Skill

| Quieres... | Usa este skill |
|---|---|
| Crear endpoints REST | `aspnet-webapi.skill.md` |
| Trabajar con base de datos | `ef-core.skill.md` |
| Agregar autenticación | `authentication.skill.md` |
| Validar entrada de datos | `validation.skill.md` |
| Escribir tests | `testing.skill.md` |
| Configurar servicios | `di-configuration.skill.md` |
| Optimizar performance | `performance.skill.md` |
| Preguntas generales .NET | `dotnet-chatmode.md` |

---

## Recursos Relacionados

- [ASP.NET Web API Skill](aspnet-webapi.skill.md)
- [Entity Framework Core Skill](ef-core.skill.md)
- [Authentication Skill](authentication.skill.md)
- [Validation Skill](validation.skill.md)
- [Testing Skill](testing.skill.md)
- [DI & Configuration Skill](di-configuration.skill.md)
- [Performance Skill](performance.skill.md)
- [.NET Chat Mode](dotnet-chatmode.md)

---

## Actualizaciones y Versionado

**Versión Actual:** 2.0  
**Última Actualización:** 2025-11-12

**Skills Incluidos:**
- ✅ aspnet-webapi.skill.md
- ✅ ef-core.skill.md
- ✅ authentication.skill.md
- ✅ validation.skill.md
- ✅ testing.skill.md
- ✅ di-configuration.skill.md
- ✅ performance.skill.md
- ✅ dotnet-chatmode.md

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET de SWO*

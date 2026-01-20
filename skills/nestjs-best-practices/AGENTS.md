# NestJS Best Practices

**Version 1.0.0**  
Xiro - The Terminal Labs Engineering  
January 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when maintaining,  
> generating, or refactoring NestJS codebases. Humans  
> may also find it useful, but guidance here is optimized for automation  
> and consistency by AI-assisted workflows.

---

## Abstract

Comprehensive performance optimization guide for NestJS with expressjs platform applications, designed for AI agents and LLMs. Contains 40+ rules across 8 categories, prioritized by impact from critical (eliminating waterfalls, reducing bundle size) to incremental (advanced patterns). Each rule includes detailed explanations, real-world examples comparing incorrect vs. correct implementations, and specific impact metrics to guide automated refactoring and code generation.

---

## Table of Contents

1. [Section 1](#1-section-1) — **CRITICAL**
   - 1.1 [Enable CORS with Whitelist Origins Only](#11-enable-cors-with-whitelist-origins-only)
   - 1.2 [Regular Dependency Security Audits](#12-regular-dependency-security-audits)
   - 1.3 [Use Helmet Middleware for Security Headers](#13-use-helmet-middleware-for-security-headers)
2. [Section 2](#2-section-2) — **HIGH**
   - 2.1 [Cache Frequently Used Data with Redis](#21-cache-frequently-used-data-with-redis)
3. [Section 3](#3-section-3) — **HIGH**
   - 3.1 [Keep Functions Short and Single Purpose](#31-keep-functions-short-and-single-purpose)
   - 3.2 [Organize Code by Feature Modules](#32-organize-code-by-feature-modules)
   - 3.3 [Remove Unused Code and Dependencies](#33-remove-unused-code-and-dependencies)
   - 3.4 [Single Responsibility - Separate Controller and Service](#34-single-responsibility---separate-controller-and-service)
   - 3.5 [Use Consistent Naming Conventions](#35-use-consistent-naming-conventions)
4. [Section 4](#4-section-4) — **CRITICAL**
   - 4.1 [Enable Global Exception Filter](#41-enable-global-exception-filter)
   - 4.2 [Implement Proper Logging Strategy](#42-implement-proper-logging-strategy)
5. [Section 5](#5-section-5) — **CRITICAL**
   - 5.1 [Validate All Inputs with DTOs and ValidationPipe](#51-validate-all-inputs-with-dtos-and-validationpipe)
6. [Section 6](#6-section-6) — **CRITICAL**
   - 6.1 [Use Parameterized Queries to Prevent SQL Injection](#61-use-parameterized-queries-to-prevent-sql-injection)
7. [Section 7](#7-section-7) — **CRITICAL**
   - 7.1 [Use Guards for Route Protection](#71-use-guards-for-route-protection)
8. [Section 8](#8-section-8) — **MEDIUM**
   - 8.1 [Generate Swagger/OpenAPI Documentation](#81-generate-swaggeropenapi-documentation)
9. [Section 9](#9-section-9) — **CRITICAL**
   - 9.1 [Never Hardcode Secrets - Use Environment Variables](#91-never-hardcode-secrets---use-environment-variables)
10. [Section 10](#10-section-10) — **MEDIUM**
   - 10.1 [Write Comprehensive Unit Tests](#101-write-comprehensive-unit-tests)
11. [Section 11](#11-section-11) — **HIGH**
   - 11.1 [Implement Health Check Endpoints](#111-implement-health-check-endpoints)
12. [Section 12](#12-section-12) — **HIGH**
   - 12.1 [Enable Compression Middleware for Responses](#121-enable-compression-middleware-for-responses)
   - 12.2 [Implement Rate Limiting for All Routes](#122-implement-rate-limiting-for-all-routes)
13. [Section 13](#13-section-13) — **HIGH**
   - 13.1 [Lazy Load Non-Critical Modules](#131-lazy-load-non-critical-modules)

---

## 1. Section 1

**Impact: CRITICAL**

### 1.1 Enable CORS with Whitelist Origins Only

**Impact: CRITICAL (Prevents unauthorized domain access)**

Wildcard CORS (`*`) allows malicious sites to make requests to your API. Explicit origin whitelist prevents cross-site attacks. **Never use wildcard in production.**

**Incorrect: vulnerable CORS**

```typescript
// main.ts
app.enableCors();  // Uses '*' - dangerous! 🚨
```

**Correct: secure CORS**

```typescript
// main.ts
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['https://yourapp.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
});
```

### 1.2 Regular Dependency Security Audits

**Impact: CRITICAL (Prevents exploitation of known vulnerabilities in packages)**

Outdated dependencies contain known vulnerabilities that attackers exploit. Automated scanning with npm audit, bun audit, and Snyk catches issues before production. **Never deploy without clean dependency audit.**

**Incorrect: vulnerable deps**

```json
// package.json - outdated packages 🚨
{
  "dependencies": {
    "lodash": "^4.17.20",  // CVE-2021-23337
    "express": "^4.17.1"    // Multiple CVEs
  }
}
```

**Correct: automated security**

```bash
# Pre-commit / CI checks
bun run audit:check
npm audit --audit-level high
bun pm audit           # Bun's audit check
snyk test --severity-threshold=high
```

**Workflow:**

- `npm audit` or `bun pm audit` weekly in CI/CD

- `npm audit fix` auto-updates patches

- Snyk/Dependabot for vulnerability alerts

- Pin major versions, auto-patch minors

### 1.3 Use Helmet Middleware for Security Headers

**Impact: CRITICAL (Protects against XSS, clickjacking, and other web attacks)**

NestJS with Express adapter exposes HTTP headers that can be exploited by attackers. Helmet automatically sets security headers like X-XSS-Protection, X-Frame-Options, and Content-Security-Policy. **Always enable it in production.**

> **Hint**: Apply `helmet` as global middleware **before** other calls to `app.use()` or route definitions. Middleware order matters - routes defined before helmet will not have security headers applied.

**Express: default adapter**

```bash
bun add helmet
```

**Fastify adapter:**

```bash
bun add @fastify/helmet
```

**Incorrect: vulnerable headers**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Define routes before helmet - insecure!
  app.route('/api', ...);

  app.use(helmet());
  await app.listen(3000);
}
```

**Correct: Express**

```typescript
// main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Apply helmet FIRST, before routes and other middleware
  app.use(helmet());

  // Now define routes
  app.route('/api', ...);

  await app.listen(3000);
}
```

**Correct: Fastify**

```typescript
// main.ts
import helmet from '@fastify/helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Fastify uses register() instead of use()
  await app.register(helmet);

  await app.listen(3000);
}
```

When using `@apollo/server` (4.x) with Apollo Sandbox, default CSP may cause issues. Configure as follows:

**Express with Apollo:**

```typescript
app.use(helmet({
  crossOriginEmbedderPolicy: false,
  contentSecurityPolicy: {
    directives: {
      imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
      scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
      manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
      frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
    },
  },
}));
```

**Fastify with Apollo:**

```typescript
await app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: [`'self'`, 'unpkg.com'],
      styleSrc: [
        `'self'`,
        `'unsafe-inline'`,
        'cdn.jsdelivr.net',
        'fonts.googleapis.com',
        'unpkg.com',
      ],
      fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
      imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
      scriptSrc: [
        `'self'`,
        `https: 'unsafe-inline'`,
        'cdn.jsdelivr.net',
        `'unsafe-eval'`,
      ],
    },
  },
});

// Or disable CSP entirely if not needed:
await app.register(helmet, {
  contentSecurityPolicy: false,
});
```

Reference: [https://docs.nestjs.com/security/helmet](https://docs.nestjs.com/security/helmet)

---

## 2. Section 2

**Impact: HIGH**

### 2.1 Cache Frequently Used Data with Redis

**Impact: HIGH (Dramatically reduces database load and improves response times)**

Every database query increases latency and costs. Redis caching stores hot data in memory for sub-millisecond access. **Cache read-heavy endpoints and expensive computations.**

**Incorrect: database every request**

```typescript
// users.service.ts - DB hit every time 🚨
async getUserProfile(id: string) {
  return this.prisma.user.findUnique({
    where: { id },
    include: { posts: true, followers: true }
  });
}
```

**Correct: Redis cached**

```typescript
// cache.service.ts
@Injectable()
export class CacheService {
  constructor(private redis: Redis) {}
  
  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }
  
  async set(key: string, data: any, ttl = 300) {
    await this.redis.setex(key, ttl, JSON.stringify(data));
  }
}

// users.service.ts
async getUserProfile(id: string) {
  const cacheKey = `user:${id}:profile`;
  let profile = await this.cacheService.get(cacheKey);
  
  if (!profile) {
    profile = await this.prisma.user.findUnique({
      where: { id },
      include: { posts: true, followers: true }
    });
    await this.cacheService.set(cacheKey, profile, 300);  // 5min ✅
  }
  
  return profile;
}
```

---

## 3. Section 3

**Impact: HIGH**

### 3.1 Keep Functions Short and Single Purpose

**Impact: MEDIUM (Improves testability and reduces bugs)**

Long functions with multiple responsibilities are hard to test, debug, and maintain. Short functions (<20 lines) with one clear purpose are easier to understand. **Extract logic into focused helper functions.**

**Incorrect: god function**

```typescript
@Post()
async createUser(@Body() data: any) {
  // 3 responsibilities mixed 🚨
  const hashedPassword = await bcrypt.hash(data.password, 10);
  const user = await this.prisma.user.create({ data: {...data, password: hashedPassword} });
  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET);
  await this.sendWelcomeEmail(user.email);
  return { user, token };
}
```

**Correct: single responsibility**

```typescript
@Post()
async createUser(@Body() data: CreateUserDto) {
  const hashedPassword = await this.hashPassword(data.password);
  const user = await this.usersRepository.create({ ...data, password: hashedPassword });
  const token = this.generateToken(user.id);
  await this.emailService.sendWelcome(user.email);
  return { user, token };  // ✅ Orchestration only
}

private async hashPassword(password: string) {
  return bcrypt.hash(password, 10);
}
```

### 3.2 Organize Code by Feature Modules

**Impact: HIGH (Improves scalability and maintainability)**

Flat controller/service structure becomes unmaintainable at scale. NestJS modules enforce separation of concerns by feature. **One module per business domain.**

> **Hint**: Use feature modules to encapsulate related controllers, services, and providers. Each module should be independently testable and reusable across the application.

Services can be shared across multiple modules by exporting them:

Use `@Global()` for modules that should be available everywhere without importing:

> **Warning**: Use global modules sparingly. Only use for truly universal services like logging, configuration, or utilities.

Create configurable modules with `register()` or `forRoot()` patterns:

Modules must explicitly declare their dependencies:

Best practice structure for a feature module:

| Practice | Description | Why |

|----------|-------------|-----|

| One module per feature | Each business domain gets its own module | Clear separation of concerns |

| Export only what's needed | Only export services meant for external use | Prevents tight coupling |

| Use DTOs in each module | Keep data transfer objects with the module | Encapsulates validation logic |

| Avoid circular dependencies | Don't have Module A import Module B if B imports A | Prevents initialization issues |

| Use barrel exports | Export from `index.ts` for clean imports | Better developer experience |

Test modules in isolation:

**Sources:**

- [Modules | NestJS - Official Documentation](https://docs.nestjs.com/modules)

- [Dynamic Modules | NestJS](https://docs.nestjs.com/fundamentals/dynamic-modules)

- [Custom Providers | NestJS](https://docs.nestjs.com/providers#custom-providers)

- [EventEmitter2 | NestJS Recipe](https://docs.nestjs.com/techniques/events)

### 3.3 Remove Unused Code and Dependencies

**Impact: MEDIUM (Reduces bundle size and maintenance burden)**

Dead code bloats the project and confuses developers. Unused dependencies increase attack surface. Regular cleanup keeps codebase lean. **No unused imports, functions, or packages.**

**Incorrect: code rot**

```json
// package.json - Unused deps
{
  "dependencies": {
    "lodash": "^4.17.20",     // Never imported
    "moment": "^2.29.4",      // Replaced by date-fns
    "axios": "^1.0.0"         // Switched to fetch
  }
}
```

**Correct: clean codebase**

```json
// package.json - Clean ✅
{
  "scripts": {
    "cleanup": "bun pm prune && bun pm cache rm && bun run lint:fix",
    "depcheck": "depcheck && ts-unused-exports tsconfig.json"
  }
}
```

### 3.4 Single Responsibility - Separate Controller and Service

**Impact: HIGH (Makes testing and maintenance easier)**

Fat controllers mix HTTP concerns with business logic, making unit testing impossible. Controllers should only parse HTTP requests and delegate. **Controllers are thin, services are smart.**

> **Hint**: Controllers handle HTTP-specific concerns (validation, parsing, status codes). Services handle business logic (calculations, workflows, data transformations). This separation makes both layers independently testable.

| Controllers (HTTP Layer) | Services (Business Layer) |

|--------------------------|---------------------------|

| Parse request bodies | Execute business logic |

| Validate with DTOs | Perform calculations |

| Return HTTP status codes | Transform data |

| Handle routing | Manage transactions |

| Set headers/cookies | Call external APIs |

| Upload/download files | Enforce business rules |

**Problems:**

```typescript
┌─────────────────────────────────────────────────────┐
│                   Controller Layer                   │
│  - Parse HTTP requests                               │
│  - Validate with DTOs                                │
│  - Set status codes and headers                      │
│  - Delegate to services                              │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                    Service Layer                     │
│  - Execute business logic                            │
│  - Enforce business rules                            │
│  - Coordinate multiple repositories                  │
│  - Emit domain events                                │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                  Repository Layer                    │
│  - Data access                                       │
│  - Database queries                                  │
│  - Entity persistence                                │
└─────────────────────────────────────────────────────┘
```

- Cannot test business logic without HTTP context

- Cannot reuse business logic in other contexts (CLI, GraphQL, WebSocket)

- Difficult to mock dependencies for testing

- Changes to business logic require HTTP layer changes

Controllers SHOULD handle HTTP-specific details:

**Sources:**

- [Controllers | NestJS - Official Documentation](https://docs.nestjs.com/controllers)

- [Providers | NestJS - Official Documentation](https://docs.nestjs.com/providers)

- [Single Responsibility Principle | Wikipedia](https://en.wikipedia.org/wiki/Single-responsibility_principle)

- [Clean Architecture | Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

### 3.5 Use Consistent Naming Conventions

**Impact: MEDIUM (Improves code readability and maintainability)**

Inconsistent naming confuses developers and breaks tooling. NestJS follows specific conventions for maximum clarity. **Standardize all filenames, classes, and methods.**

**Incorrect: mixed conventions**

```typescript
userController.ts      // camelCase file
UserService.ts         // PascalCase file  
getuserById()          // mixed case method
Userservice            // missing suffix
```

**Correct: NestJS conventions**

```typescript
users.controller.ts    // kebab-case file
users.service.ts       // kebab-case file
UsersController        // PascalCase class
UsersService           // PascalCase class
findUserById()         // camelCase method ✅
getUserProfile()       // camelCase method ✅
```

---

## 4. Section 4

**Impact: CRITICAL**

### 4.1 Enable Global Exception Filter

**Impact: CRITICAL (Prevents stack trace leaks in production)**

Default Express errors leak stack traces and database info to clients. Global filters catch all exceptions and return safe HTTP responses. **Hide internal errors from users.**

> **Hint**: Use global exception filters to standardize error responses, log errors properly, and prevent sensitive information leaks. Different responses for development vs production environments.

**Without global filters:**

```json
// 🚨 Production response - LEAKS sensitive info!
{
  "statusCode": 500,
  "message": "select * from users where id = $1 - relation \"users\" does not exist",
  "stack": "Error: relation \"users\" does not exist\n    at Connection.parseE (/app/node_modules/pg/lib/connection.js:539:11)\n    at /app/src/users/users.service.ts:42:15\n    at processTicksAndRejections (internal/process/task_queues.js:95:5)"
}
```

**With global filters:**

```typescript
// common/filters/all-exceptions.filter.ts
import { LoggerService } from '../services/logger.service';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(
    private configService: ConfigService,
    private loggerService: LoggerService,
  ) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getResponse();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    // ✅ Log to external service (Sentry, DataDog, etc.)
    this.loggerService.logError({
      status,
      path: request.url,
      method: request.method,
      exception,
      userId: request.user?.id,
      correlationId: request.headers['x-correlation-id'],
    });

    const errorResponse = this.buildErrorResponse(exception, request);
    response.status(status).json(errorResponse);
  }
}
```

Create domain-specific exceptions for better error handling:

Handle HTTP exceptions specifically:

Handle database-specific errors safely:

Use multiple filters for different exception types:

Better approach using dependency injection:

Define a consistent error response structure:

Integrate with external logging services:

| Practice | Description |

|----------|-------------|

| Hide stack traces in production | Never expose internal implementation details |

| Log everything | Always log errors server-side for debugging |

| Use environment-aware responses | Show details in dev, hide in prod |

| Create custom exceptions | Use domain-specific exceptions for business logic |

| Standardize error format | Consistent response structure across all endpoints |

| Handle database errors | Map DB errors to safe HTTP responses |

| Use correlation IDs | Track requests through distributed systems |

**Sources:**

- [Exception Filters | NestJS - Official Documentation](https://docs.nestjs.com/exception-filters)

- [Built-in HTTP exceptions | NestJS](https://docs.nestjs.com/exception-filters#built-in-http-exceptions)

- [Exceptions | NestJS](https://docs.nestjs.com/throw-exceptions)

- [Error Handling Best Practices | OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html)

### 4.2 Implement Proper Logging Strategy

**Impact: HIGH (Enables debugging, monitoring, and audit trails)**

Console.log lacks structure, levels, and persistence. NestJS Logger with Winston provides structured JSON logs for production monitoring. **Log all errors, warnings, and business events.**

**Incorrect: console.log debugging**

```typescript
// users.service.ts - No structure 🚨
console.log('Creating user:', data);
try {
  const user = await this.prisma.user.create({ data });
  console.log('User created:', user.id);
} catch (error) {
  console.error('User creation failed:', error);
}
```

**Correct: structured logging**

```typescript
// logger.service.ts
import { LoggerService } from '@nestjs/common';
import * as winston from 'winston';

@Injectable()
export class WinstonLogger implements LoggerService {
  private logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
      winston.format.timestamp(),
      winston.format.json()
    ),
    transports: [
      new winston.transports.Console(),
      new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    ],
  });
}

// users.service.ts
import { Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async createUser(data: CreateUserDto) {
    this.logger.log(`Creating user for email: ${data.email}`, 'UsersService');
    
    try {
      const user = await this.prisma.user.create({ data });
      this.logger.log(`User created successfully: ${user.id}`);
      return user;
    } catch (error) {
      this.logger.error(`Failed to create user ${data.email}: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

---

## 5. Section 5

**Impact: CRITICAL**

### 5.1 Validate All Inputs with DTOs and ValidationPipe

**Impact: CRITICAL (Prevents invalid data and injection attacks)**

Unvalidated inputs lead to runtime errors, SQL injection, and security vulnerabilities. Global ValidationPipe ensures every DTO is validated before reaching controllers. **Never trust client data.**

> **Hint**: Apply ValidationPipe globally in `main.ts` to ensure all routes are protected. Use `transform: true` to automatically convert plain objects to DTO class instances.

When implementing or reviewing code, **always** follow these steps:

**File:** `src/main.ts`

**If missing:** Add the ValidationPipe setup before `app.listen()`.

**Check pattern:** Controllers should NOT use `any` type for request bodies.

**Check pattern:** All DTO properties must have decorators from `class-validator`.

**Check pattern:** Use `@Type()` for non-string types from query params.

Use this checklist when reviewing or creating endpoints:

- [ ] Global ValidationPipe configured in `main.ts`

- [ ] All controller methods use typed DTOs (not `any`)

- [ ] All DTO properties have validation decorators

- [ ] Query params use `@Type()` decorator for number/boolean/date

- [ ] Optional fields use `@IsOptional()`

- [ ] Enums use `@IsEnum()` decorator

- [ ] Arrays use `@IsArray()` decorator

- [ ] Nested DTOs use `@ValidateNested()` with `@Type()`

| Option | What It Does | Always Use? |

|--------|--------------|------------|

| `whitelist: true` | Removes properties not in DTO | ✅ Yes |

| `forbidNonWhitelisted: true` | Throws error if extra properties sent | ✅ Yes (security) |

| `transform: true` | Converts plain objects to DTO instances | ✅ Yes (type safety) |

| `disableErrorMessages: false` | Shows detailed validation errors | ✅ Yes (debugging) |

| `transformOptions.enableImplicitConversion: true` | Auto-converts strings to types | ✅ Yes (convenience) |

For business-specific validation rules:

**Sources:**

- [NestJS Validation Documentation](https://docs.nestjs.com/techniques/validation)

- [class-validator Documentation](https://github.com/typestack/class-validator)

- [class-transformer Documentation](https://github.com/typestack/class-transformer)

---

## 6. Section 6

**Impact: CRITICAL**

### 6.1 Use Parameterized Queries to Prevent SQL Injection

**Impact: CRITICAL (Eliminates SQL injection vulnerabilities)**

Raw SQL queries with string concatenation allow attackers to inject malicious code. Prisma Client automatically parameterizes all queries, preventing injection. **Never use string concatenation with user input.**

**Incorrect: SQL injection vulnerable**

```typescript
// Raw SQL concatenation - DANGEROUS 🚨
const userId = req.params.id;
const query = `SELECT * FROM users WHERE name = '${userId}'`; 
await prisma.$executeRaw(query);
```

**Correct: Prisma safe queries**

```typescript
// Prisma ORM - Safe ✅
const user = await prisma.user.findUnique({
  where: { id: userId }
});

// Prisma raw with parameters
const users = await prisma.$queryRaw`
  SELECT * FROM users 
  WHERE email = ${email} AND status = ${status}
`;

// Or with positional params
const results = await prisma.$queryRaw`
  SELECT * FROM users WHERE id = $1
`, userId;
```

---

## 7. Section 7

**Impact: CRITICAL**

### 7.1 Use Guards for Route Protection

**Impact: CRITICAL (Enforces authentication/authorization per route)**

Unprotected routes expose sensitive data. Guards run before controllers and can short-circuit requests. **Protect endpoints explicitly.**

> **Hint**: Guards determine whether a request will be handled by the controller or not. Use them for authentication (who are you?) and authorization (what can you do?). Always use global guards with public route decorators for default-deny security.

When implementing or reviewing security, **always** follow these steps:

**Files to create/modify:**

```typescript
// ✅ REQUIRED: Global guard registration
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,  // ✅ Global authentication
    },
  ],
})
export class AppModule {}
```

- `src/auth/guards/jwt-auth.guard.ts`

- `src/auth/guards/jwt-auth.guard.spec.ts`

- `src/auth/decorators/public.decorator.ts`

- `src/app.module.ts` (for APP_GUARD)

**File:** `src/auth/decorators/public.decorator.ts`

**File:** `src/app.module.ts`

**Pattern to check:**

```typescript
// First install: bun add @nestjs/throttler

// app.module.ts
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,      // 1 minute
      limit: 5,        // 5 requests per minute
    }]),
  ],
})
export class AppModule {}

// auth/auth.controller.ts
import { Throttle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle({ default: { limit: 5, ttl: 60000 } })  // ✅ Stricter for login
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }

  @Post('register')
  @Throttle({ default: { limit: 3, ttl: 3600000 } })  // ✅ 3 per hour
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }
}
```

Use this checklist when reviewing or creating endpoints:

- [ ] Global authentication guard registered in `app.module.ts`

- [ ] `@Public()` decorator exists for public routes

- [ ] Controllers don't use `any` type for `@Req()` user

- [ ] Admin routes have `@Roles('admin')` guard

- [ ] Resource owner checks implemented (users can only access their own data)

- [ ] Rate limiting applied to auth endpoints

- [ ] Guards use `Reflector` to check metadata

| Practice | Description |

|----------|-------------|

| Use global guards | Default-deny security with explicit public routes |

| Separate auth from authorization | Different guards for authentication vs authorization |

| Use decorators | Custom decorators improve code readability |

| Check resource ownership | Users can only access their own resources |

| Combine guards | Chain multiple guards for comprehensive security |

| Rate limit auth endpoints | Prevent brute force attacks |

**Sources:**

- [Guards | NestJS - Official Documentation](https://docs.nestjs.com/guards)

- [Authentication | NestJS - Official Documentation](https://docs.nestjs.com/security/authentication)

- [Authorization | NestJS - Official Documentation](https://docs.nestjs.com/security/authorization)

- [Throttler | NestJS Documentation](https://docs.nestjs.com/security/rate-limiting)

---

## 8. Section 8

**Impact: MEDIUM**

### 8.1 Generate Swagger/OpenAPI Documentation

**Impact: MEDIUM (Enables automatic API documentation and client generation)**

Undocumented APIs force clients to guess endpoints and payloads. NestJS Swagger module auto-generates interactive OpenAPI docs from controllers. **All public APIs must be self-documenting.**

**Incorrect: no documentation**

```typescript
// users.controller.ts - Clients must guess schema 🚨
@Controller('users')
export class UsersController {
  @Post()
  create(@Body() data: any) {
    return this.usersService.create(data);
  }
}
```

**Correct: auto-generated docs**

```typescript
// main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const config = new DocumentBuilder()
    .setTitle('Users API')
    .setDescription('User management endpoints')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
    
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);  // ✅ Interactive docs
  
  await app.listen(3000);
}

// users.controller.ts
@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UsersController {
  @Post()
  @ApiOperation({ summary: 'Create new user' })
  @ApiResponse({ status: 201, description: 'User created', type: UserDto })
  @ApiResponse({ status: 400, description: 'Invalid input' })
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

---

## 9. Section 9

**Impact: CRITICAL**

### 9.1 Never Hardcode Secrets - Use Environment Variables

**Impact: CRITICAL (Prevents credential leaks in source control)**

Hardcoded credentials in source code get committed to git and exposed publicly. The `@nestjs/config` package provides a secure way to manage configuration through environment variables. **Secrets belong in environment only.**

Use Joi or Zod to validate environment variables at startup:

Use `registerAs` for type-safe, grouped configuration:

Always provide a `.env.example` file to document required variables:

**Sources:**

- [Configuration | NestJS - Official Documentation](https://docs.nestjs.com/techniques/configuration)

- [Managing Environment Variables in NestJS with ConfigModule | Medium](https://medium.com/@hashirmughal1000/managing-environment-variables-in-nestjs-with-configmodule-5b0742efb69c)

- [Managing Configuration and Environment Variables in NestJS | dev.to](https://dev.to/vishnucprasad/managing-configuration-and-environment-variables-in-nestjs-28ni)

- [NestJS Environment Configuration Using Zod | Medium](https://medium.com/@rotemdoar17/nestjs-environment-configuration-using-zod-92e3decca5ca)

---

## 10. Section 10

**Impact: MEDIUM**

### 10.1 Write Comprehensive Unit Tests

**Impact: MEDIUM (Catches bugs early and enables safe refactoring)**

Untested code leads to regressions and production bugs. NestJS + Jest provides full testing support for controllers, services, and repositories. **Aim for 80%+ coverage on business logic.**

**Incorrect: no tests**

```typescript
// users.service.ts - No tests 🚨
@Injectable()
export class UsersService {
  async createUser(data: CreateUserDto) {
    return this.prisma.user.create({ data });
  }
}
```

**Correct: full test suite**

```typescript
// users.service.spec.ts
describe('UsersService', () => {
  let service: UsersService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [UsersService, { provide: PrismaService, useValue: mockPrisma }],
    }).compile();
    service = module.get(UsersService);
  });

  it('should create user successfully', async () => {
    const dto: CreateUserDto = { email: 'test@example.com', name: 'Test' };
    const mockUser = { id: '1', ...dto };

    jest.spyOn(prisma.user, 'create').mockResolvedValue(mockUser);

    const result = await service.createUser(dto);
    expect(result).toEqual(mockUser);
    expect(prisma.user.create).toHaveBeenCalledWith(expect.any(Object));
  });
});
```

---

## 11. Section 11

**Impact: HIGH**

### 11.1 Implement Health Check Endpoints

**Impact: HIGH (Enables monitoring and automatic restarts)**

No health checks prevent container orchestrators from detecting failures. Dedicated endpoints verify database, cache, and service connectivity. **Production apps must expose health status.**

**Incorrect: no monitoring**

```typescript
// No health endpoint - can't detect failures 🚨
```

**Correct: comprehensive health checks**

```typescript
// health.controller.ts
@Controller('health')
export class HealthController {
  constructor(
    @Inject(PrismaService) private prisma: PrismaService,
    @Inject(CacheService) private cache: CacheService,
  ) {}

  @Get()
  async checkHealth() {
    // Database check
    await this.prisma.$queryRaw`SELECT 1`;
    
    // Cache check
    await this.cache.get('health-check');
    
    return {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: { database: 'ok', cache: 'ok' }
    };
  }
}

// main.ts
app.setGlobalPrefix('api');
app.use('/api/health', healthRouter);
```

---

## 12. Section 12

**Impact: HIGH**

### 12.1 Enable Compression Middleware for Responses

**Impact: HIGH (Reduces bandwidth usage and improves performance)**

Uncompressed responses waste bandwidth and slow down page loads. Compression middleware (gzip/brotli) reduces response size by 70-90%. **Always enable compression in production.**

**Incorrect: no compression**

```typescript
// main.ts - Large responses 🚨
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
```

**Correct: compressed responses**

```typescript
// main.ts
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(compression());  // Gzip/brotli compression ✅
  await app.listen(3000);
}
```

Reference: [https://docs.nestjs.com/techniques/compression](https://docs.nestjs.com/techniques/compression)

### 12.2 Implement Rate Limiting for All Routes

**Impact: CRITICAL (Prevents DDoS and brute force attacks)**

Unlimited requests per IP enable brute force attacks and DDoS. Global rate limiter throttles excessive requests. **Protect every endpoint from abuse.**

**Incorrect: no protection**

```typescript
// main.ts - Unlimited requests 🚨
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
```

**Correct: rate limited**

```typescript
// main.ts
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.useGlobalGuards(new ThrottlerGuard());
  
  await app.listen(3000);
}

// app.module.ts
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60,        // 60 seconds
      limit: 10,      // 10 requests per IP
    }]),
  ],
})
```

---

## 13. Section 13

**Impact: HIGH**

### 13.1 Lazy Load Non-Critical Modules

**Impact: HIGH (Reduces startup time and memory usage)**

All modules load at startup, wasting memory on unused features. Lazy loading defers module initialization until the first request to that module. **Load only what's needed, when it's needed.**

> **Hint**: Use lazy loading for administrative panels, analytics dashboards, reporting features, and any route that isn't accessed immediately on application startup. Core features like auth and public APIs should remain eagerly loaded.

Without lazy loading:

With lazy loading:

**Problems:**

```bash
# Application startup
[AppModule] initialized
[UsersModule] initialized

Startup time: 0.5s
Memory usage: 100MB

# After first admin request
[AdminModule] loaded
Memory usage: 150MB (+50MB)

# After first analytics request
[AnalyticsModule] loaded
Memory usage: 230MB (+80MB)
```

- Increased startup time (all modules initialize immediately)

- Higher memory usage (all providers and services loaded)

- Slower cold starts (affects serverless deployments)

- Unused code stays in memory

NestJS supports lazy loading through the `@LazyModuleDecorator` and dynamic imports:

For NestJS 10+, use the built-in `ModuleLoader`:

A more practical approach is lazy loading through routing:

Create a reusable lazy loading decorator:

Using Fastify's plugin system for lazy loading:

Configure which modules to lazy load:

❌ **Don't lazy load:**

- Authentication/Authorization modules (needed immediately)

- Public API routes (accessed by most users)

- Core business logic (used throughout the app)

- Frequently accessed features (would cause repeated loading)

✅ **DO lazy load:**

- Admin panels (accessed by few users, infrequently)

- Analytics dashboards (not accessed on every request)

- Report generators (heavy, infrequently used)

- Integration modules (external service integrations)

- Background job processors (only run periodically)

| Practice | Description |

|----------|-------------|

| Measure before optimizing | Profile startup time and memory usage first |

| Lazy load infrequently used modules | Admin, analytics, reports, integrations |

| Keep critical modules eager | Auth, core API, public routes |

| Preload in development | Faster hot module replacement during development |

| Monitor memory usage | Track actual memory savings in production |

| Consider serverless | Lazy loading is crucial for cold start optimization |

**Sources:**

- [Modules | NestJS - Official Documentation](https://docs.nestjs.com/modules)

- [Dynamic Modules | NestJS](https://docs.nestjs.com/fundamentals/dynamic-modules)

- [Performance Optimization | NestJS](https://docs.nestjs.com/techniques/performance)

- [Serverless NestJS | NestJS Blog](https://trilon.io/blog/serverless-nestjs)

---

## References

1. [https://docs.nestjs.com/](https://docs.nestjs.com/)
2. [https://expressjs.com/](https://expressjs.com/)

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

0. [Section 0](#0-section-0) — **CRITICAL**
   - 0.1 [Never Hardcode Secrets - Use Environment Variables](#01-never-hardcode-secrets---use-environment-variables)
   - 0.2 [Organize Code by Feature Modules](#02-organize-code-by-feature-modules)
   - 0.3 [Use Helmet Middleware for Security Headers](#03-use-helmet-middleware-for-security-headers)
   - 0.4 [Validate All Inputs with DTOs and ValidationPipe](#04-validate-all-inputs-with-dtos-and-validationpipe)

---

## 0. Section 0

**Impact: CRITICAL**

### 0.1 Never Hardcode Secrets - Use Environment Variables

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

### 0.2 Organize Code by Feature Modules

**Impact: HIGH (Improves scalability and maintainability)**

Flat controller/service structure becomes unmaintainable at scale. NestJS modules enforce separation of concerns by feature. **One module per business domain.**

> **Hint**: Use feature modules to encapsulate related controllers, services, and providers. Each module should be independently testable and reusable across the application.

Services can be shared across multiple modules by exporting them:

Use `@Global()` for modules that should be available everywhere without importing:

> **Warning**: Use global modules sparingly. Only use for truly universal services like logging, configuration, or utilities.

Create configurable modules with `register()` or `forRoot()` patterns:

Modules must explicitly declare their dependencies:

Best practice structure for a feature module:

For better performance in large applications:

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

### 0.3 Use Helmet Middleware for Security Headers

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

### 0.4 Validate All Inputs with DTOs and ValidationPipe

**Impact: CRITICAL (Prevents invalid data and injection attacks)**

Unvalidated inputs lead to runtime errors, SQL injection, and security vulnerabilities. Global ValidationPipe ensures every DTO is validated before reaching controllers. **Never trust client data.**

> **Hint**: Apply ValidationPipe globally in `main.ts` to ensure all routes are protected. Use `transform: true` to automatically convert plain objects to DTO class instances.

Install required packages for validation:

**Incorrect: no validation**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

// users.controller.ts
@Post()
create(@Body() createUserDto: any) {
  // No validation - vulnerable to injection attacks!
  return this.usersService.create(createUserDto);
}
```

**Correct: full validation**

```json
{
  "statusCode": 400,
  "message": [
    "email must be an email",
    "password must be longer than or equal to 8 characters"
  ],
  "error": "Bad Request"
}
```

| Option | Description | Recommended |

|--------|-------------|-------------|

| `whitelist: true` | Removes properties not defined in DTO | ✅ Always enable |

| `forbidNonWhitelisted: true` | Throws error if extra properties sent | ✅ Enable for security |

| `transform: true` | Converts plain objects to DTO instances | ✅ Enable for type safety |

| `disableErrorMessages: false` | Shows detailed validation errors | ✅ Enable for debugging |

Create custom validators for complex validation logic:

For specific validation requirements per route:

When validation fails, NestJS automatically returns:

- [NestJS Validation Documentation](https://docs.nestjs.com/techniques/validation)

- [class-validator Documentation](https://github.com/typestack/class-validator)

- [class-transformer Documentation](https://github.com/typestack/class-transformer)

---

## References

1. [https://docs.nestjs.com/](https://docs.nestjs.com/)
2. [https://expressjs.com/](https://expressjs.com/)

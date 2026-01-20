# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### nestjs-best-practices

NestJS best practices and patterns for building scalable, maintainable backend applications. Contains 26 rules across 13 categories covering security, architecture, performance, validation, database operations, authentication, and advanced patterns.

**Use when:**
- Writing new NestJS modules, controllers, or services
- Implementing authentication and authorization
- Setting up Prisma database operations
- Creating DTOs and validation pipes
- Adding middleware or guards
- Implementing error handling and logging
- Reviewing NestJS code for best practices
- Refactoring existing NestJS applications

**Categories covered:**
- Security (CRITICAL) - CORS, dependency audits, security headers
- Performance (HIGH) - Redis caching, compression
- Architecture (HIGH) - Feature modules, naming conventions, event-driven architecture
- Error Handling (HIGH) - Exception filters, structured logging
- Validation (CRITICAL) - DTOs, ValidationPipe, custom pipes
- Database (CRITICAL) - Parameterized queries, Prisma v7 patterns
- Authentication (CRITICAL) - Guards, password hashing (argon2/bcrypt)
- API (MEDIUM) - Swagger, cursor pagination
- Configuration (CRITICAL) - Environment variables, secrets management
- Testing (MEDIUM) - Unit tests, test coverage
- Deployment (MEDIUM) - Health checks, graceful shutdown
- Middleware (MEDIUM) - Rate limiting, compression
- Advanced (HIGH) - Scheduled tasks, lazy loading

**Technology stack:**
- NestJS with Express/Fastify adapter
- Bun runtime (native crypto, scheduling)
- Prisma v7 (sql template tag, interactive transactions)
- EventEmitter2 for event-driven architecture

## Installation

```bash
npx add-skill xirothedev/agent-skills
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**
```
Review my NestJS service for security issues
```
```
Create authentication guards for my NestJS app
```
```
Help me implement cursor pagination
```

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for the agent
- `rules/` - Individual rule files with agent-friendly format
- `AGENTS.md` - Compiled documentation (generated)
- `README.md` - Skill-specific documentation
- `LICENSE` - License information

## License

MIT

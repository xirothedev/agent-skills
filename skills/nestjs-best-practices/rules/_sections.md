# Sections

This file defines all sections, their ordering, impact levels, and descriptions.
The section ID (in parentheses) is the filename prefix used to group rules.

---

## 1. Security (security)

**Impact:** CRITICAL
**Description:** Security vulnerabilities can lead to data breaches, unauthorized access, and service disruption. Always prioritize security in production.

## 2. Performance & Optimization (performance)

**Impact:** CRITICAL
**Description:** Server performance directly affects user experience and infrastructure costs. Eliminating waterfalls and optimizing queries yields the largest gains.

## 3. Architecture & Code Organization (architecture)

**Impact:** HIGH
**Description:** Proper module structure, dependency injection patterns, and separation of concerns ensure maintainable and scalable applications.

## 4. Error Handling & Logging (error-handling)

**Impact:** HIGH
**Description:** Comprehensive error handling and logging enable debugging, monitoring, and graceful failure recovery in production environments.

## 5. Validation & Data Transfer (validation)

**Impact:** MEDIUM-HIGH
**Description:** Input validation prevents invalid data, reduces attack surface, and provides clear error messages to API consumers.

## 6. Database & ORM (database)

**Impact:** MEDIUM-HIGH
**Description:** Efficient database queries, proper indexing, and transaction management prevent performance bottlenecks and data integrity issues.

## 7. Authentication & Authorization (auth)

**Impact:** MEDIUM-HIGH
**Description:** Proper authentication and authorization ensure only authorized users can access protected resources and perform allowed actions.

## 8. API Design (api)

**Impact:** MEDIUM
**Description:** Well-designed APIs with consistent patterns, proper versioning, and clear documentation improve developer experience and client integration.

## 9. Configuration & Environment (config)

**Impact:** MEDIUM
**Description:** Proper configuration management prevents secrets leakage, enables environment-specific settings, and simplifies deployment across environments.

## 10. Testing (testing)

**Impact:** MEDIUM
**Description:** Comprehensive test coverage ensures code quality, prevents regressions, and documents expected behavior through executable specifications.

## 11. Deployment & Production (deployment)

**Impact:** MEDIUM
**Description:** Production readiness includes health checks, graceful shutdown, logging, monitoring, and scaling considerations.

## 12. Middleware & Interceptors (middleware)

**Impact:** LOW-MEDIUM
**Description:** Custom middleware and interceptors enable cross-cutting concerns like logging, caching, and transformation across the application.

## 13. Advanced Patterns (advanced)

**Impact:** LOW
**Description:** Advanced patterns for specific use cases that require careful implementation and deep understanding of NestJS internals.
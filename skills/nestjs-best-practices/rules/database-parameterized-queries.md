---
title: Use Parameterized Queries to Prevent SQL Injection
impact: CRITICAL
section: 6
impactDescription: Eliminates SQL injection vulnerabilities
tags: security, database, sql-injection, prisma
---

## Use Parameterized Queries to Prevent SQL Injection

Raw SQL queries with string concatenation allow attackers to inject malicious code. Prisma Client automatically parameterizes all queries, preventing injection. **Never use string concatenation with user input.**

**Incorrect (SQL injection vulnerable):**

```typescript
// Raw SQL concatenation - DANGEROUS 🚨
const userId = req.params.id;
const query = `SELECT * FROM users WHERE name = '${userId}'`; 
await prisma.$executeRaw(query);
```

**Correct (Prisma safe queries):**

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
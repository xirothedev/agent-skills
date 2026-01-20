---
title: Validate All Inputs with DTOs and ValidationPipe
impact: CRITICAL
impactDescription: Prevents invalid data and injection attacks
tags: validation, security, dto, class-validator
---

## Validate All Inputs with DTOs and ValidationPipe

Unvalidated inputs lead to runtime errors, SQL injection, and security vulnerabilities. Global ValidationPipe ensures every DTO is validated before reaching controllers. **Never trust client data.**

> **Hint**: Apply ValidationPipe globally in `main.ts` to ensure all routes are protected. Use `transform: true` to automatically convert plain objects to DTO class instances.

### Installation

Install required packages for validation:

```bash
bun add class-validator class-transformer
```

### Setup Global ValidationPipe

**Incorrect (no validation):**

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

**Correct (full validation):**

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,              // Strip properties not in DTO
    forbidNonWhitelisted: true,   // Throw error if extra properties
    transform: true,              // Convert plain objects to DTO instances
    transformOptions: {
      enableImplicitConversion: true,
    },
    disableErrorMessages: false,  // Show detailed validation errors
  }));

  await app.listen(3000);
}

// dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional, IsInt } from 'class-validator';

export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  password: string;

  @IsOptional()
  @IsString()
  @MinLength(2)
  name?: string;

  @IsOptional()
  @IsInt()
  age?: number;
}

// users.controller.ts
@Post()
create(@Body() createUserDto: CreateUserDto) {
  // DTO is automatically validated before reaching here
  return this.usersService.create(createUserDto);
}
```

### ValidationPipe Options Explained

| Option | Description | Recommended |
|--------|-------------|-------------|
| `whitelist: true` | Removes properties not defined in DTO | ✅ Always enable |
| `forbidNonWhitelisted: true` | Throws error if extra properties sent | ✅ Enable for security |
| `transform: true` | Converts plain objects to DTO instances | ✅ Enable for type safety |
| `disableErrorMessages: false` | Shows detailed validation errors | ✅ Enable for debugging |

### Common Validation Decorators

```typescript
import {
  IsEmail, IsString, IsInt, IsBoolean, IsDate,
  Min, Max, MinLength, MaxLength, IsOptional,
  Matches, IsEnum, IsArray, ArrayNotEmpty
} from 'class-validator';

export class ExampleDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(3)
  @MaxLength(20)
  username: string;

  @IsInt()
  @Min(18)
  @Max(120)
  age: number;

  @IsOptional()
  @IsEnum(['admin', 'user', 'guest'])
  role?: string;

  @IsOptional()
  @IsArray()
  @ArrayNotEmpty()
  tags?: string[];

  @IsOptional()
  @Matches(/^[A-Z]{2}[0-9]{6}$/)
  passportNumber?: string;
}
```

### Custom Validation

Create custom validators for complex validation logic:

```typescript
// decorators/is-strong-password.decorator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
} from 'class-validator';

@ValidatorConstraint({ name: 'isStrongPassword', async: false })
export class IsStrongPasswordConstraint implements ValidatorConstraintInterface {
  validate(password: string) {
    // At least 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char
    const strongPasswordRegex = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;
    return strongPasswordRegex.test(password);
  }

  defaultMessage() {
    return 'Password must contain at least 8 characters, including uppercase, lowercase, number, and special character';
  }
}

export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsStrongPasswordConstraint,
    });
  };
}

// dto/create-user.dto.ts
export class CreateUserDto {
  @IsStrongPassword()
  password: string;
}
```

### Per-Route Validation

For specific validation requirements per route:

```typescript
// users.controller.ts
import { UsePipes } from '@nestjs/common';
import { ValidationPipe } from '@nestjs/common';

@Post()
@UsePipes(new ValidationPipe({ whitelist: true }))
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

### Validation Error Response

When validation fails, NestJS automatically returns:

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

### References

- [NestJS Validation Documentation](https://docs.nestjs.com/techniques/validation)
- [class-validator Documentation](https://github.com/typestack/class-validator)
- [class-transformer Documentation](https://github.com/typestack/class-transformer)
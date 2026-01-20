---
title: Use Guards for Route Protection
impact: CRITICAL
impactDescription: Enforces authentication/authorization per route
tags: security, guards, auth, authorization
---

## Use Guards for Route Protection

Unprotected routes expose sensitive data. Guards run before controllers and can short-circuit requests. **Protect endpoints explicitly.**

> **Hint**: Guards determine whether a request will be handled by the controller or not. Use them for authentication (who are you?) and authorization (what can you do?). Always use global guards with public route decorators for default-deny security.

## Guard Execution Order

Guards execute **after** middleware but **before** interceptors and pipes:

```
Request → Middleware → Guards → Interceptors → Pipes → Controller
                              ↑
                         Short-circuit here
```

## Incorrect (No Protection)

```typescript
@Controller('users')
export class UsersController {
  @Get()
  findAll() {
    return this.usersService.findAll();  // 🚨 Publicly accessible!
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);  // 🚨 Anyone can view any user!
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);  // 🚨 Anyone can delete users!
  }
}
```

## Correct (Protected Routes)

### Basic Guard Usage

```typescript
// auth/jwt-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      return false;
    }

    try {
      const payload = this.jwtService.verify(token);
      request['user'] = payload;  // Attach user to request
    } catch {
      return false;
    }

    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}

// users/users.controller.ts
@Controller('users')
export class UsersController {
  @Get()
  @UseGuards(JwtAuthGuard)  // ✅ Requires authentication
  findAll(@Req() request: Request) {
    return this.usersService.findAll();
  }

  @Get('profile')
  @UseGuards(JwtAuthGuard)  // ✅ Requires authentication
  getProfile(@Req() request: Request) {
    return request['user'];  // Access authenticated user
  }
}
```

### Role-Based Authorization

```typescript
// auth/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>(
      'roles',
      context.getHandler(),
    );

    if (!requiredRoles) {
      return true;  // No roles required
    }

    const { user } = context.switchToHttp().getRequest();

    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}

// auth/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// users/users.controller.ts
@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)  // ✅ Apply to entire controller
export class UsersController {
  @Get()
  @Roles('user', 'admin')  // ✅ Accessible by users and admins
  findAll() {
    return this.usersService.findAll();
  }

  @Get('admin')
  @Roles('admin')  // ✅ Admins only
  getAdminData() {
    return this.usersService.getAdminData();
  }

  @Delete(':id')
  @Roles('admin')  // ✅ Only admins can delete
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

### Permission-Based Authorization

```typescript
// auth/permissions.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const REQUIRE_PERMISSIONS_KEY = 'requirePermissions';
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata(REQUIRE_PERMISSIONS_KEY, permissions);

// auth/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.get<string[]>(
      REQUIRE_PERMISSIONS_KEY,
      context.getHandler(),
    );

    if (!requiredPermissions) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    return requiredPermissions.every((permission) =>
      user.permissions?.includes(permission),
    );
  }
}

// products/products.controller.ts
@Controller('products')
@UseGuards(JwtAuthGuard, PermissionsGuard)
export class ProductsController {
  @Get()
  findAll() {
    return this.productsService.findAll();  // ✅ No specific permission needed
  }

  @Post()
  @RequirePermissions('product:create')  // ✅ Requires create permission
  create(@Body() createProductDto: CreateProductDto) {
    return this.productsService.create(createProductDto);
  }

  @Delete(':id')
  @RequirePermissions('product:delete')  // ✅ Requires delete permission
  remove(@Param('id') id: string) {
    return this.productsService.remove(id);
  }
}
```

## Global Guards with Public Routes

Best practice: Use global guards with explicit public decorators:

```typescript
// auth/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// auth/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(
      IS_PUBLIC_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (isPublic) {
      return true;  // ✅ Skip authentication for public routes
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      return false;
    }

    try {
      const payload = this.jwtService.verify(token);
      request['user'] = payload;
    } catch {
      return false;
    }

    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}

// app.module.ts
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

// auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  @Post('login')
  @Public()  // ✅ Explicitly mark as public
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  @Post('register')
  @Public()  // ✅ Explicitly mark as public
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }
}
```

## Resource Owner Guard

Protect routes so users can only access their own resources:

```typescript
// auth/owner.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';

@Injectable()
export class OwnerGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const { user, params } = context.switchToHttp().getRequest();

    // Get the resource ID parameter name from metadata
    const resourceIdParam = this.reflector.get<string>(
      'resourceIdParam',
      context.getHandler(),
    ) || 'id';

    const resourceUserId = params[resourceIdParam];

    if (user.id !== resourceUserId && !user.roles?.includes('admin')) {
      throw new ForbiddenException(
        'You can only access your own resources',
      );
    }

    return true;
  }
}

// auth/owner.decorator.ts
export const Owner = (resourceIdParam: string = 'id') =>
  SetMetadata('resourceIdParam', resourceIdParam);

// orders/orders.controller.ts
@Controller('orders')
@UseGuards(JwtAuthGuard)
export class OrdersController {
  @Get('my')
  findMyOrders(@Req() request: Request) {
    return this.ordersService.findByUserId(request['user'].id);
  }

  @Get(':id')
  @UseGuards(OwnerGuard)  // ✅ User can only view their own orders
  @Owner('id')  // ✅ Resource ID is in :id parameter
  findOne(@Param('id') id: string) {
    return this.ordersService.findOne(id);
  }

  @Patch(':id')
  @UseGuards(OwnerGuard)  // ✅ User can only update their own orders
  @Owner('id')
  update(@Param('id') id: string, @Body() updateOrderDto: UpdateOrderDto) {
    return this.ordersService.update(id, updateOrderDto);
  }
}
```

## API Key Guard

For service-to-service authentication:

```typescript
// auth/api-key.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class ApiKeyGuard implements CanActivate {
  constructor(private configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];

    const validApiKeys = this.configService.get<string>('API_KEYS')?.split(',') || [];

    if (!validApiKeys.includes(apiKey)) {
      throw new UnauthorizedException('Invalid API key');
    }

    return true;
  }
}

// webhook/webhook.controller.ts
@Controller('webhook')
@UseGuards(ApiKeyGuard)  // ✅ Requires valid API key
export class WebhookController {
  @Post('stripe')
  handleStripeWebhook(@Body() payload: any) {
    return this.webhookService.handleStripeEvent(payload);
  }
}
```

## Throttler Guard (Rate Limiting)

Prevent brute force and abuse:

```typescript
// First install: bun add @nestjs/throttler

// app.module.ts
import { ThrottlerModule } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000,      // Time window in milliseconds
        limit: 10,       // Max requests per window
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,  // ✅ Global rate limiting
    },
  ],
})
export class AppModule {}

// Custom limits per route
import { Throttle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle({ default: { limit: 5, ttl: 60000 } })  // ✅ Stricter limit for login
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  @Post('register')
  @Throttle({ default: { limit: 3, ttl: 3600000 } })  // ✅ 3 registrations per hour
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }
}
```

## Guard Composition

Combine multiple guards:

```typescript
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)  // ✅ Applied to all routes
export class AdminController {
  @Get()
  @Roles('admin')  // ✅ Must be authenticated AND have admin role
  getDashboard() {
    return this.adminService.getDashboard();
  }

  @Get('users')
  @Roles('admin')
  @RequirePermissions('user:read')  // ✅ Add permission check
  getUsers() {
    return this.adminService.getUsers();
  }

  @Delete('users/:id')
  @Roles('admin')
  @RequirePermissions('user:delete')
  deleteUser(@Param('id') id: string) {
    return this.adminService.deleteUser(id);
  }
}
```

## Custom Guard with Arguments

```typescript
// auth/max-age.guard.ts
export const MAX_AGE_KEY = 'maxAge';

export const MaxAge = (days: number) => SetMetadata(MAX_AGE_KEY, days);

@Injectable()
export class MaxAgeGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const maxAgeDays = this.reflector.get<number>(
      MAX_AGE_KEY,
      context.getHandler(),
    );

    if (!maxAgeDays) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    const accountAge = Date.now() - new Date(user.createdAt).getTime();
    const maxAgeMs = maxAgeDays * 24 * 60 * 60 * 1000;

    if (accountAge < maxAgeMs) {
      throw new ForbiddenException(
        `Account must be at least ${maxAgeDays} days old`,
      );
    }

    return true;
  }
}

// features/premium.controller.ts
@Controller('premium')
@UseGuards(JwtAuthGuard, MaxAgeGuard)
export class PremiumController {
  @Get('trading')
  @MaxAge(30)  // ✅ Account must be 30 days old
  getTradingData() {
    return this.premiumService.getTradingData();
  }
}
```

## Async Guards

Guards can be async for database checks:

```typescript
// auth/banned.guard.ts
@Injectable()
export class BannedGuard implements CanActivate {
  constructor(private usersService: UsersService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const { user } = context.switchToHttp().getRequest();

    const userWithBanStatus = await this.usersService.findOne(user.id);

    if (userWithBanStatus.isBanned) {
      throw new ForbiddenException(
        `Account banned until: ${userWithBanStatus.bannedUntil}`,
      );
    }

    return true;
  }
}
```

## Summary: Guard Best Practices

| Practice | Description |
|----------|-------------|
| Use global guards | Default-deny security with explicit public routes |
| Separate auth from authorization | Use different guards for authentication and authorization |
| Use decorators | Custom decorators improve code readability |
| Combine guards | Chain multiple guards for comprehensive security |
| Return boolean or throw | Guards can return `false` or throw exceptions |
| Keep guards focused | Single responsibility per guard |
| Consider performance | Cache user data to avoid repeated database queries |

**Sources:**
- [Guards | NestJS - Official Documentation](https://docs.nestjs.com/guards)
- [Authentication | NestJS - Official Documentation](https://docs.nestjs.com/security/authentication)
- [Authorization | NestJS - Official Documentation](https://docs.nestjs.com/security/authorization)
- [Role-based Authentication | NestJS Guide](https://docs.nestjs.com/techniques/security)
- [Throttler | NestJS Throttler Documentation](https://docs.nestjs.com/security/rate-limiting)

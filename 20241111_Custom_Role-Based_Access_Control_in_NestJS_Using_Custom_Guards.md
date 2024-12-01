In any application handling multiple roles and varying levels of access, implementing Role-Based Access Control (RBAC) becomes crucial. RBAC restricts access to resources based on a user's role, helping ensure that only authorized users can perform specific actions. In this article, we'll explore how to set up a custom role-based access guard in NestJS to secure our endpoints effectively.

### Setting Up the Environment
To implement role-based control in our NestJS application, we’ll use a custom guard. Here’s the overview of the key components involved:

- **Custom Decorator**: Specifies roles required to access specific endpoints.
- **Role Enum**: Defines the roles used in the application.
- **Guard**: Checks whether a user has the required role(s).
- **NestJS Modules and Controller Endpoints**: Protects routes with required roles and makes them accessible only to authorized users.

## Step 1: Defining the Role Enum

First, let's define the roles in an enum. This allows us to easily manage roles in one place and refer to them consistently across the app.

```typescript
export enum ApplicationUserRoleEnum {
  ADMIN = 'ADMIN',
  OWNER = 'OWNER',
  USER = 'USER',
}
```

## Step 2: Creating a Custom Role Decorator

We’ll create a custom `@RequiredRoles()` decorator to specify required roles on a per-route basis.

```typescript
import { SetMetadata } from '@nestjs/common';
import { ApplicationUserRoleEnum } from '../enum/application-user-role.enum';

export const RequiredRoles = (...roles: ApplicationUserRoleEnum[]) => SetMetadata('roles', roles);
```

## Step 3: Implementing the Roles Guard

Now, let’s build the main component: the `ApplicationUserRolesGuard`. This guard checks if the user’s role matches any of the roles defined by `@RequiredRoles()` for the specific endpoint.

```typescript
import {
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  Injectable,
  Logger,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Observable } from 'rxjs';
import { RequiredRoles } from '../decorator/roles.decorator';
import { ApplicationUserRoleEnum } from '../enum/application-user-role.enum';

@Injectable()
export class ApplicationUserRolesGuard implements CanActivate {
  private readonly logger = new Logger(ApplicationUserRolesGuard.name);

  constructor(private reflector: Reflector) {}

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const roles = this.reflector.get<ApplicationUserRoleEnum[]>(
      RequiredRoles,
      context.getHandler(),
    );

    if (!roles || roles.length === 0) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (
      !user ||
      !roles.some((role) => user.role === role)
    ) {
      this.logger.error(`Unauthorized role: required roles are ${roles.join(', ')}`);
      throw new ForbiddenException('You do not have access to perform this action');
    }

    return true;
  }
}
```

This guard retrieves the roles required for the endpoint from the `Reflector`, and compares them with the user's role. If the user’s role isn’t one of the required roles, an exception is thrown.

## Step 4: Applying the Guard Globally

To apply the guard across the application, we define it as a provider in our module configuration.

```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { ApplicationUserRolesGuard } from './guards/application-user-roles.guard';
import { ApplicationUserController } from './controllers/application-user.controller';
import { ApplicationUserService } from './services/application-user.service';

@Module({
  controllers: [ApplicationUserController],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ApplicationUserRolesGuard,
    },
    ApplicationUserService,
  ],
})
export class ApplicationUserModule {}
```

## Step 5: Protecting Routes with Role Requirements

Now, we can use the `@RequiredRoles()` decorator on our route handlers to specify role requirements.

```typescript
import { Controller, Get, Patch, Param, Delete, Body } from '@nestjs/common';
import { RequiredRoles } from '../decorator/roles.decorator';
import { ApplicationUserRoleEnum } from '../enum/application-user-role.enum';
import { UpdateUserDto } from '../dto/update-user.dto';

@Controller('users')
export class ApplicationUserController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `User ${id} details`;
  }

  @Patch(':id')
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN, ApplicationUserRoleEnum.OWNER)
  update(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return `Updated user ${id} with data ${JSON.stringify(updateUserDto)}`;
  }

  @Delete(':id')
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN)
  remove(@Param('id') id: string) {
    return `Deleted user ${id}`;
  }
}
```

Each endpoint is now restricted based on roles. For example:

- The **findOne()** route is publicly accessible.
- The **update()** route requires either **ADMIN** or **OWNER** role.
- The **remove()** route requires an **ADMIN** role.

## Wrapping Up
Implementing RBAC in NestJS using a custom guard helps to control access to resources securely and efficiently. By creating a custom guard, defining role requirements, and securing endpoints, we have established a scalable, role-based access control setup that’s flexible and maintainable.

Using this approach, your application can now handle authorization more effectively, limiting user actions based on their role and ensuring sensitive routes are only accessible to authorized roles. Happy coding!

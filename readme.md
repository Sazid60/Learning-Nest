# Comprehensive NestJS Learning Guide for Express Developers
## Built on a Miniature Todo App using NestJS, Prisma, and PostgreSQL

This guide maps the architecture and features of the Express.js-based **Eventra-Backend** to **NestJS**. It is written step-by-step, explaining concepts line-by-line as if you have no prior NestJS knowledge, while referencing your existing Express expertise.

---

## Table of Contents
1. [The Paradigm Shift: From Express to NestJS](#1-the-paradigm-shift-from-express-to-nestjs)
2. [Project Installation & Initialization](#2-project-installation--initialization)
3. [Architecture Mapping: Eventra (Express) vs. NestJS](#3-architecture-mapping-eventra-express-vs-nestjs)
4. [Step-by-Step Construction of the Miniature Todo App](#4-step-by-step-construction-of-the-miniature-todo-app)
   - [4.1 Database Design & Prisma Setup](#41-database-design--prisma-setup)
   - [4.2 Prisma Module & Service (Global Database Connection)](#42-prisma-module--service-global-database-connection)
   - [4.3 Authentication Module (JWT, Cookies, Roles)](#43-auth-module)
   - [4.4 Middlewares: Intercepting Requests (Express vs. Nest)](#44-middlewares)
   - [4.5 Guards: Authentication & Role/Permission Authorization](#45-guards)
   - [4.6 Pipes: Request Validation & Parameter Parsing](#46-pipes)
   - [4.7 Interceptors: Response Mapping & Auditing](#47-interceptors)
   - [4.8 Exception Filters: Centralized Custom & Prisma Error Handling](#48-exception-filters)
   - [4.9 Todo Module: Implementing Advanced Features](#49-todo-module)
     - [CRUD with User Isolation](#crud-isolation)
     - [Interactive Transactions & Rollback (Quotas & Limits)](#transaction-rollback)
     - [Database Aggregations & Query Math (Stats & Aggregates)](#aggregations-query-math)
     - [Pagination & Sorting Helpers](#pagination-helpers)
   - [4.10 File Upload (Multer & Cloudinary)](#410-file-upload-cloudinary)
   - [4.11 Email Dispatching (Nodemailer & EJS/HTML)](#411-email-dispatching-nodemailer--ejs)
   - [4.12 Rate Limiting (Throttler)](#412-rate-limiting)
5. [Testing in NestJS: Unit Tests & End-to-End (E2E) Tests](#5-testing-in-nestjs)
6. [Summary: Checklist for Building GasPay](#6-summary-checklist-for-building-gaspay)

---

## 1. The Paradigm Shift: From Express to NestJS

In Express, you write code *imperatively*. You manually orchestrate routers, require services, and pass handlers around. In NestJS, you write code *declaratively* using **TypeScript Decorators** and **Dependency Injection**.

### 1.1 The HTTP Lifecycle Comparison
Here is how requests flow through both systems:

**Express.js Flow:**
```
Client Request ──> Middleware Chain (cors, parser, etc.) ──> Auth Middleware (jwt verification) ──> Route Handler (Controller function) ──> Manual Response (`res.json`)
```
If an error occurs, it is forwarded manually to a central error handling middleware using `next(error)`.

**NestJS Flow:**
Nest uses an "onion-layer" lifecycle that intercepts requests at specific phases:
```
Client Request
   │
   ├──> 1. Middleware (Global / Route specific)
   │       │
   │       └──> 2. Guards (Authentication & Authorization)
   │               │
   │               └──> 3. Interceptors (Pre-handler logic)
   │                       │
   │                       └──> 4. Pipes (DTO validation & Parameter Parsing)
   │                               │
   │                               └──> 5. Controller (Route Handler method)
   │                                       │
   │                               <───────┘
   │                       │
   │               <─────── 6. Interceptors (Post-handler payload mapping / transformations)
   │               │
   └──<──────────── 7. Exception Filters (Translating thrown exceptions into structured JSON error payloads)
Client Response
```

### 1.2 Inversion of Control (IoC) & Dependency Injection (DI)
In Express, when a route handler needs a database model or helper service, you import the module directly:
```typescript
// Express Style
import { db } from '../config/db';
import { UserService } from '../services/user.service';

export const getUserProfile = async (req, res) => {
  const users = new UserService(db); // Manual construction
  const profile = await users.find(req.user.id);
  res.json(profile);
};
```
In NestJS, classes do not instantiate their own dependencies. The **IoC Container** creates singletons and manages their instantiation on startup. You request dependencies by declaring them in class constructors:
```typescript
// NestJS Style
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable() // Mark this class as a Provider that can be injected
export class UserService {
  constructor(private readonly prisma: PrismaService) {} // Automatically resolved and injected

  async find(id: bigint) {
    return this.prisma.user.findUnique({ where: { id } });
  }
}
```

---

## 2. Project Installation & Initialization

To build our NestJS application, first install the NestJS Command Line Interface (CLI) globally on your machine:

```bash
npm install -g @nestjs/cli
```

### 2.1 Bootstrapping a New Project
Run the following CLI command to initialize a new project scaffold:
```bash
nest new nest-todo-app
```
Choose `npm` as the package manager when prompted.

### 2.2 Installing Relational & Auxiliary Dependencies
Next, install the exact packages needed to mirror Eventra's functionality (JWT, validation, files, mail, rate limiting, and database ORM):
```bash
npm install @prisma/client @nestjs/jwt @nestjs/config @nestjs/throttler @nestjs/mapped-types bcrypt class-validator class-transformer cookie-parser cloudinary multer nodemailer ejs rxjs
npm install --save-dev prisma ts-node typescript @types/node @types/express @types/bcrypt @types/cookie-parser @types/multer @types/nodemailer supertest @types/supertest
```

---

## 3. Architecture Mapping: Eventra (Express) vs. NestJS

Before mapping file structures directly, let's break down the core architectural building blocks of NestJS using a structured **What is it? Why is it used? How is it used?** framework:

---

### 3.1 Modules (`@Module()`)
*   **What is it?**
    A module is a class annotated with the `@Module()` decorator. It provides metadata that NestJS uses to organize the internal application structure and draw boundaries between features.
*   **Why is it used?**
    In Express, code is organized loosely (e.g. importing database connections or route files manually anywhere). As applications grow, this leads to circular dependencies and spaghetti imports. NestJS modules group related controllers, services, and gateways into encapsulated boxes, forming a clean, modular application graph.
*   **How is it used?**
    By registering metadata properties inside `@Module()`:
    *   `imports`: List of other exported modules required by this module.
    *   `controllers`: The API endpoint route controllers registered in this module.
    *   `providers`: The services or database connections instantiated and managed by this module.
    *   `exports`: The services created in this module that should be visible/injectable inside other modules.
    
    ```typescript
    @Module({
      imports: [ConfigModule],
      controllers: [TodoController],
      providers: [TodoService],
      exports: [TodoService],
    })
    export class TodoModule {}
    ```

---

### 3.2 Controllers (`@Controller()`)
*   **What is it?**
    A controller is a class annotated with `@Controller('route-prefix')` containing methods mapped to HTTP request endpoints (e.g. GET, POST, PATCH).
*   **Why is it used?**
    In Express, routing is set up imperatively (e.g. `router.post('/register', authMiddleware, controllerMethod)`). NestJS Controllers use declarative decorators, separating endpoint routing definitions from implementation logic. It makes routes self-documenting and automatically injects query parameters, cookies, request bodies, and headers without manual extraction.
*   **How is it used?**
    Annotating a class and decorating its methods with HTTP verb decorators:
    
    ```typescript
    @Controller('api/v1/auth')
    export class AuthController {
      constructor(private readonly authService: AuthService) {} // Injected service
      
      @Post('login')
      async login(@Body() loginDto: LoginDto) { // Body parser automated
        return this.authService.login(loginDto);
      }
    }
    ```

---

### 3.3 Providers & Services (`@Injectable()`)
*   **What is it?**
    A provider is any class decorated with `@Injectable()` that can be managed and instantiated by the NestJS Dependency Injection (DI) system. Services are a type of provider containing the core business logic of your application.
*   **Why is it used?**
    In Express, you typically import static class instances or functions directly into routers (e.g. `import { authService } from './auth.service'`). This creates tight coupling, making testing and mocking dependencies extremely difficult. In NestJS, services are registered in the DI container. Nest automatically resolves, instantiates, and injects them when requested by another class constructor, facilitating decoupled code and easy unit mocking.
*   **How is it used?**
    Marking the class with `@Injectable()` and requesting it via constructor signatures:
    
    ```typescript
    @Injectable()
    export class UserService {
      constructor(private readonly prisma: PrismaService) {} // Auto-injected dependency
      
      async getUser(id: bigint) {
        return this.prisma.user.findUnique({ where: { id } });
      }
    }
    ```

---

### 3.4 Feature Translation Table
Before writing code, let's look at how the file structures translate:

| Feature in Eventra (Express) | Express Location | NestJS Equivalent | NestJS Decorator / Pattern |
| :--- | :--- | :--- | :--- |
| **Server startup** | `src/server.ts` & `src/app.ts` | `src/main.ts` | `NestFactory.create(AppModule)` |
| **Feature Routes** | `src/app/routes/index.ts` | Controllers | `@Controller('path')` |
| **HTTP Action Route** | `router.post('/login', handler)` | Controller Method | `@Post('login')`, `@Get()`, `@Delete()` |
| **JWT verification** | `src/app/middlewares/auth.ts` | Guards | `@UseGuards(JwtAuthGuard)` |
| **Request Schema Validation**| Zod schemas in middleware | DTOs & ValidationPipe | `@Body() dto: CreateTodoDto` with global `ValidationPipe` |
| **Central Database Instance**| `src/prisma.ts` | PrismaService | Injectable Service module |
| **Error Handlers** | `globalErrorHandler.ts` | Exception Filters | `@Catch(HttpException)` |
| **Response Formatter** | `sendResponse.ts` | Interceptors | `@UseInterceptors(TransformInterceptor)` |
| **File Uploader** | Multer configuration | File Interceptors | `@UseInterceptors(FileInterceptor)` |

---

### 3.5 Proposed Project Directory Structure
Below is the clean-architecture module layout mapping all directories and files built across this guide:

```
todo-app/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── test/
│   └── todos.e2e-spec.ts              ← E2E Tests
├── src/
│   ├── main.ts                        ← App Entrypoint & Global validation setup
│   ├── app.module.ts                  ← Global Module wiring
│   ├── config/
│   │   └── env.validation.ts          ← Startup configuration validation (class-validator)
│   ├── prisma/
│   │   ├── prisma.module.ts           ← Global database connection module
│   │   ├── prisma.service.ts          ← Injectable Prisma Client (with soft deletes)
│   │   └── seeder.service.ts          ← Auto-bootstrapping database seeder
│   ├── common/
│   │   ├── context/
│   │   │   └── tenant.context.ts      ← AsyncLocalStorage storage definition
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   └── roles.decorator.ts
│   │   ├── filters/
│   │   │   └── prisma-exception.filter.ts  ← Centralized db constraint translation
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── interceptors/
│   │   │   ├── bigint.interceptor.ts
│   │   │   └── transform.interceptor.ts
│   │   ├── middleware/
│   │   │   ├── logger.middleware.ts
│   │   │   └── tenant.middleware.ts   ← Subdomain-based tenant identification
│   │   ├── pipes/
│   │   │   ├── parse-bigint.pipe.ts
│   │   │   └── parse-json.pipe.ts
│   │   ├── services/
│   │   │   └── email.service.ts       ← Nodemailer + EJS compiler service
│   │   ├── templates/
│   │   │   └── todo-notification.ejs  ← Notification template layout
│   │   ├── utils/
│   │   │   └── pagination.helper.ts   ← Offset/Limit builder & metadata response
│   │   └── workers/
│   │       └── outbox.worker.ts       ← Background Transactional Outbox processor
│   └── modules/
│       ├── auth/
│       │   ├── auth.module.ts
│       │   ├── auth.controller.ts
│       │   ├── auth.service.ts
│       │   ├── client.controller.ts   ← Form-data multipart validation controller
│       │   └── dto/
│       │       ├── register.dto.ts
│       │       └── login.dto.ts
│       ├── todo/
│       │   ├── todo.module.ts
│       │   ├── todo.controller.ts
│       │   ├── todo.service.ts
│       │   └── todo.service.spec.ts   ← Service Unit tests
│       ├── file/
│       │   ├── file.module.ts         ← Cloudinary client module
│       │   ├── cloudinary.service.ts  ← Multipart image stream uploader
│       │   └── profile.controller.ts  ← Profile avatar change route
│       ├── billing/
│       │   ├── billing.service.ts
│       │   └── late-fee.service.ts    ← Cron-driven daily scheduler
│       └── payment/
│           ├── payment.controller.ts  ← Checkout redirect callbacks & IPN endpoints
│           ├── payment.service.ts     ← State transition transaction blocks
│           └── ssl-commerz.service.ts ← SSLCommerz gateway communications
```

---

### 3.6 Environment Configuration Validation on Startup
*   **What is it?**
    It is a configuration verification pipeline configured inside `@nestjs/config` that asserts all required `.env` values are present and conform to correct schema types (numbers, string formats) when the application bootstraps.
*   **Why is it used?**
    In your Eventra-Backend, you use a custom helper function `requireEnv` inside `src/config/index.ts` to assert environment variable presence:
    ```typescript
    const requireEnv = (name: string): string => {
      const value = process.env[name];
      if (!value) {
        throw new Error(`Environment variable ${name} is missing!`);
      }
      return value;
    };
    ```
    In NestJS, we handle this declaratively using **class-validator** validations. This guarantees the application fails to compile/run immediately if environment configuration criteria are not met, saving you from runtime errors later.
*   **How is it used?**
    By defining an environment validation class using class-validator, validating the configuration, and passing it to `ConfigModule.forRoot`:

In your Eventra-Backend, you use a custom helper function `requireEnv` inside `src/config/index.ts` to assert that all environment variables are present when the Node process starts:
```typescript
const requireEnv = (name: string): string => {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Environment variable ${name} is missing!`);
  }
  return value;
};
```

In NestJS, this is handled declaratively using **`@nestjs/config`**'s custom configuration validation function combined with class-validator.

#### Step 1: Create a Validation Function
Create a validation file that checks the process environment variables:

```typescript
// src/config/env.validation.ts
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, IsString, validateSync } from 'class-validator';

enum Environment {
  Development = 'development',
  Production = 'production',
  Test = 'test',
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  PORT: number = 3000;

  @IsString()
  DATABASE_URL: string;

  @IsString()
  JWT_SECRET: string;
}

export function validateEnv(config: Record<string, any>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(`Startup Config Validation Error: \n${errors.toString()}`);
  }
  return validatedConfig;
}
```

#### Step 2: Register Validation in AppModule
Register this validation function in your global config module definition:

```typescript
// src/app.module.ts (Snippet)
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { validateEnv } from './config/env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // ConfigService will be globally available across all modules
      validate: validateEnv, // Auto-validate on application bootstrap
    }),
  ],
})
export class AppModule {}
```
If any variables declared in the `envSchema` are missing or incorrect, NestJS will throw an assertion error on startup, preventing the server from listening on an invalid configuration state.

---

### 3.7 Detailed Code Translations: Express (Eventra) vs. NestJS
For developers transitioning from the **Eventra-Backend (Express)** architecture to **NestJS**, here is a detailed, code-by-code translation mapping core utilities:

#### A. CatchAsync (Asynchronous Error Handler)
In **Eventra**, async routes must be wrapped with a helper to pass rejected promises to `next()`:
```typescript
// Eventra (Express): src/shared/catchAsync.ts
type AsyncHandler = (req: Request, res: Response, next: NextFunction) => Promise<void>;
export const catchAsync = (fn: AsyncHandler) => (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
};
```
In **NestJS**, `catchAsync` is completely obsolete. NestJS controllers natively support Promises and Observables. If a controller throws an error or rejects, NestJS automatically intercepts it and forwards it to the exception filter layer:
```typescript
// NestJS: Clean Controller standard
@Get(':id')
async getTodo(@Param('id', ParseBigIntPipe) id: bigint) {
  // NestJS catches any throws/rejections here natively and forwards to global filters
  return this.todoService.findOne(id);
}
```

#### B. ValidateRequest (DTO Body Validation)
In **Eventra**, request payloads are verified via inline schema-parsing middleware:
```typescript
// Eventra (Express): src/app/middlewares/validateRequest.ts
const validateRequest = (schema: ZodObject<any>) => async (req: Request, res: Response, next: NextFunction) => {
    try {
        await schema.parseAsync({ body: req.body });
        return next();
    } catch (err) {
        next(err);
    }
};
```
In **NestJS**, request validations are handled using **Data Transfer Objects (DTOs)** and the built-in **`ValidationPipe`**. Rather than a custom pipe for each route, NestJS uses decorators on a class structure to declare validation rules:
```typescript
// NestJS: src/modules/auth/dto/register.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;

  @IsString()
  name: string;
}
```
In the main bootstrap file, we register the validation pipe globally (or at handler level):
```typescript
// src/main.ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```
The controller simply binds the DTO as a type, and NestJS validates it automatically:
```typescript
// NestJS: Controller Handler
@Post('register')
async register(@Body() registerDto: RegisterDto) {
  return this.authService.register(registerDto);
}
```

#### C. Authentication & Role Authorization (JWT + Cookie Checks)
In **Eventra**, user authentication and authorization logic are coupled inside an Express middleware:
```typescript
// Eventra (Express): src/app/middlewares/auth.ts
const auth = (...roles: string[]) => async (req: Request, res: Response, next: NextFunction) => {
    try {
        const token = req.cookies["accessToken"];
        if (!token) throw new ApiError(401, "You are not authorized!");
        
        const verifyUser = jwtHelper.verifyToken(token, config.jwt.jwt_secret);
        req.user = verifyUser;

        if (roles.length && !roles.includes(verifyUser.role)) {
            throw new ApiError(401, "You are not authorized!");
        }
        next();
    } catch (err) {
        next(err);
    }
};
```
In **NestJS**, you separate authentication, authorization, and decorator bindings into discrete, single-responsibility components:
1. **`JwtAuthGuard`**: Decodes the token, asserts authenticity, and binds the payload to the request context.
2. **`RolesGuard`**: Reads metadata (using Nest's `Reflector`) to match against authenticated users.
3. **`Roles` Decorator**: Attaches required roles metadata.
4. **`CurrentUser` Decorator**: Safely retrieves the authenticated user from the request context.

```typescript
// NestJS: Declarative Guard & Decorator Setup
@Controller('api/v1/todos')
@UseGuards(JwtAuthGuard, RolesGuard) // Authenticate and check authorization
export class TodoController {
  
  @Post()
  @Roles('ADMIN', 'CREATOR') // Apply metadata
  async createTodo(
    @CurrentUser() user: JwtPayload, // Extract from request
    @Body() dto: CreateTodoDto,
  ) {
    return this.todoService.create(dto, user.id);
  }
}
```

#### D. Pagination & Request Parameter Filtering
In **Eventra**, pagination inputs are calculated using a static helper and filtered using a custom `pick` function:
```typescript
// Eventra (Express): src/helpers/paginationHelper.ts
const calculatePagination = (options: IOptions): IOptionsResult => {
    const page = Number(options.page) || 1;
    const limit = Number(options.limit) || 9;
    const skip = (page - 1) * limit;
    const sortBy = options.sortBy || 'createdAt';
    const sortOrder = options.sortOrder || 'desc';

    return { page, limit, skip, sortBy, sortOrder };
};
```
```typescript
// Eventra (Express): Usage inside Controller
const paginationOptions = paginationHelper.calculatePagination(pick(req.query, ['page', 'limit', 'sortBy', 'sortOrder']));
```
In **NestJS**, query parameters are bound using the `@Query()` decorator, validated/parsed using pipes (or custom utilities), and mapped to pagination filters:
```typescript
// NestJS: src/common/utils/pagination.helper.ts
export class PaginationHelper {
  static getPaginationOptions(options: { page?: string; limit?: string; sortBy?: string; sortOrder?: string }) {
    const page = Math.max(Number(options.page) || 1, 1);
    const limit = Math.max(Number(options.limit) || 10, 1);
    const skip = (page - 1) * limit;

    const sortBy = options.sortBy || 'created_at';
    const sortOrder = options.sortOrder || 'desc';

    return {
      skip,
      take: limit,
      orderBy: { [sortBy]: sortOrder },
    };
  }
}
```
In your controllers, simply bind inputs using parameter annotations:
```typescript
// NestJS: Pagination usage
@Get()
async getTodos(
  @Query('page') page?: string,
  @Query('limit') limit?: string,
  @Query('sortBy') sortBy?: string,
  @Query('sortOrder') sortOrder?: 'asc' | 'desc',
) {
  const options = PaginationHelper.getPaginationOptions({ page, limit, sortBy, sortOrder });
  return this.todoService.findAll(options);
}
```
![alt text](image.png)
---

## 4. Step-by-Step Construction of the Miniature Todo App

We will write a fully operational multi-tenant Todo app. 
It supports three roles:
- `ADMIN`: Full database control.
- `CREATOR`: Can create lists and todos.
- `COLLABORATOR`: Read-only or assigned todo marking.

---

### 4.1 Database Design & Prisma Setup
Initialize Prisma inside your project directory:
```bash
npx prisma init
```
This generates a `prisma/schema.prisma` file and a `.env` configuration file. Replace the contents of `prisma/schema.prisma` with the database model below.

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum Role {
  ADMIN
  CREATOR
  COLLABORATOR
}

model User {
  id            BigInt      @id @default(autoincrement())
  email         String      @unique
  name          String
  password_hash String
  role          Role        @default(COLLABORATOR)
  created_at    DateTime    @default(now()) @map("created_at")
  updated_at    DateTime    @updated_at @map("updated_at")
  profile_photo String?     @map("profile_photo")
  
  // Relations
  todo_lists    TodoList[]  @relation("ListOwner")
  assigned_todos TodoItem[] @relation("AssignedUser")
  audit_logs    AuditLog[]

  @@map("users")
}

model TodoList {
  id          BigInt     @id @default(autoincrement())
  owner_id    BigInt     @map("owner_id")
  title       String
  description String?
  total_todos Int        @default(0) @map("total_todos")
  created_at  DateTime   @default(now()) @map("created_at")
  
  // Relations
  owner       User       @relation("ListOwner", fields: [owner_id], references: [id], onDelete: Cascade)
  todo_items  TodoItem[]

  @@map("todo_lists")
}

model TodoItem {
  id           BigInt    @id @default(autoincrement())
  list_id      BigInt    @map("list_id")
  assignee_id  BigInt?   @map("assignee_id")
  title        String
  completed    Boolean   @default(false)
  completed_at DateTime? @map("completed_at")
  due_date     DateTime? @map("due_date")
  created_at   DateTime  @default(now()) @map("created_at")

  // Relations
  list         TodoList  @relation(fields: [list_id], references: [id], onDelete: Cascade)
  assignee     User?     @relation("AssignedUser", fields: [assignee_id], references: [id], onDelete: SetNull)

  @@map("todo_items")
}

model AuditLog {
  id          BigInt   @id @default(autoincrement())
  user_id     BigInt   @map("user_id")
  action      String   // e.g. "TODO_CREATED", "TODO_COMPLETED"
  details     Json
  created_at  DateTime @default(now()) @map("created_at")

  user        User     @relation(fields: [user_id], references: [id], onDelete: Cascade)

  @@map("audit_logs")
}
```

#### Run Database Migrations:
Execute the command below to apply the schema modifications to your active PostgreSQL instance:
```bash
npx prisma migrate dev --name init_todo_schema
```

---

### 4.2 Prisma Module & Service (Global Database Connection)
To use Prisma across our Nest modules, we wrap the database connection client inside an injectable Nest Service.

Create the files inside `src/prisma/`:

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    // Establish connection on app bootstrap
    await this.$connect();
  }

  async onModuleDestroy() {
    // Gracefully disconnect on app shutdown
    await this.$disconnect();
  }
}
```

Now, write the module file. Notice the `@Global()` decorator. By marking this module global and exporting the `PrismaService`, we allow other modules to import and use the database client without importing `PrismaModule` explicitly.

```typescript
// src/prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService], // Must export to make injectable outside this module
})
export class PrismaModule {}
```

---

### 4.3 Authentication Module (JWT, Cookies, Roles)
Let's build a authentication module that handles user registration, JWT token generation, password hashing, and cookie injection.

#### Step 1: Create the User Register & Login DTOs
DTOs act as structural blueprints to validate client request bodies. Using `class-validator`, we define decorators for direct request validation.

```typescript
// src/modules/auth/dto/register.dto.ts
import { IsEmail, IsString, MinLength, IsEnum, IsOptional } from 'class-validator';
import { Role } from '@prisma/client';

export class RegisterDto {
  @IsEmail({}, { message: 'Please provide a valid email address.' })
  email: string;

  @IsString()
  @MinLength(1, { message: 'Name is required.' })
  name: string;

  @IsString()
  @MinLength(6, { message: 'Password must be at least 6 characters long.' })
  password: string;

  @IsEnum(Role)
  @IsOptional()
  role?: Role;
}
```

```typescript
// src/modules/auth/dto/login.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail({}, { message: 'Please provide a valid email address.' })
  email: string;

  @IsString()
  @MinLength(1, { message: 'Password is required.' })
  password: string;
}
```

#### Step 2: Implement the Authentication Service
This service hashes passwords, registers users, and signs JWT payloads.

```typescript
// src/modules/auth/auth.service.ts
import { Injectable, ConflictException, UnauthorizedException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly jwtService: JwtService,
  ) {}

  async register(dto: RegisterDto) {
    // 1. Verify email uniqueness
    const existing = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });
    if (existing) {
      throw new ConflictException('A user with this email address already exists.');
    }

    // 2. Hash Password
    const passwordHash = await bcrypt.hash(dto.password, 10);

    // 3. Create User in PostgreSQL
    const user = await this.prisma.user.create({
      data: {
        email: dto.email,
        name: dto.name,
        password_hash: passwordHash,
        role: dto.role,
      },
    });

    return {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
    };
  }

  async login(dto: LoginDto) {
    // 1. Find user by email
    const user = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });
    if (!user) {
      throw new UnauthorizedException('Invalid credentials.');
    }

    // 2. Verify hashed password match
    const isPasswordValid = await bcrypt.compare(dto.password, user.password_hash);
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials.');
    }

    // 3. Sign JWT Token Payload
    const payload = { 
      sub: user.id.toString(), // Convert BigInt to string for token payload safety
      email: user.email, 
      role: user.role 
    };

    return {
      accessToken: await this.jwtService.signAsync(payload),
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
    };
  }
}
```

#### Step 3: Implement the Authentication Controller
This controller exposes authentication routes and injects JWT access tokens directly into HTTP-only cookies. Incoming requests are automatically validated against DTOs using NestJS's global `ValidationPipe`.

```typescript
// src/modules/auth/auth.controller.ts
import { Controller, Post, Body, Res, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { Response } from 'express';

@Controller('api/v1/auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  async register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }

  @Post('login')
  @HttpCode(HttpStatus.OK)
  async login(
    @Body() dto: LoginDto,
    @Res({ passthrough: true }) response: Response, // { passthrough: true } lets Nest automatically serialize returned values
  ) {
    const result = await this.authService.login(dto);

    // Inject JWT into HTTP-only cookie, matching Eventra behaviors
    response.cookie('accessToken', result.accessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      maxAge: 24 * 60 * 60 * 1000, // 1 Day
      sameSite: 'lax',
    });

    return {
      success: true,
      user: result.user,
    };
  }
}
```

#### Step 4: Wire the Authentication Module
```typescript
// src/modules/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigService, ConfigModule } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';

@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get<string>('JWT_SECRET', 'DEFAULT_SECRET_KEY'),
        signOptions: { expiresIn: '1d' },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

---

### 4.4 Middlewares: Intercepting Requests (Express vs. Nest)
*   **What is it?**
    A middleware is a function or class annotated with `@Injectable()` implementing the `NestMiddleware` interface. It is executed before the route handler, with full access to the Node/Express `request` and `response` objects and the `next()` function.
*   **Why is it used?**
    It has exactly the same purpose as Express middlewares. Use it for tasks that apply globally to requests, such as request logging, cookie parsing, CORS, headers injection, or body parsing.
*   **How is it used?**
    By implementing the `use(req, res, next)` method:

**Example: Request Logger Middleware**
This middleware prints incoming request details to the server console:

```typescript
// src/common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl } = req;
    const userAgent = req.get('user-agent') || '';
    const startTime = Date.now();

    res.on('finish', () => {
      const { statusCode } = res;
      const duration = Date.now() - startTime;
      this.logger.log(
        `${method} ${originalUrl} ${statusCode} - ${userAgent} (${duration}ms)`
      );
    });

    next();
  }
}
```

#### Registering Middleware in NestJS Modules:
Unlike controllers and services, middlewares are not declared using the `@Module()` decorator. Instead, you apply them dynamically in module classes that implement `NestModule`:

```typescript
// src/app.module.ts (Snippet)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';

@Module({
  // module configurations
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*'); // Apply logger middleware globally to all routes
  }
}
```

---

### 4.5 Guards: Authentication & Role/Permission Authorization
*   **What is it?**
    A guard is an injectable class annotated with `@Injectable()` implementing the `CanActivate` interface. It determines if a request should be allowed to run the controller handler.
*   **Why is it used?**
    In Express, you write security checks as mid-route middlewares (e.g. `auth(UserRole.ADMIN)`). However, middlewares are generic and do not easily know *which* handler is running or what route metadata is attached. NestJS Guards execute *after* middlewares but *before* pipes, and have direct access to the `ExecutionContext` (allowing them to inspect route metadata, such as roles needed for the endpoint).
*   **How is it used?**
    By returning a boolean (or a promise resolving to a boolean) from the `canActivate()` method. If it returns `true`, the request proceeds; if `false` or throws, Nest blocks it.

#### 4.5.1 JWT Authentication Guard
This guard extracts the JWT token from the authorization headers or HTTP-only cookies, validates it, and attaches the active user payload to the Express request object.

```typescript
// src/common/guards/jwt-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const httpCtx = context.switchToHttp();
    const request = httpCtx.getRequest<Request>();

    // 1. Extract token from Authorization header or HTTP cookies
    let token = '';
    const authHeader = request.headers.authorization;
    if (authHeader && authHeader.startsWith('Bearer ')) {
      token = authHeader.split(' ')[1];
    } else if (request.cookies && request.cookies.accessToken) {
      token = request.cookies.accessToken;
    }

    if (!token) {
      throw new UnauthorizedException('Authentication credentials not found.');
    }

    try {
      // 2. Verify decoded JWT payload signature
      const payload = await this.jwtService.verifyAsync(token);
      
      // Convert ID back to BigInt context for internal database queries
      request['user'] = {
        id: BigInt(payload.sub),
        email: payload.email,
        role: payload.role,
      };
      
      return true;
    } catch {
      throw new UnauthorizedException('Token is expired or invalid.');
    }
  }
}
```

#### 4.5.2 Role-Based Access Control (RBAC) Guard
This guard reads role metadata assigned to specific controller endpoints and checks it against the authenticated user's role.

First, create a metadata helper decorator:
```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '@prisma/client';

export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);
```

Next, implement the guard class:
```typescript
// src/common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from '@prisma/client';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {} // Reflector reads custom metadata

  canActivate(context: ExecutionContext): boolean {
    // Read the required roles metadata from the controller route handler
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true; // If no roles metadata is set, access is permitted by default
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      throw new ForbiddenException('Access denied: Authentication context not found.');
    }

    const hasRole = requiredRoles.includes(user.role);
    if (!hasRole) {
      throw new ForbiddenException(`Access denied: Required roles: [${requiredRoles.join(', ')}].`);
    }

    return true;
  }
}
```

#### 4.5.3 Custom Parameter Decorators (`@CurrentUser`)
*   **What is it?**
    A custom parameter decorator is a custom-defined parameter annotation built using Nest's `createParamDecorator` function. It allows you to extract specific request properties directly into your controller method arguments.
*   **Why is it used?**
    In Express, you retrieve request contexts by manually typing `req.user` or `req.tenantId` in your controller logic. This ties your business controllers directly to the Express `req` signature, making unit testing harder (since you have to construct complete mock request objects). Custom parameter decorators extract these variables declaratively, keeping controller signatures clean and testable.
*   **How is it used?**
    Create the decorator using `createParamDecorator` and bind it to controller arguments:

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    if (!user) return null;
    return data ? user[data] : user;
  },
);
```

---

### 4.6 Pipes: Request Validation & Parameter Parsing
*   **What is it?**
    A pipe is an injectable helper class implementing the `PipeTransform` interface. It operates on the arguments (parameters) of a controller route handler method.
*   **Why is it used?**
    In Express, incoming params are string types by default, and body parsing/validation is handled manually inside controllers or via route middlewares (like Eventra's Zod parse). NestJS Pipes run immediately before the handler method is executed. They perform two key tasks:
    1.  **Transformation**: Mapping input strings/objects to their expected JavaScript types (e.g. converting string IDs to `BigInt`).
    2.  **Validation**: Inspecting payloads against DTO classes (using class-validator decorators) and automatically throwing HTTP 400 Bad Request if validation rules fail, preventing execution from ever reaching your service.
*   **How is it used?**
    You apply them directly to route parameters or handlers using decorators:

#### 4.6.1 ParseBigIntPipe
Since PostgreSQL uses `BIGINT` for its primary key IDs, this custom pipe parses route parameter strings into JavaScript `bigint` types.

```typescript
// src/common/pipes/parse-bigint.pipe.ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseBigIntPipe implements PipeTransform<string, bigint> {
  transform(value: string): bigint {
    try {
      return BigInt(value);
    } catch {
      throw new BadRequestException(`Validation failed. "${value}" is not a valid 64-bit integer.`);
    }
  }
}
```

#### 4.6.2 Global ValidationPipe Setup
In NestJS, instead of writing custom validation pipes, we use the built-in `ValidationPipe` provided by `@nestjs/common` which automatically handles body schema verification using the class annotations defined inside our DTOs.

Register it globally inside `src/main.ts`:
```typescript
// src/main.ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // Automatically strip non-decorated properties
      forbidNonWhitelisted: true, // Throw errors if undeclared fields are passed
      transform: true,            // Auto-transform request payloads into DTO classes
    }),
  );

  await app.listen(3000);
}
```

---

### 4.7 Interceptors: Response Mapping & Auditing
*   **What is it?**
    An interceptor is an injectable class annotated with `@Injectable()` implementing the `NestInterceptor` interface. It intercepts the request-response stream both *before* the controller executes and *after* the response is returned.
*   **Why is it used?**
    It wraps method execution, using RxJS observables to inspect or alter the final output payload. Common use cases include:
    1.  **Response Transformation**: Automatically formatting response envelopes (e.g. replacing Express's manual `sendResponse` checks).
    2.  **Payload Serialization**: Manipulating the returned object structure (e.g. converting `BigInt` types to string representations to prevent JSON stringify crashes).
    3.  **Auditing & Logging**: Logging database changes or tracking route execution time.
*   **How is it used?**
    By implementing `intercept(context, next)` and wrapping the call stream using RxJS operators:

#### 4.7.1 Global BigInt JSON Serializer Interceptor
JavaScript's `JSON.stringify` does not support serializing BigInt variables to JSON strings natively, which will cause your application to throw a `TypeError: Do not know how to serialize a BigInt` exception. 

This interceptor intercepts all outgoing responses and recursively converts BigInt values to string fields.

```typescript
// src/common/interceptors/bigint.interceptor.ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { map, Observable } from 'rxjs';

@Injectable()
export class BigIntInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => this.serializeBigInt(data))
    );
  }

  private serializeBigInt(obj: any): any {
    if (obj === null || obj === undefined) return obj;
    if (typeof obj === 'bigint') return obj.toString();
    if (Array.isArray(obj)) return obj.map(item => this.serializeBigInt(item));
    if (typeof obj === 'object') {
      const result: any = {};
      for (const key in obj) {
        if (Object.prototype.hasOwnProperty.call(obj, key)) {
          result[key] = this.serializeBigInt(obj[key]);
        }
      }
      return result;
    }
    return obj;
  }
}
```

#### 4.7.2 Global Response Wrapping Interceptor
Like Eventra, you can enforce a standard JSON API envelope (`{ success: true, statusCode: 200, message: "...", data: {...} }`) globally:

```typescript
// src/common/interceptors/transform.interceptor.ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { map, Observable } from 'rxjs';

export interface ResponseEnvelope<T> {
  success: boolean;
  statusCode: number;
  message?: string;
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ResponseEnvelope<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ResponseEnvelope<T>> {
    const httpContext = context.switchToHttp();
    const response = httpContext.getResponse();
    const statusCode = response.statusCode;

    return next.handle().pipe(
      map(data => ({
        success: true,
        statusCode,
        message: data?.message || 'Request completed successfully.',
        data: data?.data !== undefined ? data.data : data,
      }))
    );
  }
}
```

---

### 4.8 Exception Filters: Centralized Custom & Prisma Error Handling
*   **What is it?**
    An exception filter is a class annotated with the `@Catch()` decorator implementing the `ExceptionFilter` interface. It is responsible for catching unhandled exceptions thrown across your application.
*   **Why is it used?**
    In Express, you write a central error handling middleware `globalErrorHandler(err, req, res, next)`. In NestJS, Exception Filters serve this exact role. They intercept uncaught exceptions (such as standard Nest HTTP exceptions, validation errors, and raw database constraint violations from Prisma), log the details, and return a clean, structured JSON response envelope instead of letting the server crash or leak sensitive system data.
*   **How is it used?**
    By implementing the `catch(exception, host)` method and registering it globally in `main.ts` or applying it to specific controllers.

This filter handles custom NestJS HTTP exceptions and database constraints from Prisma:

```typescript
// src/common/filters/global-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus, Logger } from '@nestjs/common';
import { Response } from 'express';
import { Prisma } from '@prisma/client';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger('ExceptionFilter');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Something went wrong!';
    let errorDetails: any = exception;

    // 1. Process standard NestJS HTTP Exceptions (e.g. ForbiddenException, NotFoundException)
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const resPayload = exception.getResponse();
      message = typeof resPayload === 'object' && 'message' in resPayload
        ? (resPayload as any).message
        : exception.message;
      
      errorDetails = {
        statusCode: status,
        message: Array.isArray(message) ? message[0] : message,
      };
    } 
    // 2. Process Prisma-specific database errors (matching globalErrorHandler.ts)
    else if (exception instanceof Prisma.PrismaClientKnownRequestError) {
      switch (exception.code) {
        case 'P2000':
          message = 'Value too long for column';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2002':
          message = 'Unique constraint failed (duplicate key)';
          status = HttpStatus.CONFLICT;
          break;
        case 'P2003':
          message = 'Foreign key constraint failed';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2004':
          message = 'A constraint failed';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2005':
          message = 'Invalid type for field';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2006':
          message = 'Field required error';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2007':
          message = 'Data validation failed';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2008':
          message = 'Failed to parse value';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2009':
          message = 'Invalid query';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P2010':
          message = 'Missing required argument';
          status = HttpStatus.BAD_REQUEST;
          break;
        case 'P1000':
          message = 'Authentication failed against database server';
          status = HttpStatus.BAD_GATEWAY;
          break;
        default:
          message = 'Prisma database error occurred';
          status = HttpStatus.BAD_REQUEST;
          break;
      }
      errorDetails = {
        statusCode: status,
        message,
        meta: exception.meta,
      };
    } 
    else if (exception instanceof Prisma.PrismaClientValidationError) {
      message = 'Validation error: ' + exception.message;
      status = HttpStatus.BAD_REQUEST;
      errorDetails = {
        statusCode: status,
        message,
      };
    } 
    else if (exception instanceof Prisma.PrismaClientUnknownRequestError) {
      message = 'Unknown Prisma error occurred';
      status = HttpStatus.INTERNAL_SERVER_ERROR;
      errorDetails = {
        statusCode: status,
        message,
      };
    } 
    else if (exception instanceof Prisma.PrismaClientInitializationError) {
      message = 'Prisma client failed to initialize';
      status = HttpStatus.INTERNAL_SERVER_ERROR;
      errorDetails = {
        statusCode: status,
        message,
      };
    } 
    else if (exception instanceof Error) {
      message = exception.message;
      errorDetails = {
        statusCode: status,
        message,
        stack: exception.stack,
      };
    }

    this.logger.error(`Exception Caught [${status}]: ${message}`, exception instanceof Error ? exception.stack : '');

    // Matches the exact shape of your Eventra error responses: { success: false, message, error }
    response.status(status).json({
      success: false,
      message: Array.isArray(message) ? message[0] : message,
      error: errorDetails,
    });
  }
}
```

---

### 4.9 Todo Module: Implementing Advanced Features
Now, let's combine these concepts to build the Todo module. We will implement CRUD operations, user isolation, relational migrations, aggregations, and automatic transaction rollbacks.

#### 4.9.1 DTO Definitions
```typescript
// src/modules/todo/dto/create-todo.dto.ts
import { IsString, MinLength, IsOptional, IsDateString } from 'class-validator';

export class CreateTodoDto {
  @IsString()
  @MinLength(1, { message: 'Title is required.' })
  title: string;

  @IsString()
  @IsOptional()
  assigneeId?: string; // String type from JSON, parsed to BigInt inside the service

  @IsDateString()
  @IsOptional()
  dueDate?: string;
}
```

```typescript
// src/modules/todo/dto/create-list.dto.ts
import { IsString, MinLength, IsOptional } from 'class-validator';

export class CreateListDto {
  @IsString()
  @MinLength(1, { message: 'Title is required.' })
  title: string;

  @IsString()
  @IsOptional()
  description?: string;
}
```

---

#### 4.9.2 Implementing the Todo Service (Transactions, Aggregations, Isolation)
Before looking at the implementation, let's understand these advanced database query patterns:

---

##### A. Interactive Transactions & Rollback
*   **What is it?**
    An interactive transaction is a series of database read/write operations executed within a secure, atomic transaction block. If any step fails or throws an exception, all database modifications in that block are rolled back.
*   **Why is it used?**
    In Express, you manage transaction states manually via SQL strings (e.g. `BEGIN`, `COMMIT`, `ROLLBACK`). In NestJS and Prisma, we pass an async callback to `this.prisma.$transaction(async (tx) => { ... })`. This ensures atomic database integrity: for example, when adding a Todo item, we increment the list's `total_todos` count and insert the item. If the insert fails, the increment is automatically reverted. In **GasPay**, this prevents creating bills without ledger records.
*   **How is it used?**
    By executing queries inside the `$transaction` callback, using the transaction-scoped `tx` database client instead of the global `this.prisma` client:
    ```typescript
    await this.prisma.$transaction(async (tx) => {
      await tx.model.update(...);
      // If error occurs, throw exception to abort and roll back
    });
    ```

---

##### B. Database Aggregations & Grouping
*   **What is it?**
    Database aggregations are functions (such as `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) and grouping mechanisms (`GROUP BY`) executed directly by the database engine to summarize record fields.
*   **Why is it used?**
    Instead of fetching thousands of records into Node's memory and using JavaScript loop arrays to count or sum fields (which wastes CPU and memory), database-level aggregations calculate metrics in microseconds and return only the final scalar result values. This is critical for statistics dashboard widgets.
*   **How is it used?**
    Using Prisma's `aggregate`, `groupBy`, and `_count` query operations:
    ```typescript
    const stats = await this.prisma.todoItem.aggregate({
      _count: { id: true },
      where: { completed: true },
    });
    ```

---

##### C. Query Pagination & Sorting
*   **What is it?**
    Query pagination splits database records into smaller, manageable chunks (pages) based on a offset index and limit range, while sorting orders the output according to a specified field.
*   **Why is it used?**
    Querying and returning all records in a single request when a database table has thousands of rows will cause high request latency, browser lag, and potential server out-of-memory crashes. Pagination fetches rows in small batches (e.g. 10 at a time), ensuring fast, scalable response times.
*   **How is it used?**
    Using Prisma's `skip` (offset) and `take` (limit) keywords derived from HTTP request queries:
    ```typescript
    const todos = await this.prisma.todoItem.findMany({
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { created_at: 'desc' },
    });
    ```

---

##### D. Todo Service Implementation
This service houses the core business logic, database queries, transactions, and aggregations:

```typescript
// src/modules/todo/todo.service.ts
import { Injectable, ForbiddenException, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { CreateListDto } from './dto/create-list.dto';
import { CreateTodoDto } from './dto/create-todo.dto';

@Injectable()
export class TodoService {
  constructor(private readonly prisma: PrismaService) {}

  // 1. Create a new TodoList (Strict creator isolation)
  async createList(dto: CreateListDto, ownerId: bigint) {
    return this.prisma.todoList.create({
      data: {
        owner_id: ownerId,
        title: dto.title,
        description: dto.description,
      },
    });
  }

  // 2. Fetch Lists Owned by the User
  async fetchMyLists(ownerId: bigint) {
    return this.prisma.todoList.findMany({
      where: { owner_id: ownerId },
      include: { _count: { select: { todo_items: true } } },
    });
  }

  // 3. Interactive Transaction: Create a Todo item with auto-rollback
  async createTodoItem(listId: bigint, dto: CreateTodoDto, creatorId: bigint) {
    // Runs inside a single database transaction thread
    return this.prisma.$transaction(async (tx) => {
      
      // Step A: Fetch and verify list ownership
      const list = await tx.todoList.findUnique({
        where: { id: listId },
      });
      if (!list) {
        throw new NotFoundException('The target Todo List does not exist.');
      }
      if (list.owner_id !== creatorId) {
        throw new ForbiddenException('You do not own this list.');
      }

      // Step B: Quota validation rollback (max 10 todos per list limit)
      if (list.total_todos >= 10) {
        // Throwing an exception inside the $transaction callback aborts and rolls back all queries
        throw new BadRequestException('Transaction Aborted: This list has reached its limit of 10 todos.');
      }

      // Step C: Increment total items counter on the parent list
      await tx.todoList.update({
        where: { id: listId },
        data: { total_todos: { increment: 1 } },
      });

      // Step D: Create the TodoItem
      const assigneeId = dto.assigneeId ? BigInt(dto.assigneeId) : null;
      const todo = await tx.todoItem.create({
        data: {
          list_id: listId,
          assignee_id: assigneeId,
          title: dto.title,
          due_date: dto.dueDate ? new Date(dto.dueDate) : null,
        },
      });

      // Step E: Create an Audit Log entry for the action
      await tx.auditLog.create({
        data: {
          user_id: creatorId,
          action: 'TODO_CREATED',
          details: { todoId: todo.id.toString(), title: todo.title },
        },
      });

      return todo;
    });
  }

  // 4. Mark Todo Completed
  async completeTodo(todoId: bigint, userId: bigint) {
    return this.prisma.$transaction(async (tx) => {
      const todo = await tx.todoItem.findUnique({
        where: { id: todoId },
        include: { list: true },
      });

      if (!todo) {
        throw new NotFoundException('Todo item not found.');
      }

      // Access checks: owner or assignee can complete the task
      if (todo.list.owner_id !== userId && todo.assignee_id !== userId) {
        throw new ForbiddenException('You do not have permission to modify this todo.');
      }

      const updatedTodo = await tx.todoItem.update({
        where: { id: todoId },
        data: {
          completed: true,
          completed_at: new Date(),
        },
      });

      // Audit Log
      await tx.auditLog.create({
        data: {
          user_id: userId,
          action: 'TODO_COMPLETED',
          details: { todoId: todo.id.toString() },
        },
      });

      return updatedTodo;
    });
  }

  // 5. Database Aggregations for Stats (Equivalent to Eventra review averages)
  async getCompletionMetrics(userId: bigint) {
    // Total count of todos completed vs incomplete across all user lists
    const totalCount = await this.prisma.todoItem.count({
      where: { list: { owner_id: userId } },
    });

    const completedCount = await this.prisma.todoItem.count({
      where: { list: { owner_id: userId }, completed: true },
    });

    // Run custom group-by query math on due date attributes
    const aggregation = await this.prisma.todoItem.groupBy({
      by: ['completed'],
      where: { list: { owner_id: userId } },
      _count: {
        id: true,
      },
    });

    return {
      total: totalCount,
      completed: completedCount,
      completionRatio: totalCount > 0 ? (completedCount / totalCount) * 100 : 0,
      distribution: aggregation,
    };
  }
}
```

---

#### 4.9.3 Implementing the Todo Controller
This controller exposes API endpoints, applies security guards, reads user contexts via decorators, and validates parameters using pipes.

```typescript
// src/modules/todo/todo.controller.ts
import { Controller, Post, Get, Patch, Body, Param, UseGuards, UsePipes } from '@nestjs/common';
import { TodoService } from './todo.service';
import { CreateListDto } from './dto/create-list.dto';
import { CreateTodoDto } from './dto/create-todo.dto';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { RolesGuard } from '../../common/guards/roles.guard';
import { Roles } from '../../common/decorators/roles.decorator';
import { CurrentUser } from '../../common/decorators/current-user.decorator';
import { ParseBigIntPipe } from '../../common/pipes/parse-bigint.pipe';
import { Role } from '@prisma/client';

@Controller('api/v1/todos')
@UseGuards(JwtAuthGuard, RolesGuard) // Order is important: run auth guard first to set request.user context
export class TodoController {
  constructor(private readonly todoService: TodoService) {}

  @Post('lists')
  @Roles(Role.ADMIN, Role.CREATOR) // Only admins and creators can create lists
  async createList(
    @Body() dto: CreateListDto,
    @CurrentUser('id') userId: bigint,
  ) {
    const list = await this.todoService.createList(dto, userId);
    return {
      message: 'Todo list created successfully.',
      data: list,
    };
  }

  @Get('lists')
  async getMyLists(@CurrentUser('id') userId: bigint) {
    const lists = await this.todoService.fetchMyLists(userId);
    return {
      data: lists,
    };
  }

  @Post('lists/:listId/items')
  @Roles(Role.ADMIN, Role.CREATOR)
  async createTodoItem(
    @Param('listId', ParseBigIntPipe) listId: bigint,
    @Body() dto: CreateTodoDto,
    @CurrentUser('id') userId: bigint,
  ) {
    const todo = await this.todoService.createTodoItem(listId, dto, userId);
    return {
      message: 'Todo item added to list successfully.',
      data: todo,
    };
  }

  @Patch('items/:id/complete')
  async completeTodo(
    @Param('id', ParseBigIntPipe) id: bigint,
    @CurrentUser('id') userId: bigint,
  ) {
    const todo = await this.todoService.completeTodo(id, userId);
    return {
      message: 'Todo marked as completed.',
      data: todo,
    };
  }

  @Get('metrics')
  async getMetrics(@CurrentUser('id') userId: bigint) {
    const metrics = await this.todoService.getCompletionMetrics(userId);
    return {
      data: metrics,
    };
  }
}
```

---

#### 4.9.4 Wire the Todo Module
```typescript
// src/modules/todo/todo.module.ts
import { Module } from '@nestjs/common';
import { TodoService } from './todo.service';
import { TodoController } from './todo.controller';
import { JwtModule } from '@nestjs/jwt'; // Required inside JwtAuthGuard

@Module({
  imports: [JwtModule],
  controllers: [TodoController],
  providers: [TodoService],
})
export class TodoModule {}
```

---

### 4.10 File Upload (Multer & Cloudinary)
*   **What is it?**
    It is a file processing structure that intercepts multipart form-data requests using Nest's `FileInterceptor` (which wraps Express Multer internally) and uploads the extracted binary streams to external cloud storage (like Cloudinary).
*   **Why is it used?**
    In Express, you configure Multer directly on the routes (e.g. `router.post('/upload', upload.single('file'), handler)`), and handle file parsing manually. NestJS isolates this using `FileInterceptor` and class-level parameter bindings (`@UploadedFile()`), keeping controllers cleanly separated from file stream buffers and handling storage credentials inside an injectable service module.
*   **How is it used?**
    By using the `@UseInterceptors(FileInterceptor('fieldname'))` decorator on your route and retrieving the file via the `@UploadedFile()` decorator:

#### Step 1: Create the Cloudinary Service
```typescript
// src/modules/file/cloudinary.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { v2 as cloudinary } from 'cloudinary';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class CloudinaryService {
  constructor(private readonly configService: ConfigService) {
    // Configure Cloudinary credentials from environment variables
    cloudinary.config({
      cloud_name: this.configService.get<string>('CLOUDINARY_CLOUD_NAME'),
      api_key: this.configService.get<string>('CLOUDINARY_API_KEY'),
      api_secret: this.configService.get<string>('CLOUDINARY_API_SECRET'),
    });
  }

  async uploadImage(file: Express.Multer.File): Promise<string> {
    if (!file) {
      throw new BadRequestException('No file provided for upload.');
    }

    return new Promise((resolve, reject) => {
      cloudinary.uploader.upload_stream(
        { folder: 'nest_todo_photos' },
        (error, result) => {
          if (error) return reject(error);
          resolve(result?.secure_url || '');
        }
      ).end(file.buffer);
    });
  }
}
```

#### Step 2: Implement File Upload inside a Controller
Apply the `FileInterceptor` directly to your controller routes:

```typescript
// src/modules/file/profile.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { CloudinaryService } from './cloudinary.service';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { CurrentUser } from '../../common/decorators/current-user.decorator';
import { PrismaService } from '../../prisma/prisma.service';

@Controller('api/v1/profile')
@UseGuards(JwtAuthGuard)
export class ProfileController {
  constructor(
    private readonly cloudinaryService: CloudinaryService,
    private readonly prisma: PrismaService,
  ) {}

  @Post('avatar')
  @UseInterceptors(FileInterceptor('file')) // Intercept multipart payloads matching "file" key
  async uploadAvatar(
    @UploadedFile() file: Express.Multer.File,
    @CurrentUser('id') userId: bigint,
  ) {
    // 1. Upload to Cloudinary
    const url = await this.cloudinaryService.uploadImage(file);

    // 2. Update user profile photo field in database
    await this.prisma.user.update({
      where: { id: userId },
      data: { profile_photo: url },
    });

    return {
      message: 'Avatar image uploaded successfully.',
      data: { url },
    };
  }
}
```

#### Step 3: File Uploads with Structured Metadata (Eventra Pattern)
In your Eventra-Backend, when creating a client or updating a profile, you send a file along with stringified JSON metadata using a key named `data` (e.g., `req.body.data`). The Express router intercepts this and manually parses it:
```typescript
// Express Eventra Pattern
router.post("/create-client", multerUpload.single('file'), (req, res, next) => {
    req.body = userValidation.createClient.parse(JSON.parse(req.body.data));
    return userController.createClient(req, res, next);
});
```

To replicate this cleanly in NestJS, create a reusable `ParseJsonPipe` to automatically deserialize the text fields from multipart forms before passing them to the controller:

```typescript
// src/common/pipes/parse-json.pipe.ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { plainToInstance } from 'class-transformer';
import { validateSync } from 'class-validator';

@Injectable()
export class ParseJsonPipe implements PipeTransform {
  constructor(private readonly dtoClass?: any) {}

  transform(value: any) {
    if (typeof value !== 'string') {
      return value;
    }
    
    let parsed: any;
    try {
      parsed = JSON.parse(value);
    } catch {
      throw new BadRequestException('Validation failed. The metadata field must be a valid JSON string.');
    }

    if (this.dtoClass) {
      const validated = plainToInstance(this.dtoClass, parsed);
      const errors = validateSync(validated);
      if (errors.length > 0) {
        throw new BadRequestException({
          message: 'Validation failed',
          errors: errors.toString(),
        });
      }
      return validated;
    }

    return parsed;
  }
}
```

Now use the custom pipe inside your Controller, passing the `RegisterDto` class to `ParseJsonPipe` for automatic validation mapping:

```typescript
// src/modules/auth/client.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, Body } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { ParseJsonPipe } from '../../common/pipes/parse-json.pipe';
import { RegisterDto } from './dto/register.dto'; // Reuses RegisterDto schemas

@Controller('api/v1/clients')
export class ClientController {
  @Post('create-client')
  @UseInterceptors(FileInterceptor('file')) // Intercept multipart 'file' field
  async createClient(
    @UploadedFile() file: Express.Multer.File,
    @Body('data', new ParseJsonPipe(RegisterDto)) clientDto: RegisterDto, // Parse & validate the 'data' form field automatically
  ) {
    // clientDto is now a fully validated RegisterDto object!
    return {
      success: true,
      message: 'Client created successfully with upload.',
      data: {
        fileName: file?.originalname,
        clientName: clientDto.name,
      },
    };
  }
}
```
This avoids manual inline `JSON.parse` blocks inside your routing controllers and integrates seamlessly with NestJS's validation pipeline.

---


### 4.11 Email Dispatching (Nodemailer & EJS/HTML)
*   **What is it?**
    It is an email transmission module that integrates Nodemailer (an SMTP mail transport client) and EJS (Embedded JavaScript templates) to compile dynamic HTML layouts and deliver email alerts.
*   **Why is it used?**
    To send clean, formatted HTML emails (such as onboarding notices, invoices, and password resets) with variable data. Instead of concatenating HTML strings inside service files (which is ugly and unmaintainable), EJS templates keep layout designs separate from business logic. In NestJS, this is wrapped inside a dedicated provider service that can be injected anywhere email delivery is needed.
*   **How is it used?**
    By creating EJS templates in a directory, compiling them with EJS variables, and passing the HTML output to a Nodemailer transporter instance:

#### Step 1: Create an Email Template
Create an EJS template at `src/common/templates/todo-notification.ejs`:
```html
<!DOCTYPE html>
<html>
<head>
    <title>New Task Assigned</title>
</head>
<body>
    <h1>Hello, <%= name %>!</h1>
    <p>A new task titled "<strong><%= taskTitle %></strong>" has been assigned to your workspace.</p>
    <p>Please log in to complete the assignment before your target deadline.</p>
</body>
</html>
```

#### Step 2: Implement the Email Service
This service loads templates, compiles them with dynamic variables, and sends emails via SMTP.

```typescript
// src/common/services/email.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as nodemailer from 'nodemailer';
import * as ejs from 'ejs';
import * as path from 'path';

@Injectable()
export class EmailService {
  private readonly transporter: nodemailer.Transporter;
  private readonly logger = new Logger(EmailService.name);

  constructor(private readonly configService: ConfigService) {
    this.transporter = nodemailer.createTransport({
      host: this.configService.get<string>('SMTP_HOST'),
      port: this.configService.get<number>('SMTP_PORT', 587),
      secure: this.configService.get<boolean>('SMTP_SECURE', false),
      auth: {
        user: this.configService.get<string>('SMTP_USER'),
        pass: this.configService.get<string>('SMTP_PASS'),
      },
    });
  }

  async sendTaskAssignmentEmail(to: string, name: string, taskTitle: string) {
    try {
      const templatePath = path.join(__dirname, '../../templates/todo-notification.ejs');
      
      // Render EJS to HTML
      const html = await ejs.renderFile(templatePath, { name, taskTitle });

      await this.transporter.sendMail({
        from: this.configService.get<string>('SMTP_FROM', '"Todo Team" <noreply@todo.com>'),
        to,
        subject: 'Task Assignment Notice',
        html,
      });

      this.logger.log(`Assignment notification email successfully sent to: ${to}`);
    } catch (error) {
      this.logger.error(`Failed to send assignment notification email: ${error.message}`);
    }
  }
}
```

---

### 4.12 Rate Limiting (Throttler)
*   **What is it?**
    It is a security guard system provided by `@nestjs/throttler` that monitors incoming client request frequencies per IP address and automatically rejects clients exceeding specific thresholds.
*   **Why is it used?**
    To protect authentication endpoints from brute-force password guessing, block API scrapers, and prevent denial-of-service (DoS) resource abuse. Instead of writing custom route-level throttle maps in Express, NestJS provides a global throttler module that can be injected at the application level and overridden on specific controllers (e.g. strict limits on login, loose limits on static lists).
*   **How is it used?**
    By registering `ThrottlerModule` in your `AppModule` imports and binding `ThrottlerGuard` to the `APP_GUARD` token to apply it globally:

Config the Throttler in your `AppModule`:
```typescript
// src/app.module.ts (Throttling Wireframe)
import { Module } from '@nestjs/common';
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000, // Time to live (milliseconds)
      limit: 10,   // Limit requests per TTL window
    }]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard, // Applies rate-limiting globally to all controller routes
    },
  ],
})
export class AppModule {}
```

---

## 5. Testing in NestJS: Unit Tests & End-to-End (E2E) Tests
*   **What is it?**
    It is a verification system that checks the correctness of your application code. It uses **Jest** as the runner and includes:
    1.  **Unit Tests**: Isolating a single class or service by mocking all database and API dependencies.
    2.  **End-to-End (E2E) Tests**: Launching a complete NestJS application context to run HTTP requests against testing endpoints.
*   **Why is it used?**
    Testing guarantees that code changes don't introduce bugs or break existing workflows (like pricing calculations, validation logic, or role checking). In Express, testing requires manually configuring servers and mocking database libraries. NestJS provides a dedicated `@nestjs/testing` package that creates a simulated module container (`Test.createTestingModule`), making mocking and HTTP assertions straightforward.
*   **How is it used?**
    Create test suites next to your files with a `.spec.ts` filename (for unit tests) or in a dedicated `test/` directory (for E2E tests), and execute them:

### 5.1 Unit Testing services
Create test suites next to your services, using a `.spec.ts` filename. Use Jest to mock dependencies (like `PrismaService` or `EmailService`):

```typescript
// src/modules/todo/todo.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { TodoService } from './todo.service';
import { PrismaService } from '../../prisma/prisma.service';
import { BadRequestException } from '@nestjs/common';

const mockPrisma = {
  todoList: {
    findUnique: jest.fn(),
    update: jest.fn(),
  },
  todoItem: {
    create: jest.fn(),
  },
  auditLog: {
    create: jest.fn(),
  },
  $transaction: jest.fn((callback) => callback(mockPrisma)), // Mock transactional callbacks
};

describe('TodoService Unit Tests', () => {
  let service: TodoService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        TodoService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile();

    service = module.get<TodoService>(TodoService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('should throw BadRequestException if list item quota is exceeded', async () => {
    // Simulate a list with 10 existing items
    mockPrisma.todoList.findUnique.mockResolvedValue({
      id: BigInt(1),
      owner_id: BigInt(2),
      total_todos: 10,
    });

    await expect(
      service.createTodoItem(BigInt(1), { title: 'New task' }, BigInt(2))
    ).rejects.toThrow(BadRequestException);
  });

  it('should successfully create TodoItem when within limits', async () => {
    mockPrisma.todoList.findUnique.mockResolvedValue({
      id: BigInt(1),
      owner_id: BigInt(2),
      total_todos: 4,
    });
    mockPrisma.todoItem.create.mockResolvedValue({ id: BigInt(10), title: 'Test Item' });

    const result = await service.createTodoItem(BigInt(1), { title: 'Test Item' }, BigInt(2));
    
    expect(result.title).toBe('Test Item');
    expect(mockPrisma.todoList.update).toHaveBeenCalled();
  });
});
```
To execute unit tests:
```bash
npm run test
```

### 5.2 End-to-End (E2E) Testing
E2E tests boot up the Nest application context and send HTTP requests to test the app layer.

Create E2E tests in a separate folder:

```typescript
// test/todos.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Todo Module Controller (E2E)', () => {
  let app: INestApplication;
  let prisma: PrismaService;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    prisma = app.get<PrismaService>(PrismaService);

    // Seed temporary user and retrieve JWT token
    const registerResponse = await request(app.getHttpServer())
      .post('/api/v1/auth/register')
      .send({
        email: 'e2e@todo.com',
        name: 'E2E User',
        password: 'securePassword123',
        role: 'CREATOR',
      });

    const loginResponse = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({
        email: 'e2e@todo.com',
        password: 'securePassword123',
      });

    authToken = loginResponse.body.accessToken || loginResponse.headers['set-cookie'][0].split(';')[0].split('=')[1];
  });

  afterAll(async () => {
    // Clean up test data
    await prisma.user.delete({ where: { email: 'e2e@todo.com' } });
    await app.close();
  });

  it('/api/v1/todos/lists (POST) - successfully creates a todo list', () => {
    return request(app.getHttpServer())
      .post('/api/v1/todos/lists')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ title: 'Workspace List', description: 'Test description' })
      .expect(201)
      .expect((res) => {
        expect(res.body.success).toBe(true);
        expect(res.body.data.title).toBe('Workspace List');
      });
  });

  it('/api/v1/todos/lists (POST) - blocks request if unauthenticated', () => {
    return request(app.getHttpServer())
      .post('/api/v1/todos/lists')
      .send({ title: 'Unauthorized List' })
      .expect(401);
  });
});
```
To execute E2E tests:
```bash
npm run test:e2e
```

---

## 6. Advanced Topics: Transitioning from Todo to GasPay

The miniature Todo application covers all core NestJS lifecycle stages (Middlewares, Guards, Pipes, Interceptors, and Filters) and Prisma integration. However, to implement the full **GasPay** utility billing platform (as specified in your [readme.md](file:///e:/ARCHIOTECH/Learning-Nest/readme.md)), you need to know a few additional advanced patterns.

---

### 6.1 Subdomain-Based Multi-Tenancy (tenant_id Isolation)
*   **What is it?**
    It is a software architecture pattern where a single backend server serving multiple companies (tenants) segregates and isolates data logically. Here, each tenant has a unique subdomain (e.g. `metro.gaspay.cc` vs. `delta.gaspay.cc`), and the application determines context dynamically using Node's `AsyncLocalStorage` to set context boundaries on all database queries.
*   **Why is it used?**
    In Express, you typically extract the tenant ID from headers or subdomains on every controller call, passing the parameter down into services manually (e.g. `usersService.getUser(id, tenantId)`). This is repetitive and error-prone; forgetting to add `where: { tenant_id }` in a single query leaks tenant data to other clients. Combining Express middleware and `AsyncLocalStorage` resolves and binds the tenant ID once per request, automatically applying it across all service database query contexts.
*   **How is it used?**
    By declaring an `AsyncLocalStorage` context, setting it in a Nest Middleware, and retrieving it inside your database queries:

#### Step 1: Create the Tenant Storage Context
```typescript
// src/common/context/tenant.context.ts
import { AsyncLocalStorage } from 'async_hooks';

export interface TenantContext {
  tenantId: bigint;
  companyName: string;
}

export const tenantStorage = new AsyncLocalStorage<TenantContext>();
```

#### Step 2: Implement the Multi-Tenancy Middleware
This middleware runs on every request, parses the subdomain, queries PostgreSQL to find the tenant, and runs the rest of the request within the async storage scope:

```typescript
// src/common/middleware/tenant.middleware.ts
import { Injectable, NestMiddleware, BadRequestException, NotFoundException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { PrismaService } from '../../prisma/prisma.service';
import { tenantStorage } from '../context/tenant.context';

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly prisma: PrismaService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const host = req.headers.host || ''; // e.g., "metro.gaspay.cc"
    const hostParts = host.split('.');
    
    // Resolve subdomain
    const subdomain = hostParts[0];
    if (!subdomain || hostParts.length < 2) {
      throw new BadRequestException('Request must include a valid tenant subdomain.');
    }

    // Query database for tenant mapping
    const tenant = await this.prisma.tenant.findUnique({
      where: { subdomain },
    });

    if (!tenant) {
      throw new NotFoundException(`Tenant subdomain "${subdomain}" is not registered.`);
    }

    // Run the request execution path inside the AsyncLocalStorage scope
    tenantStorage.run({ tenantId: tenant.id, companyName: tenant.company_name }, () => {
      // Also attach to request object for Express-familiar access in controllers
      req['tenantId'] = tenant.id;
      next();
    });
  }
}
```

#### Step 3: Consume Tenant Context in Services
Any database query in your application can now query the tenant ID from storage:

```typescript
// src/modules/billing/billing.service.ts (Snippet)
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { tenantStorage } from '../../common/context/tenant.context';

@Injectable()
export class BillingService {
  constructor(private readonly prisma: PrismaService) {}

  async getBillsByBuilding(buildingId: bigint) {
    const store = tenantStorage.getStore();
    const tenantId = store?.tenantId; // Automatically populated by middleware

    return this.prisma.bill.findMany({
      where: {
        building_id: buildingId,
        tenant_id: tenantId, // Strict database isolation boundary enforced
      },
    });
  }
}
```

---

### 6.2 Database Seeding on Server Startup
Like Eventra's `seedSuperAdmin` script run in `server.ts` during startup, NestJS allows you to hook into lifecycle events. By implementing `OnApplicationBootstrap`, your seeding service will execute immediately after the application starts.

```typescript
// src/prisma/seeder.service.ts
import { Injectable, OnApplicationBootstrap, Logger } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import * as bcrypt from 'bcrypt';
import { Role } from '@prisma/client';

@Injectable()
export class SeederService implements OnApplicationBootstrap {
  private readonly logger = new Logger(SeederService.name);

  constructor(private readonly prisma: PrismaService) {}

  async onApplicationBootstrap() {
    this.logger.log('Checking database status and seeding system settings...');
    await this.seedAdmin();
  }

  private async seedAdmin() {
    try {
      const adminExists = await this.prisma.user.findFirst({
        where: { role: Role.ADMIN },
      });

      if (adminExists) {
        this.logger.log('Database verification complete: Admin account verified.');
        return;
      }

      // Create initial Admin credentials
      const passwordHash = await bcrypt.hash('Admin@12345', 10);
      const admin = await this.prisma.user.create({
        data: {
          email: 'admin@gmail.com',
          name: 'Super Admin',
          password_hash: passwordHash,
          role: Role.ADMIN,
        },
      });

      this.logger.log(`Security: Initial admin seeded successfully: ${admin.email}`);
    } catch (error) {
      this.logger.error('Database seeding failed:', error.message);
    }
  }
}
```
Register the `SeederService` inside `PrismaModule` as a provider to enable auto-running.

---

### 6.3 Soft Deletes in Prisma using Client Extensions
GasPay specifies that important records (such as Branches, Buildings, Flats, and Meters) should support **soft deletes** (`is_deleted = true`). In NestJS, we can write a Prisma client extension that intercepts all query methods and automatically filters out deleted items.

Refactor your `PrismaService` to use a client extension:

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  // Expose an extended client that automatically filters soft deleted rows
  readonly extended = this.$extends({
    query: {
      $allModels: {
        async findMany({ model, args, query }) {
          // If the model schema contains 'is_deleted' field, automatically filter it
          if (args.where && !('is_deleted' in args.where)) {
            args.where = { ...args.where, is_deleted: false };
          }
          return query(args);
        },
        async findFirst({ model, args, query }) {
          if (args.where && !('is_deleted' in args.where)) {
            args.where = { ...args.where, is_deleted: false };
          }
          return query(args);
        },
      },
    },
  });

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```
Now, call `this.prisma.extended.building.findMany(...)` in your services to query only active, non-deleted buildings.

---

### 6.4 Reusable Pagination Utility
To implement Eventra's consistent pagination pattern across list endpoints in NestJS, create a clean pagination helper:

```typescript
// src/common/utils/pagination.helper.ts
export interface PaginationOptions {
  page?: string;
  limit?: string;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}

export interface PaginationResult {
  skip: number;
  take: number;
  orderBy: Record<string, 'asc' | 'desc'>;
}

export interface MetaPayload {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

export class PaginationHelper {
  static getPaginationOptions(options: PaginationOptions): PaginationResult {
    const page = Math.max(Number(options.page) || 1, 1);
    const limit = Math.max(Number(options.limit) || 10, 1);
    const skip = (page - 1) * limit;

    const sortBy = options.sortBy || 'created_at';
    const sortOrder = options.sortOrder || 'desc';

    return {
      skip,
      take: limit,
      orderBy: { [sortBy]: sortOrder },
    };
  }

  static getMeta(page: number, limit: number, total: number): MetaPayload {
    return {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    };
  }
}
```

#### Consuming the Helper in a Controller:
```typescript
@Get()
async getTodos(
  @Query('page') page: string,
  @Query('limit') limit: string,
  @Query('sortBy') sortBy: string,
  @Query('sortOrder') sortOrder: 'asc' | 'desc',
) {
  const { skip, take, orderBy } = PaginationHelper.getPaginationOptions({ page, limit, sortBy, sortOrder });
  
  const [todos, total] = await this.prisma.$transaction([
    this.prisma.todoItem.findMany({ skip, take, orderBy }),
    this.prisma.todoItem.count(),
  ]);

  return {
    meta: PaginationHelper.getMeta(Number(page) || 1, take, total),
    data: todos,
  };
}
```

  return {
    meta: PaginationHelper.getMeta(Number(page) || 1, take, total),
    data: todos,
  };
}
```

---

### 6.5 Express Higher-Order Helpers in NestJS (catchAsync & sendResponse)
In your Express codebase, you rely heavily on two utility wrappers: `catchAsync` (to forward asynchronous route failures to `next()`) and `sendResponse` (to write uniform success JSON outputs).

Here is how their functionality is translated in NestJS:

#### A. The `catchAsync` Helper is Obsolete
In Express, any async error that is not caught inside a `try/catch` block will crash the process or leave the request hanging, unless you wrap it:
```typescript
// Express Style
const successPayment = catchAsync(async (req, res) => { ... });
```
In NestJS, **you do not need `catchAsync`**. NestJS handles JavaScript promises natively under the hood. If a controller method returns a Promise or an Observable, NestJS wraps it automatically. If it resolves, the data is sent back; if it rejects (throws an error), NestJS catches it and routes it directly to your `GlobalExceptionFilter` (from Section 4.8).
```typescript
// NestJS Style - Clean, standard async-await without manual wrapping!
@Post('success')
async successPayment(@Body() payload: any) {
  return this.paymentService.processSuccess(payload);
}
```

#### B. The `sendResponse` Helper is Replaced by Interceptors
In Express, you invoke a custom function to format all success responses:
```typescript
// Express Style
sendResponse(res, {
    statusCode: 200,
    success: true,
    message: "Payment Validated Successfully",
    data: null,
});
```
In NestJS, you map responses globally using **Interceptors**. The `TransformInterceptor` we wrote in **Section 4.7.2** intercept all outgoing return values from your controllers, automatically wrapping them inside the standard JSON envelope structure:
```json
{
  "success": true,
  "statusCode": 200,
  "message": "Request completed successfully.",
  "data": { ... }
}
```
This means your controllers can simply return raw data payloads, keeping your code cleaner and more focused.

---

### 6.6 Payment Gateway Integration: SSLCommerz in NestJS
Let's translate your SSLCommerz payment mechanism from `src/app/modules/sslCommerz` and `src/app/modules/payment` to NestJS.

#### Step 1: Create the SSLCommerz Gateway Service
This service handles HTTP communications with the SSLCommerz payment APIs using Axios.

```typescript
// src/modules/payment/ssl-commerz.service.ts
import { Injectable, BadRequestException, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios from 'axios';

export interface SSLCommerzPayload {
  amount: number;
  transactionId: string;
  name: string;
  email: string;
  phoneNumber: string;
  address: string;
}

@Injectable()
export class SslCommerzService {
  constructor(private readonly configService: ConfigService) {}

  // 1. Initialize checkout session
  async initiatePayment(payload: SSLCommerzPayload): Promise<any> {
    try {
      const data = {
        store_id: this.configService.get<string>('SSL_STORE_ID'),
        store_passwd: this.configService.get<string>('SSL_STORE_PASS'),
        total_amount: payload.amount,
        currency: 'BDT',
        tran_id: payload.transactionId,
        
        // Dynamic callback URLs pointing to your NestJS payment endpoints
        success_url: `${this.configService.get<string>('SSL_SUCCESS_BACKEND_URL')}?transactionId=${payload.transactionId}&amount=${payload.amount}&status=success`,
        fail_url: `${this.configService.get<string>('SSL_FAIL_BACKEND_URL')}?transactionId=${payload.transactionId}&amount=${payload.amount}&status=fail`,
        cancel_url: `${this.configService.get<string>('SSL_CANCEL_BACKEND_URL')}?transactionId=${payload.transactionId}&amount=${payload.amount}&status=cancel`,
        ipn_url: this.configService.get<string>('SSL_IPN_URL'),
        
        shipping_method: 'N/A',
        product_name: 'Utility Bill Payment',
        product_category: 'Billing',
        product_profile: 'general',
        cus_name: payload.name,
        cus_email: payload.email,
        cus_add1: payload.address,
        cus_city: 'Dhaka',
        cus_country: 'Bangladesh',
        cus_phone: payload.phoneNumber,
        ship_name: 'N/A',
        ship_add1: 'N/A',
        ship_country: 'N/A',
      };

      const response = await axios({
        method: 'POST',
        url: this.configService.get<string>('SSL_PAYMENT_API'),
        data: data,
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      });

      return response.data; // Contains gateway redirect URLs
    } catch (error) {
      throw new BadRequestException(`SSLCommerz Session Initialization Failed: ${error.message}`);
    }
  }

  // 2. Validate transactional integrity (IPN checks)
  async validatePayment(valId: string): Promise<any> {
    try {
      const storeId = this.configService.get<string>('SSL_STORE_ID');
      const storePass = this.configService.get<string>('SSL_STORE_PASS');
      const validationUrl = this.configService.get<string>('SSL_VALIDATION_API');

      const response = await axios({
        method: 'GET',
        url: `${validationUrl}?val_id=${valId}&store_id=${storeId}&store_passwd=${storePass}`,
      });

      return response.data;
    } catch (error) {
      throw new UnauthorizedException(`SSLCommerz Payment Validation Failed: ${error.message}`);
    }
  }
}
```

#### Step 2: Implement the Payment Logic Service
This service executes the transactional logic in database blocks.

```typescript
// src/modules/payment/payment.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { SslCommerzService } from './ssl-commerz.service';

@Injectable()
export class PaymentService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly sslCommerzService: SslCommerzService,
  ) {}

  // Process success callback
  async handleSuccess(query: Record<string, string>) {
    const transactionId = query.transactionId;
    
    // Look up the database record
    const bill = await this.prisma.bill.findUnique({
      where: { transaction_id: transactionId },
    });

    if (!bill) {
      throw new NotFoundException(`Invoice with transaction ${transactionId} not found.`);
    }

    if (bill.status === 'PAID') {
      return { success: true };
    }

    // Execute state changes inside a transaction block
    return this.prisma.$transaction(async (tx) => {
      // 1. Update bill status
      await tx.bill.update({
        where: { id: bill.id },
        data: { status: 'PAID', paid_at: new Date() },
      });

      // 2. Log Payment Ledger entry
      await tx.paymentLedger.create({
        data: {
          bill_id: bill.id,
          amount_paid: bill.total_amount,
          payment_method: 'SSLCOMMERZ',
          transaction_reference: transactionId,
        },
      });

      return { success: true };
    });
  }

  // Process failure callback
  async handleFailure(query: Record<string, string>) {
    const transactionId = query.transactionId;
    await this.prisma.bill.update({
      where: { transaction_id: transactionId },
      data: { status: 'FAILED' },
    });
    return { success: true };
  }
}
```

#### Step 3: Implement the Payment Gateway Controller
This controller processes redirect requests from the payment client and redirects users back to the frontend.

```typescript
// src/modules/payment/payment.controller.ts
import { Controller, Post, Get, Query, Body, Res, HttpCode, HttpStatus } from '@nestjs/common';
import { PaymentService } from './payment.service';
import { SslCommerzService } from './ssl-commerz.service';
import { Response } from 'express';
import { ConfigService } from '@nestjs/config';

@Controller('api/v1/payments')
export class PaymentController {
  constructor(
    private readonly paymentService: PaymentService,
    private readonly sslCommerzService: SslCommerzService,
    private readonly configService: ConfigService,
  ) {}

  // 1. Success Callback Redirect (SSLCommerz GET/POST callback)
  @Post('success')
  @HttpCode(HttpStatus.FOUND)
  async paymentSuccess(
    @Query() query: Record<string, string>,
    @Res() res: Response, // Request raw Express response to execute HTML redirections
  ) {
    const result = await this.paymentService.handleSuccess(query);
    const frontendUrl = this.configService.get<string>('FRONTEND_URL');

    if (result.success) {
      // Redirect users back to frontend transaction logs dashboard
      return res.redirect(`${frontendUrl}/billing/dashboard?status=success`);
    }
  }

  // 2. Failure Callback Redirect
  @Post('fail')
  @HttpCode(HttpStatus.FOUND)
  async paymentFail(
    @Query() query: Record<string, string>,
    @Res() res: Response,
  ) {
    await this.paymentService.handleFailure(query);
    const frontendUrl = this.configService.get<string>('FRONTEND_URL');
    return res.redirect(`${frontendUrl}/billing/dashboard?status=failed`);
  }

  // 3. IPN Webhook Verification (SSLCommerz validation server ping)
  @Post('ipn')
  @HttpCode(HttpStatus.OK)
  async paymentIPN(@Body() body: any) {
    // Validate request parameter parameters with SSLCommerz servers
    const validationResult = await this.sslCommerzService.validatePayment(body.val_id);
    
    if (validationResult.status === 'VALID' || validationResult.status === 'VALIDATED') {
      await this.paymentService.handleSuccess({ transactionId: body.tran_id });
    }
    
    return { success: true };
  }
}
  }
}
```

---

### 6.7 Background Tasks & Cron Scheduling (Late Fee Processing)
*   **What is it?**
    It is a background scheduling manager provided by NestJS (`@nestjs/schedule`) that automatically triggers specific service methods at designated intervals (using crontab syntax or cron expressions) on the server process thread.
*   **Why is it used?**
    In Express or standard Node applications, scheduling background tasks requires installing external libraries (like `node-cron` or `agenda`), setting up custom runners, and manually booting task runner processes. NestJS provides a built-in decorator-based scheduler, allowing you to easily run background tasks (like nightly bill updates or system sanity reports) on the primary Node process.
*   **How is it used?**
    By importing `ScheduleModule.forRoot()` in your `AppModule` and applying the `@Cron()` decorator to target methods:

In NestJS, you can implement this with the official scheduler package `@nestjs/schedule`.

#### Step 1: Install Scheduler Package
```bash
npm install @nestjs/schedule
```

#### Step 2: Register ScheduleModule in AppModule
```typescript
// src/app.module.ts (Snippet)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(), // Registers the scheduler runner across the app
  ],
})
export class AppModule {}
```

#### Step 3: Implement the Overdue Bill Cron Job
Create an injectable service and use the `@Cron()` decorator to schedule task execution automatically. NestJS provides pre-built cron expressions for common intervals.

```typescript
// src/modules/billing/late-fee.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class LateFeeService {
  private readonly logger = new Logger(LateFeeService.name);

  constructor(private readonly prisma: PrismaService) {}

  // Executes every day at midnight (00:00:00) server time
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  async applyLateFees() {
    this.logger.log('--- STARTING OVERDUE BILLS CORRECTION SCAN ---');
    const today = new Date();

    try {
      // Find all UNPAID bills where the due date is in the past, and no late fee has been applied yet
      const overdueBills = await this.prisma.bill.findMany({
        where: {
          status: 'UNPAID',
          due_date: { lt: today },
          late_fee_applied: false,
        },
      });

      if (overdueBills.length === 0) {
        this.logger.log('No new overdue bills found. Execution complete.');
        return;
      }

      this.logger.log(`Found ${overdueBills.length} overdue bills. Commencing penalty calculation...`);

      // Run corrections in a database transaction block
      await this.prisma.$transaction(async (tx) => {
        for (const bill of overdueBills) {
          // E.g. Apply a flat penalty fee of 150 BDT, or query volumetric percentage math
          const penaltyFee = 150.00;
          const newTotal = Number(bill.total_amount) + penaltyFee;

          await tx.bill.update({
            where: { id: bill.id },
            data: {
              late_fee_applied: true,
              total_amount: newTotal,
              
              // Record penalty logs inside the double-entry transactional ledger
              ledger_records: {
                create: {
                  amount: penaltyFee,
                  entry_type: 'DEBIT',
                  description: 'Late Fee Interest Penalty Applied',
                },
              },
            },
          });
        }
      });

      this.logger.log(`Success: Applied overdue late fees to ${overdueBills.length} bills.`);
    } catch (error) {
      this.logger.error(`Late fee execution failed: ${error.message}`);
    }
  }
}
```
This runs entirely in the background on the Node server thread, removing the need to configure external server-level crontabs.

---

### 6.8 Asynchronous Outbox Worker (The Notification Queue)
*   **What is it?**
    It is a background messaging worker that operates on the **Transactional Outbox Pattern**. It periodically polls a database table (e.g. `notification_queue`), processes unsent records (such as email or SMS notifications), and dispatches them asynchronously using external communication services.
*   **Why is it used?**
    In Express and NestJS, making network requests to external APIs (like sending an SMS or email via Nodemailer) directly inside controller route handlers blocks execution. If the third-party API is slow or offline, your user's request hangs, leading to a poor user experience. Instead, we write notifications into a database outbox table within our main transaction. The outbox worker runs in the background to handle the transmission, retry logic, and logging without blocking the client.
*   **How is it used?**
    By defining an outbox queue table in Prisma, implementing a polling service decorated with `@Interval()`, and executing the dispatches:

#### Step 1: Design the Database Queue Model
Ensure your Prisma schema includes a table to hold pending messages:
```prisma
model NotificationQueue {
  id         BigInt    @id @default(autoincrement())
  recipient  String    // e.g. Phone number or Email
  message    String    // Text payload
  status     String    // "PENDING", "SENT", "FAILED"
  attempts   Int       @default(0)
  error_logs String?
  created_at DateTime  @default(now())
  updated_at DateTime  @updated_at
  
  @@map("notification_queue")
}
```

#### Step 2: Implement the Outbox Worker Service
This worker runs periodically (every 10 seconds), fetches unsent notifications, dispatches them concurrently, and updates status values:

```typescript
// src/common/workers/outbox.worker.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { Interval } from '@nestjs/schedule';
import axios from 'axios';

@Injectable()
export class OutboxWorker {
  private readonly logger = new Logger(OutboxWorker.name);
  private isProcessing = false;

  constructor(private readonly prisma: PrismaService) {}

  // Executes automatically every 10 seconds
  @Interval(10000)
  async processNotificationQueue() {
    // Prevent concurrency overlap if the previous processing loop is still running
    if (this.isProcessing) return;
    this.isProcessing = true;

    try {
      // 1. Fetch pending notifications with less than 3 failed attempts
      const pendingJobs = await this.prisma.notificationQueue.findMany({
        where: {
          status: 'PENDING',
          attempts: { lt: 3 },
        },
        take: 10, // Process in batches of 10
      });

      if (pendingJobs.length === 0) {
        this.isProcessing = false;
        return;
      }

      this.logger.log(`Outbox Queue processing ${pendingJobs.length} notifications...`);

      // 2. Dispatch notifications concurrently
      await Promise.all(
        pendingJobs.map(async (job) => {
          await this.prisma.notificationQueue.update({
            where: { id: job.id },
            data: { attempts: { increment: 1 } },
          });

          try {
            // E.g. Send SMS via external telecom gateway API
            await this.sendSmsGateway(job.recipient, job.message);

            // Update status on success
            await this.prisma.notificationQueue.update({
              where: { id: job.id },
              data: { status: 'SENT' },
            });
          } catch (error) {
            this.logger.error(`Failed to send outbox notification ${job.id}: ${error.message}`);
            
            // Mark as failed or leave pending for retry
            const status = job.attempts >= 2 ? 'FAILED' : 'PENDING';
            await this.prisma.notificationQueue.update({
              where: { id: job.id },
              data: { 
                status,
                error_logs: error.message 
              },
            });
          }
        }),
      );
    } catch (err) {
      this.logger.error(`Outbox worker loop failed: ${err.message}`);
    } finally {
      this.isProcessing = false;
    }
  }

  private async sendSmsGateway(recipient: string, message: string): Promise<void> {
    // Replaces this block with actual dynamic SMS provider integration (e.g. Twilio, Greenweb)
    await axios.post('https://api.sms-provider.com/send', {
      apiKey: 'DUMMY_KEY',
      to: recipient,
      text: message,
    });
  }
}
```

This pattern ensures that even if your billing engine creates 10,000 invoices at once, your endpoints remain lightning fast, and communication alerts are dispatched steadily in the background.

---

## 7. Summary: Checklist for Building GasPay

Use this checklist when building your **GasPay** multi-tenant billing platform:

1. **Initialize Prisma Database Schemas**: Configure standard tables (Branch, Building, Flat, Bills, etc.) referencing `tenant_id` for multi-tenant isolation.
2. **Install NestJS Global Modules**: Wire core modules (`PrismaModule`, `AuthModule`, `MailModule`, `ThrottlerModule`).
3. **Register Request Lifecycle Components globally in `main.ts`**:
   - `ValidationPipe` for input validation.
   - `GlobalExceptionFilter` to map Prisma constraints to HTTP responses.
   - `BigIntInterceptor` to serialize `BIGINT` primary keys to strings.
   - `TransformInterceptor` to wrap API responses in standard JSON envelopes.
4. **Implement Multi-Tenancy Middleware**: Extract tenant ID values from subdomains or headers and bind them to the request context or `AsyncLocalStorage`.
5. **Secure Routes using JWT and Role Guards**: Lock administrative routes behind role-based access checks.
6. **Implement Transactions for Billing Logic**: Wrap all billing processes (dues calculation, slab calculations, invoice generation, line items creation) in a transactional block to ensure database integrity.
7. **Protect Infrastructure using Throttling**: Limit API requests using Throttler to protect against denial of service or credential stuffing attacks.
8. **Verify System Integrity using Jest**: Write unit tests for pricing calculations and E2E tests for billing endpoints.


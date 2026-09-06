# Supabase JWT Authentication with Login API

আগের গাইডগুলোতে Supabase PostgreSQL দিয়ে CRUD API বানানো হয়েছিল। এবার সেই API-কে **Supabase Authentication** দিয়ে সুরক্ষিত করা হবে — মানে Supabase-এ login করে যে JWT token পাওয়া যায়, সেটা verify করে তবেই route access দেওয়া হবে, একটা custom **Guard** বানিয়ে।

---

## ধাপ ১: `jose` Install করা

JWT token verify করার জন্য এই package লাগবে:

```bash
npm i jose
```

---

## ধাপ ২: Supabase থেকে প্রয়োজনীয় তথ্য নেওয়া

Supabase project-এর **Settings → JWT Keys**-এ গিয়ে **Legacy JWT Secret** key **Reveal** করে copy করো।

![JWT Keys সেটিংস](https://github.com/user-attachments/assets/5cbc111f-01af-4395-93c2-15c9e87249d6)

![Legacy JWT Secret reveal করা](https://github.com/user-attachments/assets/b8e058f1-ef51-4dfb-aa0f-84975460dae6)

এছাড়াও **Settings → General → Project Settings**-এ গিয়ে **Project ID** copy করো (এটা দিয়েই project-এর URL বসবে)।

### `.env`

```bash
SUPABASE_JWT_SECRET=
SUPABASE_URL=
```

- `SUPABASE_JWT_SECRET` — Legacy JWT Secret বসাতে হবে (ভবিষ্যতে fallback বা অন্য কোনো verification পদ্ধতির জন্য রাখা, বর্তমান JWKS-ভিত্তিক Guard-এ এটা সরাসরি ব্যবহার হচ্ছে না)
- `SUPABASE_URL` — এই ফরম্যাটে বসবে: `https://<project-id>.supabase.co` (Project ID বসিয়ে)

---

## ধাপ ৩: `app.module.ts`-এ Config Global করা

> **নোট:** `ConfigModule.forRoot({ isGlobal: true })` — এভাবে `isGlobal: true` দিলে `ConfigModule` পুরো application জুড়ে available হয়ে যায়, প্রতিটা module-এ আলাদা করে import করা লাগবে না। যেহেতু Guard-টা `auth` নামের আলাদা folder-এ থাকবে, সেখানে `ConfigService` ব্যবহার করতে হলে এই global setting-টা দরকার।

### `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { EmployeeModule } from './employee/employee.module';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [ ConfigModule.forRoot({ isGlobal: true }), TypeOrmModule.forRoot({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    autoLoadEntities: true,
    synchronize: true,
  }), EmployeeModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

---

## ধাপ ৪: Guard তৈরি করা

```bash
nest g guard auth/supabase-auth
```

এই command চালালে `auth` নামের folder-এর ভিতরে `supabase-auth.guard.ts` তৈরি হবে।

### `supabase-auth.guard.ts`

```ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Request } from 'express';
import { createRemoteJWKSet, jwtVerify } from 'jose';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SupabaseAuthGuard implements CanActivate {
  private readonly JWKS: ReturnType<typeof createRemoteJWKSet>;

  constructor(private configService: ConfigService) {
    const supabaseUrl = this.configService.get<string>('SUPABASE_URL');
    this.JWKS = createRemoteJWKSet(
      new URL(`${supabaseUrl}/auth/v1/.well-known/jwks.json`)
    );
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const authHeader = request.headers['authorization'];
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedException('No token provided');
    }
    const token = authHeader.split(' ')[1];

    try {
      const { payload } = await jwtVerify(token, this.JWKS);
      request['user'] = payload;
      return true;
    } catch (error: unknown) {
      const message = error instanceof Error ? error.message : String(error);
      console.log('JWT VERIFY ERROR:', message);
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

### এই Guard কী করছে, ধাপে ধাপে

1. Constructor-এ `ConfigService` থেকে `SUPABASE_URL` নেয়, আর সেটা দিয়ে `createRemoteJWKSet` ব্যবহার করে একটা `JWKS` object তৈরি করে রাখে — এটা Supabase-এর `/auth/v1/.well-known/jwks.json` endpoint থেকে signing key(গুলো) fetch ও cache করে রাখে (প্রতি request-এ নতুন করে fetch করে না)
2. Request-এর `authorization` header বের করে
3. Header না থাকলে বা `Bearer ` দিয়ে শুরু না হলে → `UnauthorizedException('No token provided')`
4. Header থেকে `Bearer ` অংশটুকু বাদ দিয়ে আসল token বের করে (`split(' ')[1]`)
5. `jwtVerify(token, JWKS)` দিয়ে token verify করে — `jose` নিজে থেকেই token-এর header দেখে সঠিক public key বেছে নেয় JWKS থেকে, এবং signature ও expiry (`exp`) চেক করে
6. Verify সফল হলে decode হওয়া `payload`-টা `request['user']`-এ বসিয়ে দেয় — যাতে পরে controller/service-এ `request.user` দিয়ে সেই logged-in user-এর তথ্য পাওয়া যায়
7. Verify ব্যর্থ হলে (ভুল token, মেয়াদ শেষ, signature না মেলা ইত্যাদি) → error message log করে `UnauthorizedException('Invalid token')` throw করে

> **গুরুত্বপূর্ণ:** এই JWKS-ভিত্তিক approach তখনই কাজ করে যখন Supabase project-এর JWT signing **asymmetric (ES256/RS256)** হয়। প্রজেক্টের JWT algorithm Supabase Dashboard-এর **Settings → API → JWT Settings**-এ গিয়ে দেখে নেওয়া ভালো।

---

## ধাপ ৫: Controller-এ Guard বসানো

Route-টাকে সুরক্ষিত করতে `@UseGuards(SupabaseAuthGuard)` decorator বসাতে হবে।

### `employee.controller.ts`

```ts
import { Body, Controller, Delete, Get, Param, Post, Put, Query, UseGuards } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { Employee } from './employees.entity';
import { SupabaseAuthGuard } from '../auth/supabase-auth/supabase-auth.guard';

@Controller('employee')
export class EmployeeController {
    constructor(private readonly employeeService: EmployeeService) {}

    @Post()
    async createEmployee(@Body() employeeData: Partial<Employee>) {
        return this.employeeService.createEmployee(employeeData);
    }

    @Get()
    @UseGuards(SupabaseAuthGuard)
    async findAllEmployees(): Promise<Employee[]> {
        return this.employeeService.findAllEmployees();
    }

    @Get('filter')
    async filterEmployees(
        @Query('name') name?: string,
        @Query('department') department?: string,
    ): Promise<Employee[]> {
        return this.employeeService.search({ name, department });
    }

    @Get(':id')
    async findEmployeeById(@Param('id') id: number): Promise<Employee | null> {
        return this.employeeService.findOneEmployee(id);
    }

    // find by name
    @Get('name/:name')
    async findEmployeeByName(@Param('name') name: string): Promise<Employee[]> {
        return this.employeeService.findByName(name);
    }

    // search by keyword (partial match)
    @Get('search/:keyword')
    async searchEmployeeByKeyword(@Param('keyword') keyword: string): Promise<Employee[]> {
        return this.employeeService.searchByKeyword(keyword);
    }

    @Put(':id')
    async updateEmployee(
        @Param('id') id: number,
        @Body() updateData: Partial<Employee>): Promise<Employee> {
        return this.employeeService.update(id, updateData);
    }

    @Delete(':id')
    async deleteEmployee(@Param('id') id: number): Promise<{ message: string }> {
        return this.employeeService.delete(id);
    }
}
```

এখানে শুধু `findAllEmployees()` (মানে `GET /employee`) route-টাই `SupabaseAuthGuard` দিয়ে সুরক্ষিত — এটা **method-scoped** guard। বাকি route-গুলো (filter, id দিয়ে খোঁজা, name/keyword দিয়ে খোঁজা, update, delete) এখনো unprotected — চাইলে সবগুলো route-এই বা পুরো controller-এই (`@Controller`-এর উপরে বসিয়ে) guard লাগানো যায়।

---

## ধাপ ৬: Supabase-এ Test User তৈরি করা

Supabase dashboard-এ **Authentication** ট্যাবে ক্লিক করো।

![Authentication ট্যাব](https://github.com/user-attachments/assets/def141d7-a778-463d-a69b-7c52759f24ae)

**"Create new user"** বাটনে ক্লিক করো।

![Create new user বাটন](https://github.com/user-attachments/assets/cada2d2b-3c8c-4eaf-b116-c94e2a0d42f7)

Email/password দিয়ে user তৈরি করো।

![New user তৈরি করা](https://github.com/user-attachments/assets/6052c419-e82d-444a-b0d2-5de7328f297c)

User সফলভাবে তৈরি হয়ে গেলে এমন দেখাবে।

![User তৈরি সম্পন্ন](https://github.com/user-attachments/assets/9ddc88a4-df8f-4b4d-8dea-01aac4ac40c9)

---

## ধাপ ৭: Postman দিয়ে Login করে Token নেওয়া

Supabase project-এর **Project ID** copy করো।

![Project ID copy করা](https://github.com/user-attachments/assets/c4ac7bbc-8557-40dc-8f7b-cb515ca2c10f)

Postman-এ URL-এ সেই Project ID বসাও।

![Postman URL-এ Project ID বসানো](https://github.com/user-attachments/assets/35d3919e-a8d8-415b-af9a-bf9faf97ae99)

```bash
https://
```

URL-এর শেষে যোগ করো `.supabase.co/auth/v1/token?grant_type=password`, এবং Body-তে email/password দাও (login credentials, যেটা user তৈরির সময় দেওয়া হয়েছিল)।

![URL-এ endpoint যোগ ও Body দেওয়া](https://github.com/user-attachments/assets/10b0d4fa-936e-434b-b164-dacdb4003075)

Supabase project থেকে **API key** copy করো।

![API key copy করা](https://github.com/user-attachments/assets/b4a1542b-3a26-43f3-89a7-70ee6df1c14b)

Postman-এর **Headers**-এ `apikey` আর `Content-Type` যোগ করো।

![Headers-এ apikey ও Content-Type](https://github.com/user-attachments/assets/200894b7-d0b0-49b4-a454-b6332dfcb5d4)

**Send** বাটনে ক্লিক করলে response-এ `access_token` পাওয়া যাবে — এটাই আসল JWT token।

![Send করে response পাওয়া](https://github.com/user-attachments/assets/63e5cc67-b6c8-416a-8106-bdb724ce2abb)

---

## ধাপ ৮: সেই Token দিয়ে Protected Route Access করা

এখন `GET /employee` route (যেটাতে `SupabaseAuthGuard` বসানো আছে) call করার সময়, Postman-এর **Authorization** header-এ বসাও:

```
Bearer <access_token>
```

(`Bearer`-এর পর একটা স্পেস দিয়ে token বসাতে হবে)

এরপর **Send** ক্লিক করলে, Guard token verify করে request পাস করে দেবে, আর employee list ফেরত আসবে।

![Authorization-এ Bearer token বসানো](https://github.com/user-attachments/assets/c43c4701-d501-4d63-a43b-af20c9f337aa)

> **যদি token ছাড়াই বা ভুল token দিয়ে চেষ্টা করো**, তাহলে Guard-এর ভিতরের `UnauthorizedException` throw হবে এবং response হবে `401 Unauthorized` — এটাই প্রমাণ করে Guard ঠিকমতো কাজ করছে।

---

## যা পরিবর্তন হলো (এই আপডেটের সারসংক্ষেপ)

- `jsonwebtoken` বাদ দিয়ে **`jose`** package install করা হয়েছে
- `.env`-এ `SUPABASE_JWT_SECRET`-এর পাশাপাশি নতুন করে **`SUPABASE_URL`** যোগ করা হয়েছে
- Guard-এর ভিতরের verification পদ্ধতি বদলে গেছে: আগে `jwt.verify(token, secret)` (shared-secret) ব্যবহার হতো, এখন `jwtVerify(token, JWKS)` ব্যবহার হচ্ছে, যেখানে `JWKS` টা Supabase-এর `/auth/v1/.well-known/jwks.json` endpoint থেকে asymmetric public key নিয়ে আসে (`createRemoteJWKSet` দিয়ে)
- এই নতুন পদ্ধতি live test করে (Postman দিয়ে `GET /employee` call করে) `200 OK` সহ verify করা হয়েছে — token সঠিকভাবে verify হয়ে `request['user']` populate হচ্ছে

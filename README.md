## Supabase JWT Authentication with Login API


#### Install jsonwebtoken
```bash
npm i jsonwebtoken
```
---


>#### Note: project settings theke JWT Keys theke Legacy JWT Secret keys Reveal kore copy koro
><img width="306" height="574" alt="image" src="https://github.com/user-attachments/assets/5cbc111f-01af-4395-93c2-15c9e87249d6" />
><img width="1543" height="568" alt="image" src="https://github.com/user-attachments/assets/b8e058f1-ef51-4dfb-aa0f-84975460dae6" />

#### `.env`
```bash
SUPABASE_JWT_SECRET=''
```
---


>#### Note: add koro- ConfigModule.forRoot({isGlobal: true,})
#### `app.module.ts`
```bash
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


#### Create guard
```bash
nest g guard auth/supabase-auth
```
---


#### `supabase-auth.guard.ts`
```bash
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Observable } from 'rxjs';
import * as jwt from 'jsonwebtoken';
import { Request } from 'express';
import { ConfigService } from '@nestjs/config';


@Injectable()
export class SupabaseAuthGuard implements CanActivate {

  constructor(
    private configService: ConfigService
  ) {}

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const authHeader = request.headers['authorization'];
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedException('No token provided');
    }
    const token = authHeader.split(' ')[1];
    const jwtSecret = this.configService.get<string>('SUPABASE_JWT_SECRET');
    if (!jwtSecret) {
      throw new UnauthorizedException('JWT secret not configured');
    }
    try {
     const decode = jwt.verify(token, jwtSecret);
     request['user'] = decode; // Attach decoded token to request object
     return true;
    }
    catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```
---


>#### add- @UseGuards(SupabaseAuthGuard)
#### `employee.controller.ts`
```bash
import { Body, Controller, Delete, Get, Param, Post, Put, Query } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { Employee } from './employees.entity';

@Controller('employee')
export class EmployeeController {
    constructor(private readonly employeeService: EmployeeService) {}

    @Post()
    async createEmployee(@Body() employeeData: Partial<Employee>) {
        return this.employeeService.createEmployee(employeeData);
    }

    @UseGuards(SupabaseAuthGuard)
    @Get()
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
---


>#### Click Authentication
><img width="1599" height="559" alt="image" src="https://github.com/user-attachments/assets/def141d7-a778-463d-a69b-7c52759f24ae" />
>
>#
>#### click create new user
><img width="269" height="198" alt="image" src="https://github.com/user-attachments/assets/cada2d2b-3c8c-4eaf-b116-c94e2a0d42f7" />
>
>#
>#### create new user
><img width="408" height="396" alt="image" src="https://github.com/user-attachments/assets/6052c419-e82d-444a-b0d2-5de7328f297c" />
>
>#
>#### created
><img width="1599" height="765" alt="image" src="https://github.com/user-attachments/assets/9ddc88a4-df8f-4b4d-8dea-01aac4ac40c9" />
>
>#
>#### copy project ID
><img width="752" height="425" alt="image" src="https://github.com/user-attachments/assets/c4ac7bbc-8557-40dc-8f7b-cb515ca2c10f" />
>
>#
>#### postman e project id url e boshao
><img width="1154" height="56" alt="image" src="https://github.com/user-attachments/assets/35d3919e-a8d8-415b-af9a-bf9faf97ae99" />
>
>#
>#### url e add koro- .supabase.co/auth/v1/token?grant_type=password abong Body daw
><img width="1177" height="233" alt="image" src="https://github.com/user-attachments/assets/10b0d4fa-936e-434b-b164-dacdb4003075" />
>
>#
>#### api key copy koro-
><img width="1500" height="595" alt="image" src="https://github.com/user-attachments/assets/b4a1542b-3a26-43f3-89a7-70ee6df1c14b" />
>#
>#### Headers e apikey ar Content-Type daw
><img width="1166" height="277" alt="image" src="https://github.com/user-attachments/assets/200894b7-d0b0-49b4-a454-b6332dfcb5d4" />
>#
>#### Send click koro-
><img width="1193" height="697" alt="image" src="https://github.com/user-attachments/assets/63e5cc67-b6c8-416a-8106-bdb724ce2abb" />
>
>#
>#### Authorization e Bearer(space diye) token add koro- then Send click koro-
><img width="1173" height="285" alt="image" src="https://github.com/user-attachments/assets/c43c4701-d501-4d63-a43b-af20c9f337aa" />
>---

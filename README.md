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

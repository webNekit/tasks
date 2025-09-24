### Техническое задание: Student Management System (Система учета студентов)

**Цель проекта:** Разработать бэкенд-приложение для управления студентами учебного заведения с системой ролевой авторизации и отслеживанием статусов обучения.

**Функциональные требования:**

#### 1. Модуль аутентификации и авторизации
- **Регистрация и вход:** Эндпоинты `POST /auth/signup`, `POST /auth/login`
- **Роли пользователей:** Система должна поддерживать две роли:
  - `ADMIN` - полный доступ ко всем операциям
  - `USER` - может только просматривать информацию о студентах
- **Защита маршрутов:** Использовать JWT-аутентификацию с ролевыми guards

#### 2. Модуль управления студентами
- **Сущность Student** должна содержать:
  - `id` - идентификатор
  - `email` - электронная почта (уникальная)
  - `fullName` - полное имя
  - `studentId` - номер студенческого билета (уникальный)
  - `status` - статус обучения (Enum)
  - `groupId` - идентификатор группы
  - `createdAt`, `updatedAt` - даты создания и обновления

- **Статусы студента (Enum):**
  - `STUDYING` - обучается
  - `ACADEMIC_LEAVE` - академический отпуск
  - `EXPELLED` - отчислен
  - `NOT_STUDYING` - не обучается

#### 3. Модуль управления группами
- **Сущность Group** должна содержать:
  - `id` - идентификатор
  - `name` - название группы (например, "ПИ-201")
  - `faculty` - факультет
  - `createdAt`, `updatedAt`

#### 4. API эндпоинты

**Для студентов (`/students`):**
- `GET /students` - получить список всех студентов (доступно USER и ADMIN)
- `GET /students/:id` - получить студента по ID
- `POST /students` - создать нового студента (только ADMIN)
- `PATCH /students/:id` - обновить данные студента (только ADMIN)
- `PATCH /students/:id/status` - изменить статус студента (только ADMIN)
- `DELETE /students/:id` - удалить студента (только ADMIN)

**Для групп (`/groups`):**
- `GET /groups` - получить список групп
- `POST /groups` - создать группу (только ADMIN)

---

### Пример реализации функции изменения статуса студента

#### 1. Enum для статусов (`src/students/student-status.enum.ts`)
```typescript
export enum StudentStatus {
  STUDYING = 'STUDYING',
  ACADEMIC_LEAVE = 'ACADEMIC_LEAVE', 
  EXPELLED = 'EXPELLED',
  NOT_STUDYING = 'NOT_STUDYING'
}
```

#### 2. DTO для изменения статуса (`src/students/dto/update-status.dto.ts`)
```typescript
import { IsEnum } from 'class-validator';
import { StudentStatus } from '../student-status.enum';

export class UpdateStudentStatusDto {
  @IsEnum(StudentStatus)
  status: StudentStatus;
}
```

#### 3. Сервис студентов (`src/students/students.service.ts`)
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { StudentStatus } from './student-status.enum';
import { UpdateStudentStatusDto } from './dto/update-status.dto';

@Injectable()
export class StudentsService {
  constructor(private prisma: PrismaService) {}

  // Пример метода для изменения статуса
  async updateStudentStatus(id: number, updateStatusDto: UpdateStudentStatusDto) {
    const { status } = updateStatusDto;
    
    // Проверяем существует ли студент
    const student = await this.prisma.student.findUnique({
      where: { id }
    });

    if (!student) {
      throw new NotFoundException(`Student with ID ${id} not found`);
    }

    // Обновляем статус студента
    const updatedStudent = await this.prisma.student.update({
      where: { id },
      data: { 
        status,
        updatedAt: new Date() // Обновляем дату изменения
      },
      include: {
        group: true // Включаем информацию о группе в ответ
      }
    });

    // Здесь можно добавить дополнительную логику:
    // - Отправка уведомлений при отчислении
    // - Логирование изменений статуса
    // - Проверка бизнес-правил (например, нельзя изменить статус отчисленного)

    return updatedStudent;
  }

  // Другие методы сервиса...
  async findAll() {
    return this.prisma.student.findMany({
      include: {
        group: true
      },
      orderBy: {
        createdAt: 'desc'
      }
    });
  }

  async findOne(id: number) {
    const student = await this.prisma.student.findUnique({
      where: { id },
      include: { group: true }
    });

    if (!student) {
      throw new NotFoundException(`Student with ID ${id} not found`);
    }

    return student;
  }
}
```

#### 4. Контроллер студентов (`src/students/students.controller.ts`)
```typescript
import { 
  Controller, 
  Get, 
  Param, 
  Patch, 
  Body, 
  UseGuards,
  ParseIntPipe 
} from '@nestjs/common';
import { StudentsService } from './students.service';
import { UpdateStudentStatusDto } from './dto/update-status.dto';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { RolesGuard } from '../auth/roles.guard';
import { Roles } from '../auth/roles.decorator';
import { UserRole } from '../auth/user-role.enum';

@Controller('students')
@UseGuards(JwtAuthGuard, RolesGuard)
export class StudentsController {
  constructor(private readonly studentsService: StudentsService) {}

  @Get()
  @Roles(UserRole.USER, UserRole.ADMIN) // Доступно USER и ADMIN
  findAll() {
    return this.studentsService.findAll();
  }

  @Get(':id')
  @Roles(UserRole.USER, UserRole.ADMIN)
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.studentsService.findOne(id);
  }

  @Patch(':id/status')
  @Roles(UserRole.ADMIN) // Только ADMIN может менять статусы
  updateStatus(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateStatusDto: UpdateStudentStatusDto
  ) {
    return this.studentsService.updateStudentStatus(id, updateStatusDto);
  }

  // Другие методы контроллера...
}
```

#### 5. Ролевой Guard (`src/auth/roles.guard.ts`)
```typescript
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
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user; // user добавляется в JwtAuthGuard
    
    return requiredRoles.includes(user.role);
  }
}
```
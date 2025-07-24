---
applyTo: "**/k8s/**"
---

# Criando Aplicações Node.js/Nest.js para Testes em Kubernetes

Este documento fornece o código e as instruções para criar três aplicações de exemplo usando Node.js e Nest.js. O objetivo é ter um conjunto de microserviços para testar a implantação e orquestração em um ambiente Kubernetes local (Minikube/Kind).

## Estrutura do Projeto

Para cada aplicação, você pode usar o Nest CLI para gerar a estrutura básica:
```bash
# Instalar o Nest CLI
npm i -g @nestjs/cli

# Criar uma nova aplicação
nest new <nome-da-aplicacao>
```
Depois de criar a aplicação, você pode adicionar os módulos, controllers, services e testes conforme descrito abaixo.

---

## 1. Aplicação: `app-controle-acesso`

Esta aplicação será responsável pelo controle de acesso, validando credenciais e emitindo um JSON Web Token (JWT).

### 1.1. Estrutura de Arquivos

```
app-controle-acesso/
└── src/
    ├── app.module.ts
    ├── main.ts
    └── modules/
        └── auth/
            ├── dto/
            │   └── login.dto.ts
            ├── entities/
            │   └── user.entity.ts
            ├── strategies/
            │   └── jwt.strategy.ts
            ├── auth.controller.ts
            ├── auth.module.ts
            ├── auth.service.ts
            ├── auth.controller.spec.ts
            └── auth.service.spec.ts
```

### 1.2. Dependências

Instale as dependências necessárias para JWT e Passport:
```bash
npm install --save @nestjs/jwt @nestjs/passport passport passport-jwt
npm install --save-dev @types/passport-jwt
```

### 1.3. Código-Fonte

**`src/app.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { AuthModule } from './modules/auth/auth.module';

@Module({
  imports: [AuthModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

**`src/modules/auth/auth.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtStrategy } from './strategies/jwt.strategy';

@Module({
  imports: [
    PassportModule,
    JwtModule.register({
      secret: 'SECRET_KEY', // Em produção, use uma variável de ambiente
      signOptions: { expiresIn: '60m' },
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
})
export class AuthModule {}
```

**`src/modules/auth/auth.controller.ts`**
```typescript
import { Controller, Post, Body, UseGuards, Request } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  async login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }
}
```

**`src/modules/auth/auth.service.ts`**
```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { LoginDto } from './dto/login.dto';
import { User } from './entities/user.entity';

@Injectable()
export class AuthService {
  // Simulação de um banco de dados de usuários
  private readonly users: User[] = [
    new User(1, 'user@test.com', 'password123', ['user']),
  ];

  constructor(private readonly jwtService: JwtService) {}

  async validateUser(email: string, pass: string): Promise<any> {
    const user = this.users.find(u => u.email === email);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(loginDto: LoginDto) {
    const user = await this.validateUser(loginDto.email, loginDto.password);
    if (!user) {
      throw new UnauthorizedException('Credenciais inválidas');
    }
    const payload = { email: user.email, sub: user.id, roles: user.roles };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

**`src/modules/auth/strategies/jwt.strategy.ts`**
```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: 'SECRET_KEY', // Deve ser a mesma chave do módulo
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, email: payload.email, roles: payload.roles };
  }
}
```

### 1.4. Testes

**`src/modules/auth/auth.controller.spec.ts`**
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtService } from '@nestjs/jwt';

describe('AuthController', () => {
  let controller: AuthController;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [AuthController],
      providers: [AuthService, JwtService],
    }).compile();

    controller = module.get<AuthController>(AuthController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });
});
```

**`src/modules/auth/auth.service.spec.ts`**
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { AuthService } from './auth.service';
import { JwtService } from '@nestjs/jwt';

describe('AuthService', () => {
  let service: AuthService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [AuthService, JwtService],
    }).compile();

    service = module.get<AuthService>(AuthService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

---

## 2. Aplicação: `app-produto`

Esta aplicação gerencia o cadastro e o estoque de produtos.

### 2.1. Estrutura de Arquivos

```
app-produto/
└── src/
    ├── app.module.ts
    ├── main.ts
    └── modules/
        └── product/
            ├── dto/
            │   ├── create-product.dto.ts
            │   └── update-stock.dto.ts
            ├── entities/
            │   └── product.entity.ts
            ├── product.controller.ts
            ├── product.module.ts
            ├── product.service.ts
            ├── product.controller.spec.ts
            └── product.service.spec.ts
```

### 2.2. Código-Fonte

**`src/app.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { ProductModule } from './modules/product/product.module';

@Module({
  imports: [ProductModule],
})
export class AppModule {}
```

**`src/modules/product/product.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { ProductController } from './product.controller';
import { ProductService } from './product.service';

@Module({
  controllers: [ProductController],
  providers: [ProductService],
})
export class ProductModule {}
```

**`src/modules/product/product.controller.ts`**
```typescript
import { Controller, Get, Post, Body, Param, Patch, UseGuards } from '@nestjs/common';
import { ProductService } from './product.service';
import { CreateProductDto } from './dto/create-product.dto';
import { UpdateStockDto } from './dto/update-stock.dto';
// import { JwtAuthGuard } from '.../auth/guards/jwt-auth.guard'; // Supondo um guard JWT

@Controller('products')
// @UseGuards(JwtAuthGuard) // Proteger todas as rotas
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  @Post()
  create(@Body() createProductDto: CreateProductDto) {
    return this.productService.create(createProductDto);
  }

  @Get()
  findAll() {
    return this.productService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.productService.findOne(+id);
  }

  @Patch(':id/stock')
  updateStock(@Param('id') id: string, @Body() updateStockDto: UpdateStockDto) {
    return this.productService.updateStock(+id, updateStockDto.quantity);
  }
}
```

**`src/modules/product/product.service.ts`**
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateProductDto } from './dto/create-product.dto';
import { Product } from './entities/product.entity';

@Injectable()
export class ProductService {
  private readonly products: Product[] = [];
  private idCounter = 1;

  create(createProductDto: CreateProductDto): Product {
    const newProduct = new Product(
      this.idCounter++,
      createProductDto.name,
      createProductDto.price,
      createProductDto.stock,
    );
    this.products.push(newProduct);
    return newProduct;
  }

  findAll(): Product[] {
    return this.products;
  }

  findOne(id: number): Product {
    const product = this.products.find(p => p.id === id);
    if (!product) {
      throw new NotFoundException(`Produto com ID ${id} não encontrado`);
    }
    return product;
  }

  updateStock(id: number, quantity: number): Product {
    const product = this.findOne(id);
    product.stock += quantity;
    return product;
  }
}
```

**`src/modules/product/entities/product.entity.ts`**
```typescript
export class Product {
  constructor(
    public id: number,
    public name: string,
    public price: number,
    public stock: number,
  ) {}
}
```

---

## 3. Aplicação: `app-pdv`

Esta aplicação é o Ponto de Venda (PDV), responsável por registrar as vendas.

### 3.1. Estrutura de Arquivos

```
app-pdv/
└── src/
    ├── app.module.ts
    ├── main.ts
    └── modules/
        └── sale/
            ├── dto/
            │   └── create-sale.dto.ts
            ├── entities/
            │   └── sale.entity.ts
            ├── sale.controller.ts
            ├── sale.module.ts
            ├── sale.service.ts
            ├── sale.controller.spec.ts
            └── sale.service.spec.ts
```

### 3.2. Código-Fonte

**`src/app.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { SaleModule } from './modules/sale/sale.module';
import { HttpModule } from '@nestjs/axios';

@Module({
  imports: [SaleModule, HttpModule],
})
export class AppModule {}
```

**`src/modules/sale/sale.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { SaleController } from './sale.controller';
import { SaleService } from './sale.service';

@Module({
  imports: [HttpModule],
  controllers: [SaleController],
  providers: [SaleService],
})
export class SaleModule {}
```

**`src/modules/sale/sale.controller.ts`**
```typescript
import { Controller, Post, Body, UseGuards } from '@nestjs/common';
import { SaleService } from './sale.service';
import { CreateSaleDto } from './dto/create-sale.dto';
// import { JwtAuthGuard } from '.../auth/guards/jwt-auth.guard';

@Controller('sales')
// @UseGuards(JwtAuthGuard)
export class SaleController {
  constructor(private readonly saleService: SaleService) {}

  @Post()
  create(@Body() createSaleDto: CreateSaleDto) {
    // Em um cenário real, o ID do usuário viria do token JWT
    const userId = 1; 
    return this.saleService.create(userId, createSaleDto);
  }
}
```

**`src/modules/sale/sale.service.ts`**
```typescript
import { Injectable, NotFoundException, InternalServerErrorException } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import { CreateSaleDto } from './dto/create-sale.dto';
import { Sale } from './entities/sale.entity';

@Injectable()
export class SaleService {
  private readonly sales: Sale[] = [];
  private idCounter = 1;
  // URL do microserviço de produto - em produção, viria de uma config
  private readonly productServiceUrl = 'http://app-produto-service:3000/products';

  constructor(private readonly httpService: HttpService) {}

  async create(userId: number, createSaleDto: CreateSaleDto): Promise<Sale> {
    let totalAmount = 0;

    for (const item of createSaleDto.items) {
      try {
        // 1. Verifica o produto e o estoque no `app-produto`
        const productUrl = `${this.productServiceUrl}/${item.productId}`;
        const { data: product } = await firstValueFrom(this.httpService.get(productUrl));

        if (!product || product.stock < item.quantity) {
          throw new NotFoundException(`Produto ${item.productId} indisponível ou sem estoque.`);
        }

        // 2. Calcula o total
        totalAmount += product.price * item.quantity;

        // 3. Atualiza o estoque no `app-produto`
        await firstValueFrom(
          this.httpService.patch(`${productUrl}/stock`, {
            quantity: -item.quantity,
          }),
        );
      } catch (error) {
        console.error(error);
        throw new InternalServerErrorException('Falha ao processar a venda.');
      }
    }

    const newSale = new Sale(
      this.idCounter++,
      userId,
      createSaleDto.items,
      totalAmount,
      new Date(),
    );

    this.sales.push(newSale);
    return newSale;
  }
}
```

**`src/modules/sale/entities/sale.entity.ts`**
```typescript
export interface SaleItem {
  productId: number;
  quantity: number;
  price: number; // Preço no momento da venda
}

export class Sale {
  constructor(
    public id: number,
    public userId: number,
    public items: SaleItem[],
    public totalAmount: number,
    public createdAt: Date,
  ) {}
}
```

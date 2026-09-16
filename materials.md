MỤC LỤC
Mục 1: Controller, Route Shadowing & Specificity Strategy
Mục 2: Dependency Injection, Singleton & Custom Token (Symbol)
Mục 3: Modules Nâng Cao & Tính Singleton của Module
Mục 4: Middleware & Tích Hợp RequestId với LoggingInterceptor
Mục 5: Exception Filter Toàn Diện (HttpAdapterHost)
Mục 6: Pipes Chuẩn Production
Mục 7: Guards Toàn Diện, JWT & Mô Hình Mặc Định Đóng
Mục 8: Interceptors Toàn Diện & Chuẩn Hóa Response
Mục 9: Custom Decorators (@CurrentUser & @Auth Composite)
Mục 10: Tích Hợp Cuối - API POST /orders Đi Qua Đủ 7 Lớp
Mục 1: Controller, Route Shadowing & Specificity Strategy
1. Hiện tượng "Route Shadowing" là gì?
Trong Express, router kiểm tra các đường dẫn theo thứ tự khai báo từ trên xuống dưới. Nếu bạn đặt một route có tham số động (:id) trước route tĩnh (profile), route động sẽ "nuốt chửng" route tĩnh:

typescript


// ❌ SAI: Route tĩnh bị che khuất (Shadowed)
@Get(':id')
findOne(@Param('id') id: string) { ... } // Khi gọi GET /users/profile, nó hiểu :id = "profile"!
@Get('profile')
getProfile() { ... } // Route này KHÔNG BAO GIỜ được gọi tới!
// ✅ ĐÚNG: Luôn đặt route tĩnh ở trên, route động ở dưới
@Get('profile')
getProfile() { ... }
@Get(':id')
findOne(@Param('id') id: string) { ... }
2. Bật routeResolutionStrategy: 'specificity' trong NestJS
Để tránh lỗi do vô tình xếp nhầm thứ tự, NestJS cung cấp cơ chế tự động ưu tiên route theo độ cụ thể (Literal 
→
→ Parametric 
→
→ Wildcard). Cấu hình tại src/main.ts:

typescript


// src/main.ts
const app = await NestFactory.create(AppModule, {
  routeResolutionStrategy: 'specificity', // 👈 Tự động ưu tiên route cụ thể
});
3. Tại sao DTO bắt buộc phải dùng class mà không dùng interface?
interface trong TypeScript: Chỉ tồn tại lúc viết code để kiểm tra kiểu, khi biên dịch sang JavaScript (build) thì nó bị xóa sạch 100%. NestJS và các thư viện như class-validator, class-transformer không thể đọc được metadata để validate dữ liệu.
class: Vẫn tồn tại sau khi biên dịch sang JavaScript, cho phép NestJS dùng Reflection metadata để kiểm tra từng trường dữ liệu khi client gửi lên.
Mục 2: Dependency Injection, Singleton & Custom Token (Symbol)
1. Định nghĩa Dependency Injection (DI) & Singleton
Dependency Injection (DI): Thay vì một Class phải tự mình khởi tạo các đối tượng phụ thuộc (const service = new UsersService()), NestJS sẽ làm việc này thay bạn. NestJS tạo sẵn các Provider và tự động "tiêm" (inject) vào constructor khi cần.
Singleton: Mặc định trong NestJS, mỗi Service/Provider chỉ được tạo đúng 1 instance duy nhất trong toàn bộ vòng đời ứng dụng. Mọi Controller gọi Service này đều dùng chung 1 địa chỉ bộ nhớ đó.
2. Tạo Token bằng Symbol và tiêm bằng @Inject()
Symbol đảm bảo token là duy nhất tuyệt đối, không sợ bị trùng tên chuỗi.

Bước 1: Định nghĩa token (src/common/constants/tokens.ts)

typescript


export const APP_CONFIG = Symbol('APP_CONFIG');
Bước 2: Đăng ký trong AppModule (src/app.module.ts)

typescript


@Module({
  providers: [
    AppService,
    {
      provide: APP_CONFIG,
      useValue: { appName: 'Cửa Hàng Trực Tuyến', version: '1.0.0' },
    },
  ],
})
export class AppModule {}
Bước 3: Tiêm vào Service bằng @Inject() (src/app.service.ts)

typescript


import { Inject, Injectable } from '@nestjs/common';
import { APP_CONFIG } from './common/constants/tokens';
@Injectable()
export class AppService {
  constructor(
    @Inject(APP_CONFIG) private readonly config: { appName: string; version: string },
  ) {}
  getAppInfo() {
    return `${this.config.appName} - v${this.config.version}`;
  }
}
⚠️ Cách đọc hiểu lỗi "Nest can't resolve dependencies":
Khi bạn thấy lỗi này, NestJS đang nói: "Tôi thấy trong constructor của bạn đòi hỏi một thứ, nhưng trong mảng providers của Module hiện tại chưa có ai đăng ký cung cấp thứ đó cả!".
Cách sửa: Kiểm tra xem đã thêm Service đó vào providers của Module chưa, hoặc đã import Module chứa Service đó vào chưa.

Mục 3: Modules Nâng Cao & Tính Singleton của Module
1. Tạo PromotionsModule
Tạo tính năng tính khuyến mãi để sau này các module khác (như OrdersModule) có thể tái sử dụng.

Tạo Service: src/promotions/promotions.service.ts

typescript


import { Injectable } from '@nestjs/common';
@Injectable()
export class PromotionsService {
  calculateDiscount(amount: number): number {
    return amount >= 500000 ? 50000 : 0; // Giảm 50.000đ cho đơn từ 500.000đ
  }
}
Tạo Module: src/promotions/promotions.module.ts

typescript


import { Module } from '@nestjs/common';
import { PromotionsService } from './promotions.service';
@Module({
  providers: [PromotionsService],
  exports: [PromotionsService], // 👈 QUAN TRỌNG: Phải export thì module khác mới dùng được!
})
export class PromotionsModule {}
2. Tính Singleton của Module
Dù PromotionsModule được import ở 5 Module khác nhau (OrdersModule, ProductsModule, UsersModule...), NestJS vẫn chỉ tạo duy nhất 1 phiên bản PromotionsService. Dữ liệu nội tại của Service này được chia sẻ đồng nhất trong toàn hệ thống.

Mục 4: Middleware & Tích Hợp RequestId với LoggingInterceptor
1. Viết RequestMiddleware gắn UUID
Gán một mã định danh duy nhất (requestId) cho từng request gửi đến để dễ theo dõi log (traceability).

src/common/middleware/request-id.middleware.ts:

typescript


import { Injectable, NestMiddleware } from '@nestjs/common';
import { randomUUID } from 'crypto';
import { Request, Response, NextFunction } from 'express';
@Injectable()
export class RequestMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const requestId = (req.headers['x-request-id'] as string) || randomUUID();
    req['requestId'] = requestId; // Gắn vào request
    res.setHeader('x-request-id', requestId); // Trả ngược lại header cho client
    next();
  }
}
2. Tích hợp đọc requestId trong LoggingInterceptor
src/common/interceptors/logging.interceptor.ts:

typescript


import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const requestId = req['requestId'];
    const startTime = Date.now();
    console.log(`[START] RequestId: ${requestId} - ${req.method} ${req.url}`);
    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        console.log(`[END] RequestId: ${requestId} - Thời gian xử lý: ${duration}ms`);
      }),
    );
  }
}
Đăng ký Middleware trong 
app.module.ts
:

typescript


export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(RequestMiddleware).forRoutes('*');
  }
}
Mục 5: Exception Filter Toàn Diện (HttpAdapterHost)
1. Khi nào dùng và thứ tự chạy?
Khi nào dùng: Dùng để bắt tất cả các ngoại lệ (throw Error, BadRequestException, lỗi SQL 500...) và biến đổi chúng thành định dạng JSON chuẩn mực trước khi trả về cho Client.
Thứ tự: Exception Filter luôn là chốt chặn cuối cùng khi có lỗi văng ra từ bất kỳ tầng nào (Guard, Pipe, Controller, Service).
2. Viết AllExceptionsFilter an toàn (Không để lộ stack trace)
src/common/exception-filters/all-exceptions.filter.ts:

typescript


import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';
@Catch() // Bắt tất cả mọi lỗi
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}
  catch(exception: unknown, host: ArgumentsHost): void {
    const { httpAdapter } = this.httpAdapterHost;
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    // 1. Phân loại mã HTTP Status Code
    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
    // 2. Bóc tách message an toàn
    let message: any = 'Lỗi hệ thống nội bộ. Vui lòng thử lại sau!';
    if (exception instanceof HttpException) {
      const res = exception.getResponse();
      message = typeof res === 'object' ? (res as any).message || res : res;
    } else if (exception instanceof Error) {
      // Ghi log lỗi nội bộ ra terminal để dev sửa, KHÔNG trả stack trace cho client
      this.logger.error(`System Error: ${exception.message}`, exception.stack);
    }
    // 3. Chuẩn hóa format trả về
    const responseBody = {
      statusCode: httpStatus,
      message,
      data: null,
      requestId: request['requestId'] || null,
      path: httpAdapter.getRequestUrl(request),
      timestamp: new Date().toISOString(),
    };
    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
Đăng ký qua APP_FILTER trong AppModule:

typescript


providers: [
  {
    provide: APP_GUARD,
    useClass: AllExceptionsFilter,
  },
]
Mục 6: Pipes Chuẩn Production
Trong src/main.ts, cấu hình 3 cờ quan trọng nhất của ValidationPipe:

typescript


app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true, // 1. Tự động loại bỏ các field thừa không khai báo trong DTO
    forbidNonWhitelisted: true, // 2. Nếu client cố tình gửi field lạ lên -> Ném lỗi 400 ngay lập tức!
    transform: true, // 3. Tự động ép kiểu (VD: query string "10" tự chuyển thành số 10)
  }),
);
Mục 7: Guards Toàn Diện, JWT & Mô Hình Mặc Định Đóng
1. Ẩn dụ "Vé xem phim" về JWT 🎟️
Session truyền thống giống như thẻ gửi đồ: Server phải mở sổ (Database) ra tra cứu xem thẻ này là của ai.
JWT giống như vé xem phim điện tử có mã QR & Chữ ký chống giả:
Tự chứa thông tin: Tên phim, ghế số mấy (userId, role), giờ chiếu (exp).
Có con dấu chữ ký bảo mật (secret key): Khách không tự sửa được từ ghế thường thành ghế VIP.
Người soát vé (JwtAuthGuard) chỉ cần nhìn chữ ký và ngày giờ là cho vào ngay, không cần gọi điện thoại hỏi lại quầy vé.
2. Viết Custom Decorator @Public()
src/auth/decorators/public.decorator.ts:

typescript


import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
3. Viết JwtAuthGuard hỗ trợ @Public()
src/auth/guards/jwt-auth.guard.ts:

typescript


import { ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }
  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true; // Có nhãn @Public -> Cho qua cửa luôn!
    return super.canActivate(context); // Không có nhãn -> Bắt buộc check Token
  }
}
4. Viết @Roles(...) & RolesGuard
src/auth/decorators/roles.decorator.ts:

typescript


import { SetMetadata } from '@nestjs/common';
import { Role } from '../../common/enums/role.enum';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
src/auth/guards/roles.guard.ts:

typescript


import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from '../../common/enums/role.enum';
import { ROLES_KEY } from '../decorators/roles.decorator';
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles || requiredRoles.length === 0) return true;
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => role === user?.role);
  }
}
5. Áp dụng Mô hình Mặc định đóng trong AppModule
Tất cả các route mặc định đều bị khóa, chỉ mở khi có @Public().

typescript


// src/app.module.ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard }, // Gác cổng 1: Check Token
  { provide: APP_GUARD, useClass: RolesGuard },    // Gác cổng 2: Check Quyền
]
Mục 8: Interceptors Toàn Diện & Chuẩn Hóa Response
Tự động bọc mọi kết quả thành công vào vỏ hộp JSON chuẩn mực.

src/common/interceptors/transform.interceptor.ts:

typescript


import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
export interface ApiResponse<T> {
  statusCode: number;
  message: string;
  data: T;
  timestamp: string;
}
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    const response = context.switchToHttp().getResponse();
    const statusCode = response.statusCode;
    return next.handle().pipe(
      map((data) => ({
        statusCode,
        message: data?.message || 'Success',
        data: data && typeof data === 'object' && 'data' in data ? data.data : data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
Đăng ký toàn cục trong AppModule:

typescript


providers: [
  { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
]
Mục 9: Custom Decorators (@CurrentUser & @Auth Composite)
1. Bản chất của @CurrentUser()
@CurrentUser() không tự sinh ra user. Nó chỉ là hàm viết tắt lấy từ request.user do JwtAuthGuard gán vào từ trước.

src/auth/decorators/current-user.decorator.ts:

typescript


import { createParamDecorator, ExecutionContext } from '@nestjs/common';
export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    // Nếu gọi @CurrentUser('userId') thì lấy user['userId'], ngược lại lấy cả object user
    return data ? user?.[data] : user;
  },
);
2. Viết Composite Decorator @Auth() (Gộp nhiều decorator làm 1)
src/auth/decorators/auth.decorator.ts:

typescript


import { applyDecorators, UseGuards } from '@nestjs/common';
import { Role } from '../../common/enums/role.enum';
import { JwtAuthGuard } from '../guards/jwt-auth.guard';
import { RolesGuard } from '../guards/roles.guard';
import { Roles } from './roles.decorator';
export function Auth(...roles: Role[]) {
  return applyDecorators(
    Roles(...roles),
    UseGuards(JwtAuthGuard, RolesGuard),
  );
}
Mục 10: Tích Hợp Cuối - API POST /orders Đi Qua Đủ 7 Lớp
1. Code trọn vẹn Module Orders
1. DTO: src/orders/dto/create-order.dto.ts

typescript


import { IsInt, IsNotEmpty, Min } from 'class-validator';
export class CreateOrderDto {
  @IsInt({ message: 'productId phải là số nguyên' })
  @IsNotEmpty({ message: 'productId không được để trống' })
  productId: number;
  @IsInt({ message: 'quantity phải là số nguyên' })
  @Min(1, { message: 'Số lượng mua tối thiểu là 1' })
  quantity: number;
}
2. Service: src/orders/orders.service.ts

typescript


import { BadRequestException, Injectable } from '@nestjs/common';
import { CreateOrderDto } from './dto/create-order.dto';
import { PromotionsService } from '../promotions/promotions.service';
@Injectable()
export class OrdersService {
  constructor(private readonly promotionsService: PromotionsService) {}
  createOrder(userId: number, dto: CreateOrderDto) {
    if (dto.productId <= 0) {
      throw new BadRequestException('Mã sản phẩm không hợp lệ!');
    }
    const unitPrice = 150000;
    const totalAmount = unitPrice * dto.quantity;
    const discount = this.promotionsService.calculateDiscount(totalAmount);
    return {
      orderId: 'ORD-' + Date.now(),
      customerId: userId,
      productId: dto.productId,
      quantity: dto.quantity,
      unitPrice,
      totalAmount: totalAmount - discount,
      discount,
      status: 'SUCCESS',
    };
  }
}
3. Controller: src/orders/orders.controller.ts

typescript


import { Body, Controller, Post } from '@nestjs/common';
import { OrdersService } from './orders.service';
import { CreateOrderDto } from './dto/create-order.dto';
import { Auth } from '../auth/decorators/auth.decorator';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { Role } from '../common/enums/role.enum';
@Controller('orders')
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}
  @Post()
  @Auth(Role.USER, Role.ADMIN) // 👈 Tầng 2: Guard bảo vệ
  createOrder(
    @CurrentUser('userId') userId: number, // 👈 Tầng 5: Decorator lấy userId
    @Body() dto: CreateOrderDto,           // 👈 Tầng 4: Pipe validate dữ liệu
  ) {
    return this.ordersService.createOrder(userId, dto); // 👈 Tầng 6: Service xử lý
  }
}
4. Module: src/orders/orders.module.ts

typescript


import { Module } from '@nestjs/common';
import { OrdersController } from './orders.controller';
import { OrdersService } from './orders.service';
import { PromotionsModule } from '../promotions/promotions.module';
@Module({
  imports: [PromotionsModule],
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}
2. Sơ Đồ Request Lifecycle 7 Lớp Của POST /orders
Mermaid diagram
💡 Bảng Tóm Tắt Nhiệm Vụ 7 Tầng Trong Vòng Đời Request
Thứ tự	Tầng (Layer)	Nhiệm vụ chính trong API POST /orders
1	Middleware	Can thiệp sớm nhất: sinh mã requestId và gắn vào request.
2	Guard	Người gác cổng: giải mã JWT token, kiểm tra role USER/ADMIN.
3	Interceptor (Vào)	Bắt đầu bấm giờ đo hiệu năng (Logging).
4	Pipe	Soi dữ liệu body: kiểm tra productId là số, quantity >= 1.
5	Controller	Điểm tiếp nhận: lấy userId từ @CurrentUser, chuyển tiếp sang Service.
6	Service	Xử lý logic chính: tính giá, áp dụng khuyến mãi từ PromotionsService.
7	Interceptor (Ra) & Filter	Thành công: Interceptor bọc vỏ { statusCode, message, data }.
Thất bại: Filter tóm lỗi, chuẩn hóa response an toàn.
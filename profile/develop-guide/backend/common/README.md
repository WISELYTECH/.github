# Nest.js 공통가이드

## 1. 상속보다는 조합 사용하기

상속은 강결합을 만들고 비즈니스 규칙이 늘어나면 유연성을 떨어뜨림. 조합이 더 유연하고, 테스트하기 쉬운 코드가 만들어짐

### 안좋은 예: 상속 사용

```ts
// 상속 구조 - 강한 결합, 재사용성 낮음
abstract class BaseService {
  constructor(protected readonly repository: Repository<any>) {}
  
  async findById(id: number) {
    return this.repository.findOne({ where: { id } });
  }
}

class UserService extends BaseService {
  async getUserWithPosts(id: number) {
    return this.repository.findOne({
      where: { id },
      relations: ['posts'],
    });
  }
}

class ProductService extends BaseService {
  async getProductWithReviews(id: number) {
    return this.repository.findOne({
      where: { id },
      relations: ['reviews'],
    });
  }
}
```

### 좋은 예: 조합 사용

```typescript
// 조합 구조 - 느슨한 결합, 높은 재사용성
@Injectable()
class EntityFinder<T> {
  constructor(private readonly repository: Repository<T>) {}
  
  async findById(id: number): Promise<T | null> {
    return this.repository.findOne({ where: { id } as any });
  }
  
  async findByIdWith(id: number, relations: string[]): Promise<T | null> {
    return this.repository.findOne({
      where: { id } as any,
      populate: [...relations]
    });
  }
}

@Injectable()
class UserService {
  constructor(
    private readonly userFinder: EntityFinder<User>,
    private readonly userValidator: UserValidator,
  ) {}
  
  async getUserWithPosts(id: number): Promise<User> {
    const user = await this.userFinder.findByIdWith(id, ['posts']);
    await this.userValidator.validateExists(user);
    return user;
  }
}

@Injectable()
class ProductService {
  constructor(
    private readonly productFinder: EntityFinder<Product>,
    private readonly productValidator: ProductValidator,
  ) {}
  
  async getProductWithReviews(id: number): Promise<Product> {
    const product = await this.productFinder.findByIdWith(id, ['reviews']);
    await this.productValidator.validateExists(product);
    return product;
  }
}
```

---

## 2. Private 메서드 지양하기

클래스에 Private 메서드가 많다는 것은 코드 스멜의 신호 (너무 많은 책임을 가지고 있을 가능성이 높음). 별도의 컴포넌트로 분리하세요.

### 안좋은 예: 과도하게 많은 Private 메서드들

```typescript
@Injectable()
class OrderService {
  constructor(
    private readonly orderRepository: Repository<Order>,
    private readonly userRepository: Repository<User>,
    private readonly productRepository: Repository<Product>,
  ) {}
  
  async create(userId: number, productIds: number[]) {
    const user = await this.validateUser(userId);
    const products = await this.validateProducts(productIds);
    const totalPrice = this.calculateTotalPrice(products);
    await this.validateUserBalance(user, totalPrice);
    
    const order = this.buildOrder(user, products, totalPrice);
    return this.orderRepository.save(order);
  }
  
  // 코드 스멜!
  private async validateUser(userId: number) {
    const user = await this.userRepository.findOne({ where: { id: userId } });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
  
  private async validateProducts(productIds: number[]) {
    const products = await this.productRepository.findByIds(productIds);
    if (products.length !== productIds.length) {
      throw new BadRequestException('Some products not found');
    }
    return products;
  }
  
  private calculateTotalPrice(products: Product[]): number {
    return products.reduce((sum, p) => sum + p.price, 0);
  }
  
  private async validateUserBalance(user: User, totalPrice: number) {
    if (user.balance < totalPrice) {
      throw new BadRequestException('Insufficient balance');
    }
  }
  
  private buildOrder(user: User, products: Product[], totalPrice: number): Order {
    const order = new Order();
    order.user = user;
    order.products = products;
    order.totalPrice = totalPrice;
    order.status = OrderStatus.PENDING;
    return order;
  }
}
```
- 위 예시에서 validate , build 등은 여러 서비스에서 재사용 가능하게 구성하는게 더 유연함
- 다른 서비스에서 같은 검증 로직이 필요할 때 코드 중복이 발생할 여지가 있거나 서비스 - 서비스 의존이 발생하고 의존성 트리 관리가 안됨
- 검증 로직 변경 시 여러 서비스를 수정해야 함
- 테스트 시 OrderService 전체를 mocking 해야 함

### 좋은 예: 책임을 분리한 컴포넌트

```typescript
// 각 책임을 담당하는 독립적인 컴포넌트들
@Injectable()
class UserFinder {
  constructor(private readonly userRepository: Repository<User>) {}
  
  async findById(id: number): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
}

@Injectable()
class ProductFinder {
  constructor(private readonly productRepository: Repository<Product>) {}
  
  async findByIds(ids: number[]): Promise<Product[]> {
    const products = await this.productRepository.findByIds(ids);
    if (products.length !== ids.length) {
      throw new BadRequestException('Some products not found');
    }
    return products;
  }
}

@Injectable()
class OrderValidator {
  validateUserBalance(user: User, totalPrice: number): void {
    if (user.balance < totalPrice) {
      throw new BadRequestException('Insufficient balance');
    }
  }
}

@Injectable()
class OrderCreator {
  constructor(private readonly orderRepository: Repository<Order>) {}
  
  async create(user: User, products: Product[], totalPrice: number): Promise<Order> {
    const order = new Order();
    order.user = user;
    order.products = products;
    order.totalPrice = totalPrice;
    order.status = OrderStatus.PENDING;
    
    return this.orderRepository.save(order);
  }
}

// Service는 오케스트레이터 역할만 수행
@Injectable()
class OrderService {
  constructor(
    private readonly userFinder: UserFinder,
    private readonly productFinder: ProductFinder,
    private readonly orderValidator: OrderValidator,
    private readonly orderCreator: OrderCreator,
  ) {}
  
  async create(userId: number, productIds: number[]): Promise<Order> {
    const user = await this.userFinder.findById(userId);
    const products = await this.productFinder.findByIds(productIds);
    
    const totalPrice = products.reduce((sum, p) => sum + p.price, 0);
    this.orderValidator.validateUserBalance(user, totalPrice);
    
    return this.orderCreator.create(user, products, totalPrice);
  }
}
```

**장점:**
- 각 컴포넌트를 독립적으로 테스트 가능
- 재사용성 증가
- 단일 책임 원칙 준수
- 코드 가독성 향상

---

## 3. 순환 의존성 지양하기

순환 의존성 (의존성을 그렸을 때 그래프 모양이 나오는 경우) 코드의 복잡도를 높이고 테스트를 어렵게 만듬. 트리 구조의 의존성을 유지하세요.

### 안좋은 예: 순환 의존성

```typescript
// 순환 의존성 발생!
@Module({
  imports: [
    forwardRef(() => PostModule), // forwardRef 사용 - 설계 문제의 징후
  ],
  providers: [UserService],
  exports: [UserService],
})
class UserModule {}

@Module({
  imports: [
    forwardRef(() => UserModule), // 순환 참조
  ],
  providers: [PostService],
  exports: [PostService],
})
class PostModule {}

// UserService와 PostService가 서로를 의존
@Injectable()
class UserService {
  constructor(
    @Inject(forwardRef(() => PostService))
    private readonly postService: PostService,
  ) {}
  
  async getUserWithPosts(userId: number) {
    return this.postService.getPostsByUserId(userId);
  }
}

@Injectable()
class PostService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private readonly userService: UserService,
  ) {}
  
  async getPostWithAuthor(postId: number) {
    return this.userService.getUserById(postId);
  }
}
```
이 구조의 문제점
```
UserModule ←→ PostModule
    ↓              ↓
UserService ←→ PostService
```
- forwardRef 필요
- 테스트 어려움: 하나를 테스트하려면 다른 하나를 반드시 목킹해야 함
- 변경의 파급효과: 한쪽 변경이 다른 쪽에 즉시 영향
- 초기화 순서 문제: 누가 먼저 생성되어야 하는지 불명확
- 설계 스멜: 책임 분리가 제대로 안 된 신호

### 좋은 예
---

## 1. 공통 모듈로 의존성 분리

### 의존성 구조
```
       UserPostModule
      /            \
  UserModule    PostModule
  (독립적)      (독립적)
```

### 모듈 구성

```typescript
// shared.module.ts
@Module({
  providers: [
    // 공통 Repository
    UserRepository,
    PostRepository,
    
    // 공통 Finder (데이터 조회만 담당)
    UserFinder,
    PostFinder,
    
    // 공통 Validator
    UserValidator,
    PostValidator,
  ],
  exports: [
    UserRepository,
    PostRepository,
    UserFinder,
    PostFinder,
    UserValidator,
    PostValidator,
  ],
})
export class SharedModule {}
```

```typescript
// user.module.ts
@Module({
  imports: [SharedModule],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

```typescript
// post.module.ts
@Module({
  imports: [SharedModule],
  providers: [PostService],
  exports: [PostService],
})
export class PostModule {}
```

### 서비스 구현

```typescript
// user.finder.ts
@Injectable()
export class UserFinder {
  constructor(private readonly userRepository: UserRepository) {}
  
  async findById(userId: number): Promise<User | null> {
    return this.userRepository.findOne({ where: { id: userId } });
  }
  
  async findByIds(userIds: number[]): Promise<User[]> {
    return this.userRepository.findByIds(userIds);
  }
  
  async findByEmail(email: string): Promise<User | null> {
    return this.userRepository.findOne({ where: { email } });
  }
}
```

```typescript
// post.finder.ts
@Injectable()
export class PostFinder {
  constructor(private readonly postRepository: PostRepository) {}
  
  async findById(postId: number): Promise<Post | null> {
    return this.postRepository.findOne({ where: { id: postId } });
  }
  
  async findByUserId(userId: number): Promise<Post[]> {
    return this.postRepository.find({ where: { userId } });
  }
  
  async findPublishedPosts(): Promise<Post[]> {
    return this.postRepository.find({ where: { published: true } });
  }
}
```

```typescript
// user.validator.ts
@Injectable()
export class UserValidator {
  validateExists(user: User | null): user is User {
    if (!user) {
      throw new NotFoundException('User not found');
    }
  }
  
  validateActive(user: User): void {
    if (!user.isActive) {
      throw new BadRequestException('User is not active');
    }
  }
  
  validateEmail(email: string): void {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      throw new BadRequestException('Invalid email format');
    }
  }
}
```

```typescript
// post.validator.ts
@Injectable()
export class PostValidator {
  validateExists(post: Post | null): post is Post {
    if (!post) {
      throw new NotFoundException('Post not found');
    }
  }
  
  validateOwnership(post: Post, userId: number): void {
    if (post.userId !== userId) {
      throw new ForbiddenException('You do not own this post');
    }
  }
  
  validateContent(content: string): void {
    if (content.length < 10) {
      throw new BadRequestException('Content is too short');
    }
    if (content.length > 10000) {
      throw new BadRequestException('Content is too long');
    }
  }
}
```

```typescript
// user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userFinder: UserFinder,       // SharedModule
    private readonly userValidator: UserValidator, // SharedModule
    private readonly userRepository: UserRepository,
  ) {}
  
  async findOne(userId: number): Promise<User> {
    const user = await this.userFinder.findById(userId);
    this.userValidator.validateExists(user);
    return user;
  }
  
  async create(dto: CreateUserDto): Promise<User> {
    this.userValidator.validateEmail(dto.email);
    
    const existingUser = await this.userFinder.findByEmail(dto.email);
    if (existingUser) {
      throw new ConflictException('Email already exists');
    }
    
    return this.userRepository.save(dto);
  }
}
```

```typescript
// post.service.ts
@Injectable()
export class PostService {
  constructor(
    private readonly postFinder: PostFinder,       // SharedModule
    private readonly postValidator: PostValidator, // SharedModule
    private readonly userFinder: UserFinder,       // SharedModule
    private readonly postRepository: PostRepository,
  ) {}
  
  async findWithAuthor(postId: number): Promise<PostWithAuthor> {
    const post = await this.postFinder.findById(postId);
    this.postValidator.validateExists(post);
    
    // UserService 대신 UserFinder 직접 사용
    const author = await this.userFinder.findById(post.userId);
    
    return {
      ...post,
      author: author ? { id: author.id, username: author.username } : null,
    };
  }
  
  async create(userId: number, dto: CreatePostDto): Promise<Post> {
    const user = await this.userFinder.findById(userId);
    if (!user) {
      throw new NotFoundException('User not found');
    }
    
    this.postValidator.validateContent(dto.content);
    
    return this.postRepository.save({
      ...dto,
      userId,
    });
  }
  
  async findMany(userId: number): Promise<Post[]> {
    return this.postFinder.findByUserId(userId);
  }
}
```

---

## 2. 단방향 의존 구조

### 의존성 구조
```
  UserModule (저수준)
      ↑
  PostModule (고수준)
```

### 모듈 구성

```typescript
// user.module.ts
@Module({
  providers: [UserService, UserRepository],
  exports: [UserService],
})
export class UserModule {}
```

```typescript
// post.module.ts
@Module({
  imports: [UserModule], // forwardRef 없음
  providers: [PostService, PostRepository],
  exports: [PostService],
})
export class PostModule {}
```

### 서비스 구현

```typescript
// user.service.ts
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}
  
  async getUser(userId: number): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id: userId } });
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return user;
  }
  
  async getUsersByIds(userIds: number[]): Promise<User[]> {
    return this.userRepository.findByIds(userIds);
  }
  
  async createUser(dto: CreateUserDto): Promise<User> {
    return this.userRepository.save(dto);
  }
}
```

```typescript
// post.service.ts
@Injectable()
export class PostService {
  constructor(
    private readonly postRepository: PostRepository,
    private readonly userService: UserService, // 단방향 의존
  ) {}
  
  async findWithAuthor(postId: number): Promise<PostWithAuthor> {
    const post = await this.postRepository.findOne({ where: { id: postId } });
    if (!post) {
      throw new NotFoundException('Post not found');
    }
    
    const author = await this.userService.fineOne(post.userId);
    
    return { ...post, author };
  }
  
  async findWithAuthors(postIds: number[]): Promise<PostWithAuthor[]> {
    const posts = await this.postRepository.findByIds(postIds);
    const userIds = [...new Set(posts.map(p => p.userId))];
    const users = await this.userService.getUsersByIds(userIds);
    
    const userMap = new Map(users.map(u => [u.id, u]));
    
    return posts.map(post => ({
      ...post,
      author: userMap.get(post.userId) || null,
    }));
  }
  
  async create(userId: number, dto: CreatePostDto): Promise<Post> {
    // 사용자 존재 여부 확인
    await this.userService.getUser(userId);
    
    return this.postRepository.save({
      ...dto,
      userId,
    });
  }
}
```

---
### 3. 이벤트 기반 아키텍처

의존성 구조
```
UserModule → EventBus ← PostModule
   (발행)       ↕️        (구독)
```
```ts
// event-bus.interface.ts
export interface IEventBus {
  publish(eventName: string, payload: any): Promise<void>;
  subscribe(eventName: string, handler: (payload: any) => Promise<void>): void;
}
```
```ts
// event-bus.service.ts, 구현체는 아무 이벤트 시스템이나 사용 가능 
@Injectable()
export class EventBus implements IEventBus {
  private readonly logger = new Logger(EventBus.name);
  private handlers = new Map<string, Array<(payload: any) => Promise<void>>>();
  
  async publish(eventName: string, payload: any): Promise<void> {
    this.logger.log(`Publishing event: ${eventName}`);
    
    const eventHandlers = this.handlers.get(eventName) || [];
    
    // 비동기로 모든 핸들러 실행
    await Promise.allSettled(
      eventHandlers.map(handler => 
        handler(payload).catch(error => {
          this.logger.error(`Handler error for ${eventName}: ${error.message}`);
        })
      )
    );
  }
  
  subscribe(eventName: string, handler: (payload: any) => Promise<void>): void {
    if (!this.handlers.has(eventName)) {
      this.handlers.set(eventName, []);
    }
    
    this.handlers.get(eventName).push(handler);
    this.logger.log(`Subscribed to event: ${eventName}`);
  }
}
```

```ts
이벤트 정의
// events/user.events.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: number,
    public readonly username: string,
    public readonly email: string,
  ) {}
}

export class UserDeletedEvent {
  constructor(public readonly userId: number) {}
}
```

모듈 구성
```ts
// event.module.ts
@Module({
  providers: [EventBus],
  exports: [EventBus],
})
export class EventModule {}
// user.module.ts
@Module({
  imports: [EventModule],
  providers: [UserService, UserRepository],
  exports: [UserService],
})
export class UserModule {}
// post.module.ts
@Module({
  imports: [EventModule],
  providers: [
    PostService, 
    PostRepository, 
    UserCreatedHandler,
    UserDeletedHandler,
  ],
  exports: [PostService],
})
export class PostModule {}
```

UserService - 이벤트 발행

```ts
// user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: EventBus,
  ) {}
  
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.save(dto);
    
    // 이벤트 발행 
    await this.eventBus.publish(
      'user.created',
      new UserCreatedEvent(user.id, user.username, user.email)
    );
    
    return user;
  }
  
  async deleteUser(userId: number): Promise<void> {
    await this.userRepository.delete(userId);
    
    await this.eventBus.publish(
      'user.deleted',
      new UserDeletedEvent(userId)
    );
  }
}
```
PostModule - 이벤트 구독
```ts
// handlers/user-created.handler.ts
@Injectable()
export class UserCreatedHandler implements OnModuleInit {
  private readonly logger = new Logger(UserCreatedHandler.name);
  
  constructor(
    private readonly postRepository: PostRepository,
    private readonly eventBus: EventBus,
  ) {}
  
  onModuleInit() {
    this.eventBus.subscribe('user.created', this.handle.bind(this));
  }
  
  async handle(event: UserCreatedEvent): Promise<void> {
    this.logger.log(`Creating welcome post for user ${event.userId}`);
    
    try {
      await this.postRepository.save({
        userId: event.userId,
        title: `Welcome ${event.username}!`,
        content: 'This is your first post. Start sharing your thoughts!',
        published: true,
      });
      
      this.logger.log(`Welcome post created for user ${event.userId}`);
    } catch (error) {
      this.logger.error(`Failed to create welcome post: ${error.message}`);
      throw error;
    }
  }
}

// handlers/user-deleted.handler.ts
@Injectable()
export class UserDeletedHandler implements OnModuleInit {
  private readonly logger = new Logger(UserDeletedHandler.name);
  
  constructor(
    private readonly postRepository: PostRepository,
    private readonly eventBus: EventBus,
  ) {}
  
  onModuleInit() {
    this.eventBus.subscribe('user.deleted', this.handle.bind(this));
  }
  
  async handle(event: UserDeletedEvent): Promise<void> {
    this.logger.log(`Deleting all posts for user ${event.userId}`);
    
    try {
      const result = await this.postRepository.delete({ userId: event.userId });
      this.logger.log(`Deleted ${result.affected} posts for user ${event.userId}`);
    } catch (error) {
      this.logger.error(`Failed to delete posts: ${error.message}`);
      throw error;
    }
  }
}

// post.service.ts
@Injectable()
export class PostService {
  constructor(
    private readonly postRepository: PostRepository,
    // UserService 의존 없음! 
  ) {}
  
  async getPostsByUserId(userId: number): Promise<Post[]> {
    return this.postRepository.find({ where: { userId } });
  }
  
  async createPost(userId: number, dto: CreatePostDto): Promise<Post> {
    return this.postRepository.save({ ...dto, userId });
  }
}
```
---

## 핵심 원칙

> **"forwardRef를 사용해야 한다면, 설계를 다시 생각해야 한다는 신호"**
- 가급적 forwardRef 를 지양하고 시간이 없으면 두 모듈의 공통 모듈 만들기 / 시간 있으면 이벤트로 분리하기 
---

## 4. 효율적인 아키텍처 구성하기
과도하게 미리 추상화보다는 명확한 책임 분리에 집중
```
┌─────────────────────────────────────────┐
│         Controller Layer                │
│      (요청/응답 처리, HTTP)                │
└──────────────────┬──────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────┐
│          Service Layer                  │
│    (비즈니스 플로우 오케스트레이션)             │
│                                         │
│  UserService                            │
│  - createUser()                         │
│  - getUser()                            │
└────┬──────┬──────┬──────┬──────┬────────┘
     │      │      │      │      │
     ↓      ↓      ↓      ↓      ↓
┌─────────────────────────────────────────┐
│     Concrete Components Layer           │
│   (구체화된 컴포넌트 - 단일 책임)              │
│                                         │
│  Finder    Validator   Creator  Modifier│
│  - findById  - validate  - save  - update│
│  - findByIds - Unique    - saveMany      │
└──────────────────┬──────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────┐
│         Entity Layer                    │
│   (리치 엔티티 - 비즈니스 로직 포함)           │
│                                         │
│  User (Rich Domain Entity)              │
│  ┌──────────────────────────┐           │
│  │ 데이터 (상태)                │          │
│  │ - id, email, name         │          │
│  │ - role, balance           │          │
│  │ - activationToken         │          │
│  └──────────────────────────┘           │
│  ┌──────────────────────────┐           │
│  │ 비즈니스 로직 (행위)          │           │
│  │ - create()                │           │
│  │ - assignDefaultRole()     │           │
│  │ - promoteToAdmin()        │           │
│  │ - generateActivationToken()│          │
│  │ - activate()              │           │
│  │ - addBalance()            │           │
│  │ - deductBalance()         │           │
│  │ - canPurchase()           │           │
│  └──────────────────────────┘           │
└─────────────────────────────────────────┘
```


### 클래스 별 역할

#### Controller: 요청/응답 처리

```typescript
@Controller('users')
class UserController {
  constructor(private readonly userService: UserService) {}
  
  @Post()
  async createUser(@Body() dto: CreateUserDto): Promise<UserResponse> {
    const user = await this.userService.createUser(dto);
    return new UserResponse(user);
  }
  
  @Get(':id')
  async getUser(@Param('id') id: number): Promise<UserResponse> {
    const user = await this.userService.getUser(id);
    return new UserResponse(user);
  }
}
```

#### Service: 오케스트레이터

Service는 비즈니스 플로우를 조율하고, 리치 엔티티의 메서드와 구체화된 컴포넌트들을 호출

```typescript
@Injectable()
class UserService {
  constructor(
    private readonly userFinder: UserFinder,
    private readonly userCreator: UserCreator,
    private readonly userValidator: UserValidator,
    private readonly emailSender: EmailSender,
  ) {}
  
  async createUser(dto: CreateUserDto): Promise<User> {
    // 1. 검증
    await this.userValidator.validateEmailUnique(dto.email);
    
    // 2. 엔티티 생성 (리치 엔티티 활용)
    const user = User.create(dto.email, dto.name);
    
    // 3. 비즈니스 로직 실행 (엔티티 메서드)
    user.assignDefaultRole();
    user.generateActivationToken();
    
    // 4. 저장
    const savedUser = await this.userCreator.save(user);
    
    // 5. 부가 작업
    await this.emailSender.sendWelcomeEmail(savedUser);
    
    return savedUser;
  }
  
  async getUser(id: number): Promise<User> {
    return this.userFinder.findById(id);
  }
}
```

#### Entity: 리치 엔티티 (비즈니스 로직 포함)

엔티티는 단순 데이터 모델이 아닌, 비즈니스 로직을 포함한 리치 모델 객체

```typescript
@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  email: string;
  
  @Column()
  name: string;
  
  @Column()
  role: UserRole;
  
  @Column()
  balance: number;
  
  @Column({ nullable: true })
  activationToken: string;
  
  @Column({ default: false })
  isActivated: boolean;
  
  // 팩토리 메서드
  static create(email: string, name: string): User {
    const user = new User();
    user.email = email;
    user.name = name;
    user.balance = 0;
    user.isActivated = false;
    return user;
  }
  
  // 비즈니스 로직 - 역할 할당
  assignDefaultRole(): void {
    this.role = UserRole.USER;
  }
  
  // 비즈니스 로직 - 관리자 권한 부여
  promoteToAdmin(): void {
    if (this.role === UserRole.ADMIN) {
      throw new BadRequestException('Already an admin');
    }
    this.role = UserRole.ADMIN;
  }
  
  // 비즈니스 로직 - 활성화 토큰 생성
  generateActivationToken(): void {
    this.activationToken = Math.random().toString(36).substring(7);
  }
  
  // 비즈니스 로직 - 계정 활성화
  activate(token: string): void {
    if (this.isActivated) {
      throw new BadRequestException('Already activated');
    }
    if (this.activationToken !== token) {
      throw new BadRequestException('Invalid token');
    }
    this.isActivated = true;
    this.activationToken = null;
  }
  
  // 비즈니스 로직 - 잔액 추가
  addBalance(amount: number): void {
    if (amount <= 0) {
      throw new BadRequestException('Amount must be positive');
    }
    this.balance += amount;
  }
  
  // 비즈니스 로직 - 잔액 차감
  deductBalance(amount: number): void {
    if (amount <= 0) {
      throw new BadRequestException('Amount must be positive');
    }
    if (this.balance < amount) {
      throw new BadRequestException('Insufficient balance');
    }
    this.balance -= amount;
  }
  
  // 비즈니스 로직 - 구매 가능 여부 확인
  canPurchase(price: number): boolean {
    return this.isActivated && this.balance >= price;
  }
}
```

#### 구체화된 컴포넌트들

각 컴포넌트는 명확한 단일 책임을 가짐 (이런 저런 서비스에서 필요시 di 해서 쓸 수 있다고 가정해야함)

```typescript
// Finder: 조회 로직
@Injectable()
class UserFinder {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}
  
  async findById(id: number): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
  
  async findByEmail(email: string): Promise<User | null> {
    return this.userRepository.findOne({ where: { email } });
  }
  
  async findActiveUsers(): Promise<User[]> {
    return this.userRepository.find({ where: { isActivated: true } });
  }
}

// Validator: 검증 로직
@Injectable()
class UserValidator {
  constructor(private readonly userFinder: UserFinder) {}
  
  async validateEmailUnique(email: string): Promise<void> {
    const existing = await this.userFinder.findByEmail(email);
    if (existing) {
      throw new ConflictException('Email already exists');
    }
  }
  
  validateExists(user: User | null): asserts user is User {
    if (!user) {
      throw new NotFoundException('User not found');
    }
  }
  
  validateActivated(user: User): void {
    if (!user.isActivated) {
      throw new BadRequestException('User not activated');
    }
  }
}

// Creator: 생성 및 저장 로직
@Injectable()
class UserCreator {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}
  
  async save(user: User): Promise<User> {
    return this.userRepository.save(user);
  }
  
  async saveMany(users: User[]): Promise<User[]> {
    return this.userRepository.save(users);
  }
}

// Modifier: 수정 로직
@Injectable()
class UserModifier {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly userFinder: UserFinder,
  ) {}
  
  async updateEmail(userId: number, newEmail: string): Promise<User> {
    const user = await this.userFinder.findById(userId);
    user.email = newEmail;
    return this.userRepository.save(user);
  }
  
  async activateUser(userId: number, token: string): Promise<User> {
    const user = await this.userFinder.findById(userId);
    user.activate(token); // 엔티티의 비즈니스 로직 활용
    return this.userRepository.save(user);
  }
}
```

### 전체 구조 예시

```typescript
// 모듈 구조
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [
    // Service (오케스트레이터)
    UserService,
    
    // 구체화된 컴포넌트들
    UserFinder,
    UserValidator,
    UserCreator,
    UserModifier,
    
    // 외부 서비스
    EmailSender,
  ],
  exports: [UserService],
})
class UserModule {}

// 비즈니스 로직 예시
@Injectable()
class OrderService {
  constructor(
    private readonly orderFinder: OrderFinder,
    private readonly orderCreator: OrderCreator,
    private readonly orderValidator: OrderValidator,
    private readonly userFinder: UserFinder,
    private readonly productFinder: ProductFinder,
    private readonly paymentProcessor: PaymentProcessor,
    private readonly inventoryModifier: InventoryModifier,
  ) {}
  
  async createOrder(userId: number, items: OrderItemDto[]): Promise<Order> {
    // 1. 사용자 조회 및 검증
    const user = await this.userFinder.findById(userId);
    
    // 2. 상품 조회
    const productIds = items.map(item => item.productId);
    const products = await this.productFinder.findByIds(productIds);
    
    // 3. 주문 엔티티 생성 (리치 엔티티)
    const order = Order.create(user, products, items);
    
    // 4. 엔티티 비즈니스 로직 실행
    order.calculateTotalPrice();
    order.applyDiscount(user.membershipLevel);
    
    // 5. 검증
    this.orderValidator.validateStock(order, products);
    this.orderValidator.validateUserCanOrder(user, order.totalPrice);
    
    // 6. 결제 처리
    await this.paymentProcessor.process(user, order.totalPrice);
    
    // 7. 재고 차감 (엔티티 메서드 활용)
    for (const product of products) {
      const quantity = order.getQuantityForProduct(product.id);
      product.decreaseStock(quantity);
    }
    await this.inventoryModifier.updateProducts(products);
    
    // 8. 주문 저장
    order.markAsPaid();
    return this.orderCreator.save(order);
  }
}
```

---

## 핵심 원칙 요약

1. **조합 우선**: 상속 대신 컴포넌트 조합으로 유연성 확보
2. **책임 분리**: Private 함수 대신 독립적인 컴포넌트로 분리
3. **단방향 의존성**: 순환 참조 없는 트리 구조 유지
4. **명확한 역할**: Controller, Service, Entity, Components의 책임을 명확히 구분
5. **리치 엔티티**: 비즈니스 로직을 엔티티에 포함시켜 도메인 중심 설계
6. **테스트 가능성**: 각 컴포넌트를 독립적으로 테스트 가능하도록 설계

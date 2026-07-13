---
title: 学习笔记：用 Idiomatic Go 设计电商平台架构
abbrlink: learning-note-20260713-02
date: 2026-07-13T16:31:52
updated: 2026-07-13T16:31:52
tags:
  - 学习笔记
  - 学习
  - Go
  - 软件架构
categories:
  - 学习笔记
desc: 结合 spf13 的 go-skills，以电商平台为例理解领域包、显式装配、小接口和标准库优先的 Idiomatic Go 架构。
---

# 学习笔记：用 Idiomatic Go 设计电商平台架构

今天主要学习了 spf13 在 `go-skills` 中总结的 Idiomatic Go 工程规范，并尝试把这些原则应用到一个电商平台中。

这套规范最重要的价值，不是提供又一份固定的 Go 项目目录，而是建立一套判断标准：代码应该按什么边界组织、什么时候需要接口、依赖关系应该在哪里出现，以及如何避免把 Java 或 Spring Boot 的习惯机械地翻译成 Go。

整套思想可以浓缩成一句话：

> Clear is better than clever. 清晰胜于巧妙。

Go 代码应当直接、可预测，并且容易导航。无法确定一层抽象是否必要时，先尝试删除它，观察代码是否反而变得更容易理解。

## 一、为什么不直接套用传统三层架构

很多后端项目会按照技术角色组织目录：

```text
controller/
service/
repository/
domain/
model/
utils/
```

在这种结构中，一个“创建订单”功能可能被拆散到多个目录：

```text
controller/order_controller.go
service/order_service.go
repository/order_repository.go
model/order.go
```

这种分层并非绝对错误，但它容易带来几个问题：

1. 阅读一个功能时需要在多个目录之间来回跳转。
2. 包名表达的是技术层次，而不是业务含义。
3. 为了保持层次形式，容易提前创建只有一个实现的接口。
4. `service`、`repository` 等包会不断膨胀，并产生横向依赖。
5. 不同业务的代码因为“都属于服务层”而被放进同一个包，降低内聚性。

spf13 更推荐按“代码做什么”组织，而不是按“代码属于哪一层”组织。对于电商平台，订单、商品、库存和支付就是比 Controller、Service、Repository 更稳定、更容易理解的边界。

## 二、电商平台的领域包结构

假设当前系统是一个由单个团队维护的中等规模电商后端，还没有独立部署各个业务的要求。此时可以先构建一个模块化单体：

```text
shop/
├── main.go                 # 程序入口，启动 HTTP Server
├── wiring.go               # 创建依赖和跨领域适配器
├── config/
│   ├── config.go           # 强类型配置
│   └── load.go             # 加载环境变量和配置文件
├── database/
│   └── database.go         # 创建和配置 *sql.DB
├── account/
│   ├── account.go          # 用户模型与业务行为
│   ├── handler.go          # 注册、登录等 HTTP 接口
│   ├── store.go            # 用户相关 SQL
│   └── account_test.go
├── catalog/
│   ├── product.go          # 商品模型与查询逻辑
│   ├── handler.go          # 商品 HTTP 接口
│   ├── store.go            # 商品相关 SQL
│   └── catalog_test.go
├── cart/
│   ├── cart.go             # 购物车生命周期
│   ├── handler.go          # 加购、删除、查询购物车
│   ├── store.go            # Redis 操作
│   └── cart_test.go
├── inventory/
│   ├── inventory.go        # 查询、预占和释放库存
│   ├── store.go            # 库存相关 SQL
│   └── inventory_test.go
├── payment/
│   ├── payment.go          # 支付状态与业务逻辑
│   ├── provider.go         # 第三方支付能力接口
│   ├── handler.go          # 支付回调
│   ├── store.go
│   └── payment_test.go
├── order/
│   ├── order.go            # Order、OrderItem 等领域类型
│   ├── checkout.go         # 下单流程
│   ├── handler.go          # 创建、查询、取消订单
│   ├── store.go            # 订单相关 SQL
│   └── order_test.go
├── migrations/
├── testdata/
├── go.mod
└── go.sum
```

这套结构有几个明显特点：

- `catalog`、`cart`、`inventory`、`order` 和 `payment` 都是业务语言。
- 一个领域相关的类型、行为、存储和测试放在一起。
- `main.go` 只负责启动，`wiring.go` 负责显式装配依赖。
- 没有默认创建 `internal`、`pkg`、`utils` 或 `common`。
- 系统仍然是一个可执行程序，不因为划分领域包就变成了微服务。

## 三、领域包的边界如何判断

并不是每个名词都值得建立一个包。判断是否需要独立领域包，可以问两个问题：

1. 这个领域能否用一句话说明自己的职责？
2. 它能否在不了解其他领域内部实现的情况下独立工作和测试？

例如：

- `catalog` 负责维护和查询可销售商品。
- `cart` 负责管理用户准备购买的商品集合。
- `inventory` 负责库存数量、预占和释放。
- `payment` 负责发起支付、处理回调和退款。
- `order` 负责订单状态及其生命周期。

相反，`utils` 无法用一句明确的业务职责描述。它通常只是“暂时不知道该把代码放在哪里”的结果。随着项目增长，`utils` 很容易变成所有包都依赖的杂物箱。

扁平也不意味着把几百个文件全部堆在根目录。正确做法是从简单结构起步，在真实的领域边界出现后再拆包，而不是在第一天就创建完整的企业级分层。

## 四、具体类型优先，接口由消费方定义

以库存为例，如果系统当前只有 MySQL 一种实现，可以先写具体类型：

```go
package inventory

type Store struct {
	db *sql.DB
}

func NewStore(db *sql.DB) *Store {
	return &Store{db: db}
}

func (s *Store) Reserve(
	ctx context.Context,
	skuID string,
	quantity int,
) error {
	// 在事务中检查并预占库存
	return nil
}

func (s *Store) Release(
	ctx context.Context,
	skuID string,
	quantity int,
) error {
	// 释放已预占的库存
	return nil
}
```

这里没有提前创建 `InventoryRepository` 和 `InventoryRepositoryImpl`。只有一个实现时，接口通常没有带来多态，只增加了需要理解的名字。

但是，订单领域确实需要调用库存能力。此时接口应该定义在消费这些能力的 `order` 包中：

```go
package order

type CartLoader interface {
	LoadCart(ctx context.Context, userID string) (CheckoutCart, error)
}

type StockReserver interface {
	Reserve(ctx context.Context, skuID string, quantity int) error
	Release(ctx context.Context, skuID string, quantity int) error
}

type PaymentCharger interface {
	Charge(ctx context.Context, req ChargeRequest) (PaymentResult, error)
}
```

订单服务只声明完成下单所需的最小能力：

```go
type Service struct {
	carts    CartLoader
	stock    StockReserver
	payments PaymentCharger
	store    *Store
}
```

这样做有三个好处：

1. 接口描述的是订单领域的真实需求。
2. 实现方不需要提前知道这个接口的存在，方法集合匹配即可。
3. 测试时可以注入很小的 fake，而不必实现庞大的通用 Repository 接口。

如果不同领域的数据类型不完全一致，可以在根目录的 `wiring.go` 中加入薄适配器进行转换。适配器的职责是连接边界，而不是承载业务规则。

## 五、main 是依赖关系的地图

在这种架构中，`main.go` 和 `wiring.go` 会显式创建数据库、各领域服务和 HTTP 路由：

```go
func main() {
	ctx := context.Background()

	cfg, err := config.Load()
	if err != nil {
		log.Fatal(err)
	}

	db, err := database.Open(ctx, cfg.Database)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	inventoryStore := inventory.NewStore(db)
	paymentService := payment.NewService(db, newPaymentProvider(cfg))
	cartService := cart.NewService(newRedisClient(cfg))
	orderStore := order.NewStore(db)

	orderService := order.NewService(
		orderStore,
		newCartAdapter(cartService),
		newInventoryAdapter(inventoryStore),
		newPaymentAdapter(paymentService),
	)

	mux := http.NewServeMux()
	catalog.RegisterRoutes(mux, catalog.NewService(db))
	cart.RegisterRoutes(mux, cartService)
	order.RegisterRoutes(mux, orderService)
	payment.RegisterRoutes(mux, paymentService)

	srv := &http.Server{
		Addr:              cfg.HTTP.Addr,
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       120 * time.Second,
	}

	log.Fatal(srv.ListenAndServe())
}
```

入口代码看起来比使用依赖注入框架更直接，但这正是它的优点：打开一个文件就可以看到系统有哪些组件，以及它们如何连接。

这里还体现了“标准库优先”的原则。Go 1.22 之后，`http.ServeMux` 已经支持方法和路径参数路由。如果项目没有正则路由、具名路由生成等额外需求，就不必条件反射式地引入第三方路由框架。

同时，生产环境的 HTTP Server 必须设置超时。直接调用 `http.ListenAndServe` 不带超时，慢客户端可能长时间占用连接。

## 六、一次下单请求如何流动

一次简化的结算流程如下：

```text
客户端 POST /orders
        ↓
order.Handler 解析和校验请求
        ↓
order.Service.LoadCart(userID)
        ↓
逐项预占 inventory
        ↓
创建 Pending 订单
        ↓
调用 payment 发起支付
        ├── 成功：订单改为 Paid
        └── 失败：释放库存，订单改为 PaymentFailed
        ↓
返回订单或带上下文的错误
```

这条流程应该尽量保持线性。错误和边界情况尽早返回，正常路径不被多层 `if` 和 `else` 包裹。

例如，错误返回时应补充当前操作的上下文：

```go
cart, err := s.carts.LoadCart(ctx, userID)
if err != nil {
	return Order{}, fmt.Errorf("loading cart for user %s: %w", userID, err)
}
```

错误是值，不是等待捕获的异常。调用方可以通过 `errors.Is` 或 `errors.As` 判断需要分支处理的错误，同时日志仍然保留“当时正在做什么”的信息。

## 七、并发不是默认答案

电商系统中很多操作看起来可以并发，例如对多个 SKU 同时预占库存。但是否并发必须由真实的性能需求和一致性方案决定。

如果确实需要并发，应满足以下条件：

- 使用 `context.Context` 传播取消和超时。
- 限制并发数量，避免按输入规模无限创建 goroutine。
- 每个 goroutine 都有明确的退出条件。
- 任一步失败时，能够取消剩余任务并处理已完成操作的补偿。

限制并发可以使用 `errgroup.SetLimit`，而不是默认创建一套固定 worker pool：

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)

for _, item := range items {
	g.Go(func() error {
		return stock.Reserve(ctx, item.SKUID, item.Quantity)
	})
}

if err := g.Wait(); err != nil {
	return fmt.Errorf("reserving inventory: %w", err)
}
```

需要注意：Go 1.22 或更高版本的循环变量已经按每次迭代绑定，不需要再添加旧式的 `item := item` 捕获补丁。实际编码应先查看 `go.mod` 的 Go 版本。

“channel 优先于 mutex”也不能理解为禁止锁。channel 适合传递数据和编排执行，mutex 适合保护确实需要共享的内存状态。关键是根据语义选择，而不是把其中一种写法当成教条。

## 八、测试应该围绕行为

订单测试不需要启动完整的 MySQL、Redis 和第三方支付服务。因为订单只依赖几个小接口，可以直接注入简单 fake：

```go
func TestCheckout(t *testing.T) {
	tests := []struct {
		name        string
		paymentErr  error
		wantStatus  order.Status
		wantRelease bool
	}{
		{
			name:       "payment succeeds",
			wantStatus: order.StatusPaid,
		},
		{
			name:        "payment fails and releases stock",
			paymentErr:  errors.New("payment declined"),
			wantStatus:  order.StatusPaymentFailed,
			wantRelease: true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			stock := &fakeStock{}
			payments := &fakePayment{err: tt.paymentErr}

			svc := order.NewService(
				newFakeOrderStore(),
				newFakeCart(),
				stock,
				payments,
			)

			got, err := svc.Checkout(t.Context(), "user-1")
			// 检查错误、订单状态以及失败时是否释放库存
		})
	}
}
```

测试重点是可观察行为：支付成功时订单是否变为已支付，支付失败时是否释放库存，而不是某个内部私有方法被调用了几次。

适合这类项目的测试实践还包括：

- 使用表驱动测试覆盖正常和异常路径。
- 测试辅助函数调用 `t.Helper()`。
- 文件输入输出使用 `testdata` 或内存文件系统。
- 复杂结构使用 `go-cmp` 输出可读差异。
- 并发测试使用 channel、显式同步或 `testing/synctest`，不要通过 `time.Sleep` 猜测执行进度。

## 九、容易混淆的几个问题

### 1. 领域包是否等于 DDD

不等于。这里借用了领域语言组织代码，但没有要求聚合根、领域服务、仓储接口、工厂等完整战术模式。只在实际问题需要时采用对应概念。

### 2. 领域包是否等于微服务

不等于。包是源码和依赖边界，微服务是部署和运行边界。应先把单体内部边界理清，再根据团队规模、负载和独立发布需求决定是否拆服务。

### 3. 扁平结构是否适合所有规模

不是。扁平是默认起点，不是最终目标。代码出现清晰、稳定、可独立测试的领域后，就应该拆成顶层领域包。

### 4. 是否完全不能使用 `internal`

不是。`internal` 有明确的编译器访问限制语义。当库需要在内部多个包之间共享导出类型，又不希望外部用户依赖这些类型时，它很有价值。问题在于无论项目性质如何都默认创建它。

### 5. 是否永远不该使用 Repository

不是。如果调用方确实需要在多种存储实现之间切换，或者存储边界具有独立业务意义，可以定义小接口。问题是只有一个实现时，就为了保持层次形式创建一整套 Repository 接口。

## 十、实践时的检查清单

设计或审查 Go 电商项目时，可以依次检查：

1. 包名表达的是业务职责，还是 Controller、Service 等技术层次？
2. 每个包能否用一句话说明自己的职责？
3. 是否存在 `utils`、`common` 这类归属不清的包？
4. 接口是否由消费方定义，并且只包含真正需要的方法？
5. 是否存在只有一个实现、也没有测试替换需求的大接口？
6. `main` 是否清楚展示依赖装配关系？
7. I/O 和长时间操作是否传递 `context.Context`？
8. goroutine 是否有明确的所有者、并发上限和退出路径？
9. 错误是否显式处理并使用 `%w` 补充上下文？
10. 标准库能否满足需求，第三方依赖是否有明确理由？
11. 测试是否验证行为，而不是过度绑定内部实现？
12. 当前抽象解决的是已经发生的问题，还是想象中的未来需求？

## 十一、总结

用 Idiomatic Go 设计电商平台，不是把某个目录模板原样复制过来，而是让代码结构跟随业务边界自然生长。

初期可以从简单单体开始，把商品、购物车、库存、订单和支付组织成清晰的领域包；由 `main` 显式装配依赖；由消费方定义小接口；用具体类型表达真实实现；优先使用现代标准库；通过表驱动测试和轻量 fake 验证行为。

这套方法真正要避免的不是某个特定目录名，而是没有实际收益的复杂度。随着业务和团队发展，架构当然可以演化，但每一次拆包、抽象或引入框架，都应该能够回答一个具体问题：它究竟让哪部分代码变得更清楚、更可靠或更容易改变？

参考资料：

- [spf13/go-skills](https://github.com/spf13/go-skills)
- [Idiomatic Go: The Go Way](https://github.com/spf13/go-skills/blob/main/go/SKILL.md)
- [Go CLI Architecture: Cobra & Viper](https://github.com/spf13/go-skills/blob/main/cobra-viper/SKILL.md)

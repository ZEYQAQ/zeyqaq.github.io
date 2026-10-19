为了解决多种语言之间相互调用接口的问题，IDL应运而生。在分布式系统、微服务或者是跨语言/跨进程通信中，不同的模块可能用了不同的技术栈（如Go、Java等）。IDL就类似于一份技术契约（Contact），它不包含实际的业务代码，仅只是定义有哪些字段类型编号以及提供了哪些接口方法。

IDL都定义了哪些内容？可以拿一份标准的示例（以Protobuf为例）来看：

```
syntax = "proto3"; //声明当前 .proto 文件使用的是 Protocol Buffers v3 版本语法。

package order.service.v1; //定义 Protobuf 内部的逻辑包名（命名空间），这里表示当前文件属于 order（订单域）下的 service 服务，版本为 v1。

option go_package = "example.com/order/service/v1;orderv1";//专门针对 Go 语言的代码生成配置，指定生成的 Go 代码应该放在哪个 Go 模块路径下，以及对应的 Go Package 名称。
option java_multiple_files = true;//每个 message、enum、service 都会被拆分生成为独立的 .java 源文件（例如单独生成 OrderInfo.java、BaseResp.java、OrderServiceGrpc.java 等），结构更清晰、便于维护与引用。
option java_package = "com.example.order.service.v1";//专门针对 Java 语言的代码生成配置，指定生成的 Java 类文件所在的 package。

// 1. 业务错误码定义（枚举）
enum ErrorCode {
  SUCCESS = 0;                  // 成功
  INVALID_ARGUMENT = 10001;     // 参数校验失败
  ORDER_NOT_FOUND = 20001;      // 订单不存在
  ORDER_ALREADY_PAID = 20002;   // 订单已支付，无法重复操作
  INSUFFICIENT_STOCK = 30001;   // 库存不足
  INTERNAL_SERVER_ERROR = 50001;// 服务端内部异常
}

// 2. 订单状态枚举
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  PENDING_PAYMENT = 1;
  PAID = 2;
  SHIPPED = 3;
  CANCELLED = 4;
}

// 3. 通用响应结构（BaseResp，承载错误码与错误信息）
message BaseResp {
  ErrorCode code = 1;          // 业务错误码
  string message = 2;          // 错误或提示信息
  map<string, string> extra = 3;// 链路追踪 trace_id 或诊断透传信息
}

// 4. 复杂业务数据结构
message OrderItem {
  int64 sku_id = 1;
  int32 quantity = 2;
  int64 unit_price_cents = 3;  // 单价（分）
  string sku_name = 4;
}

message OrderInfo {
  string order_id = 1;
  int64 user_id = 2;
  OrderStatus status = 3;
  int64 total_amount_cents = 4;// 总金额（分）
  repeated OrderItem items = 5;// 订单商品列表（集合/数组）
  int64 created_at = 6;        // 秒级时间戳
}

// 5. 接口 1：创建订单（复杂入参 & 出参）
message CreateOrderRequest {
  int64 user_id = 1;
  repeated OrderItem items = 2; // 购买项列表
  string client_ip = 3;
  string idempotency_key = 4;   // 幂等 Token（防止重试导致重复下单）
}

message CreateOrderResponse {
  BaseResp base_resp = 1;       // 统一错误码/状态
  string order_id = 2;          // 创建成功的订单号
  int64 expire_time = 3;        // 支付过期时间
}

// 6. 接口 2：批量/分页查询订单（分页入参 & 列表出参）
message QueryOrdersRequest {
  int64 user_id = 1;
  OrderStatus filter_status = 2;// 状态筛选
  int32 page_size = 3;          // 每页数量（默认 20）
  string page_token = 4;        // 分页游标/Token
}

message QueryOrdersResponse {
  BaseResp base_resp = 1;
  repeated OrderInfo orders = 2;
  string next_page_token = 3;   // 下一页游标
  bool has_more = 4;            // 是否还有更多数据
}

// 7. 服务定义（包含多个 RPC 接口）
service OrderService {
  // 创建订单
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
  // 分页查询订单
  rpc QueryOrders (QueryOrdersRequest) returns (QueryOrdersResponse);
}
```

可以看到IDL实际上并没有具体的代码实现，只是显示声明了自己对应什么语言以及参数、接口实现。以Protobuf为例，实际上这个IDL会给到protoc（Protobuf 编译器）。protoc会根据目标编程语言和插件（Plugin）来生成目标语言的原生源代码文件。

具体有哪些？上面这份，如果是面向go语言编译，

```
protoc --go_out=. --go-grpc_out=. order.proto
```

会产出：

1、结构体

```
type CreateOrderRequest struct {
    UserId         int64        `protobuf:"varint,1,opt,name=user_id,json=userId,proto3"`
    Items          []*OrderItem `protobuf:"bytes,2,rep,name=items,proto3"`
    ClientIp       string       `protobuf:"bytes,3,opt,name=client_ip,json=clientIp,proto3"`
    IdempotencyKey string       `protobuf:"bytes,4,opt,name=idempotency_key,json=idempotencyKey,proto3"`
}
```

2、gRPC的通信骨架

服务端接口（Server Interface）：

```
type OrderServiceServer interface {
    CreateOrder(context.Context, *CreateOrderRequest) (*CreateOrderResponse, error)
    QueryOrders(context.Context, *QueryOrdersRequest) (*QueryOrdersResponse, error)
    mustEmbedUnimplementedOrderServiceServer()
}
```

客户端 Client 桩代码：

```
type OrderServiceClient interface {
    CreateOrder(ctx context.Context, in *CreateOrderRequest, opts ...grpc.CallOption) (*CreateOrderResponse, error)
}
// 实现：内部封装了底层网络封包、发送和接收逻辑
```





另一种IDL类型：Thrift。

举例：

```
namespace go order.service
namespace java com.example.order.service

// 1. 通用响应结构 BaseResp
struct BaseResp {
    1: required i32 code,                // 0 表示成功，非 0 表示具体错误码
    2: required string message,          // 错误信息描述
    3: optional map<string, string> extra // 追踪/扩展字段
}

// 2. 订单状态
enum OrderStatus {
    PENDING = 1,
    PAID = 2,
    CANCELLED = 3
}

struct OrderItem {
    1: required i64 sku_id (api.body="sku_id"),
    2: required i32 quantity (vt.gt="0"),             // 字段校验注解：数量必须 > 0
    3: required i64 unit_price_cents
}

// 3. 入参与出参结构
struct CreateOrderReq {
    1: required i64 user_id,
    2: required list<OrderItem> items (vt.min_size="1"), // 至少包含 1 个商品
    3: optional string remark
}

struct CreateOrderResp {
    1: required BaseResp base_resp,
    2: optional string order_id
}

// 4. RPC 服务接口
service OrderService {
    CreateOrderResp CreateOrder(1: CreateOrderReq req)
}
```



相比较，Protobuf仅做了序列化和数据定义，而Thift定义了全部的：从IDL、编解码（Proctocol）到底层TCP读写（Transport）再到多线程并发模型（Server）。



Go1.8版本后加入了plugin特性，允许主程序动态加载另外一段go代码，即将额外的go代码单独编译成一个动态链接库文件（.so文件），让主程序运行期间动态加载并执行而无需重新编译主程序。

在压测领域，发压机只需要常驻运行主程序，而发压逻辑可以随时用plugin机制替换，这样可以热替换，提高发压机运行效率。



对于IDL，直接用git做版本管理是正常的，因为这种文件有天然的代码属性。但是通常存在网络隔离、依赖其他仓库的Thrift/Proto导致解析失败、随分支变动无法精确固定 等问题。现在存在一种专门用于api展示调用的平台，可以使用这种平台，这一段属于封装上层的私有内容，不再赘述。


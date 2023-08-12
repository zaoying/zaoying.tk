前段时间，使用构造器模式重构了`http` 工具类库，顺带优化一下`AuditLog` 模块的代码，然后很意外地被领导发现，他跟我说：这就是链式调用。这句话勾起我多年前的回忆

```go
type LogLevel = string 
const (
    Debug LogLevel = "Debug" 
    Info LogLevel = "Info" 
    Warn LogLevel = "Warn" 
    Error LogLevel = "Error"
)

struct AuditLog {
    User string
    Operation string
    Level LogLevel
    Result string
    Timestamp time.Date
}

// 重构前 
sendAuditLog({
    User: "admin",
    Operation: "delete user",
    Result: "failed",
    Level: "Error",
    Timestamp: "2022-10-13"
})

// 重构后 
Builder().User("admin").Operation("delete user").Error("Failed")
```

早在十多年前，也就是2012年左右的光景，那时候前端还是jQuery的天下，大家非常熟悉的React、Angular和Vue三剑客还没兴起。我作为一个刚入学的大学生接触前端的第一个框架就是jQuery，那时候IE浏览器的份额还很高，jQuery抹平IE、Chrome和FireFox之间的差异，而且它特有的**链式调用** 更是操作DOM的一把利器。

```js
$("selector").html("abcd")
```

但随着技术的演进，尤其是V8引擎的强大性能改进，使得Javascript从一个玩具语言逐渐成为最受欢迎的编程语言，像React、Angular和Vue这样的MVVM类库框架逐渐兴起，jQuery逐渐退出舞台。

```js
const eventNumbers = \[1,2,3,4,5,6\].filter(num => num % 2 == 0)
    .map(item => \`${item} is event number\`)
```

虽然像Array、Map这样的数据结构还支持`map`、`filter`等链式操作，但很少人再提起**链式调用**，反而是隔壁Java用一个全新的词汇 **Stream API**，但Java的实现是通过Stream的抽象接口，局限于实现Collection接口的数据结构。

直到后来专门去学习函数式编程，接触`柯里化` 等概念。可惜Go 对函数式编程的支持非常一般，连最基本的箭头函数都不支持，所以不打算像Rust那样使用宏实现自动柯里化，退而求其次，选择中规中矩的`OOP` 常见的`构造器模式`，笨是笨点，但至少容易理解和维护。

```go
type LogLevel = string
const (
    Debug LogLevel = "Debug"
    Info LogLevel = "Info"
    Warn LogLevel = "Warn"
    Error LogLevel = "Error"
)

struct AuditLog {
    User string
    Operation string
    Result string
    Level LogLevel
    Timestamp Date
}

func (log *AuditLog) User(user string) OperationInterface {
    log.User = user
    return log
}

func (log *AuditLog) Operation(op string) LevelInterface {
    log.Operation = op
    return log
}

func (log *AuditLog) Debug(result string) *AuditLog {
    log.Level = Debug
    log.Result = result
    return log
}

func (log *AuditLog) Info(result string) *AuditLog {
    log.Level = Info
    log.Result = result
    return log
}

func (log *AuditLog) Warn(result string) *AuditLog {
    log.Level = Warn
    log.Result = result
    return log
}

func (log *AuditLog) Error(result string) *AuditLog {
    log.Level = Error
    log.Result = result
    return log
}

// 构造函数
func NewAuditLog() *AuditLog {
    return &AuditLog{
        User: "",
        Operation: "",
        Result: "",
        Level: Error,
        Timestamp: time.Now()
    }
}

// 构造器模式 
func Builder() UserInterface {
    return NewAuditLog()
}

// 针对User字段封装的setter函数 
interface UserInterface {
    User(string) OperationInterface
}

// 针对Operation字段封装的setter函数
interface OperationInterface {
    Operation(string)
}

// 针对Level字段封装的setter函数，对枚举进行特殊处理 
interface LevelInterface {
    Debug(string) *AuditLog
    Info(string) *AuditLog
    Warn(string) *AuditLog
    Error(string) *AuditLog
}
```

重构后的代码看起来更简洁且条理清晰，最重要的是看起来像符合自然语言的阅读习惯，这被称之为代码的 `语义化`

```go
// 重构后 
Builder().User("admin").Operation("delete user").Error("Failed")
```

以上的重构过程需要遵守哪些规则或者诀窍，我列出来以下5点供大家参考：

1. 每个字段都抽取出一个 `interface` ，interface 包裹着 `setter` 方法
2. 通过控制 `interface` 内部的 `setter` 方法返回值类型来控制方法的调用顺序，比如 `UserInterface` 的返回值是 `OperationInterface`
3. 对于类型是`枚举`的字段，比如说 `Level` 字段的类型就是只有 `Debug`、`Info`等有限个数的，可以分别创建别名方法，比如 `Debug()`、`Info()`
4. 每个枚举类型的 `setter` 方法都可以额外设置一个非枚举的字段，比如说 `Debug(string)` 能够同时设置 `Level` 和 `Result` 两个字段的值。
5. 最后一个字段的 `setter` 方法要返回完整的结构体，或者被特定interface包裹起来的结构体，目的是隐藏实现细节。

以上过程非常公式化，完全可以通过代码自动生成，然后再后期人为调整一下顺序，使得方法调用顺序更适合阅读习惯。

我认为比较值得讨论的点，就是使用 `interface` 代替原来的 `枚举`，让代码看起来更简洁易懂且语义化。

围绕着 *使用 `interface` 代替 `枚举`* 的话题，我想进一步探讨 `Go` 语言中如何实现复杂类型枚举

由于Go语言的限制，Go语言中的枚举都是通过数字、字符串等基本类型来模拟的，不存在真正意义上的枚举。

要实现复杂结构的枚举，往往需要借助全局变量，但是全局变量并不是只读，在运行过程中容易被篡改，存在数据竞争的风险。

```go
struct ResponseEnum {
    Code int
    MsgEn string
    MsgCn string
    Extra Extra
}

struct Extra {
    StatusCode string
    Message string
}

var Succeed = ResponseEnum {
    Code: 200,
    MsgEn: "request succeed",
    MsgCn: "请求成功"
}
var BadRequest = ResponseEnum {
    Code: 400,
    MsgEn: "invalid request",
    MsgCn: "请求参数有误"
}
var Unauthorized = ResponseEnum {
    Code: 401,
    MsgEn: "needs login",
    MsgCn: "需要登陆"
}
var NotFound = ResponseEnum {
    Code: 404,
    MsgEn: "the requested resource is not existed",
    MsgCn: "资源不存在"
}
```

使用 `interface` 替换 `枚举` 重构：

```go
interface Response {
    Code() int
    MsgEn() string
    MsgCn() string
    Extra() *Extra
}

struct httpResponse {
    Code int `json: "code"`
    MsgEn string `json: "msgEn"`
    MsgCn string `json: "msgCn"`
    Extra Extra `json: "extra"`
}

func (resp *httpResponse) Code() int {
    return resp.Code
}

func (resp *httpResponse) MsgEn() string {
    return resp.MsgEn
}

func (resp *httpResponse) MsgCn() string {
    return resp.MsgCn
}

func (resp *httpResponse) Extra() Extra {
    return resp.Extra
}

func Succeed(extra Extra) Response {
    return &httpResponse {
        Code: 200,
        MsgEn: "request succeed",
        MsgCn: "请求成功", 
        Extra: extra
    }
}

func BadRequest(extra Extra) Response {
    return &httpResponse {
        Code: 400,
        MsgEn: "invalid request",
        MsgCn: "请求参数有误" 
        Extra: extra
    }
}

func Unauthorized(extra Extra) Response {
    return &httpResponse {
        Code: 401,
        MsgEn: "needs login",
        MsgCn: "需要登陆" 
        Extra: extra
    }
}

func NotFound(extra Extra) Response {
    return &httpResponse {
        Code: 401,
        MsgEn: "the requested resource is not existed",
        MsgCn: "资源不存在" 
        Extra: extra
    }
}
```

重构完后，原本的全局变量都变成公开函数，每次调用函数都会返回一个全新结构体示例，自然就不会存在数据竞争的场景。

而且返回的结果类型都是 `interface` ，只提供读取操作屏蔽修改操作，消除运行中篡改数据的可能，提高代码的健壮性。

并且可以通过创建 `函数别名` 的方式自定义枚举：

```go

// DuplicatedName 所有场景都通用的自定义重名资源错误枚举
func DuplicatedName() Response {
    return BadRequest(Extra{StatusCode: "CustomizedCode001", Message: "资源名称已存在"})
}

// ImageError 跟镜像相关的错误枚举
func ImageError(code string, msg string) Response {
    // 状态码统一加上IMG前缀
    return BadRequest(Extra{StatusCode: "IMG" + code, Message: msg})
}
```

## 总结

链式调用 `Chain Call` 更像是一种俗称，它常见的实现方式有两种：CPS(Continuation Passing Style) 和 State Machine两种，
前者是函数式的柯里化 `Currying` ，一般都是延迟求值 `lazy evaluation`，具体做法是把中间状态保存到函数栈帧中，默认是线程安全的；
后者是面向对象的状态机 `State Machine` ，也可以做到延迟求值，具体做法是把中间状态保存到结构体中，但因为函数栈帧的大小有限，最终都会逃逸到堆上，因此性能不是最佳，默认也不是线程安全的，优点是易于理解。


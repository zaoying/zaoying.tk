## useResource：声明式API与useMock基于依赖注入的mock工具

前不久组内的萌新用不知道从哪里学来的技术，说要封装一套 `axios` 库供大家使用。

等他开发完，在 `code review` 环节，大家看到他写的代码都面面相觑，不知道该如何评价。

我一时间也不知道该如何评价，只能提醒他不要写死代码，目前 `axios` 还没入选开源库，后期有可能换成其他替代品。

会后我专门到网上搜一番，发现二次封装 `axios` 的案例确实不少，但给我感觉其实都半斤八两，不见得哪个更优秀。

当时我们刚从Java切换到Go，由于Go对于 swagger 支持不够好，前后端对接的接口文档需要手写。

有时候后端修改了接口没有通知前端，经常遇到相互扯皮的事情。

我突发奇想，既然Go对注解、装饰器的支持很不好，前端的 `typescript` 语法跟 `Java` 十分相似，为什么不把Java那套照搬到前端？

不仅能解决前端接口封装的问题，还能规避go不支持swagger文档的问题。

### useResource：声明式API

说干就干，我参考 `Open Feign` 的设计，`Feign` 的设计很大程度上借鉴了 `Spring MVC` 。

只是 `Feign` 主要面向客户端，而 `Spring MVC` 面向服务端，两者的注解大同小异，Feign 兼容后者而已。

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);
}
```

显然这种声明式API的设计，比那些二次封装 `axios` 的方案优雅太多了，真正做到抽象接口与具体实现分离。

声明式API可以不改动业务代码的前提下，根据实际情况把具体实现在原生 `fetch` 和 `axios` 之间切换。

#### 装饰器

其实说照搬Java的说法是不正确的，Typescript 只有装饰器的说法，并没有注解。

而且两者差别还挺大的，Java是先定义注解`Annotation`，然后在运行时通过反射获得注解的元数据`metadata`；

然而装饰器 `Decorator` 的做法就非常直接白，直接一次性把所有的事情做完了。

```ts
export type Method = "GET" | "POST" | "PUT" | "PATCH" | "DELETE" | "OPTION"

export interface Exchange {
    (...args: any[]): Promise<Response>
}

export interface Operation {
    method?: Method
    path?: string
    headers?: Record<string, string>
    pathVariables: {name: string, order: number}[]
    requestBody?: {order: number, encode: (body: any) => BodyInit}
}

export interface Operations {
    [key: string]: Operation
}

export interface Resource {
    exchange: Exchange
    endpoint?: string
    resourceName?: string
    headers?: Record<string, string>
    operations?: Operations
}

export const RESTfulHeader: Record<string, string> = {
    "Content-Type": "application/json"
}

export function RESTful(endpoint: string, resource?: string, headers?: Record<string, string>) {
    return function<T extends { new (...args: any[]): Resource}>(target: T) {
        return class extends target {
            constructor(...args: any[]) {
                super(...args)
                this.endpoint = endpoint
                this.resourceName = resource
                this.headers = headers ? {...headers, ...RESTfulHeader} : {...RESTfulHeader}
            }
        }
    }
}

export function RequestMapping(method: Method, path: string, headers?: Record<string, string>) {
    return function(target: Resource, methodName: string, descriptor: PropertyDescriptor) {
        if (!target.operations) {
            target.operations = {}
        }
        const op = target.operations[methodName] ?? {pathVariables: []}
        op.method = method
        op.path = path
        op.headers = headers
        target.operations[methodName] = op
    }
}

export function Get(path: string, headers?: Record<string, string>) {
    return RequestMapping("GET", path, headers)
}

export function Post(path: string, headers?: Record<string, string>) {
    return RequestMapping("POST", path, headers)
}

export function Put(path: string, headers?: Record<string, string>) {
    return RequestMapping("PUT", path, headers)
}

export function Patch(path: string, headers?: Record<string, string>) {
    return RequestMapping("PATCH", path, headers)
}

export function Delete(path: string, headers?: Record<string, string>) {
    return RequestMapping("DELETE", path, headers)
}

export function Option(path: string, headers?: Record<string, string>) {
    return RequestMapping("OPTION", path, headers)
}

export function PathVariable(name: string) {
    return function(target: Resource, propertyKey: string | symbol, parameterIndex: number) {
        if (!target.operations) {
            target.operations = {}
        }
        const methodName = String(propertyKey)
        const op = target.operations[methodName] ?? {pathVariables: []}
        const pv = {name: name, order: parameterIndex}
        op.pathVariables.push(pv)
        target.operations[methodName] = op
    }
}

export const PV = PathVariable

export interface Encoder<T> {
    (src: T): BodyInit
}

export function RequestBody<T>(encoder: Encoder<T>) {
    return function(target: Resource, propertyKey: string | symbol, parameterIndex: number) {
        if (!target.operations) {
            target.operations = {}
        }
        const methodName = String(propertyKey)
        const op = target.operations[methodName] ?? {pathVariables: []}
        op.requestBody = {order: parameterIndex, encode: encoder}
        target.operations[methodName] = op
    }
}

export function JSONBody() {
    return RequestBody<Object>(JSON.stringify)
}

export function PlainBody() {
    return RequestBody<Object>(String)
}

export function FileBody() {
    return RequestBody<Blob>((src) => src)
}
```

然而我在实现的过程，还是坚持把这个过程给解耦了，装饰器只是单纯地把元数据保存到目标的 `Resource` 中。

#### useResource

接下来就是把保存在 `Resource` 的元数据读取出来，然后把 `exchange` 函数替换掉。

```ts
import { Delete, Exchange, Get, JSONBody, PV, Post, Put, RESTful, Resource } from "../annotations/restful"
import { useIoC } from "./ioc"

export interface Provider<T extends Resource> {
    (exchange: Exchange): T
}

export interface RequestInterceptor {
    (req: RequestInit): RequestInit
}

export interface ResponseInterceptor {
    (res: Response): Response
}

const globalRequestInterceptor: RequestInterceptor[] = []
const globalResponseInterceptor: ResponseInterceptor[] = []

export function addRequestInterceptor(interceptor: RequestInterceptor) {
    globalRequestInterceptor.push(interceptor)
}

export function addResponseInterceptor(interceptor: ResponseInterceptor) {
    globalResponseInterceptor.push(interceptor)
}

export function useResource<T extends Resource>(provider: (exchange: Exchange) => T): T {
    const context = useIoC()
    const exchange = context.inject(DefaultExchange)
    const sub = context.inject(provider)
    const resource = sub(exchange)
    invoke(resource, resource)
    return resource
}

function DefaultExchange(...args: any[]) {
    return Promise.resolve(new Response("{}"))
}

function invoke<T extends Resource>(resource: T, top: T) {
    const proto = Object.getPrototypeOf(resource)
    if (!proto) {
        return
    }
    invoke(proto, top)
    const props = Object.getOwnPropertyDescriptors(resource)
    for (const key in props) {
        const prop = props[key].value
        if (typeof prop == "function") {
            const exchange = sendRequest(key, resource, top)
            if (exchange) {
                const replace = prop.bind({...resource, exchange: exchange})
                const map = new Map([[key, replace]])
                Object.assign(resource, Object.fromEntries(map.entries()))
            }
        }
    }
}

function sendRequest<T>(methodName: string, res: Resource, top: Resource): Exchange | undefined {
    if (!res.operations) {
        return 
    }
    const op = res.operations[methodName]
    if (!op) {
        return
    }
    const headers = top.headers ?? {}
    const opHeaders = op.headers ?? {}
    return async (...args: any[]) => {
        let path = op.path
        if (path && op.pathVariables) {
            for (const pv of op.pathVariables) {
                path = path.replace("{" + pv.name + "}", String(args[pv.order]))
            }
        }
        const url = `${top.endpoint}/${top.resourceName}/${path}`
        let request: RequestInit = {
            method: op.method,
            headers: {...headers, ...opHeaders}
        }
        if (op.requestBody) {
            const order = op.requestBody.order
            request.body = op.requestBody.encode(args[order])
        }
        try {
            for (const interceptor of globalRequestInterceptor) {
                request = interceptor(request)
            }
            let response = await fetch(url, request)
            for (const interceptor of globalResponseInterceptor) {
                response = interceptor(response)
            }
            return Promise.resolve(response)
        } catch (e) {
            return Promise.reject(e)
        }
    }
}
```

一时间看不懂所有代码实现也没关系，可以先看看怎么使用：

先编写一个实现增删改查的基类 `CURD<T>`，`T` 由子类决定，再继承基类编写 `UserResource`

```ts
import { Delete, Exchange, Get, JSONBody, PV, Post, Put, RESTful, Resource } from "../annotations/restful"

@RESTful("example.com", "resource")
export class CURD<T> implements Resource {
    exchange: Exchange
    constructor(exchange: Exchange) {
        this.exchange = exchange
    }

    @Get("?page={page}&pageSize={pageSize}")
    async list(@PV("page") page?: number, @PV("pageSize") pageSize?: number): Promise<T[]> {
        return (await this.exchange(page ?? 1, pageSize ?? 10)).json()
    }

    @Post("")
    async create(@JSONBody() t: T): Promise<Response> {
        return this.exchange(t)
    }

    @Get("{id}")
    async get(@PV("id") id: string): Promise<T> {
        return (await this.exchange(id)).json()
    }

    @Put("{id}")
    async update(@PV("id") id: string, @JSONBody() t: T): Promise<Response> {
        return this.exchange(id, t)
    }

    @Delete("{id}")
    async delete(@PV("id") id: string): Promise<Response> {
        return this.exchange(id)
    }
}

export interface User {
    username: string
    password: string
    role: string[]
}

@RESTful("localhost", "users")
export class UserResource extends CURD<User> {
}

export function UserResourceProvider(exchange: Exchange): UserResource {
    return new UserResource(exchange)
}
```

接着，通过注入 `UserResourceProvider` 获得 `UserResource` 的实例，最后通过实例方法调用后端的接口：

```ts
const userRes = useResource(UserResourceProvider)
userRes.list().then(console.info)
const user = {username: "", password: "", role: []}
userRes.get('1').then(console.info)
userRes.create(user).then(console.info)
userRes.update('1', user).then(console.info)
userRes.delete('1').then(console.info)
```

#### 拦截器

给每个request设置token

```ts
addRequestInterceptor((req) => {
    const authToken = {}
    if (req.headers) {
        const headers = new Headers()
        headers.append("Authorization", "bear:xxxxx")
        if (req.headers instanceof Array) {
            for (const h of req.headers) {
                headers.append(h[0], h[1])
            }
        }
        req.headers = headers
    }
    req.headers = authToken
    return req
})
```

### useMock：基于依赖注入的mock工具

组内的成员都是搞前端开发的新手，不知道如何 `mock` 后端接口。

我想起以前从没有为这件事情发过愁，原因是后端接口都接入 `swagger/openapi` ，可以直接生成mock server。
只是后端切换到Go以后，他们不知道该如何接入 `swagger` ，只能每个人都在本地维护一套 mock server。

关键是他们都担心 mock 代码会影响到生产环境，所以都没有提交代码仓库。
结果遇到某个问题需要定位，还得一个个找他们要 mock 数据。

现在有了依赖注入，要实现 mock 功能简直不要太容易，几行代码就封装一个 `useMock` ：

```ts
import { Resource } from "../annotations/restful";
import { useIoC } from "./ioc";
import { Provider } from "./resource";

export function useMock<T extends Resource>(provider: Provider<T>, sub: Provider<T>) {
    const context = useIoC()
    context.define(provider, sub)
}
```

#### mockServer

对于已经在使用 `mock Server` 的接口，可以继承派生出一个子类： `XXXResourceForMock`，
然后通过 `RESTful` 设置新的 `endpoint` 和 `resource`，就可以就把请求转发到指定的服务器。

```ts
useMock(UserResourceProvider, (exchange: Exchange) => {
    @RESTful("http://mock-server:8080/backend", "users")
    class UserResourceForMock extends UserResource {
        
    }
    return new UserResourceForMock(exchange)
})
```

> 如果遇到问题，仔细观察endpoint是否为绝对路径，以及是否包含http://

#### mockOperation

如果 `mock server` 返回结果无法满足需求，可以单独 `mock` 某个方法，可以根据实际需求返回特定的结果。

```ts
useMock(UserResourceProvider, (exchange: Exchange) => {
    @RESTful("http://mock-server:8080/backend", "users")
    class UserResourceForMock extends UserResource {
        async list(page: number, pageSize: number): Promise<User[]> {
            return Promise.resolve([])
        }
        async create(user: User): Promise<Response> {
            return Promise.resolve(new Response("{}"))
        }
        async get(id: string): Promise<User> {
            return Promise.resolve({username: "", password: "", role: []})
        }
        async update(id: string, user: User): Promise<Response> {
            return Promise.resolve(new Response("{}"))
        }
        async delete(id: string): Promise<Response> {
            return Promise.resolve(new Response("{}"))
        }
    }
    return new UserResourceForMock(exchange)
})
```

#### pure_func

为了防止以上 `mock` 操作一不小心影响到生产环境，可以定义一个 `developMockOnly` 函数：

```ts
// 只用于开发环境的mock操作
function developMockOnly() {
}
```

把所有的 `mock` 操作都放到上面的函数内部，然后修改生产环境的 `webpack` 配置：

```js
{
 minimizer: [
      new TerserPlugin({
        terserOptions: {
           extractComments: 'all',
           compress: {
               pure_funcs: ['console.info', 'developMockOnly']
           },
	    }
      }),
    ]
}
```

把 `developMockOnly` 加到 `pure_funcs` 数组中。

这样即便把 `mock` 操作提交到主干分支，也不会出现开发环境的mock操作不会影响到生产环境的问题。

## 总结

以上代码早在半年前就已经写好，奈何公司的保密措施非常严格，没有办法把代码带出来。

出来之后，想重新实现一遍的想法在脑海中酝酿许久，终于在上周末花了一天的时间就写出来大部分代码。

然后又额外花了一天时间，解决一些潜在的问题，然后写了本文分享给大家，希望大家都能从中受到启发。
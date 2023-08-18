## useIoC答疑
距离上次发表 [useIoC：仅一百多行代码实现前端的依赖注入](/frontend/dependenccy-inject.html) 已经过去一个多月，期间网友和我进行多次深入且友好的互动。网友给我提了不少疑问和建议，我经过一个多月的思考和实践，决定将这些疑问和建议分享给大家。

### 1. 为什么不能照搬Angular的依赖注入？
> Angular依赖注入的对象是**实例**，并且要求必须提供**无参数的构造函数**。

这个要求和 React 对 `Function Component` 的定义产生非常严重冲突：

```tsx
type FC<P = {}> = FunctionComponent<P>;

interface FunctionComponent<P = {}> {
    (props: P, context?: any): ReactNode;
    propTypes?: WeakValidationMap<p> | undefined;
    contextTypes?: ValidationMap<any> | undefined;
    defaultProps?: Partial<any> | undefined;
    displayName?: string | undefined;
}
```

强行照搬Angular的依赖注入，需要对现有代码进行重构，把入参全部去掉或改成可选，这显然不符合实际。

其次，稍微具有亿点观察力的人，很容易发现 React 规定所有函数组件返回值必须是 `ReactNode` 。

而 `ReactNode` 的定义更是万金油，地位相当于 Java 的 `Object` 和 Go 的 `interface{}` 。


```
type ReactNode =
        | ReactElement
        | string
        | number
        | Iterable<reactnode>
        | ReactPortal
        | boolean
        | null
        | undefined
        | DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES[keyof DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES];
```

想象一下，注入的实例类型都是 ReactNode ，原来是想注入一个Input，结果注入的是Button。
当你无法限制具体的注入变量的类型时，强行使用依赖注入只会产生更严重的后果！

### 2. 为什么依赖注入不直接使用React Context，反而要重新造轮子？

简短地说，就是React Context并不好用，因为每个组件注入的依赖类型和数量都不相同，相当于每个组件都需要重新定义一个Context，操作起来非常麻烦。

但可以基于 React Context 实现一个 `IoCContext` ：

```
// 通过React useContext在组件树传递ioc context
export const IoCContext = createContext<Context>(NewIoCContext())

// 从父级组件中获取ioc context
export const useIoC = function(): Context {
    return useContext(IoCContext)
}
```

再经过 `IoCContext` 进行依赖注入，就不需要每个组件都创建新的 Context：

```tsx
import { NewIoCContext, useIoC } from 'Com/app/hooks/ioc';
import { FC, ReactNode, useId } from 'react';
import { FormPropsDispatcher, InputType, NewFormController } from "./components/form/form";

const {define} = NewIoCContext()

export type LabelProps = {
    label: ReactNode
    for?: string
    children?: ReactNode
}

export const GetLabelID = define(() => "")

export const Label: FC<labelprops> = define((props) => {
    const context = useIoC()
    const id = useId()
    context.define(GetLabelID, () => id)
    
    return <>;
        <label htmlfor={id}>{props.label}</label>
        {props.children}
    </>
})

export type InputProps = {
    name: string
    value?: InputType
    type?: string
    placeholder?: string
    readonly?: boolean
    onChange?: (val: InputType) =&gt; void
    onBlur?: (e: any) =&gt; void
    validate?: (val: InputType) =&gt; string
}

function transformValue(val: InputType)  {
    if (val instanceof File) {
        return undefined
    }
    return val
}

export const Input: FC<inputprops> = define((props) =&gt; {    
    const context = useIoC()
    const labelId = context.inject(GetLabelID)({})

    return <input id={labelId} name={props.name} type={props.type} value={transformValue(props.value)} onblur={props.onBlur} placeholder={props.placeholder} readonly={props.readonly}/>
})
```

> 原来的 `useIoC` 被重命名为 `NewIoCContext` ，原因是 React 规定所有自定义 `hook` 都必须以 `useXXX` 的方式命名，而 hook 只能在组件内部调用，但实际情况却是NewIoCContext并没有使用 React 官方的hook，所以只能改名 ；而新的 `useIoC` 由于内部调用 `useContext` 是货真价实的 `hook` ，但只能在组件内部被调用。

上述代码，主要是 `Label` 和 `Input` 两个组件，**HTML5** 规范要求 label 和 input 需要通过 id 绑定，所以 `Label` 足迹通过 `useId` 生成 id，然后通过 `GetLabelID` 注入 `Input` 组件中。
    
本来一切正常，但几乎所有的依赖注入工具都似乎建立在 `单例模式` 假设下，一旦遇到 `单例模式` 以外的场景就行不通，这其实也是为什么 Angular 早就支持依赖注入，但迟迟没能推广开来的原因之一。

以下面代码为例：
```tsx
<Form action="">
    <Label label="用户名">
        <Input name="username" value="admin">
    </Label>
    <Label label="旧密码">
        <Input name="oldPWD" type="password" validate="{checkPassword}">
    </Label>
    <Label label="新密码">
        <Input name="newPWD" type="password" validate="{checkPassword}">
    </Label>
    <Label label="重复密码">
        <Input name="repeat" type="password" validate="{checkPassword}">
    </Label>
</Form>
```

上述代码，一共出现四次 `Label` 组件，意味着注入四个不同 `GetLabelID` 函数，然而 React Fiber 并发可能使得四个组件更新的顺序不一样，也就有可能导致 `Input` 组件被注入的依赖是随机，从而引发一系列的问题。

有位网友建议给每个组件都套上一层 `IoCContext.Provider` ：

```tsx
export const Label: FC<LabelProps> = define((props) =&gt; {
    const parent = useIoC()
    const context = NewIoCContext(parent)
    const id = useId()
    context.define(GetLabelID, () =&gt; id)
    
    return &lt;&gt;
        <Label htmlfor={id}>{props.label}</Label>
        <IoCContext.Provider value="{context}">
            {props.children}
        </Poccontext.Provider>
    
})
</labelprops>
```

这样的做法是暂时解决问题，但并不是所有的场景都需要套上一层额外的 Provider。
尽管在数据量少的时候可以手动套上，避免所有场景都套上的尴尬情形；
但在数据量大的场景，比如说某个列表需要渲染上万个子项，手动套就非常麻烦，而且成为性能瓶颈。

## 对children属性的深入挖掘

熟悉 `Angular` 或 `Vue` 的开发者都知道有个 `template` 标签非常好用。

其实 React 也能实现类似的功能，函数式组件有一个很特别的属性 `children` 。

一般情况下，习惯把 `children` 的类型设置为 `ReactNode` ，这样就可以接受任意组件作为子组件，功能类似于 `Vue` 的静态 `template`，缺点是没办法向子组件传递参数。

后面发现改成 `Function Component` ，也就是 `FC` ，就可以传递参数给子组件，起到类似于 `Vue` 的 `scoped template` 的作用：

```tsx
export type LabelProps = {
    label: ReactNode
    children: FC<{id: string}>
}

export const Label: FC<LabelProps> = define((props) => {
    const id = useId()
    return <>
        <label htmlfor={id}>{props.label}</label>
        {
            props.children({id})
        }
    </>
})
```

父级组件 `Label` 通过闭包的方式向子组件 `Input` 传递参数 `id` ：

```tsx
<Form action="">
    <Label label="用户名">
        {({id}) => <Input id={id} name="username" value="admin">}
    </Label>
    <Label label="旧密码">
        {({id}) => <input id={id} name="oldPWD" type="password" validate={checkPassword}>}
    </Label>
    <Label label="新密码">
        {({id}) => <Input id={id} name="newPWD" type="password" validate={checkPassword}>}
    </Label>
    <Label label="重复密码">
        {({id}) => <Input id={id} name="repeat" type="password" validate={checkPassword}>}
    </label>
</Form>
```

这种方式，可以很好地解决 React Context 在 `非单例模式` 场景下，父级组件向子组件传递参数的问题。

React 官方对于 `children` 的相关使用一直非常模糊，以至于在开发过程中，经常受到编译器的错误信息误导；

别看现在的代码看起来很直观，但第一把就能写对的可能性很低，主要原因还是官方并没有考虑挖掘一下 children 的潜力。

大多数人一开始能写出来的代码：

```tsx
<Label label="用户名">
    <Input id={id} name="username" value="admin">
</Label>
```

然后编译报错提示：变量 `id` 没有定义，然后只能把 `id` 提取出来：

```tsx
const id = useId()
const field = <Label id={id} label="用户名">
    <Input id={id} name="username" value="admin">
</Label>
```

但是这样的写法违反 `迪米特里原则` ，显然 `id` 只在 `Label` 和 `Input` 之间传递，但却暴露给更多的组件。
假如需要在同一个地方写很多 `Label` 和 `Input` ，那还得给 `id` 取别名才能避免相互干扰。

### 异构children

有不少的组件，需要传递不同的children，但一个组件的只有一个 `children` 属性。

像 `Vue` 就提供了 `named slot` 的解决方案，但 React 没有，所以很多 `React based` 的UI库就经常这样写：

以标签页 `Tab` 为例，需要同时传递 `TabHeader` 和 `TabContent` ，并且还要求两者的长度和顺序都完全一致：

```tsx
<Tabs>
  <TabList>
    <Tab>One</tab>
    <Tab>Two</tab>
    <Tab>Three</tab>
  </Tablist>

  Tabpanels>
    <TabPanel>
      <p>one!</p>
    </TabPanel>
    <TabPanel>
      <p>two!</p>
    </TabPanel>
    <TabpPnel>
      <p>three!</p>
    </TabpPnel>
  </TabPanels>
</Tabs>
```

以上是 `Chakra UI` 的示例代码，这样写法不利于后续的维护，尤其是数据逐渐增加后，经常会出现忘记添加 `TabHeader` 或 `TabContent` 导致长度不一致，某个页签只有标题没有内容，又或者是有内容但标签为孔；而且必须要保证顺序是正确的，否则会出现标签和内容不一致的尴尬场面！

面对这种问题，常见的解决方法，把数据提取到一个数组，然后通过数组的 `map` 函数一同映射。

可惜，这样的做法又像前面的例子一样违反 `迪米特里原则` 。

以下是我的做法：
```tsx
<Tab activetab="abc">
    <Tabitem title="abc">123</tabitem>
    <Tabitem title="def">456</tabitem>
    <Tabitem title="ghi">789</tabitem>
</Tab>
```

`title` 是页签的标题，`children` 是页签的内容，`TabItem` 完整代码如下：

```
export type TabItemProps = {
    title: string,
    closeable?: boolean,
    children: ReactNode
}

export const TabItem: FC<TabItemProps> = define((tab) => {
    const context = useIoC()
    const setTab = context.inject(TabPropsDispatcher)
    useEffect(() => {
        setTab(p => {
            if (p.tabs) {
                if (p.tabs.find(t => t.title == title)) {
                    return p
                }
                return {...p, tabs: [...p.tabs, tab], activeTab: p.activeTab ?? title}
            }
            return {...p, tabs: [tab]}
        })
  })
    return <></>;
})
```

`TabPropsDispatcher` 是父组件 `Tab` 定义的，就是 `const [props, setProps] = useState()` 的 `setProps`。

第一步是 `Tab` 组件定义 `TabPropsDispatcher` 注入到 `TabItem` ，
然后 `TabItem` 通过依赖注入获得 `TabPorpsDispather` ，
最后把当前页签的标题 `title` 和内容 `content` 注册到父级组件。

> 最后一步，要求 `title` 必须是唯一的，注册到父组件那一步出现 **在组件渲染的过程中使用hook修改属性** ，按照 React 官方的说法，这么做是会导致无限循环，所以注册到父组件的函数会判断一下 `title` 是否已存在，如果已存在就停止修改属性，从而避免渲染陷入无限循环。

为了解决报错，注册组件必须被包裹在 `useEffect` 中运行，否则 React 会直接报错。
只有向父组件注册子组件的信息，`Tab` 组件才能正确的切换或关闭页签。
如果需要自定义标签页的样式，可以通过 `define(TabHeader, () => <自定义标签头>)` ，向 Tab 组件注入自定义的组件

### 更复杂的例子

表格组件 `Table` 是前端 UI 不可或缺的重要组件， 不少 `React based` 的 UI 库例如 `antd` 在封装 Table 组件的时候，由于不能使用 `named slot` 等原因，封装出来的 `Table` 组件在使用的过程中稍显啰嗦，头重脚轻：

```tsx
const dataSource = [
  {
    key: '1',
    name: '胡彦斌',
    age: 32,
    address: '西湖区湖底公园1号',
  },
  {
    key: '2',
    name: '胡彦祖',
    age: 42,
    address: '西湖区湖底公园1号',
  },
];

const columns = [
  {
    title: '姓名',
    dataIndex: 'name',
    key: 'name',
  },
  {
    title: '年龄',
    dataIndex: 'age',
    key: 'age',
  },
  {
    title: '住址',
    dataIndex: 'address',
    key: 'address',
  },
];

;
<Table datasource="{dataSource}" columns="{columns}"></Table>
```

下面通过把 children 类型改成 `FC<{name: string, rowNum: number, data: any}>` ，就可以让代码简洁不少：

```tsx
<Table data="{[]}">
    <Tablecolumn name="id" title="编号" width="{10}">
        {({data}) => <input type="checkbox" name="ids" value={data.id}>}
    </Tablecolumn>
    <Tablecolumn name="name" title="名字" width="{40}">
        {({data}) => data.name}
    </Tablecolumn>
    <Tablecolumn name="age" title="年龄" width="{20}">
        {({data}) => data.age}
    </Tablecolumn>
    <Tablecolumn name="operation" title="操作" width="{20}">
        {
            ({rowNum}) => {
                const setTable = inject(TablePropsDispatcher)
                const ctl =NewTableController(setTable)
                return <button type="danger" onclick={()=> ctl.removeData(rowNum)}>删除</button>
            }
        }
    </Tablecolumn>
</Table>
```

> `TableColumn` 组件的 `title` 类型是 `ReactNode` ，这意味着你可以使用任何组件定制表格头。

基于 `Vue` 的 `element-ui` 也有类似的写法，可以点击 [组件 | Element](https://element.eleme.cn/#/zh-CN/component/table) 。

### 小结

通过将 `children` 类型改成 `FC` 的方式模拟 Vue 的 `named slot` ，不失为弥补 `React Context` 不足之处的有效手段。

我个人比较倾向于这样写前端代码：

```tsx
<ParentComponent>
  {
      // 向子组件传递参数，不需要对外暴露任何内部属性，功能更内聚
      (props) => <ChilrdComponent>{props.fromParent}</ChilrdComponent>
  }
  {
      // 返回空的子组件，做一些带有副作用的操作：从远端服务获取数据
      (props) => {
          useEffect(() => accessRemoteServer(props))
          return <></>
      }
  }
</ParentComponent>
```

这样的代码看起来更直观易懂，因为所有与 `ParentComponet` 相关的代码都在 `ParentComponent` 组件内部，没有对外暴露任何的内部属性，使得组件的功能更加内聚，提供代码的健壮性。

## 总结

`useIoC` 和 `IoCContext` 是基于 `React Context` 的依赖注入工具，它克服了 `React Context` 需要针对每个组件的不同属性新增 `Context` 的缺点，使得各组件之间的耦合关系进一步降低，提高组件的可扩展性。

促使前端 UI 库或框架的代码水平更进一步，诸如 `SOLID` 等常见编码规范、编程模式，也能在前端落地。

而对 `children: FC` 的改造，使得功能模块更加内聚，与 `useIoC` 相互配合，就可以编写出 `高内聚` 、`低耦合` 的代码。

### 适用范围比较

`useIoC` 的优点是灵活，可以注入任意数量和类型的依赖，不破坏组件的函数签名，适合嵌套层次比较深的组件

`useIoC` 的缺点源自于 `React Context` ，定义的依赖默认是全局只有一个实例，在多实例的情况需要增加 `IoCContext.Provider` 才能避免相互干扰，在数据量比较大的情况下容易出现性能瓶颈。


`children: FC` 的优点是简单可靠且容易维护，默认并发安全，性能开销少。

`children: FC` 的缺点是扩展性差，能传递的参数类型和数量比较固定，想增加/修改属性会破坏函数签名

掌握 `useIoC` 和 `children: FC` 两种父组件向子组件传递参数的方式，并根据两者的适用范围灵活运用，就可以写出高质量的前端代码。

完整的源代码在这里[zaoying/uikit (github.com)](https://github.com/zaoying/uikit)
## useI18n——基于依赖注入实现国际化功能

### 前情提要
一个月前，我基于React hook 实现前端组件的依赖注入，前文：[useIoC：仅一百多行代码实现前端的依赖注入](/frontend/dependenccy-inject.md)。

同时也尝试基于依赖注入实现一套UI库，目的是验证前端依赖注入的可行性，然后意外解锁 React children 的全新用法：[useIoC答疑 & 对children属性的深入挖掘](./useIoC-vs-chidlren.md)。

> UI库已在Github开源：https://github.com/zaoying/uikit

在造轮子的过程中，经过不断尝试又找到一些前有未见的最佳实践，还封装一些好用的工具。

下面逐个分享给大家

### 国际化：useI18n

业务经常会遇到页面支持国际化的需求，然而组内对于国际化功能的实现却非常简单粗暴，直接在 `assets` 目录新建 `i18n` 目录，然后新增 `zh-cn.json` 或 `en-us.json` ，然后往里面写对应的中文、英文，如下：

```json
{
    "modal": {
        "confirm": "Confirm",
        "cancel": "Cancel"
    }
}
```

这样的做法一开始没啥问题，可随着页面的不断增加，渐渐地文件越来越大，有些翻译太长自动换行后就严重影响可读性。

除此之外，还会经常遇到忘记翻译的尴尬情形；或者是文件太长，想找到对应的翻译十分困难。

更令人反感的是，使用时还需要把 `json` 文件转换成 `Object` 对象，然后在使用 `TranslateService` 翻译一遍，这样的做法实在是过于鸡肋和繁琐。

我在外网以 `react i18n` 搜索一番，发现有个比较受欢迎的项目 [react-i18next](https://react.i18next.com/)，官方的代码演示如下：

```tsx
<div>{t('simpleContent')}</div>
<Trans i18nKey="userMessagesUnread" count={count}>
  Hello <strong title={t('nameTitle')}>{{name}}</strong>, you have {{count}} unread message(s). <Link to="/msgs">Go to messages</Link>.
</Trans>
```

呃。。。说实话，这个项目的做法和组内大差不差，水平都是半斤八两。

作为一个七年经验的开发，背过的八股文数不胜数，我很自然就想到通过依赖注入的方式实现国际化功能，并且理所当然地认为业界应该有很多的优秀实践，然后现实却狠狠地打我的脸。

> 为什么现在前端连个优雅的国际化工具都找不到？

> 难道那些在面试的时候，靠八股文为难别人的“高手”，估计也是死记硬背、滥竽充数的水货，目的仅仅就是为了PUA老实人？

吐槽完毕，回到正题，我的解决思路是根据不同 `locale` 分别创建独立的 `context` ，然后通过 `navigator.locales` 获取对应的 `context` 。

首先，先定义一个词条 `ModalDict` ，并且指定 `locale` 为 `en-us`：

```tsx
export const ModalDict = i18n("en-us", () => {
    return {
        confirm: "Confirm",
        cancel: "Cancel"
    }
})
```

接着通过 `register` 函数，给特定的 `locale` 注册翻译：

```ts
register("zh-cn", (locale) => {
    locale.define(ModalDict, () => ({
        confirm: "确定", cancel: "取消"
    }))
})
```


最后通过 `useI18n` 注入到组件中，通过 `dict.key` 的方式访问属性有语法提示，可以有效渐少通过 `translate(key)` 传入字符串时因为拼写错误而导致翻译失败的尴尬场面，更符合人体工程学。

```tsx
export type FooterProps = {
    confirm?: ReactNode
    cancel?: ReactNode
    onConfirm?: () => boolean,
    onCancel?: () => boolean
}

export const Footer: FC<FooterProps> = (props) => {

    const dict = useI18n(ModalDict)({})

    return <div className="two buttons">
        <Button type="primary" onClick={props.onConfirm}>
            {dict.confirm}
        </Button>
        <Button type="grey" onClick={props.onCancel}>
            {dict.cancel}
        </Button>
    </div>
}
```

细心的你可能注意到，`ModalDict` 是个无参数的函数。为什么不直接用Object呢？

使用函数的优点是可以同时支持根据不同的 `locale` 注入不同的国际化组件，而不局限于文字信息。

举个例子，有个需求需要你根据不同地区输出不同的日期：大中华地区就按年月日输出，而欧美地区的就需要反过来，按照日月年的顺序输出；不仅“年”、“月”、“日”要被翻译成 "Year" / "Month" / "Day" ，而且顺序还刚好相反。

这种情况，如果还是逐字翻译，必然要写一大堆的 `if else` ，还不如直接根据 `locale` 注入不同的组件，代码实现起来更简洁易懂。

接下来，给大家看看 `useI18n` 的具体实现：

```ts
import { Context, Func, NewIoCContext } from "./ioc"

const GlobalDict = new Map<string, any>()

/**
 * 批量注册国际化组件
 * @param locale 地域
 * @param callback 用于注册组件的回调函数
 */
export function register(locale: string, callback: (locale: Context) => void) {
    const key = new Intl.Locale(locale).toString()
    const localizeCtx = GlobalDict.get(key) ?? NewIoCContext()
    callback(localizeCtx)
    GlobalDict.set(key, localizeCtx)
}

/**
 * 根据不同的地区获取组件的上下文
 * @param locale 地域
 * @returns 组件的上下文
 */
export function get(locale: Intl.Locale) {
    const key = locale.toString()
    return GlobalDict.get(key) ?? NewIoCContext()
}

/**
 * 注册单个国际化组件
 * @param locale 地域
 * @param component 组件
 * @returns 返回带有componentId的组件
 */
export function i18n<I, O>(locale: string, component: Func<I, O>): Func<I, O> {
    const canonical = new Intl.Locale(locale)
    const localizeCtx = get(canonical)
    return localizeCtx.define(component)
}

// 默认地域
let defaultLocale: Intl.Locale = new Intl.Locale("en-us")

/**
 * 初始化默认地域
 */
export function initLocale() {
    const locale = navigator.languages?.length ? navigator.languages[0] : navigator.language;
    defaultLocale = new Intl.Locale(locale)
}

/**
 * 手动设置默认地域
 * @param locale 地域
 */
export function setLocale(locale: string) {
    defaultLocale = new Intl.Locale(locale)
}

/**
 * 根据默认地区获取国际化组件
 * @param component 国际化组件 
 * @returns 默认地区的国际化组件
 */
export function useI18n<I, O>(component: Func<I, O>): Func<I,O> {
    const localizeCtx = get(defaultLocale)
    return localizeCtx.inject(component)
}
```
算上注释总共才60多行代码，就能优雅地实现基本的国际化功能，这就是依赖注入的便利性。

### 组件之间共享状态：WithState

使用 React 搞前端开发，经常会遇到组件之间需要共享状态的场景，比如：填写地址时省份和地级市的联动、填写日期时年月日之间的联动等等，然后就参考 React 官方文档，将状态通过 `useState` 的方式将变量提升到父级组件。

一次偶然的机会，我在编写 `Stepper` 组件时，随便点点一个联动组件，结果 `Stepper` 组件意外重新渲染了，如果不是 `Stepper` 组件重新渲染数据超级加倍了，我甚至都没意识到这个问题。

```tsx
export default function Home() {
    const [state, useState] = useState("bottom")
    return <div>
        <Tooltip message="普通按钮" direction={state}>
            <Button>普通按钮</Button>
        </Tooltip>
        <Dropdown trigger="click">
            <Button type="grey">请选择方向<i className="icon">﹀</i></Button>
            <a key="top" onClick={()=> setState("top")}>上</a>
            <a key="bottom"  onClick={()=> setState("bottom")}>下</a>
            <a key="left"  onClick={()=> setState("left")}>左</a>
            <a key="right"  onClick={()=> setState("right")}>右</a>
        </Dropdown>
        
        <Stepper></Stepper>
    </div>
}
```

从源码来看，联动组件跟 `Stepper` 组件之间毫无瓜葛，仅仅是因为两者在同一个父组件，然后联动组件修改state就会触发父级组件下所有的子组件全部刷新，这实在太离谱!

然后我就尝试把联动组件提取出来单独封装一个组件：`ButtonAndDropdown` ，看这冗长的名字就知道违背了单一原则。

然而当我尝试通过Closure闭包，把 `useState` 包裹起来时，却被React警告，不能在闭包内使用hook；因此我只能封装一个hook，然而还是没用，因为没有闭包无法限制state的作用域，闭包又不能使用hook。

我只能退而求其次，把自定义hook改造成一个组件，结果出乎意外的好：

```tsx
<WithState state={"bottom" as Direction}>{
    ({state, setState}) => <>
        <Tooltip message="普通按钮" direction={state}>
            <Button>普通按钮</Button>
        </Tooltip>
        <Dropdown trigger="click">
            <Button type="grey">请选择方向<i className="icon">﹀</i></Button>
            <a key="top" onClick={()=> setState("top")}>上</a>
            <a key="bottom"  onClick={()=> setState("bottom")}>下</a>
            <a key="left"  onClick={()=> setState("left")}>左</a>
            <a key="right"  onClick={()=> setState("right")}>右</a>
        </Dropdown>
    </>
}</WithState>
```
无论是从代码实现，还是实际效果，看起来跟使用闭包毫无差别。关键是 WithState 组件的实现也很简单：

```tsx
export type StateProps<S> = {
    state: S
    children: FC<{state: S, setState: Dispatch<SetStateAction<S>>}>
}

export function WithState<S>(props: StateProps<S>) {
    const [state, setState] = useState(props.state)
    return props.children({state, setState})
}
```

> 因此，我也学到一个技巧：当自定义hook不好使的时候，可以试试用组件包裹起来

简单一个 `WithState` 组件，却可以解决困扰许久的问题，可以将组件之间的状态包裹起来，避免影响到其他的组件。

我的个人看法：React 官方应该提供如此简洁有效的方案，而不是推行那种反模式的做法。

### 防止无限渲染：Once

在封装 `Modal` 组件的过程，发现设置 `onConfirm` 或 `onCancel` 回调时，总是会陷入无限循环的渲染

```tsx
<Modal width={360} title="修改用户资料">{
    ({ctl, ctx}) => {
        ctl.onConfirm(() => {
            const setForm = ctx.inject(FormPropsDispatcher)
            const formCtl = NewFormController(setForm)
            const formRef = ctx.inject(FormReference)({})
            formCtl.validate(formRef);
            return true
        })
        return <Button onClick={ctl.open}>
            <span><i>🎨</i>打开模态框</span>
        </Button>
    }
}</Modal>
```

之前封装的组件也遇到类似的问题，但之前的组件是通过判断某个字段是否已存在。

如果该字段已经存在，就不修改任何 `state` ，从而阻断无限循环。

但这就要求 `state` 必须具备某个唯一的字段。

然而 `Modal` 组件的 `onConfirm` 和 `onCancel` 都是函数，连字段都没有。

所以我想到自定义一个hook：`useOnce`，然而还是遇到闭包不能使用hook的限制，所以我又把它改造成一个组件：

```tsx
import { FC, ReactNode, useEffect, useState } from "react"

export type OnceProps = {
    children: FC
}

export function Once(props: OnceProps) {
    const [children, setChildren] = useState<ReactNode>()
    useEffect(() => {
        if (!children) setChildren(props.children({}))
    }, [children, props])
    return <>
        {children}
    </>
}
```

然后给 `Modal` 组件用上:

```tsx
export const Modal: FC<ModalProps> = (old) => {
    const [props, setProps] = useState<ModalProps>(old)
    const context = useIoC()
    context.define(ModalPropsDispatcher, setProps)

    const className = props.className ?? "modal"
    return <>
        <Once>{
            () => props.children && props.children({
                ctx: context,
                ctl: NewModalController(setProps)
            })
        }</Once>
        <div className={`dimmer ${props.visible ? "show" : ""}`}>
        <div className={className} style={{width: props.width, height: props.height}}>
                <div className="header">{context.inject(Header)(props)}</div>
                <div className="body">
                    {context.inject(Body)(props)}
                </div>
                <div className="center footer">
                    {context.inject(Footer)(props)}
                </div>
        </div>
    </div>
    </>
}
```

套上 `Once` 组件后，效果确实如我所料，终于不会再陷入无限循环渲染。

在编写 `Stepper` 组件的时候，还是发现一些问题：虽然 `Once` 确实让当前组件只渲染一次。

但在某些特殊场景，子组件还是多次渲染，只不过不会陷入无限循环。

所以这就要子组件必须满足 `幂等` ，简单来说就是不管执行多少次的结果，都第一次执行的结果一致。

以 `Table` 组件为例，无论重复调用 `setData` 多少次，结果都是一样的：

```tsx
<Table data={new Array<{id: number, name: string, age: number}>()}>{
    ({ctl}) => {
        return <TableColumn name="id" title={<input type="checkbox" name="ids" value="*"/>} width={10}>
            {({data}) => {
                ctl.setData([
                    {id: 0, name: "张三", age: 35}, 
                    {id: 1, name: "李四", age: 28},
                    {id: 2, name: "王五", age: 40}
                ])
                return <input type="checkbox" name="ids" value={data.id}/>
            }
        </TableColumn>
    }
}</Table>
```
但调用 `appendData` 则不同，重复执行n次 `appendData` ，数据就是会重复n次。

`Stepper` 组件的子组件 `StepperItem` 就存在这样的问题，解决方法就是通过 `useId` 生成唯一ID去重。

> 对于没有唯一字段的情况 ，可以通过 `useId` 生成唯一字段进行去重。

### 泛型化组件

在开发 `Form` 组件时，曾经想过给组件加上泛型，如此在使用的时候还能享受IDE的智能提示。

但是我很快发现 `input` 的 `value` 早就被限制得死死的：`string | ReadonlyArray<string> | number | undefined` 。

而且我也发现闭包形式的 `function(){}` 以及 箭头函数形式 `() => {}` 都不支持泛型。

要使用泛型，只能老老实实地回到常规的函数定义： `function Xxx() {}` 。

所以我就把 `Table` 组件改成最常见的函数定义：

```tsx
export function TableColumn<T>(props: TableColumnProps<T>) {
    const context = useIoC()
    const setProps = context.inject(TablePropsDispatcher)
    const ctl = NewTableController<T>(setProps)
    useEffect(() => ctl.insert(props))
    return <></>
}

export function Table<T>(old: TableProps<T>) {
}
```

细心的网友可能会发现，`Table` 和 `TableColumn` 竟然不需要 `define` 函数包裹一层，那还能使用依赖注入吗？

答案就是丝毫不影响，因为我参考 React 官方对 `FunctionComponent` 的定义，重构 `Func` 的定义：

```ts
// 组件的构造函数定义
export interface Func<I, O> {
    (props: I, context?: Context): O;
    componentId?: string
    displayName?: string | undefined;
    originFunction?: Func<I, O>
}
```

这样做的好处有很多：

首先， React 在打印错误日志时能正确显示组件名称，而不是千篇一律的 `wrapper` ；

其次，所有的函数式组件都不需要通过 `define` 函数包裹，调用方可以直接通过 `inject` 进行依赖注入。

最后，被调用方还是通过 `define(component, subType)` 注入不同的子类型。

在使用的时候，泛型化的组件跟普通的组件几乎没有区别：

```tsx
<Table data={new Array<{id: number, name: string, age: number}>()}>{
    ({ctl, Column}) => {
        ctl.setData(
            {id: 0, name: "张三", age: 35}, 
            {id: 1, name: "李四", age: 28},
            {id: 2, name: "王五", age: 40}
        )
        return <Column name="id" title={<input type="checkbox" name="ids" value="*"/>} width={10}>
            {({data}) => <input type="checkbox" name="ids" value={data.id}/>}
        </Column>
    }
</Table>
```

唯一需要注意的是，`TableColumn` 被改成 `Column` ，原因是 `Table` 父组件并不能把具体的泛型 `T` 穿给子组件 `TableColumn` 。

而且 `TableColumn<T>` 的 `<T>` 和 `JSX/TSX` 语法冲突，所以只能通过以下的方式：

```tsx
export type TableArgs<T> = {
    ctx: Context
    ctl: TableController<T>
    Column: FC<TableColumnProps<T>>
}

export type TableProps<T> = {
    data: T[]
    children: FC<TableArgs<T>>
}

export function Table<T>(old: TableProps<T>) {
    const [props, setProps] = useState<TP<T>>({data: old.data, columns: []})
    const context = useIoC()
    context.define(TablePropsDispatcher, setProps)

    const tabHeader = context.inject(TableHeader<T>)
    const tabBody = context.inject(TableBody<T>)
    return <div className="table">
        <table>
            {tabHeader(props)}
            {tabBody(props)}
        </table>
        <div className="footer">
            <Once>{
                () => old.children({
                    ctx: context,
                    ctl: NewTableController<T>(setProps),
                    Column: TableColumn<T>
                })
            }</Once>
        </div>
    </div>
}
```

> 这里的 `Column` 就等价于 `TableColumn<T>`。

这种做法其实解锁一个前所未有的用法，既然可以通过闭包向子类型传递泛型 `T` ，自然也可以传递更多的依赖。

这种传递依赖的方式，比依赖注入更高效，而且直观易懂，并非没有对外暴露内部的属性。。

## 总结

* 通过依赖注入的方式来实现国际化功能特性，高效又优雅
* 将 useState 封装成 WithState 组件，可以优雅实现组件之间的共享状态
* 将 useEffect 封装成 Once 组件，可以确保 effect 只执行一次
* 子组件如果无法满足唯一性的要求，可以通过 useId 生成唯一ID
* 父级组件可以通过修改 children 的类型，向子组件传递包括但不限于泛型等信息或依赖
# Easy

## MyPick

- extends类型约束

```tsx
//示例
interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPreview = MyPick<Todo, 'title' | 'completed'>

const todo: TodoPreview = {
    title: 'Clean room',
    completed: false,
}

//实现

type MyPick<T, K extends keyof T> = {
  [p in K]: T[p] 
}
```

## MyReadonly

```tsx
//示例
interface Todo {
  title: string
  description: string
}

const todo: MyReadonly<Todo> = {
  title: "Hey",
  description: "foobar"
}

todo.title = "Hello" // Error: cannot reassign a readonly property
todo.description = "barFoo" // Error: cannot reassign a readonly property

//实现

type MyReadonly<T> = {
  readonly [p in keyof T]: T[p]
}
```

## TupleToObject

- T[number]获取元组值联合类型
- PropertyKey为内置属性类型，即string | number | symbol

```tsx
//示例
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type result = TupleToObject<typeof tuple> 
// expected { tesla: 'tesla', 'model 3': 'model 3', 'model X': 'model X', 'model Y': 'model Y'}
//实现

type TupleToObject<T extends readonly PropertyKey[]> = {
  [p in T[number]] : p
}
```

## First

```tsx
//示例
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]

type head1 = First<arr1> // expected to be 'a'
type head2 = First<arr2> // expected to be 3
//实现
type First<T extends any[]> = T extends [] ? never : T[0]

/*
type Nth<T extends any[], Num extends number = 0> = T extends []
? never
: T[Num]
*/
```

## Length

```tsx
//示例
type tesla = ['tesla', 'model 3', 'model X', 'model Y']
type spaceX = ['FALCON 9', 'FALCON HEAVY', 'DRAGON', 'STARSHIP', 'HUMAN SPACEFLIGHT']

type teslaLength = Length<tesla> // expected 4
type spaceXLength = Length<spaceX> // expected 5
//实现
type Length<T extends readonly any[]> = T["length"]

```

## MyExclude

- 分布式条件类型：联合类型的分支被单独的拿出来依次进行条件类型语句的判断，判断结果在进行合并
    1. 是联合类型
    2. 联合类型是通过泛型参数的形式传入
    3. 泛型参数在条件类型语句中需要是裸类型参数，即没有被 [] 包裹
    
    ```tsx
    Exclude<'a' | 'b' | 'c', 'a'>
    //等价于
    ('a' extends 'a' ? never : T) | ('b' extends 'a' ? never : T) | ('c' extends 'a' ? never : T)
    //等价于
    never | 'b' | 'c'
    由于 never 是底层类型(bottom type)，在联合类型中不参与运算
    因此得到 'b' | 'c'
    ```
    

```tsx
//示例
type Result = MyExclude<'a' | 'b' | 'c', 'a'> // 'b' | 'c'
//实现
type MyExclude<T, U> =  T extends U ? never : T

```

## Includes

- 为了实现类型完全相等需要使用Equal判断
- 使用infer+扩展运算符逐位递归遍历

```tsx
//示例
type isPillarMen = Includes<['Kars', 'Esidisi', 'Wamuu', 'Santana'], 'Dio'> // expected to be `false`
//实现
type Includes<T extends readonly any[], U> = 
T extends [infer L, ...infer R] 
	? Equal<U, L> extends true ? true : Includes<R, U> 
	: false;
```

## MyAwaited

- infer
- 递归

```tsx
//示例
type ExampleType = Promise<string>

type Result = MyAwaited<ExampleType> // string
//实现
type MyAwaited<T extends PromiseLike<**any**> > = T extends PromiseLike<infer P> ? (P extends PromiseLike<unknown> ? MyAwaited<P> : P) : never;

//如果改为:
type MyAwaited<T extends PromiseLike<**unknown**> > = T extends PromiseLike<infer P> ? (P extends PromiseLike<unknown> ? MyAwaited<P> : P) : never;
//则无法通过样例：
type T = { then: (onfulfilled: (arg: number) => any) => any }
```

## If

```tsx
//示例
type A = If<true, 'a', 'b'>  // expected to be 'a'
type B = If<false, 'a', 'b'> // expected to be 'b'
//实现
type If<C extends boolean, T, F> = C extends true ? T : F;
```

## Concat  Push  Unshift

- …扩展运算符可以扩展元组

```tsx
//示例
type Result = Concat<[1], [2]> // expected to be [1, 2]
//实现
type Concat<T extends Array<unknown>, U extends unknown[]> = [...T , ...U]

//示例
type Result = Push<[1, 2], '3'> // [1, 2, '3']
//实现
type Push<T extends Array<unknown>, U> = [...T, U];

//示例
type Result = Unshift<[1, 2], 0> // [0, 1, 2,]
//实现
type Unshift<T extends Array<unknown>, U> = [U, ...T]
```

## MyParameters
```tsx
//示例
const foo = (arg1: string, arg2: number): void => {}
type FunctionParamsType = MyParameters<typeof foo> // [arg1: string, arg2: number]//实现
//实现
type MyParameters<T extends (...args: any[]) => any> = 
T extends (...args: infer P) => any ? P : never
```

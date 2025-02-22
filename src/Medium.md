# Medium
## 2 MyReturnType

```tsx
//示例
const fn = (v: boolean) => {
  if (v)
    return 1
  else
    return 2
}

type a = MyReturnType<typeof fn> // 应推导出 "1 | 2"
//实现
type MyReturnType<T> = 
T extends (...args: **any**[]) => infer P ? P : never

//若为
type MyReturnType<T> = 
T extends (...args: **unknown**[]) => infer P ? P : never
//则示例过不了
```

## 3 MyOmit

- Exclude必须抽离出来，不能写成[p in (keyof T extends K ? never : keyof T)]: T[p]，因为这样不会进行分布式条件类型（联合类型不是泛型参数）

```tsx
//示例
interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPreview = MyOmit<Todo, 'description' | 'title'>

const todo: TodoPreview = {
  completed: false,
}
//实现
type Exclude<T, K> = T extends K ? never : T

type MyOmit<T, K extends keyof T> = {
  [p in Exclude<keyof T, K>]: T[p]
}
```

## 8 Readonly 2

- 交叉类型

```tsx
//示例
interface Todo {
  title: string
  description: string
  completed: boolean
}

const todo: MyReadonly2<Todo, 'title' | 'description'> = {
  title: "Hey",
  description: "foobar",
  completed: false,
}

todo.title = "Hello" // Error: cannot reassign a readonly property
todo.description = "barFoo" // Error: cannot reassign a readonly property
todo.completed = true // OK
//实现
type MyReadonly2<T, K extends keyof T = keyof T> = {
  readonly [key in keyof Pick<T, K>]: T[key]
} & {
  [key in keyof Omit<T, K>]: T[key]
}
//等价于
type MyReadonly2<T, K extends keyof T = keyof T> = 
Readonly<Pick<T, K>> & Omit<T, K>

//若这样写：
type MyReadonly2<T, K extends keyof T = keyof T> = Readonly<T> & {
  -readonly [p in Exclude<keyof T, K>]: T[p]
}
//则会把原先readonly去掉，使得下面样例不通过
interface Todo2 {
  readonly title: string
  description?: string
  completed: boolean
}
MyReadonly2<Todo2, 'description'>
```

## 9 17973 DeepReadonly / DeepMutable

```tsx
//示例
type X = {
  x: {
    a: 1
    b: 'hi'
  }
  y: 'hey'
}
type Expected = {
  readonly x: {
    readonly a: 1
    readonly b: 'hi'
  }
  readonly y: 'hey'
}
type Todo = DeepReadonly<X> // should be same as `Expected`

type X = {
  readonly a: () => 1
  readonly b: string
  readonly c: {
    readonly d: boolean
    readonly e: {
      readonly g: {
        readonly h: {
          readonly i: true
          readonly j: "s"
        }
        readonly k: "hello"
      }
    }
  }
}
type Expected = {
  a: () => 1
  b: string
  c: {
    d: boolean
    e: {
      g: {
        h: {
          i: true
          j: "s"
        }
        k: "hello"
      }
    }
  }
}
type Todo = DeepMutable<X> // should be same as `Expected`

//实现
type DeepReadonly<T> = {
  readonly [p in keyof T]: 
		T[p] extends Object 
			? T[p] extends Function ? T[p] : DeepReadonly<T[p]> 
			: T[p];
}

type DeepMutable<T extends object> = {
  -readonly [p in keyof T]: 
		T[p] extends Object 
			? T[p] extends Function ? T[p] : DeepMutable<T[p]> 
			: T[p];
}
```

## 10 TupleToUnion

```tsx
//示例
type Arr = ['1', '2', '3']
type Test = TupleToUnion<Arr> // expected to be '1' | '2' | '3'

//实现
type TupleToUnion<T extends Array<unknown>> = T[number]
```

## 12 Chainable

```tsx
//示例
const result = config
  .option('foo', 123)
  .option('name', 'type-challenges')
  .option('bar', { value: 'Hello World' })
  .get()

// 期望 result 的类型是：
interface Result {
  foo: number
  name: string
  bar: {
    value: string
  }
}
//实现
type Exclude<T, K> = T extends K ? never : T

type Chainable<T = {}> = {
  option<K extends PropertyKey, V extends unknown>(
    key: Exclude<K, keyof T>,  //实现不能重复配置
    value: V
  ): Chainable<Omit<T, K> & { [P in K]: V }>; //这里P in K虽然像map，但实际K只是一个PropertyKey而不是联合类型，只增加一个属性
	//Omit去除T中 extends K的属性，实现不能重复配置
  get(): T;
};

//Exclude与Omit处理下面的情况：
const result3 = a
  .option('name', 'another name')
  // @ts-expect-error
  .option('name', 123)
  .get()

type Expected3 = {
  name: number
}

//如果将Omit<T, K>替换为T则不能实现覆盖，Expected3应为:
type Expected3 = {
  name: string
} & {
  name: number
}
```

## 15 Last

```tsx
//示例
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]

type tail1 = Last<arr1> // expected to be 'c'
type tail2 = Last<arr2> // expected to be 1
//实现
type Last<T extends any[]> = T extends [...any[], infer U] ? U : never;
```

## 16 Pop / Shift

```tsx
//示例
type re1 = Pop<['a', 'b', 'c', 'd']> // ['a', 'b', 'c']
type re2 = Pop<[3, 2, 1]> // [3, 2]

type Result = Shift<[3, 2, 1]> // [2, 1]

//实现
type Pop<T extends any[]> = T extends [...infer L, any] ? L : [];
type Shift<T extends any[]> = T extends [any, ...infer R] ? R : []
```

## 20 PromiseAll

```tsx
//示例
const promise1 = Promise.resolve(3);
const promise2 = 42;
const promise3 = new Promise<string>((resolve, reject) => {
  setTimeout(resolve, 100, 'foo');
});
// expected to be `Promise<[number, 42, string]>`
const p = PromiseAll([promise1, promise2, promise3] as const)

//实现
declare function PromiseAll<T extends unknown[]>(values: readonly [...T]): Promise<{
  [p in keyof T]: Awaited<T[p]>
}>
```

## 62 LookUp

- 用extends在联合类型中筛选

```tsx
//示例
interface Cat {
  type: 'cat'
  breeds: 'Abyssinian' | 'Shorthair' | 'Curl' | 'Bengal'
}

interface Dog {
  type: 'dog'
  breeds: 'Hound' | 'Brittany' | 'Bulldog' | 'Boxer'
  color: 'brown' | 'white' | 'black'
}

type MyDog = LookUp<Cat | Dog, 'dog'> // expected to be `Dog`

//实现
type LookUp<U extends { type: unknown }, T extends U['type']> = 
U extends { type: T } ? U : never;
```

## 106 4803 108 TrimLeft / TrimRight / Trim

- 模板字符串、逐字符递归操作

```tsx
//示例
type trimed = TrimLeft<'  Hello World  '> // 应推导出 'Hello World  '
type Trimed = TrimRight<'  Hello World  '> // 应推导出 '  Hello World'
type trimed = Trim<'  Hello World  '> // 应推导出 'Hello World'
//实现
type Whitespace = ' ' | '\n' | '\t'

type TrimLeft<S extends string> = 
S extends `${Whitespace}${infer T}` ? TrimLeft<T> : S;

type TrimRight<S extends string> = 
S extends `${infer T}${Whitespace}` ? TrimRight<T> : S;

type Trim<S extends string> = 
S extends `${Whitespace}${infer T}` | `${infer T}${Whitespace}` 
	? Trim<T> : S;
//或
type Trim<S extends string> = TrimRight<TrimLeft<S>>
```

## 110 MyCapitalize

- 内置的**Uppercase，Lowercase**类型

```tsx
//示例
type capitalized = Capitalize<'hello world'> // 'Hello world'

//实现
type MyCapitalize<S extends string> = 
S extends `${infer L}${infer R}` ? `${Uppercase<L>}${R}` : '';
```

## 116  119 Replace / ReplaceAll

```tsx
//示例
type replaced = Replace<'types are fun!', 'fun', 'awesome'> // 'types are awesome!'
type replaced = ReplaceAll<'t y p e s', ' ', ''> // 'types'

//实现
type Replace<S extends string, From extends string, To extends string> = 
S extends `${infer L}${From extends '' ? never : From}${infer R}` 
	? `${L}${To}${R}` : S;

type ReplaceAll<S extends string, From extends string, To extends string> = 
From extends '' 
	? S 
	: S extends `${infer L}${From}${infer R}` 
		? `${L}${To}${ReplaceAll<R, From, To>}` : S;
```

## 191 AppendArgument

```tsx
//示例
type Fn = (a: number, b: string) => number

type Result = AppendArgument<Fn, boolean> // 期望是 (a: number, b: string, x: boolean) => number

//实现
type AppendArgument<Fn extends Function, A> = 
Fn extends (...args: infer P)=>infer R 
	? (...args: [...P, A]) => R : never;
//或  (注意这里Type 'Function' is not assignable to type '(...args: any) => any'，注意参数的any不能替换为unknown)
type AppendArgument<Fn extends (...args: any[])=>any, A> = 
(...args: [...Parameters<Fn>, A]) => ReturnType<Fn>;
```

## 1042 IsNever

- 对于T = never来说，应该使用`[T] extends [never]` 而不是`T extends never`,否则会按照分布式条件类型来执行，又由于never视为空的Union，分布式条件类型无法应用导致最终返回never

```tsx
//示例
type A = IsNever<never>  // expected to be true
type B = IsNever<undefined> // expected to be false
type C = IsNever<null> // expected to be false
type D = IsNever<[]> // expected to be false
type E = IsNever<number> // expected to be false
//实现
type IsNever<T> = [T] extends [never] ? true : false;
```

## 296 21220 Permutation/PermutationsOfTuple

- K extends T 利用分布式条件类型
- `K extends T ? [K, ...Permutation<Exclude<T, K>>]` 中第二个与第三个K是逐项拆分的子项

```tsx
//示例
type perm = Permutation<'A' | 'B' | 'C'>; // ['A', 'B', 'C'] | ['A', 'C', 'B'] | ['B', 'A', 'C'] | ['B', 'C', 'A'] | ['C', 'A', 'B'] | ['C', 'B', 'A']

//实现
type Permutation<T, K = T> = 
[T] extends [never] 
	? [] 
	: K extends T ? [K, ...Permutation<Exclude<T, K>>] : never;
	
	
// 示例
PermutationsOfTuple<[any, unknown, never]> 
/**
| [any, unknown, never]
| [unknown, any, never]
| [unknown, never, any]
| [any, never, unknown]
| [never, any, unknown]
| [never, unknown, any]
 */

// 实现
type Insert<T extends unknown[], U> = 
T extends [infer F,...infer L]
  ? [F, U, ...L] | [F, ...Insert<L,U>] : [U]

type PermutationsOfTuple<T extends unknown[], R extends unknown[] = []> = 
T extends [infer F,...infer L] 
  ? PermutationsOfTuple<L, Insert<R, F> | [F, ...R] >
  : R
```

## 298 LengthOfString

- 只有元组的length类型才是具体数字，数组与字符串的length类型是number，因此需要转为元组再求length

```tsx
//示例
type strlen = LengthOfString<'tjz'>; // 3

//实现
//递归有长度限制(45个字符)
type Split<S extends string> = S extends `${infer L}${infer R}` ? [L, ...Split<R>] : []

type LengthOfString<S extends string> = Split<S>['length']

//合并实现，可以支持更长的字符串
type LengthOfString<S extends string, T extends any[] = []> = 
S extends `${infer L}${infer R}` 
	? LengthOfString<R, [...T, L]> 
	: T['length']
```

## **459 Flatten**

```tsx
//示例
type flatten = Flatten<[1, 2, [3, 4], [[[5]]]]> // [1, 2, 3, 4, 5]

//实现
type Flatten<T extends unknown[]> = 
T extends [infer L, ...infer R] 
  ? L extends unknown[] 
      ? [...Flatten<L>, ...Flatten<R>] 
      : [L, ...Flatten<R>] 
  : []
```

## 527 AppendToObject

- 映射类型不能同时有两个[p in q]
- 交叉类型与同时具备两类object属性的obj类型不等价

```tsx
//示例
type Test = { id: '1' }
type Result = AppendToObject<Test, 'value', 4> // expected to be { id: '1', value: 4 }

//实现
type AppendToObject<T extends object, U extends PropertyKey, V> = {
  [p in keyof T | U]: p extends keyof T ? T[p] : V;
}

//下面的实现虽然实际应用可以等价，但不能通过Equal类型测试
type AppendToObject<T extends object, U extends PropertyKey, V> = {
  [p in keyof T]: T[p];
} & {
  [p in U]: V
}
```

## 529 Absolute

```tsx
//示例
type Test = -100;
type Result = Absolute<Test>; // expected to be "100"
//实现
type Absolute<T extends number | string | bigint> = 
`${T}` extends `-${infer R}` ? R : `${T}`;
```

## 531 StringToUnion

```tsx
//示例
type Test = '123';
type Result = StringToUnion<Test>; // expected to be "1" | "2" | "3"
//实现
type StringToUnion<S extends string> = 
S extends `${infer L}${infer R}` ? L | StringToUnion<R> : never;
```

## 599 Merge

```tsx
//示例
type foo = {
  name: string;
  age: string;
}

type coo = {
  age: number;
  sex: string
}

type Result = Merge<foo,coo>; // expected to be {name: string, age: number, sex: string}
//实现
type Merge<F, S> = {   //优先S再F，且不能直接F[p]，否则报错
  [p in keyof F | keyof S]: 
		p extends keyof S 
			? S[p] 
			: p extends keyof F ? F[p] : never;
}
//也可以使用交叉类型
type Merge<F, S> = {
  [p in keyof (F & S)]: 
		p extends keyof S 
			? S[p] 
			: p extends keyof F ? F[p] : never;
}
```

## 612 KebabCase

- 首字母大写不用加’-’，使用需要一层判断(借助泛型参数T)
- L extends Lowercase<L> 而不是Uppercase<L>，因为其他非字母字符 L === Uppercase<L> === Lowercase<L>，此时应不加’-’

```tsx
//示例
type FooBarBaz = KebabCase<"FooBarBaz">;
const foobarbaz: FooBarBaz = "foo-bar-baz";

type DoNothing = KebabCase<"do-nothing">;
const doNothing: DoNothing = "do-nothing";

//实现
//借助泛型参数T
type KebabCase<S extends string, T extends string = ""> = 
  S extends `${infer L}${infer R}` 
    ? L extends Lowercase<L> 
      ? KebabCase<R, `${T}${L}`> 
      : KebabCase<R, T extends '' ? Lowercase<L> :`${T}-${Lowercase<L>}`> 
    : T;
//仅一个泛型参数，使用Uncapitalize
type KebabCase<S extends string> = 
  S extends `${infer L}${infer R}` 
    ? R extends Uncapitalize<R> 
      ? `${Lowercase<L>}${KebabCase<R>}`
      : `${Lowercase<L>}-${KebabCase<R>}`
    : S;
```

## **645 Diff**

```tsx
//示例
type Foo = {
  a: string;
  b: number;
}
type Bar = {
  a: string;
  c: boolean
}

type Result1 = Diff<Foo,Bar> // { b: number, c: boolean }
type Result2 = Diff<Bar,Foo> // { b: number, c: boolean }

//实现
type Diff<T extends object, U extends object> = {
  [p in Exclude<keyof (T & U), keyof (T | U) >]: (T & U)[p]
}
//或
type Diff<T extends object, U extends object> = Omit<T & U, keyof (T | U)>
```

## 949 AnyOf

- 可以不必使用infer逐个比较，而是用分布式条件类型

```tsx
//示例
type Sample1 = AnyOf<[1, '', false, [], {}]> // expected to be true.
type Sample2 = AnyOf<[0, '', false, [], {}]> // expected to be false.
//实现
type Falsy = false | 0 | '' | [] | Record<PropertyKey, never> | undefined | null | never

type AnyOf<T extends readonly any[]> = 
true extends (T[number] extends Falsy ? never : true) ? true : false;
```

## 1097 IsUnion

- `T extends T` 触发条件式分布类型，后面的`[U] extends [T]` 中的T已经是逐项拆分的子项了：
    
    ```tsx
    type T = 'a' | 'b'
    type U = T
    T extends T ? [U] extends [T] ....
    //拆分为
    ('a' extends 'a' | 'b' ? ['a' | 'b'] extends ['a'] ...) |  ('b' extends 'a' | 'b' ? ['a' | 'b'] extends ['b'] ...)
    ```
    
    `['a' | 'b'] extends ['a']` 返回false，`[U] extends [T]` 而不是`U extends T` 是为了防止触发U的条件式分布类型
    
    ```tsx
    //最终
    T extends T 
        ? [U] extends [T] //U为联合类型则false，选择true分支
          ? false 
          : true 
        : never;
    //解析为
    true | true | true | ... 
    //即
    true
    ```
    
    如果T不是联合类型的话不会触发分布式条件类型，因此`T extends T` 、`[U] extends [T]` 都为true，结果返回false
    

```tsx
 //示例
type case1 = IsUnion<string>  // false
type case2 = IsUnion<string|number>  // true
type case3 = IsUnion<[string|number]>  // false
//实现
type IsUnion<T, U = T> = 
[T] extends [never] 
  ? false 
  : T extends T 
    ? [U] extends [T] ? false : true 
    : never;
```

## 1130 ReplaceKeys

```tsx
//示例
type NodeA = {
  type: 'A'
  name: string
  flag: number
}
type NodeB = {
  type: 'B'
  id: number
  flag: number
}
type NodeC = {
  type: 'C'
  name: string
  flag: number
}
type Nodes = NodeA | NodeB | NodeC
type ReplacedNodes = ReplaceKeys<Nodes, 'name' | 'flag', {name: number, flag: string}> 
/* {type: 'A', name: number, flag: string} 
 | {type: 'B', id: number, flag: string} 
 | {type: 'C', name: number, flag: string} 
would replace name from string to number, replace flag from number to string.
*/
type ReplacedNotExistKeys = ReplaceKeys<Nodes, 'name', {aa: number}> 
/* {type: 'A', name: never, flag: number} 
 | NodeB
 | {type: 'C', name: never, flag: number} 
 would replace name to never
*/
//实现
type ReplaceKeys<U extends object, T extends PropertyKey, Y extends object> = 
U extends object   //分布式条件类型
  ? {
    [p in keyof U]:   //只选存在于U的
      p extends T     //是否需要替换
      ? p extends keyof Y  //是否存在于Y
        ? Y[p]
        : never
      : U[p]          //不需替换
} : never             //不会触发

//等价于 (U extends object)分布式条件类型可以删除，同样可以触发分布式条件类型
//为什么等价暂且未知
type ReplaceKeys<U extends object, T extends PropertyKey, Y extends object> = 
{
  [p in keyof U]: 
    p extends T 
    ? p extends keyof Y
      ? Y[p]
      : never
    : U[p]
}
/* 
type P<T> = keyof T 
当T为Union时返回never

type P<T> = {
  [k in keyof T]: T[k]
}
当T为Union时返回P<T1> | P<T2> | ...，//分布式条件类型
*/
```

## 1367 RemoveIndexSignature

- as可以对K进一步操作
- `PropertyKey extends keyof T[K]` 作用暂且未知

```tsx
 //示例
type Foo = {
  [key: string]: any;
  foo(): void;
}
type A = RemoveIndexSignature<Foo>  // expected { foo(): void }

//实现
type RemoveIndexSignature<T> = {
  [K in keyof T as PropertyKey extends keyof T[K] ? never : K]: T[K];
}
```

## 1978 PercentageParser

- 使用泛型来复用L与R： `infer L extends '+' | '-' ? **L** : never;`会报错，L不能复用，因此使用泛型来复用

```tsx
 //示例
type PString1 = ''
type PString2 = '+85%'
type PString3 = '-85%'
type PString4 = '85%'
type PString5 = '85'

type R1 = PercentageParser<PString1> // expected ['', '', '']
type R2 = PercentageParser<PString2> // expected ["+", "85", "%"]
type R3 = PercentageParser<PString3> // expected ["-", "85", "%"]
type R4 = PercentageParser<PString4> // expected ["", "85", "%"]
type R5 = PercentageParser<PString5> // expected ["", "85", ""]

//实现
type CheckPrefix<T> = T extends '+' | '-' ? T : never;
type CheckSuffix<T> =  T extends `${infer P}%` ? [P, '%'] : [T, ''];
type PercentageParser<A extends string> = 
A extends `${CheckPrefix<infer L>}${infer R}` 
	? [L, ...CheckSuffix<R>] : ['', ...CheckSuffix<A>];
```

## 2070 DropChar

- 字符串匹配时L与R都可能为空字符串’’，若都可以匹配则L优先匹配最左边第一个字符，R匹配剩下的(可以是’’)

```tsx
 //示例
type Butterfly = DropChar<' b u t t e r f l y ! ', ' '> // 'butterfly!'

//实现
type DropChar<S extends string, C extends string> = 
S extends `${infer L}${C}${infer R}` ? DropChar<`${L}${R}`, C> : S
```

## 2257 MinusOne

```tsx
 //示例
type Zero = MinusOne<1> // 0
type FiftyFour = MinusOne<55> // 54

//实现
//支持number范围内整数减1操作（若用字符串表示则是不限范围的整数）
type Reverse<A extends string, Res extends string = ""> = 
  A extends `${infer L}${infer R}` ? Reverse<R,`${L}${Res}`> : Res

type ToNumber<T extends string> = 
T extends `0${infer L}` 
  ? L extends '' ? 0 : ToNumber<`${L}`> 
  : T extends `${infer R extends number}` ? R : never

type DigsNext<S = '0123456789', Res = {}> = 
S extends `${infer L}${infer M}${infer R}`
  ? DigsNext<`${M}${R}`, Res & Record<L, M>> : Omit<Res, never>
type DigsPrev = {[K in keyof DigsNext as DigsNext[K]]: K}

type AddOne<A extends string, Res extends string = ""> = 
A extends `${infer L}${infer R}` 
  ? L extends keyof DigsNext ? `${Res}${DigsNext[L]}${R}` : AddOne<R, `${Res}0`>
  : `${Res}1`
type SubOne<A extends string, Res extends string = ""> = 
A extends `${infer L}${infer R}`
  ? L extends keyof DigsPrev ? `${Res}${DigsPrev[L]}${R}` : SubOne<R, `${Res}9`>
  : never

type MinusOne<T extends number> = 
ToNumber<
  `${T}` extends `-${infer N}` | '0'
  ? `-${Reverse<AddOne<Reverse<N>>>}` : Reverse<SubOne<Reverse<`${T}`>>>
>
```

## 2595 2852 PickByType / OmitByType

- as 再加层判断即可

```tsx
 //示例
type OnlyBoolean = PickByType<{
  name: string
  count: number
  isReadonly: boolean
  isEnable: boolean
}, boolean> // { isReadonly: boolean; isEnable: boolean; }

type OmitBoolean = OmitByType<{
  name: string
  count: number
  isReadonly: boolean
  isEnable: boolean
}, boolean> // { name: string; count: number }

//实现
type PickByType<T extends object, U> = {
  [p in keyof T as T[p] extends U ? p : never]: T[p]
}

type OmitByType<T extends object, U> = {
  [p in keyof T as T[p] extends U ? never : p]: T[p];
}
```

## 2688 2693 StartsWith / **EndsWith**

```tsx
 //示例
type a = StartsWith<'abc', 'ac'> // expected to be false
type b = StartsWith<'abc', 'ab'> // expected to be true
type c = StartsWith<'abc', 'abcd'> // expected to be false

type a = EndsWith<'abc', 'bc'> // expected to be true
type b = EndsWith<'abc', 'abc'> // expected to be true
type c = EndsWith<'abc', 'd'> // expected to be false

//实现
type StartsWith<T extends string, U extends string> = 
T extends `${U}${infer Rest}` ? true : false;
type EndsWith<T extends string, U extends string> = 
T extends `${infer Rest}${U}` ? true : false;
```

## 2757 2759 PartialByKeys / RequiredByKeys

- 使用泛型参数U进行保存以便复用
- `Omit<T, K> & Partial<Pick<T, K>` 虽然能达到效果，但交叉类型与普通interface不同，需要再map一下成interface

```tsx
 //示例
interface User {
  name: string
  age: number
  address: string
}
type UserPartialName = PartialByKeys<User, 'name'> // { name?:string; age:number; address:string }

interface User {
  name?: string
  age?: number
  address?: string
}
type UserRequiredName = RequiredByKeys<User, 'name'> // { name: string; age?: number; address?: string }

//实现
type PartialByKeys<
	T extends object, 
	K extends keyof T = keyof T, 
	U = Omit<T, K> & Partial<Pick<T, K>>
> = {
  [p in keyof U]: U[p]
}

type RequiredByKeys<
  T extends object, 
  K extends keyof T = keyof T, 
  U = Omit<T, K> & Required<Pick<T, K>>
> = {
  [p in keyof U]: U[p]
}
```

## 2793 Mutable

- -号的应用

```tsx
 //示例
interface Todo {
  readonly title: string
  readonly description: string
  readonly completed: boolean
}
type MutableTodo = Mutable<Todo> 
// { title: string; description: string; completed: boolean; }

//实现
type Mutable<T extends object> = {
  -readonly [p in keyof T]: T[p]
}
```

## 2946 ObjectEntries

```tsx
 //示例
interface Model {
  name: string;
  age: number;
  locations: string[] | null;
}
type modelEntries = ObjectEntries<Model> 
// ['name', string] | ['age', number] | ['locations', string[] | null];

//ObjectEntries<Partial<Model>> === ['name', string] | ['age', number] | ['locations', string[] | null]
//ObjectEntries<{ key?: undefined }> === ['key', undefined]

//实现
type ObjectEntries<T extends object, U = keyof T> = 
U extends keyof T 
  ? [U, T[U] extends undefined | infer R ? R : undefined] 
  : never;
```

## 3188 TupleToNestedObject

```tsx
 //示例
type a = TupleToNestedObject<['a'], string> // {a: string}
type b = TupleToNestedObject<['a', 'b'], number> // {a: {b: number}}
type c = TupleToNestedObject<[], boolean> 
// boolean. if the tuple is empty, just return the U type

//实现
type TupleToNestedObject<T extends any[], U> = 
T extends [infer L extends PropertyKey, ...infer R] ? {
  [p in L]: TupleToNestedObject<R, U>
} : U;
```

## 3192 Reverse

```tsx
 //示例
type a = Reverse<['a', 'b']> // ['b', 'a']
type b = Reverse<['a', 'b', 'c']> // ['c', 'b', 'a']

//实现
type Reverse<T extends any[]> = 
T extends [infer L, ...infer R] ? [...Reverse<R>, L] : []
```

## 3196 FlipArguments

```tsx
 //示例
type Flipped = FlipArguments<(arg0: string, arg1: number, arg2: boolean) => void>
// (arg0: boolean, arg1: number, arg2: string) => void

//实现
type Reverse<T extends any[]> = 
T extends [infer L, ...infer R] ? [...Reverse<R>, L] : []

type FlipArguments<T extends (...args: any[])=>any> = 
T extends (...args: infer U) => infer R 
	? (...args: Reverse<U>) => R : never;
```

## 3243 FlattenDepth

```tsx
 //示例
type Flipped = FlipArguments<(arg0: string, arg1: number, arg2: boolean) => void>
// (arg0: boolean, arg1: number, arg2: string) => void

//实现
type MinusOne<...> = ... //前面的MinusOne实现

type FlattenDepth<T extends any[], D extends number = 1> = 
D extends 0 
  ? T 
  : T extends [infer L, ...infer R] 
    ? L extends any[] 
      ? [...FlattenDepth<L, MinusOne<D>>, ...FlattenDepth<R, D>]
      : [L, ...FlattenDepth<R, D>]
    : T
```

## 3326 BEM

- 联合类型在模板字符串中会将不同的模板字符串组成联合类型
    
    ```tsx
    type a = `${'a' | 'b'}` // "a | "b"
    ```
    

```tsx
//示例
BEM<'btn', ['price'], []> // 'btn__price'
BEM<'btn', ['price'], ['warning', 'success']>
// 'btn__price--warning' | 'btn__price--success'
BEM<'btn', [], ['small', 'medium', 'large']>
// 'btn--small' | 'btn--medium' | 'btn--large'

//实现
type BEM<B extends string, E extends string[], M extends string[]> =
`${B}${
	E['length'] extends 0 ? '' : `__${E[number]}`
}${
	M['length'] extends 0 ? '' : `--${M[number]}`
}`
```

## 3376 InorderTraversal

- `[T] extends [TreeNode]` 防止分布式条件类型，否则虽然可以工作，但TS报错：Type instantiation is excessively deep and possibly infinite.

```tsx
 //示例
const tree1 = {
  val: 1,
  left: null,
  right: {
    val: 2,
    left: {
      val: 3,
      left: null,
      right: null,
    },
    right: null,
  },
} as const

type A = InorderTraversal<typeof tree1> // [1, 3, 2]

//实现
type InorderTraversal<T extends TreeNode | null> = 
[T] extends [TreeNode] 
  ? [...InorderTraversal<T['left']>, T['val'], ...InorderTraversal<T['right']>] 
  : []
```

## 4179 Flip

- 使用record映射
- any作为subType可以赋值给普通类型，如果是`T extends object` 或`T extends unknown` 会报错：Type 'T[p]' is not assignable to type 'string | number | bigint | boolean | null | undefined'.

```tsx
 //示例
Flip<{ a: "x", b: "y", c: "z" }>; // {x: 'a', y: 'b', z: 'c'}
Flip<{ a: 1, b: 2, c: 3 }>; // {1: 'a', 2: 'b', 3: 'c'}
Flip<{ a: false, b: true }>; // {false: 'a', true: 'b'}

//实现
type Flip<T extends Record<PropertyKey, any>> = {
  [p in keyof T as `${T[p]}`]: p;
}
```

## 4182 Fibonacci

- 利用元组长度进行计数，只适用于数据量小的情况

```tsx
 //示例
type Result1 = Fibonacci<3> // 2
type Result2 = Fibonacci<8> // 21

//实现
type Fibonacci<
  T extends number,
  A extends any[] = [1],
  B extends any[] = [],
  Count extends any[] = [1]
> = T extends Count["length"]
  ? [...A, ...B]["length"]
  : Fibonacci<T, B, [...A, ...B], [1, ...Count]>;
```

## 4260 AllCombinations

```tsx
 //示例
type AllCombinations_ABC = AllCombinations<'ABC'>;
// '' | 'A' | 'B' | 'C' | 'AB' | 'AC' | 'BA' | 'BC' | 'CA' | 'CB' | 'ABC' | 'ACB' | 'BAC' | 'BCA' | 'CAB' | 'CBA'

//实现
type AllCombinations<S, T extends string = "", U extends string = T> =
S extends `${infer L}${infer R extends string}`
  ? AllCombinations<R, T | L>
  : T extends "" ? "" : `${T}${AllCombinations<"", U extends T ? "" : U>}`;
```

## 4425 Greater Than

```tsx
 //示例
GreaterThan<2, 1> //should be true
GreaterThan<1, 1> //should be false
GreaterThan<10, 100> //should be false
GreaterThan<1111111111111111111, 1111111111111111> //should be true

//实现
type StringToArray<T extends string> = T extends `${infer F}${infer R}` ? [F, ...StringToArray<R>] : []
type NumberLength<T extends number> = StringToArray<`${T}`>['length']

type DigitArray = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
type CompareDigit<T extends string, U extends string, Index extends any[] = []> = 
T extends DigitArray[Index['length']] 
  ? false 
  : U extends DigitArray[Index['length']] 
    ? true 
    : CompareDigit<T, U, [...Index, any]>
type CompareDigitArray<T extends string[], U extends string[]> = 
T extends [infer TF extends string, ...infer TR extends string[]] 
  ? U extends [infer UF extends string, ...infer UR extends string[]] 
    ? CompareDigit<TF, UF> extends true 
      ? true : CompareDigitArray<TR, UR> 
    : false
  : false;

type GreaterThan<T extends number, U extends number> = 
NumberLength<T> extends NumberLength<U> 
  ? CompareDigitArray<StringToArray<`${T}`>, StringToArray<`${U}`>> 
  :  GreaterThan<NumberLength<T>, NumberLength<U>>
```

## 4471 Zip

```tsx
 //示例
type exp = Zip<[1, 2], [true, false]> // [[1, true], [2, false]]
type exp1 = Zip<[[1, 2]], [3]> // [[[1, 2], 3]]
type exp2 = Zip<[1, 2, 3], ['1', '2']> // [[1, '1'], [2, '2']]
//实现
type Zip<T extends any[], U extends any[]> = 
T extends [infer TL, ...infer TR]
  ? U extends [infer UL, ...infer UR] ? [[TL, UL], ...Zip<TR, UR>] : []
  : []
```

## 4484 IsTuple

```tsx
 //示例
type case1 = IsTuple<[number]> // true
type case2 = IsTuple<readonly [number]> // true
type case3 = IsTuple<number[]> // false

//实现
type IsTuple<T extends readonly any[] | { length: number }> = 
[T] extends [never]
  ? false
  : number extends T['length']
    ? false
    : T extends readonly any [] ? true : false;
```

## 4499 Chunk

```tsx
 //示例
type exp1 = Chunk<[1, 2, 3], 2> // [[1, 2], [3]]
type exp2 = Chunk<[1, 2, 3], 4> // [[1, 2, 3]]
type exp3 = Chunk<[1, 2, 3], 1> // [[1], [2], [3]]

//实现
type Chunk<T extends unknown[], N extends number, R extends unknown[] = []> =
T extends [infer First, ...infer Rest]
  ? R['length'] extends N
    ? [R, ...Chunk<Rest, N, [First]>]
    : [...Chunk<Rest, N, [...R, First]>]
  : R['length'] extends 0 ? [] : [R]
```

## 4518 Fill

```tsx
 //示例
Fill<[1, 2, 3], true, 0, 1> // [true, 2, 3]
FIll<[1, 2, 3], true, 10, 20> // [1, 2, 3]

//实现
type Fill<
  T extends unknown[],
  N,
  Start extends number = 0,
  End extends number = T['length'],
  I extends number[] = [],
  AfterStart extends boolean = false
> = 
T extends [infer F,...infer R] 
  ? I['length'] extends End 
    ? T
    : I['length'] extends Start
      ? [N,...Fill<R, N, Start, End, [...I, 1], true>]
      : [
          AfterStart extends true ? N : F,
          ...Fill<R, N, Start, End, [...I, 1], AfterStart>
        ]
  : T
```

## 5117 Without

```tsx
 //示例
type Res = Without<[1, 2], 1>; // [2]
type Res1 = Without<[1, 2, 4, 1, 5], [1, 2]>; // [4, 5]
type Res2 = Without<[2, 3, 2, 3, 2, 3, 2, 3], [2, 3]>; // []

//实现
type Without<T extends any[], U> = 
T extends [infer L, ...infer R]
  ? L extends (U extends any[] ? U[number] : U)
    ? Without<R, U> : [L, ...Without<R, U>]
  : T
```

## 5140 Trunc

```tsx
 //示例
type A = Trunc<12.34> // 12

//实现
type Trunc<T extends number | string> = 
`${T}` extends `${infer L}.${infer R}`
  ? L extends '' ? '0' : L
  : `${T}`
```

## 5153 5317 IndexOf / **LastIndexOf**

- 使用Equal是为了保证搜索为any等类型时可以正确匹配
- LastIndexOf由于索引还是从左到右递增，所以前面数组长度即为返回的索引

```tsx
 //示例
type Res = IndexOf<[1, 2, 3], 2>; // expected to be 1
type Res1 = IndexOf<[2,6, 3,8,4,1,7, 3,9], 3>; // expected to be 2
type Res2 = IndexOf<[0, 0, 0], 2>; // expected to be -1

//实现
type Equal<X, Y> = ...

type IndexOf<T extends any[], U, K extends any[] = []> = 
T extends [infer L, ...infer R]
  ? Equal<L, U> extends true ? K['length'] : IndexOf<R, U, [...K, 1]>
  : -1;

type LastIndexOf<T extends any[], U> = 
T extends [...infer L, infer R]
  ? Equal<R, U> extends true ? L['length'] : LastIndexOf<L, U>
  : -1;
```

## 5310 Join

```tsx
 //示例
type Res = Join<["a", "p", "p", "l", "e"], "-">; // 'a-p-p-l-e'
type Res1 = Join<["Hello", "World"], " ">; // 'Hello World'
type Res2 = Join<["2", "2", "2"], 1>; // '21212'
type Res3 = Join<["o"], "u">; // 'o'

//实现
type Join<T extends any[], U extends string | number> = 
T extends [infer L extends string | number, ...infer R]
  ? R extends [] ? `${L}` : `${L}${U}${Join<R, U>}`
  : ''
```

## 5360 Unique

```tsx
 //示例
type Res = Unique<[1, 1, 2, 2, 3, 3]>; // [1, 2, 3]
type Res1 = Unique<[1, 2, 3, 4, 4, 5, 6, 7]>; // [1, 2, 3, 4, 5, 6, 7]
type Res2 = Unique<[1, "a", 2, "b", 2, "a"]>; // [1, "a", 2, "b"]
type Res3 = Unique<[string, number, 1, "a", 1, string, 2, "b", 2, number]>; 
// [string, number, 1, "a", 2, "b"]
type Res4 = Unique<[unknown, unknown, any, any, never, never]>; 
// [unknown, any, never]

//实现
type Includes = ...

//从前往后
type Unique<T extends any[], U extends any[] = []> = 
T extends [infer L, ...infer R]
  ? Includes<U, L> extends true ? Unique<R, U> : Unique<R, [...U, L]>
  : U
//从后往前，节省一个泛型参数
type Unique<T extends any[]> = 
T extends [...infer L, infer R]
  ? Includes<L, R> extends true ? Unique<L> : [...Unique<L>, R]
  : T
```

## 5821 MapTypes

```tsx
 //示例
type StringToNumber = { mapFrom: string; mapTo: number;}
type StringToDate = { mapFrom: string; mapTo: Date;}
MapTypes<{iWillBeNumberOrDate: string}, StringToDate | StringToNumber> 
// { iWillBeNumberOrDate: number | Date; }

//实现
type MapTypes<T extends object, R extends { mapFrom: any; mapTo: any;}> = {
  [p in keyof T]: 
    T[p] extends R['mapFrom'] 
      ? R extends { mapFrom: T[p] } ? R['mapTo'] : never
      : T[p]
}
```

## 7544 ConstructTuple

```tsx
 //示例
type result = ConstructTuple<2> // [unknown, unkonwn]

//实现
type ConstructTuple<L extends number, T extends unknown[] = []> = 
T['length'] extends L ? T : ConstructTuple<L, [...T, unknown]>
```

## 8640 NumberRange

- 方法一直接逐个去除会因为递归深度过大报错，因此采用分布式条件类型
- 方法二直接从低位开始构造联合

```tsx
 //示例
type result = NumberRange<2 , 9> //  | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9

//实现

//true表示结果包含N，false结果不含N
type AscendArr<N extends number, K extends boolean = true, T extends number[] = []> = 
T['length'] extends N 
  ? K extends true ? [...T, T['length']] : T 
  : AscendArr<N, K, [...T, T['length']]>;

type NumberRange<
  L extends number, 
  H extends number, 
  T extends number[] = AscendArr<L, false>, 
  U extends number[] = AscendArr<H>
> = Exclude<U[number], T[number]>

//方法二
type ConstructTuple<L extends number, T extends any[] = []> = 
T['length'] extends L ? T : ConstructTuple<L, [...T, 1]>

type NumberRange<
  L extends number, 
  H extends number, 
  T = never,
  U extends any[] = ConstructTuple<L>
> = 
U['length'] extends H 
	? T | U['length'] 
	: NumberRange<L, H, T | U['length'], [...U, 1]>
```

## 8767 Combination

```tsx
 //示例
type Keys = Combination<['foo', 'bar']>
// `"foo" | "bar" | "foo bar" | "bar foo" 

//实现
type Combination<T extends string[], U = T[number], V = U> = 
U extends string ? U | `${U} ${Combination<T, Exclude<V, U>>}` : never
```

## 8987 Subsequence

```tsx
 //示例
type A = Subsequence<[1, 2]> // [] | [1] | [2] | [1, 2] 

//实现
type Subsequence<T extends any[], U extends any[] = []> = 
T extends [infer L, ...infer R] 
	? Subsequence<R, U> | Subsequence<R, [...U, L]> 
	: U

type Subsequence<T extends any[]> = 
T extends [infer L, ...infer R] 
	? [L, ...Subsequence<R>] | Subsequence<R> 
	: T
```

## 9142 CheckRepeatedChars

```tsx
 //示例
type CheckRepeatedChars<'abc'>   // false
type CheckRepeatedChars<'aba'>   // true

//实现
type CheckRepeatedChars<T extends string, U = never> = 
T extends `${infer L}${infer R}`
  ? L extends U ? true : CheckRepeatedChars<R, U | L>
  : false

type CheckRepeatedChars<T extends string> = 
T extends `${infer L}${infer R}` 
  ? R extends `${any}${L}${any}` ? true : CheckRepeatedChars<R>
  : false;
```

## 9286 FirstUniqueCharIndex

```tsx
 //示例
type res1 = FirstUniqueCharIndex<'loveleetcode'> // 2
type res2 = FirstUniqueCharIndex<'aabb'>// -1

//实现
type FirstUniqueCharIndex<T extends string, U extends string[] = []> = 
T extends `${infer L}${infer R}`
  ? R extends `${string}${L}${string}`
    ? FirstUniqueCharIndex<R, [...U, L]> 
    : L extends U[number] ? FirstUniqueCharIndex<R, [...U, L]> : U['length']
  : -1;
```

## 9616 ParseUrlParams

```tsx
// 示例
ParseUrlParams<':id'> // 'id'
ParseUrlParams<'posts/:id/:user/like'> // 'id' | 'user'

// 实现
type ParseUrlParams<T extends string, Res = never> = 
`${T}/` extends `${string}:${infer L}/${infer R}`
  ? ParseUrlParams<R, L | Res> : Res
```

## 9896 GetMiddleElement

```tsx
 //示例
type simple1 = GetMiddleElement<[1, 2, 3, 4, 5]>, // 返回 [3]
type simple2 = GetMiddleElement<[1, 2, 3, 4, 5, 6]> // 返回 [3, 4]

//实现
type GetMiddleElement<T extends any[]> = 
T extends [infer L, ...infer M, infer R]
  ? M extends [] ? T : GetMiddleElement<M>
  : T
```

## 9898 FindEles

```tsx
 //示例
type res = FindEles<[1, 2, 2, 3, 3, 4, 5, 6, 6, 6]> //[1, 4, 5]

//实现
type FindEles<T extends any[], U extends any[] = []> = 
T extends [infer L, ...infer R]
  ? L extends U[number] | R[number] ? FindEles<R, [...U, L]> : [L, ...FindEles<R, U>]
  : T
```

## 9989 CountElementNumberToObject

```tsx
// 示例
CountElementNumberToObject<[1,2,3,4,5,[1,2,3]]> 
/**
{
  1: 2,
  2: 2,
  3: 2,
  4: 1,
}
 */

// 实现
type CountToObjArr<T, Res extends Record<PropertyKey, 1[]> = {}> = 
T extends [infer L extends (PropertyKey | unknown[]), ...infer R]
  ? CountToObjArr<R, L extends PropertyKey
      ? L extends keyof Res 
        ? Omit<Res, L> & Record<L, [...Res[L], 1]> 
        : Res & Record<L, [1]>
      : CountToObjArr<L, Res>
    >
  : Res

type CountElementNumberToObject<
  T, Res extends Record<PropertyKey, 1[]> = CountToObjArr<T>
> = [Res] extends [never] ? {} : { [p in keyof Res]: Res[p]['length'] }
```

## 10969 Integer

```tsx
 //示例
type a = Integer<1.1> //never
type b = Integer<1.0000000000> //1

//实现
type Integer<T extends number> = 
`${T}` extends `${any}.${any}`
  ? never
  : number extends T ? never : T

type Integer<T extends number> = `${T}` extends `${bigint}` ? T : never
```

## 16259 ToPrimitive

- boolean需要特判，否则在分布式条件类型中解析为`true | false`

```tsx
 //示例
type PersonInfo = {
  name: 'Tom',
  age: 30,
  married: false,
  addr: {
    home: '123456',
    phone: '13111111111'
  }
}

// 要求结果如下：
type PersonInfo = {
  name: string,
  age: number,
  married: boolean,
  addr: {
    home: string,
    phone: string
  }
}

//实现
type ToPrimitive<
T extends object, 
P = string | number | null | undefined | symbol | bigint
> = {
  [K in keyof T]: 
    T[K] extends object 
      ? T[K] extends Function 
        ? Function 
        : T[K] extends readonly any[] 
          ? ToPrimitive<T[K]>
          : { [X in keyof ToPrimitive<T[K]>]: ToPrimitive<T[K]>[X] }
      : T[K] extends boolean 
        ? boolean 
        : P extends any ? T[K] extends P ? P : never : never
}
```

## 18142 All

```tsx
 //示例
type Test1 = [1, 1, 1]
type Test2 = [1, 1, 2]

type Todo = All<Test1, 1> // should be same as true
type Todo2 = All<Test2, 1> // should be same as false

//实现
type All<T extends any[], U> = 
T extends [infer L, ...infer R]
  ? Equal<L, U> extends true ? All<R, U> : false
  : true
```

## 18220 Filter

```tsx
//示例
type res = Filter<[0, 1, 2] // 0 | 1

//实现
type Filter<T extends any[], P> = 
T extends [infer L, ...infer R] 
  ? L extends P ? [L, ...Filter<R, P>] : Filter<R, P>
  : T
```

## 21104 FindAll

```tsx
// 示例
FindAll<'AAAA', 'AA'> // [0, 1, 2]
FindAll<'abc def', ''> // []

// 实现
type FindAll<
  T extends string, P extends string,
  Cnt extends 0[] = [], R extends number[] = [],
> = 
P extends '' 
  ? [] 
  : T extends `${string}${infer L}` ?
    FindAll<L, P, [...Cnt, 0], 
      T extends `${P}${string}` ? [...R, Cnt['length']] : R
    >
  : R
```

## 21106 Combs

```tsx
 //示例
//任取两个按先后顺序组合
type a = Combs<['cmd', 'ctrl', 'opt', 'fn']>
//'cmd ctrl' | 'cmd opt' | 'cmd fn' | 'ctrl opt' | 'ctrl fn' | 'opt fn'

//实现
type Combs<T extends string[], P = never> = 
T extends [infer L extends string, ...infer R extends string[]]
  ? Combs<R, P | `${L} ${R[number]}`> : P
```

## 25170 ReplaceFirst

```tsx
 //示例
type a = ReplaceFirst<[1, 'two', 3], string, 2> // [1, 2, 3]

//实现
type ReplaceFirst<T extends readonly unknown[], S, P> = 
T extends readonly [infer L, ...infer R]
  ? L extends S ? [P, ...R] : [L, ...ReplaceFirst<R, S, P>]
  : T;
```

## 25270 Transpose

- 非递归法`X extends keyof M[Y]` 让`M[Y}[X]`顺利执行
- 若不需改动数组长度的情况下可以直接用map产生数组

```tsx
 //示例
type Matrix = Transpose <[[1]]>; // [[1]]
type Matrix1 = Transpose <[[1, 2], [3, 4]]>; // [[1, 3], [2, 4]]
type Matrix2 = Transpose <[[1, 2, 3], [4, 5, 6]]>; 
// [[1, 4], [2, 5], [3, 6]]

//实现

//递归法：
type Transpose<
M extends number[][], 
R extends number[][] = [], 
X extends any[] = [], 
Y extends any[] = []> = 
Y['length'] extends M[0]['length']
  ? R
  : R['length'] extends Y['length']
    ? Transpose<M, [...R, []], X, Y>
    : X['length'] extends M['length']
      ? Transpose<M, R, [], [...Y, 1]>
      : R extends [...infer A extends number[][], infer B extends number[]]
        ? Transpose<M, [...A, [...B, M[X['length']][Y['length']]]], [...X, 1], Y>
        : R

//非递归法：
type Transpose<M extends number[][],R = M['length'] extends 0?[]:M[0]> = {
  [X in keyof R]:{
    [Y in keyof M]:X extends keyof M[Y]? M[Y][X] : never
  }
}

//获取每行第idx个元素的值组成数组
type GetIdxValByRows<M extends number[][], Idx extends number> = 
M extends [infer L extends number[], ...infer R extends number[][]]
  ? [L[Idx], ...GetIdxValByRows<R, Idx>]
  : []

type Transpose<M extends number[][], R extends number[][] = []> = 
M extends []
  ? []
  : M[0]['length'] extends R['length']
    ? R
    : Transpose<M, [...R, GetIdxValByRows<M, R['length']>]>
```

## 26401 JSONSchema2TS

```tsx
 //示例
type Type14 = JSONSchema2TS<{
  type: 'object'
  properties: {
    req1: { type: 'string' }
    req2: {
      type: 'object'
      properties: {
        a: {
          type: 'number'
        }
      }
      required: ['a']
    }
    add1: { type: 'string' }
    add2: {
      type: 'array'
      items: {
        type: 'number'
      }
    }
  }
  required: ['req1', 'req2']
}>

type Expected14 = {
  req1: string
  req2: { a: number }
  add1?: string
  add2?: number[]
}

//实现
type RequiredByKeys<
  T extends object, 
  K extends keyof T = keyof T, 
  U = Omit<T, K> & Required<Pick<T, K>>
> = {
  [p in keyof U]: U[p]
}

type JSONSchema2TS<T> = 
T extends { type: 'object' }
  ? T extends { properties: infer P extends object }
    ? RequiredByKeys<
      { [K in keyof P]?: JSONSchema2TS<P[K]> }, 
      T extends { required: infer R extends any[] } ? R[number] : never
    >
    : Record<string, unknown>
  : T extends { type: 'array', items?: infer P }
    ? P extends object ? JSONSchema2TS<P>[] : P[]
    : T extends { enum: infer E extends any[] } ? E[number] 
      : T extends { type: 'string' } ? string
        : T extends { type: 'number' } ? number 
          : T extends { type: 'boolean' } ? boolean : T
```

## 27133 Square

- 要通过100的样例需要实现Extreme的Multiply

```tsx
 //示例
type a = Square<100> // 10000

//实现
type Multiply = ... 

type ToNumber<S extends string> = 
S extends `${infer N extends number}` ? N : never

type Absolute<N extends number> = 
`${N}` extends `-${infer R extends number}` ? R : N;

type Square<N extends number> = ToNumber<Multiply<Absolute<N>, Absolute<N>>>
```

## 27152 Triangular

```tsx
 //示例
Triangular<10>  // 55

//实现
type Triangular<N extends number, Cnt extends any[] = [], Res extends any[] = []> = 
N extends Cnt['length']
  ? Res['length'] : Triangular<N, [...Cnt, 1], [...Res, ...Cnt, 1]>
```

## 27862 CartesianProduct

```tsx
 //示例
CartesianProduct<1 | 2, 'a' | 'b'> 
// [1, 'a'] | [2, 'a'] | [1, 'b'] | [2, 'b']

//实现
type CartesianProduct<T, U> = 
T extends any
  ? U extends any ? [T, U] : never
  : never
```

## 27932 MergeAll

```tsx
 //示例
type Foo = { a: 1; b: 2 }
type Bar = { a: 2 }
type Baz = { c: 3 }
type Result = MergeAll<[Foo, Bar, Baz]> // { a: 1 | 2; b: 2; c: 3 }

//实现
type MergeAll<XS extends object[], Res = {}> = 
XS extends [infer L, ...infer R extends object[]]
  ? MergeAll<R, Omit<Res, keyof L> & { 
      [p in keyof L]: p extends keyof Res ? L[p] | Res[p] : L[p] 
    }>
  : Omit<Res, never>;
```

## 27958 CheckRepeatedTuple

```tsx
 //示例
type CheckRepeatedTuple<[1, 2, 3]>   // false
type CheckRepeatedTuple<[1, 2, 1]>   // true

//实现1
type CheckRepeatedTuple<T extends unknown[], Pre = never> = 
T extends [infer L, ...infer R]
  ? L extends Pre ? true : CheckRepeatedTuple<R, Pre | L>
  : false
//上述实现无法通过以下样例：
CheckRepeatedTuple<[number, 1]> //true 实际应为false
CheckRepeatedTuple<[never]> //never 实际应为false
CheckRepeatedTuple<[any]> //boolean 实际应为false

//实现2
type isInTuple<T extends unknown, U extends unknown[]> = 
U extends [infer L, ...infer R]
  ? Equal<T, L> extends true ? true : isInTuple<T, R>
  : false

type CheckRepeatedTuple<T extends unknown[], Pre extends unknown[] = []> = 
T extends [infer L, ...infer R]
  ? isInTuple<L, Pre> extends true ? true : CheckRepeatedTuple<R, [...Pre, L]>
  : false
```

## 28333 PublicType

```tsx
 //示例
PublicType<{ _h: number; i: unknown }> // { i: unknown }

//实现
type PublicType<T extends object> = {
  [p in keyof T as p extends `_${string}` ? never : p]: T[p]; 
}
```

## 29650 ExtractToObject

```tsx
 //示例
type Test = { id: '1', myProp: { foo: '2' }}
type Result = ExtractToObject<Test, 'myProp'> // { id: '1', foo: '2' }

//实现
type ExtractToObject<T extends Object, U extends keyof T> = 
Omit<T[U] & Omit<T, U>, never>
```

## 29785 DeepOmit

```tsx
 //示例
type obj = {
  person: {
    name: string;
    age: {
      value: number
    }
  }
}

type test1 = DeepOmit<obj, 'person'>    // {}
type test2 = DeepOmit<obj, 'person.name'> // { person: { age: { value: number } } }
type test3 = DeepOmit<obj, 'name'> // { person: { name: string; age: { value: number } } }
type test4 = DeepOmit<obj, 'person.age.value'> // { person: { name: string; age: {} } }

//实现
type DeepOmit<T, K extends string> =
K extends `${infer L}.${infer R}`
  ? { [p in keyof T]: p extends L ? DeepOmit<T[p], R> : T[p] }
  : Omit<T, K>;
```

## 30301 IsOdd

```tsx
 //示例
IsOdd<2023> // true
IsOdd<number> // false

//实现
type IsOdd<T extends number> = 
`${T}` extends `${number | ''}${1 | 3 | 5 | 7 | 9}` ? true : false;
```

## 30430 Hanoi

```tsx
Hanoi<0> // []
Hanoi<1> // [['A', 'B']]
Hanoi<2> // [['A', 'C'], ['A', 'B'], ['C', 'B']]
Hanoi<3> 
// [['A', 'B'], ['A', 'C'], ['B', 'C'], ['A', 'B'], ['C', 'A'], ['C', 'B'], ['A', 'B']]

//实现
type Hanoi<
  N extends number, From = 'A', To = 'B',
  Intermediate = 'C', Cnt extends 1[] = []
> = Cnt['length'] extends N
  ? []
  : [
      ...Hanoi<N, From, Intermediate, To, [...Cnt, 1]>, 
      [From, To], 
      ...Hanoi<N, Intermediate, To, From, [...Cnt, 1]>
    ]
```

## 30958 Pascal

```tsx
Pascal<3>
/*
[
  [1],
  [1, 1],
  [1, 2, 1],
]
*/

//实现
type Sum<
A extends number, B extends number, 
CntA extends any[] = [], CntB extends any[] = []
> = 
CntA['length'] extends A 
  ? CntB['length'] extends B 
    ? [...CntA, ...CntB]['length'] 
    : Sum<A, B, CntA, [...CntB, 1]>
  : Sum<A, B, [...CntA, 1], CntB>;

type GetArr<PreArr extends number[], Res extends number[] = [1]> = 
PreArr extends [infer L extends number, ...infer P extends number[]]
? P extends [infer M extends number, ...infer R extends number[]]
  ? GetArr<[M, ...R],[...Res, Sum<L, M>]>
  : GetArr<P, [...Res, 1]>
: Res

type Pascal<N extends number, Cnt extends 1[] = [], Res extends number[][] = [[1]]> = 
Res['length'] extends N
? Res
: Pascal<N, [...Cnt, 1], [...Res, GetArr<Res[Cnt['length']]>]>
```

## 30970 IsFixedStringLiteralType

```tsx
IsFixedStringLiteralType<`ABC${boolean}`> // false
IsFixedStringLiteralType<`ABC${true}`> // true
IsFixedStringLiteralType<`ABC${false}`> // true

//实现
type SingleCheck<S> = S extends ''
  ? true
  : S extends `${infer C}${infer T}`
    ? '0' | '1' extends C
      ? false : SingleCheck<T>
    : false;

type IsFixedStringLiteralType<S extends string, T = S> = [S] extends [never]
  ? false
  : S extends unknown
    ? [T] extends [S] 
      ? SingleCheck<S> : false
    : false;
```

## 34007 CompareArrayLength

```tsx
CompareArrayLength<[1, 2, 3, 4], [5, 6]> // 1
CompareArrayLength<[1, 2], [3, 4, 5, 6]> // -1
CompareArrayLength<[1, 2, 3], [4, 5, 6]> // 0

//实现
type CompareArrayLength<T extends any[], U extends any[]> = 
T extends [infer A, ...infer B]
? U extends [infer X, ...infer Y]
  ? CompareArrayLength<B, Y>
  : 1
: U extends [infer X, ...infer Y] ? -1 : 0
```

## 34857 DefinedPartial

```tsx
type A2 = Record<'a' | 'b' | 'c', string>
type E2 = { a: string } |
  { b: string } |
  { c: string } |
  { a: string, b: string } |
  { a: string, c: string } |
  { b: string, c: string } |
  { a: string, b: string, c: string }
type D2 = DefinedPartial<A2>
type C2 = Expect<Equal<D2, E2>>

// 实现
type DefinedPartial<T, K extends keyof T = keyof T> = K extends unknown
  ? T | DefinedPartial<Omit<T, K>>
  : never;
```

## 35045 LongestCommonPrefix

```tsx
LongestCommonPrefix<['flower', 'flow', 'flight']> // 'fl'
LongestCommonPrefix<['flower', 'flow', '']> // ''
LongestCommonPrefix<['flower']> // 'flower'
LongestCommonPrefix<[]> // ''

// 实现
type GetPreFix<A extends string, B extends string, Res extends string = ''> = 
A extends `${infer AL}${infer AR}`
  ? B extends `${AL}${infer BR}` ? GetPreFix<AR, BR, `${Res}${AL}`> : Res
  : Res;

type LongestCommonPrefix<T extends string[]> = 
T extends [infer TL extends string, infer TM extends string, ...infer TR extends string[]]
  ? LongestCommonPrefix<[GetPreFix<TL, TM>, ...TR]>
  : T['length'] extends 0 ? '' : T[0];
  
// 实现
type LongestCommonPrefix<T extends string[], P extends string = ''> = 
T extends [`${P}${infer Next}${any}`, ...any]
  ? T extends `${P}${Next}${any}`[]
    ? LongestCommonPrefix<T, `${P}${Next}`> : P
  : P
```

## 35191 Trace

```tsx
Trace<[[1, 2], [3, 4]] // 1 | 4
Trace<[[0, 1, 1], [2, 0, 2], [3, 3, 0]] // 0
Trace<[['a', 'b', ''], ['c', '', ''], ['d', 'e', 'f']] // 'a' | '' | 'f'

// 实现
type Trace<T extends any[][], Cnt extends 1[] = [], Res = never> = 
Cnt['length'] extends T['length']
 ? Res
 : Trace<T, [...Cnt, 1], T[Cnt['length']][Cnt['length']] | Res>
 
 // 实现
 type Trace<T extends any[][]> = {[P in keyof T]: T[P][P & keyof T[P]]}[number]
```

## 35252 IsAlphabet

```tsx
IsAlphabet<'A'> // true
IsAlphabet<'z'> // true
IsAlphabet<'9'> // false
IsAlphabet<'!'> // false
IsAlphabet<'😂'> // false
IsAlphabet<''> // false

// 实现
type IsAlphabet<S extends string> = Lowercase<S> extends Uppercase<S> ? false : true
```

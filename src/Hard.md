# Hard
## 6 SimpleVue

- Data、Computed、Methods泛型参数进行同类型的约束
- `ThisType<T & U>` 约束方法时方法的this为`T & U` ThisType可以与其他类型交叉使用
- 方法首个参数使用this可以约束方法内this的类型

```tsx
 //示例
SimpleVue({
  data() {
    // @ts-expect-error
    this.firstname
    // @ts-expect-error
    this.getRandom()
    // @ts-expect-error
    this.data()

    return {
      firstname: 'Type',
      lastname: 'Challenges',
      amount: 10,
    }
  },
  computed: {
    fullname() {
      return `${this.firstname} ${this.lastname}`
    },
  },
  methods: {
    getRandom() {
      return Math.random()
    },
    hi() {
      alert(this.amount)
      alert(this.fullname.toLowerCase())
      alert(this.getRandom())
    },
    test() {
      const fullname = this.fullname
      const cases: [Expect<Equal<typeof fullname, string>>] = [] as any
    },
  },
})

//实现
declare function SimpleVue<
  Data,
  Computed extends Record<PropertyKey, (...args: unknown[]) => unknown>,
  Methods
>(options: {
  data: (this: void) => Data
  computed: Computed & ThisType<Data>
  methods: Methods & ThisType<Data & Methods & { [K in keyof Computed]: ReturnType<Computed[K]> }>
}): unknown
```

## 213 VueBasicProps

- `ResultType` 实现构造函数到实例的转换

```tsx
 //示例
VueBasicProps({
  props: {
    propA: {},
    propB: { type: String },
    propC: { type: Boolean },
    propD: { type: ClassA },
    propE: { type: [String, Number] },
    propF: RegExp,
  },
  data(this) {
    type PropsType = Debug<typeof this>
    type cases = [
      Expect<IsAny<PropsType['propA']>>,
      Expect<Equal<PropsType['propB'], string>>,
      Expect<Equal<PropsType['propC'], boolean>>,
      Expect<Equal<PropsType['propD'], ClassA>>,
      Expect<Equal<PropsType['propE'], string | number>>,
      Expect<Equal<PropsType['propF'], RegExp>>,
    ]

    // @ts-expect-error
    this.firstname
    // @ts-expect-error
    this.getRandom()
    // @ts-expect-error
    this.data()

    return {
      firstname: 'Type',
      lastname: 'Challenges',
      amount: 10,
    }
  },
  computed: {
    fullname() {
      return `${this.firstname} ${this.lastname}`
    },
  },
  methods: {
    getRandom() {
      return Math.random()
    },
    hi() {
      alert(this.fullname.toLowerCase())
      alert(this.getRandom())
    },
    test() {
      const fullname = this.fullname
      const propE = this.propE
      type cases = [
        Expect<Equal<typeof fullname, string>>,
        Expect<Equal<typeof propE, string | number>>,
      ]
    },
  },
})

//实现
type ResultType<T> =
  T extends (...args: any[]) => infer R
  ? R
  : T extends new (...args: any[]) => infer S
    ? S
    : any;

type PropsType<Props> = {
    [K in keyof Props]: 
      Props[K] extends { type: infer V } 
        ? V extends any[] ? ResultType<V[number]> : ResultType<V> 
        : ResultType<Props[K]>
  }

declare function VueBasicProps<
  Props,
  Data,
  Computed extends Record<PropertyKey, (...args: unknown[]) => unknown>,
  Methods
>(options: {
  props: Props
  data: (this: PropsType<Props>) => Data
  computed: Computed & ThisType<Data>
  methods: Methods & ThisType<
    Data &  PropsType<Props> &  Methods & 
    { [K in keyof Computed]: ReturnType<Computed[K]> }
	>
}): unknown
```

## 1290 defineStore

```tsx
 //示例
const store = defineStore({
  id: '',
  state: () => ({
    num: 0,
    str: '',
  }),
  getters: {
    stringifiedNum() {
      // @ts-expect-error
      this.num += 1

      return this.num.toString()
    },
    parsedNum() {
      return parseInt(this.stringifiedNum)
    },
  },
  actions: {
    init() {
      this.reset()
      this.increment()
    },
    increment(step = 1) {
      this.num += step
    },
    reset() {
      this.num = 0

      // @ts-expect-error
      this.parsedNum = 0

      return true
    },
    setNum(value: number) {
      this.num = value
    },
  },
})

// @ts-expect-error
store.nopeStateProp
// @ts-expect-error
store.nopeGetter
// @ts-expect-error
store.stringifiedNum()
store.init()
// @ts-expect-error
store.init(0)
store.increment()
store.increment(2)
// @ts-expect-error
store.setNum()
// @ts-expect-error
store.setNum('3')
store.setNum(3)
const r = store.reset()

Expect<Equal<typeof store.num, number>>,
Expect<Equal<typeof store.str, string>>,
Expect<Equal<typeof store.stringifiedNum, string>>,
Expect<Equal<typeof store.parsedNum, number>>,
Expect<Equal<typeof r, true>>,

//实现
type GettersReturnType<T> = {
  readonly [p in keyof T]: ReturnType<T[p] extends ()=>any ? T[p] : never>
}

declare function defineStore<State, Getters, Actions>
(store: {
  id: string
  state: () => State
  getters: Getters & ThisType<GettersReturnType<Getters> & Readonly<State>>
  actions: Actions & ThisType<Actions & GettersReturnType<Getters> & State>
}): Actions & State & GettersReturnType<Getters>
```

## 17 Currying

```tsx
 //示例
const add = (a: number, b: number) => a + b
const three = add(1, 2)

const curriedAdd = Currying(add)
const five = curriedAdd(2)(3)

//实现
type Curried<T> = 
T extends (args: never) => any //传入的函数参数为空直接返回
  ? T
  : T extends (...args: infer A) => infer R
    ? A extends [infer X, ...infer Y] 
      ? (arg: X) => Curried<(...args: Y) => R> : R
    : never
declare function Currying<T>(fn: T): Curried<T>
```

## 55 UnionToIntersection

- 利用函数参数的性质：
    
    ```tsx
    type Bar<T> = 
    	T extends {
    		a: (x: infer U) => void;
    		b: (x: infer U) => void;
    	}
     ? U : never
    
    type T1 = Bar<{
    	a: (x: string) => void;
    	b: (x: string) => void;
    }> // string
    
    type T2 = Bar<{
    	a: (x: string) => void;
    	b: (x: number) => void;
    }> // string & numbe
    ```
    
    在本题中函数参数T同时推断了不同的U，最终合成了交叉类型
    

```tsx
 //示例
type I = UnionToIntersection<'foo' | 42 | true> // 'foo' & 42 & true
type b = UnionToIntersection<never> // unknown
//实现
type UnionToIntersection<U> = 
(U extends any ? (a: U) => any : never) extends (a: infer T) => any
  ? T : never
```

## 730 UnionToTuple

- 利用函数重载返回值特性
    
    ```tsx
    //函数重载：
    type Overload = {
    	(): number;
    	(): string;
    }
    //函数交叉
    type Intersect = (() => string) & (() => number)
    
    //判断时不看交叉顺序
    type res1 = Overload extends Intersect ? true : false; //true
    type res2 = Intersect extends Overload ? true : false; //true
    type res3 = Equal<Overload, Intersect> 
    //但Equal返回false(即使交叉顺序相同也返回false)
    ```
    
    当重载或交叉获取其ReturnType时取的是交叉类型最后一个子段
    
    ```tsx
    //交叉顺序不同返回类型不同
    type ret1 = GetIntersectionLast<Overload> //string
    type ret2 = GetIntersectionLast<Intersect> //number
    
    //GetIntersectionLast与ReturnType不同之处：
    //ReturnType限制传入的参数类型为(...args: any) => any，函数交叉类型不能符合约束
    //GetIntersectionLast不对参数类型限制
    ```
    
    UnionToIntersectionFn将Union转成函数交叉类型
    
    ```tsx
    type IntersectFn = UnionToIntersectionFn<'a' | 'b'>
    //(() => "a") & (() => "b")
    ```
    
    Union转IntersectFn再逐个提取出最后一个子段Last，用Exclude除Last后递归
    

```tsx
 //示例
UnionToTuple<1>           // 正确：[1]
UnionToTuple<'b' | 'a'> 
// 正确：['b','a']；错误：['a','b'] | ['b','a']
UnionToTuple<any | 1> // [any] (需要吸收子类型)

//实现
type UnionToIntersectionFn<U> = 
(U extends any ? (a: () => U) => any : never) extends (a: infer T) => any
  ? T : never

type GetIntersectionLast<T> = T extends () => infer R ? R : never

type UnionToTuple<T, Last = GetIntersectionLast<UnionToIntersectionFn<T>>> = 
[T] extends [never] ? [] : [...UnionToTuple<Exclude<T, Last>>, Last]
```

## 2949 ObjectFromEntries

- `Omit<T, never>`可以代替`{[p in keyof T]: T[p]}`
- 可以直接遍历T

```tsx
 //示例
interface Model {
  name: string;
  age: number;
  locations: string[] | null;
}

type ModelEntries = ['name', string] | ['age', number] | ['locations', string[] | null];

type result = ObjectFromEntries<ModelEntries> // expected to be Model

//实现
type UnionToIntersection = ...
type ObjectFromEntries<T> = 
Omit<
  UnionToIntersection<
    T extends [infer K extends PropertyKey, infer V] ? Record<K, V> : never
  >,
  never
>;

type ObjectFromEntries<T extends [PropertyKey, unknown]> = {
  [K in T as K[0]]: K[1]
}
```

## 57 59 GetRequired / GetOptional

- 可选的值的类型实际解释为联合一个undefined，但如果本身为undefined则不好区分
- 本身undefined -?后变为never，所以Required解决了上述问题

```tsx
 //示例
type a = GetRequired<{ foo: number; bar?: string }> //{ foo: number }
type b = GetRequired<{ foo: undefined; bar?: undefined }> //{ foo: undefined }

type c = GetOptional<{ foo: number; bar?: string }> //{ bar?: string }
type d = GetOptional<{ foo: undefined; bar?: undefined }> //{ bar?: undefined }

//实现
type GetRequired<T extends object> = {
  [p in keyof T as T[p] extends Required<T>[p] ? p : never]: T[p];
}
type GetOptional<T extends object> = {
  [p in keyof T as T[p] extends Required<T>[p] ? never : p]: T[p];
}

type GetOptional<T extends object> = Omit<T, keyof GetRequired<T>>
```

## 89 90 RequiredKeys / OptionalKeys

```tsx
 //示例
type Result = RequiredKeys<{ foo: number; bar?: string }>
  // expected to be “foo”
type Result = OptionalKeys<{ foo: number; bar?: string }>
  // expected to be “bar”

//实现
type GetRequired = ...
type RequiredKeys<T extends object> = keyof GetRequired<T>

type GetOptional = ...
type OptionalKeys<T extends object> = keyof GetOptional<T>
```

## 2857 IsRequiredKey

- 套一个[]防止分布式条件类型

```tsx
 //示例
type A = IsRequiredKey<{ a: number, b?: string },'a'> // true
type B = IsRequiredKey<{ a: number, b?: string },'b'> // false
type C = IsRequiredKey<{ a: number, b?: string },'b' | 'a'> // false

//实现
type GetRequired = ...
type IsRequiredKey<T extends object, K extends keyof T> = 
[K] extends [keyof GetRequired<T>] ? true :false

type GetOptional = ...
type OptionalKeys<T extends object> = keyof GetOptional<T>
```

## 112 CapitalizeWords

- Uppercase<Ch> extends Lowercase<Ch>判断是否为字母

```tsx
 //示例
CapitalizeWords<'foo bar.hello,world'> // 'Foo Bar.Hello,World'

//实现
type IsLetter<Ch extends string> = Uppercase<Ch> extends Lowercase<Ch> ? false : true

type CapitalizeWords<S extends string, P extends string = ""> = 
S extends `${infer L}${infer R}`
  ? IsLetter<L> extends true 
    ? CapitalizeWords<R, `${P}${L}`>
    : `${Capitalize<`${P}`>}${L}${CapitalizeWords<R>}`
  : Capitalize<P>
```

## 9775 CapitalizeNestObjectKeys

```tsx
 //示例
type foo = {
  foo: string
  bars: [{ foo: string }]
  1: 3
}

type Foo = {
  Foo: string
  Bars: [{
    Foo: string
  }]
  1: 3
}

//实现
type CapitalizeNestObjectKeys<T> = 
T extends any[]
  ? { [p in keyof T]: CapitalizeNestObjectKeys<T[p]> }
  : T extends object 
    ? { 
        [p in keyof T as p extends string ? Capitalize<p> : p]: 
          CapitalizeNestObjectKeys<T[p]> 
      }
    : T
```

## 19458 SnakeCase

- `L extends Lowercase<L>` 而不是`UpperCase`以区分大写字母与非字母

```tsx
 //示例
SnakeCase<'getElementById' | 'getElementByClassNames'>
// 'get_element_by_id' | 'get_element_by_class_names'

//实现
type SnakeCase<S extends string, Res extends string = ""> = 
S extends `${infer L}${infer R}`
  ? SnakeCase<R, `${Res}${L extends Lowercase<L> ? '' : '_'}${Lowercase<L>}`>
  : Res
```

## 114 CamelCase

```tsx
 //示例
type camelCase1 = CamelCase<"hello_world_with_types"> 
// 预期为 'helloWorldWithTypes'
type camelCase2 = CamelCase<"HELLO_WORLD_WITH_TYPES"> 
// 期望与前一个相同

//实现
type IsLetter<Ch extends string> = Uppercase<Ch> extends Lowercase<Ch> ? false : true

type CamelCase<S extends string> = 
S extends `${infer L}_${infer M}${infer R}`
  ? IsLetter<M> extends true
    ? `${Lowercase<L>}${Uppercase<M>}${CamelCase<R>}`
    :  `${Lowercase<L>}_${CamelCase<`${M}${R}`>}`
  : Lowercase<S>
```

## 1383 Camelize

```tsx
 //示例
Camelize<{
  some_prop: string,
  prop: { another_prop: string },
  array: [{ snake_case: string }]
}>

// {
//   someProp: string,
//   prop: { anotherProp: string },
//   array: [{ snakeCase: string }]
// }

//实现
type CamelCase = ...

type CamelizeArray<T extends object[]> = 
T extends [infer L extends object, ...infer R extends object[]] 
  ? [Camelize<L>, ...CamelizeArray<R>] : T;

type Camelize<T extends object> = {
  [p in keyof T as p extends string ? CamelCase<p> : p]: 
    T[p] extends object[] 
      ? CamelizeArray<T[p]> 
      : T[p] extends object ? Camelize<T[p]> : T[p]
```

## 147 ParsePrintFormat

```tsx
 //示例
type res1 = ParsePrintFormat<'The result is %%%d.'> // ['dec']
type res2 = CParsePrintFormat<'Hello %s: %d.'>// ['string', 'dec']

//实现
type ControlsMap = {
  c: 'char'
  s: 'string'
  d: 'dec'
  o: 'oct'
  h: 'hex'
  f: 'float'
  p: 'pointer'
}

type ParsePrintFormat<S extends string, T extends string[] = []> = 
S extends `${infer L}%${infer M}${infer R}`
  ? M extends keyof ControlsMap
    ? ParsePrintFormat<R, [...T , ControlsMap[M]]>
    : ParsePrintFormat<R, T>
  : T
```

## 545 Format

```tsx
 //示例
type FormatCase1 = Format<"%sabc"> // string => string
type FormatCase2 = Format<"%s%dabc"> // string => number => string
type FormatCase3 = Format<"sdabc"> // string
type FormatCase4 = Format<"sd%abc"> // string

//实现
type Format<T extends string, FormatMap = { d: number, s: string }> =
T extends `${infer L}%${infer M}${infer R}`
  ? M extends keyof FormatMap 
		? (M: FormatMap[M]) => Format<R> : Format<R>
  : string
```

## 223 IsAny

```tsx
 //示例
IsAny<any> // true
IsAny<unknown>// false

//实现
type IsAny<T> = Equal<T, any>

type IsAny<T> = 0 extends 1 & T ? true : false

type IsAny<T> = [{}, T] extends [T, null] ? true : false;

//any可以换为unknown
type IsAny<T> = 
((a: [any]) => [any]) extends (a: T) => [T] ? true : false;

type IsAny<T> =
[T] extends [never] ? false : keyof any extends keyof T ? true : false;
```

## 270 Get

```tsx
 //示例
type Data = {
  foo: {
    bar: {
      value: 'foobar',
      count: 6,
    },
    included: true,
  },
  hello: 'world'
}

type A = Get<Data, 'hello'> // 'world'
type B = Get<Data, 'foo.bar.count'> // 6
type C = Get<Data, 'foo.bar'> // { value: 'foobar', count: 6 }

//实现
type Get<T, K extends string> = 
K extends keyof T 
  ? T[K]
  : K extends `${infer L}.${infer R}`
    ? L extends keyof T ? Get<T[L], R> : never
    : never
```

## 300 ToNumber

- `S extends `${infer L extends number}`` 当S含前导0时只能推断出number，所以需要进行前导0的特判

```tsx
 //示例
ToNumber<'0'>// 0
ToNumber<'0@7'>// never

//实现
type ToNumber<S extends string> = S extends `${infer L extends number}` ? L : never

//忽略前导0的实现
type ToNumber<T extends string> = 
T extends `0${infer L}` 
  ? L extends '' ? 0 : ToNumber<`${L}`> 
  : T extends `${infer R extends number}` ? R : never

```

## 9155 ValidDate

```tsx
 //示例
ValidDate<'0102'> // true
ValidDate<'0131'> // true
ValidDate<'1231'> // true
ValidDate<'0229'> // false
ValidDate<'0100'> // false
ValidDate<'0132'> // false
ValidDate<'1301'> // false

//实现
type ToNumber<T extends string> = ...

type Day = {
  '01': 31;
  '02': 28;
  '03': 31;
  '04': 30;
  '05': 31;
  '06': 30;
  '07': 31;
  '08': 31;
  '09': 30;
  '10': 31;
  '11': 30;
  '12': 31;
};

type CheckDay<
  Day extends number,
  MaxDay extends number,
  Count extends 1[] = [1] //从1号开始
> =
Count['length'] extends Day
    ? true
    : Count['length'] extends MaxDay
      ? false
      : CheckDay<Day, MaxDay, [...Count, 1]>

type ValidDate<T extends string> =
  T extends `${infer M1}${infer M2}${infer D}`
  ? `${M1}${M2}` extends keyof Day
    ? CheckDay<ToNumber<D>, Day[`${M1}${M2}`]> : false
  : false;
```

## 399 FilterOut

- `[L] extends [F]` 用[]包裹防止never参与分布式条件类型

```tsx
 //示例
type Filtered = FilterOut<[1, 2, null, 3], null> // [1, 2, 3]

//实现
type FilterOut<T extends any[], F> = 
T extends [infer L, ...infer R]
  ? [L] extends [F] ? FilterOut<R, F> : [L, ...FilterOut<R, F>]
  : T;
```

## 472 Enum

- 注意交叉的顺序不同，最后结果顺序也不同

```tsx
 //示例
Enum<['macOS', 'Windows', 'Linux']>
  // -> { readonly MacOS: "macOS", readonly Windows: "Windows", readonly Linux: "Linux" }
Enum<['macOS', 'Windows', 'Linux'], true>
  // -> { readonly MacOS: 0, readonly Windows: 1, readonly Linux: 2 }

//实现
type Enum<T extends readonly string[], N extends boolean = false, U = {}> = 
T extends readonly [...infer L extends readonly string[], infer R extends string]
  ? Enum<L, N, { [p in Capitalize<R>]: N extends true ? L['length'] : R } & U>
  : Readonly<U>
```

## 553 DeepObjectToUniq

- 通过交叉一个symbol+不同的value类型达到unique效果

```tsx
 //示例
import { Equal } from "@type-challenges/utils"

type Foo = { foo: 2; bar: { 0: 1 }; baz: { 0: 1 } }

type UniqFoo = DeepObjectToUniq<Foo>

declare let foo: Foo
declare let uniqFoo: UniqFoo

uniqFoo = foo // ok
foo = uniqFoo // ok

type T0 = Equal<UniqFoo, Foo> // false
type T1 = UniqFoo["foo"] // 2
type T2 = Equal<UniqFoo["bar"], UniqFoo["baz"]> // false
type T3 = UniqFoo["bar"][0] // 1
type T4 = Equal<keyof Foo & string, keyof UniqFoo & string> // true

//实现
type DeepObjectToUniq<O extends object> = {
  [p in keyof O]: 
		O[p] extends object 
			? DeepObjectToUniq<O[p]> & { [k in symbol]: [O, p]} : O[p]
}  & { [k in symbol]: O}
```

## 651 31824 LengthOfString

```tsx
 //示例
type T0 = LengthOfString<"foo"> // 3
//需要支持上百个字符的长度（递归只能到45个）

//实现
type LengthOfString<S extends string, Res extends string[] = []> = 
S extends `${infer L}${infer R}`
  ? LengthOfString<R, [...Res, L]> : Res['length']
  
// 示例
// 需要支持1e6级别的字符数
type Deced = [10, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
type Signum = Deced[number]
type Reped<
  S extends string,
  C extends Signum,
  R extends string = '',
>
  = (C extends 0
    ? R
    : Reped<S, Deced[C], `${R}${S}`>
  )
type t0 = 'k'
type t1 = Reped<t0, 10>
type t2 = Reped<t1, 10>
type t3 = Reped<t2, 10>
type t4 = Reped<t3, 10>
type t5 = Reped<t4, 10>
type t6 = Reped<t5, 10>
type Gened<N extends string> = N extends `${''
  }${infer N6 extends Signum
  }${infer N5 extends Signum
  }${infer N4 extends Signum
  }${infer N3 extends Signum
  }${infer N2 extends Signum
  }${infer N1 extends Signum
  }${infer N0 extends Signum
  }` ? `${''
  }${Reped<t6, N6>
  }${Reped<t5, N5>
  }${Reped<t4, N4>
  }${Reped<t3, N3>
  }${Reped<t2, N2>
  }${Reped<t1, N1>
  }${Reped<t0, N0>
  }` : never
  
LengthOfString<Gened<'8414001'>> // 8414001

// 实现
type ToInt<S> = S extends `0${infer X}` ? ToInt<X> : S extends `${infer N extends number}` ? N : 0;

type Q1 = string;
type Q10 = `${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1 & {}}`;
type Q100 = `${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}`;
type Q1000 = `${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}`;
type Q10k = `${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}`;
type Q100k = `${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}`;

type Len<S, Q extends string, R extends 1[] = []> = S extends `${Q}${infer T}`
  ? Len<T, Q, [...R, 1]>
  : [R['length'], S];

type LengthOfString<S extends string> = Len<S, Q100k> extends [infer A extends number, infer S1]
  ? Len<S1, Q10k> extends [infer B extends number, infer S2]
    ? Len<S2, Q1000> extends [infer C extends number, infer S3]
      ? Len<S3, Q100> extends [infer D extends number, infer S4]
        ? Len<S4, Q10> extends [infer E extends number, infer S5]
          ? Len<S5, Q1> extends [infer F extends number, string]
            ? ToInt<`${A}${B}${C}${D}${E}${F}`>
            : 0
          : 0
        : 0
      : 0
    : 0
  : 0;
```

## 847 Join

- 函数泛型参数应该设置在对应的函数才有用
    
    ```tsx
    //如果T设置位置如下，则T将推断为string[]，而不是具体传入的元组
    declare function join<T extends string[], D extends string>(delimiter: D): 
    (...parts: T) => Join<T, D>
    ```
    

```tsx
 //示例
join('#')('a', 'b', 'c') // 'a#b#c'

//实现
type Join<T extends string[], D extends string> = 
T extends [infer L extends string, ...infer R extends string[]]
  ? R extends [] ? L : `${L}${D}${Join<R, D>}`
  : ''

declare function join<D extends string>(delimiter: D): 
<T extends string[]>(...parts: T) => Join<T, D>
```

## 956 DeepPick

```tsx
 //示例
type obj = {
  name: 'hoge',
  age: 20,
  friend: {
    name: 'fuga',
    age: 30,
    family: {
      name: 'baz',
      age: 1
    }
  }
}

type T1 = DeepPick<obj, 'name'>   // { name : 'hoge' }
type T2 = DeepPick<obj, 'name' | 'friend.name'>  
// { name : 'hoge' } & { friend: { name: 'fuga' }}
type T3 = DeepPick<obj, 'name' | 'friend.name' | 'friend.family.name'> 
/* { name : 'hoge' } &  { friend: { name: 'fuga' }} 
 & { friend: { family: { name: 'baz' }}} */

//实现
type UnionToIntersection<U> = 
(U extends any ? (a: U) => any : never) extends (a: infer T) => any
  ? T : never

type DeepPick<T, S extends string> = UnionToIntersection<
S extends `${infer L}.${infer R}`
  ? L extends keyof T ? { [p in L]: DeepPick<T[p], R> } : never
  : S extends keyof T ? { [p in S]: T[p] } : never
>

//使用现有工具简写：
//T extends Record<PropertyKey, any>让T[L]不报错
//若为T extends object则T[string]与object不兼容
type DeepPick<T extends Record<PropertyKey, any>, S extends string> = 
UnionToIntersection<
S extends `${infer L}.${infer R}`
  ? Record<L, DeepPick<T[L], R>>
  : S extends keyof T ? Pick<T, S> : never
>
```

## 2059 DropString

```tsx
 //示例
type Butterfly = DropString<'foobar!', 'fb'> // 'ooar!'

//实现
type DropString<S extends string, V extends string> = 
S extends `${infer L}${infer R}`
  ? `${(V extends `${string}${L}${string}` ? '' : L)}${DropString<R, V>}`
  : S
```

## 2822 Split

```tsx
 //示例
type result = Split<'Hi! How are you?', ' '> 
// ['Hi!', 'How', 'are', 'you?']

//实现
type Split<S extends string, SEP extends string> = 
string extends S
 ? string[]
 : S extends `${infer L}${SEP}${infer R}`
  ? [L,  ...Split<R, SEP>]
  : SEP extends '' ? [] : [S]
```

## 2828 ClassPublicKeys

- keyof 操作符返回public的key

```tsx
 //示例
class A {
  public str: string
  protected num: number
  private bool: boolean
  getNum() {
    return Math.random()
  }
}

type publicKyes = ClassPublicKeys<A> // 'str' | 'getNum'

//实现
type ClassPublicKeys<T> = keyof T
```

## 4037 IsPalindrome

- 字符串也能直接比较首尾，只需先infer出L再在后面判断extends即可
- ``${infer L}${string}${infer R}`` 特判一个字符的情况（直接不符合格式而返回true）

```tsx
 //示例
IsPalindrome<'abc'> // false
IsPalindrome<121> // true

//实现1
//缺点：IsPalindrome<string>返回true
type StrToTuple<S extends string, Res extends string[] = []> = 
S extends `${infer L}${infer R}` ? StrToTuple<R, [...Res, L]> : Res

type IsPalindrome<T extends string | number, U extends any[] = StrToTuple<`${T}`>> = 
U extends [infer L, ...infer M, infer R]
  ? Equal<L, R> extends true ? IsPalindrome<T, M> : false
  : true

//实现2
//缺点：IsPalindrome<string>返回true
type IsPalindrome<T extends string | number> = 
`${T}` extends `${infer L}${string}${infer R}`
  ? `${T}` extends `${L}${infer M}${L}` ? IsPalindrome<M> : false
  : true

//实现3
type ReverseStr<S extends string, T extends string = ""> = 
S extends `${infer L}${infer R}`
  ? ReverseStr<R, `${L}${T}`> 
  : T
type IsPalindrome<T extends string | number> = 
`${T}` extends ReverseStr<`${T}`> ? true : false
```

## 5181 MutableKeys

- extends不能区分，必须用Equal

```tsx
 //示例
type Keys = MutableKeys<{ readonly foo: string; bar: number }>; // "bar"

//实现
type MutableKeys<T> = keyof {
  [p in keyof T as 
    Equal<Readonly<Pick<T, p>>, Pick<T, p>> extends true ? never : p
  ]: T[p]
}
```

## 5423 Intersection

- unknown不能换成any, 任何包含any的交叉类型结果都是any

```tsx
 //示例
type Res = Intersection<[[1, 2], [2, 3], [2, 2]]>; // 2
type Res1 = Intersection<[[1, 2, 3], [2, 3, 4], [2, 2, 3]]>; // 2 | 3
type Res2 = Intersection<[[1, 2], [3, 4], [5, 6]]>; // never
type Res3 = Intersection<[[1, 2, 3], [2, 3, 4], 3]>; // 3
type Res4 = Intersection<[[1, 2, 3], 2 | 3 | 4, 2 | 3]>; // 2 | 3
type Res5 = Intersection<[[1, 2, 3], 2, 3]>; // never

//实现
type Intersection<T extends any[]> = 
T extends [infer L, ...infer R]
  ? (L extends any[] ? L[number] : L) & Intersection<R>  : unknown
```

## 6141 BinaryToDecimal

```tsx
 //示例
type Res1 = BinaryToDecimal<'10'>; // 2
type Res2 = BinaryToDecimal<'0011'>; // 3

//实现
type BinaryToDecimal<S extends string, Cnt extends any[] = []> = 
S extends `${infer L extends '0' | '1'}${infer R}`
  ? BinaryToDecimal<R, [...(L extends '1' ? [1] : []), ...Cnt, ...Cnt]> 
  : Cnt['length'];
```

## 7258 ObjectKeyPaths

```tsx
 //示例
type T1 = ObjectKeyPaths<{ name: string; age: number }>; 
// 'name' | 'age'
type T2 = ObjectKeyPaths<{
  refCount: number;
  person: { name: string; age: number };
}>; // 'refCount' | 'person' | 'person.name' | 'person.age'
type T3 = ObjectKeyPaths<{ books: [{ name: string; price: number }] }>;
// 'books' | 'books.0' | 'books[0]' | 'books.[0]' | 'books.0.name' |
// 'books.0.price' | 'books.length' | 'books.find'

//实现
type Keys<T, IsTop, K extends string | number> =
IsTop extends true
  ? K | (T extends unknown[] ? `[${K}]` : never)
  : `.${K}` | (T extends unknown[] ? `[${K}]` | `.[${K}]` : never)

type ObjectKeyPaths<T, IsTop = true, K extends keyof T = keyof T> =
K extends string | number
  ? `${Keys<T, IsTop, K>}${
    '' | (T[K] extends object ? ObjectKeyPaths<T[K], false> : '')
    }`
  : never
```

## 15260 Path

```tsx
 //示例
declare const example: {
  foo: {
    bar: {
      a: string;
    };
    baz: {
      b: number
      c: number
    }
  };
}
// Possible solutions:
// []
// ['foo']
// ['foo', 'bar']
// ['foo', 'bar', 'a']
// ['foo', 'baz']
// ['foo', 'baz', 'b']
// ['foo', 'baz', 'c']

//实现
type Path<
	T extends object, P extends PropertyKey[] = [], 
	K extends keyof T = keyof T
> = 
K extends any
  ? P | (T[K] extends object ? Path<T[K], [...P, K]> : [...P, K])
  : never
```

## 8804 TwoSum

```tsx
 //示例
TwoSum<[3, 3], 6>// true

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

type TwoSum<T extends number[], U extends number, P extends number = -1> =
T extends [infer L extends number, ...infer R extends number[]]
  ? P extends -1 
    ? TwoSum<R, U, L> 
    : Sum<P, L> extends U 
      ? true 
      : TwoSum<R, U, P> extends true ? true : TwoSum<R, U, L>
  : false;

//方法二
type NumtoTuple<T extends number, U extends number[] = []> = 
U['length'] extends T ? U : NumtoTuple<T, [...U, 1]>;

type SumUnion<T extends number, U extends number> = 
U extends any ? [...NumtoTuple<T>, ...NumtoTuple<U>]['length'] : never;

type TwoSum<T extends number[], U extends number> = 
T extends [infer L extends number, ...infer R extends number[]]
  ? U extends SumUnion<L, R[number]> ? true : TwoSum<R, U>
  : false;
```

## 9160 Assign

```tsx
 //示例
type Target = {
  a: 'a'
  d: {
    hi: 'hi'
  }
}
type Origin1 = {
  a: 'a1',
  b: 'b'
}
type Origin2 = {
  b: 'b2',
  c: 'c'
}

type Answer = {
   a: 'a1',
   b: 'b2',
   c: 'c'
   d: {
      hi: 'hi'
  }
}

//实现
type Assign<T extends Record<string, unknown>, U extends any[]> = 
U extends [infer L, ...infer R]
  ? Assign<(L extends object ? Omit<T, keyof L> & L : T), R>
  : Omit<T, never>

//方法三,原理类似
type Assign<T extends Record<string, unknown>, U extends any[]> = 
U extends [infer L, ...infer R]
  ? L extends object
    ? Assign<{ 
        [p in keyof T | keyof L]: 
          p extends keyof L ? L[p] : p extends keyof T ? T[p] :never 
      }, R>
    : Assign<T, R>
  : T
```

## 9384 Maximum / Minimum

- `[U] extends [C['length']]` 套[]防止分布式条件类型，只有U只剩一个子段时才可能成立，若U为never，[never] extends [K] 永远为true（K是任意类型）
- 当T为[]时T[number]为never
- U为联合类型时逐渐Exclude掉，由于C[’length’]递增，最后必剩下最大的

```tsx
 //示例
Maximum<[]> // never
Maximum<[0, 2, 1]> // 2
Maximum<[1, 20, 200, 150]> // 200

Minimum<[]> // never
Minimum<[0, 2, 1]> // 0
Minimum<[1, 20, 200, 150]> // 1

//实现
type Maximum<T extends number[], U = T[number], C extends 1[] = []> = 
[U] extends [C['length']] 
	? U : Maximum<T, Exclude<U, C['length']>, [...C, 1]>;

type Minimum<T extends number[], C extends 1[] = []> = 
T extends [] 
  ? never 
  : C['length'] extends T[number] ? C['length'] : Minimum<T, [...C, 1]>;
```

## 13580 UnionReplace

```tsx
 //示例
UnionReplace<number | string, [[string, null]]> // number | null

//实现
//单层map
type UnionReplace<T, U extends [any, any][]> = 
T extends T
  ? U extends [[infer K, infer V], ...infer R extends [any, any][]]
    ? T extends K ? V : UnionReplace<T, R>
    : T
  : never

UnionReplace<1, [[1, 2], [2, 3]]> // 2
UnionReplace<1, [[2, 3], [1, 2]]> // 2

//有序多层map
type UnionReplace<T, U extends [any, any][]> = 
U extends [[infer K, infer V], ...infer R extends [any, any][]]
  ? UnionReplace<T extends K ? V : T, R>
  : T

UnionReplace<1, [[1, 2], [2, 3]]> // 3
UnionReplace<1, [[2, 3], [1, 2]]> // 2

//无序多层map（有向图不能有环，否则结果不确定）
type UnionReplace<
T, 
U extends [any, any][], 
Pre extends [any, any][] = []
> = 
U extends [[infer K, infer V], ...infer R extends [any, any][]]
  ? UnionReplace<
      T extends K 
        ? UnionReplace<V, Pre> //map成功则往前找是否能进一步map
        : T, //map失败不递归再遍历
      R, 
      [...Pre, [K, V]]
    >
  : T

UnionReplace<1, [[4, 5], [2, 3], [1, 2], [3, 4], [5, 6]]> // 6
UnionReplace<0 | 1 | 2, [[3, 4], [1, 2], [5, 6], [4, 5], [2, 3]]>
// 0 | 6

//有环情况：
UnionReplace<1, [[1, 2], [2, 1]]> // 2
UnionReplace<1, [[2, 1], [1, 2]]> // 1

//要通过下面的测试则需要使用Equal
type UnionReplace<
T, 
U extends [any, any][], 
Pre extends [any, any][] = []
> = 
U extends [[infer K, infer V], ...infer R extends [any, any][]]
  ? UnionReplace<
      T extends K 
        ? Equal<T, K> extends true ? UnionReplace<V, Pre> : T
        : T, 
      R, 
      [...Pre, [K, V]]
    >
  : T

UnionReplace<(()=>number) | (()=>0), [[()=>number, never]]> // ()=>0
```

## 14080 FizzBuzz

```tsx
 //示例
FizzBuzz<15> /* ["1", "2", "Fizz", "4", "Buzz", "Fizz", "7", "8", 
 "Fizz", "Buzz", "11", "Fizz", "13", "14", "FizzBuzz"]  */

//实现
type IsDivisible<T extends 1[], U extends 1[]> = 
T extends [...U, ...infer R extends 1[]]
  ? R extends [] ? true : IsDivisible<R, U>
  : false

type FizzBuzz<N extends number, Res extends string[] = [], Cnt extends 1[] = [1]> = 
Res['length'] extends N
  ? Res
  : IsDivisible<Cnt, [1, 1, 1]> extends true
    ? IsDivisible<Cnt, [1, 1, 1, 1, 1]> extends true
      ? FizzBuzz<N, [...Res, 'FizzBuzz'], [...Cnt, 1]>
      : FizzBuzz<N, [...Res, 'Fizz'], [...Cnt, 1]>
    : IsDivisible<Cnt, [1, 1, 1, 1, 1]> extends true
      ? FizzBuzz<N, [...Res, 'Buzz'], [...Cnt, 1]>
      : FizzBuzz<N, [...Res, `${Cnt['length']}`], [...Cnt, 1]>
```

## 14188 Run-length encoding

- ``${infer L extends number}${infer R}`` L每次只能取一个字符，所以需要GetPreNum抽出前缀所有位数字

```tsx
 //示例
type a = RLE.Encode<'AAAAAAAAAAACDXX'> // '11ACD2X'
type b = RLE.Decode<'11ACD2X'> // 'AAAAAAAAAAACDXX'

//实现
namespace RLE {
  export type Encode<
		S extends string, Res extends string = "", Cnt extends 1[] = [1]
	> = S extends `${infer L}${infer M}${infer R}`
	    ? L extends M 
	      ? Encode<`${M}${R}`, Res, [...Cnt, 1]>
	      : Encode<`${M}${R}`, `${Res}${Cnt['length'] extends 1 
					? '' :  Cnt['length']}${L}`, [1]>
	    : `${Res}${Cnt['length'] extends 1 ? '' :  Cnt['length']}${S}`
  
  type Repeat<
    S extends string, N extends Number,
    Cnt extends 1[] = [], Res extends string = ""
	> = Cnt['length'] extends N 
			? Res : Repeat<S, N, [...Cnt, 1], `${Res}${S}`>

  type GetPreNum<S extends string, Res extends string = ""> = 
  S extends `${infer L extends number}${infer R}`
    ? GetPreNum<R, `${Res}${L}`> 
		: Res extends `${infer T extends number}` ? T : 1;

  export type Decode<
    S extends string, Res extends string = "",
    N extends number = GetPreNum<S>
  > = S extends `${N extends 1 ? '' : N}${infer L}${infer R}`
	    ? Decode<R, `${Res}${Repeat<L, N>}`> : `${Res}${S}`
}
```

## 25747 IsNegativeNumber

```tsx
 //示例
IsNegativeNumber<number>  // never
IsNegativeNumber<-1 | -2>  // never
IsNegativeNumber<-1> //  true
IsNegativeNumber<-1.9> //  true
IsNegativeNumber<-100_000_000> //  true
IsNegativeNumber<1>  // false

//实现
type IsUnion<T, U = T> = 
[T] extends [never] 
  ? false 
  : T extends T 
    ? [U] extends [T] ? false : true 
    : never;

type IsNegativeNumber<T extends number> = 
IsUnion<T> extends true 
  ? never
  : `${T}` extends `-${number}` 
    ? true 
    : number extends T ? never : false
```

## 28143 OptionalUndefined

```tsx
 //示例
OptionalUndefined<{ value: string | undefined, description: string }>
// { value?: string | undefined; description: string }

OptionalUndefined<{ 
	value: string | undefined, 
	description: string | undefined, 
	author: string | undefined 
}, 'description' | 'author'>
/* { 
	value: string | undefined; 
	description?: string | undefined; 
	author?: string | undefined; 
} */

//实现
type GetUndefinedKey<T> = 
keyof { [p in keyof T as undefined extends T[p] ? p : never]: T[p]; }

type OptionalUndefined<
  T, Props extends keyof T = keyof T, 
  U extends keyof T = GetUndefinedKey<T> & Props
> = Omit<Omit<T, U> & Partial<Pick<T, U>>, never>
```

## 30178 uniqueItems

使用函数对传入的参数进行校验

```tsx
 //示例
declare const readonlyEqual: <A>() => <T>(value: T) => Equal<Readonly<A>, Readonly<T>>
declare const expect: (value: true) => void

// Should work
expect(readonlyEqual<[1, 2, 3]>()(uniqueItems([1, 2, 3])))

// @ts-expect-error
uniqueItems([1, 2, 2, 3, 4, 4, 5, 6])

//实现
type UniqueItems<T, P extends unknown[] = []> =
T extends [infer F, ...infer R]
  ? UniqueItems<R, [...P, F extends P[number] ? never : F]>
  : P extends T ? P : P

function uniqueItems<const T>(items: UniqueItems<T>) {
  return items
}
```

## 30575 BitwiseXOR

```tsx
 //示例
BitwiseXOR<'0','1'> // '1'
BitwiseXOR<'1','1'> // '0'
BitwiseXOR<'10','1'>  // '11'

//实现
type Reverse<S extends string, Res extends string = ""> = 
S extends `${infer L}${infer R}`
  ? Reverse<R, `${L}${Res}`> : Res

type XOR<S1 extends string, S2 extends string, Res extends string = ""> = 
S1 extends `${infer L1}${infer R1}`
  ? S2 extends `${infer L2}${infer R2}`
    ? XOR<R1, R2, `${Res}${L1 extends L2 ? 0 : 1}`>
    : `${Res}${S1}`
  : `${Res}${S2}`

type BitwiseXOR<S1 extends string, S2 extends string> = 
Reverse<XOR<Reverse<S1>, Reverse<S2>>>
```

## 31797 SudokuSolved

```tsx
 //示例
SudokuSolved<[
  [[1, 2, 3], [5, 6, 7], [4, 8, 9]],
  [[4, 8, 9], [1, 2, 3], [5, 6, 7]],
  [[5, 6, 7], [4, 8, 9], [1, 2, 3]],
  [[3, 1, 2], [8, 5, 6], [9, 7, 4]],
  [[7, 9, 4], [3, 1, 2], [8, 5, 6]],
  [[8, 5, 6], [7, 9, 4], [3, 1, 2]],
  [[2, 3, 1], [6, 4, 5], [7, 9, 8]],
  [[9, 7, 8], [2, 3, 1], [6, 4, 5]],
  [[6, 4, 5], [9, 7, 8], [2, 3, 1]],
]> // true

//实现
type Digits = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9

type SudokuSolved<Grid extends number[][][]> =
  | CheckSubGrids<Grid>
  | CheckRows<Grid[number]>
  | CheckColumns<Grid[number]> extends true
  ? true
  : false

type CheckSubGrids<Grid extends number[][][]> = Grid extends [
  infer Grid1 extends number[][],
  infer Gird2 extends number[][],
  infer Grid3 extends number[][],
  ...infer Rest extends number[][][],
]
  ? CheckSubGrid<Grid1 | Gird2 | Grid3> extends true
    ? CheckSubGrids<Rest> : false
  : never

type CheckSubGrid<
  SubGrid extends number[][],
  C extends number = 0 | 1 | 2,
> = C extends C
  ? Digits extends SubGrid[C][number]
    ? true : false
  : never

type CheckRows<Rows extends number[][]> = 
Rows extends Rows
  ? Digits extends Rows[number][number]
    ? true : false
  : never

type CheckColumns<
  Rows extends number[][],
  I extends number = 0 | 1 | 2,
  J extends number = 0 | 1 | 2,
> = I extends I
  ? J extends J
    ? Digits extends Rows[I][J] ? true : false
    : never
  : never
```

## **32427 Unbox**

```tsx
 //示例
Unbox<string> // string
Unbox<() => number> // number
Unbox<boolean[]> // boolean
Unbox<Promise<boolean>> // boolean
Unbox<() => () => () => () => number> // number
Unbox<() => () => () => () => number, 3> // () => number

//实现
type Unbox<T, N = 0, Cnt extends any[] = [1]> = 
T extends (() => infer R) | Array<infer R> | Promise<infer R>
  ? Count['length'] extends N ? R : Unbox<R, N, [...Cnt, 1]>
  : T
```

## 32532 BinaryAdd

```tsx
 //示例
BinaryAdd<[1], [1]> // [1, 0]
BinaryAdd<[0], [1]> // [1]

//实现
type Bit = 1 | 0;
type Shifts = '011' | '101' | '110' | '111';
type Ones = '001' | '010' | '100' | '111';

type BinaryAdd<A extends Bit[], B extends Bit[], S extends Bit = 0, R extends Bit[] = []> = 
  [A, B] extends [[...infer AR extends Bit[], infer X extends Bit], [...infer BR extends Bit[], infer Y extends Bit]]
    ? BinaryAdd<AR, BR, `${S}${X}${Y}` extends Shifts ? 1 : 0, [`${S}${X}${Y}` extends Ones ? 1 : 0, ...R]>
    : S extends 1 ? [1, ...R] : R;
```

## 33763 UnionToObjectFromKey

```tsx
 //示例
type Foo = {
  foo: string
  common: boolean
}

type Bar = {
  bar: number
  common: boolean
}

type Other = {
  other: string
}

UnionToObjectFromKey<Foo | Bar | Other, 'common'> 
/*
{
  foo: string
  common: boolean
} | {
  bar: number
  common: boolean
}
*/

//实现
type UnionToObjectFromKey<Union, Key> = 
Union extends any
? Key extends keyof Union ? Union : never 
: never
```

## 34286 Take

```tsx
 //示例
BinaryAdd<[1], [1]> // [1, 0]
BinaryAdd<[0], [1]> // [1]

//实现
type Reverse<Arr extends any[], Res extends any[] = []> =
Arr extends [infer L, ...infer R] ? Reverse<R, [L, ...Res]> : Res

type PosTake<
  N extends number,
  Arr extends any[],
  Res extends any[] = []
> = Res['length'] extends N
? Res : Arr extends [infer L, ...infer R]
? PosTake<N, R, [...Res, L]>
: Res;

type Take<N extends number, Arr extends any[]> = 
`${N}` extends `-${infer T extends number}`
? Reverse<PosTake<T, Reverse<Arr>>> : PosTake<N, Arr>
```

## 35314 ValidSudoku

```tsx
 //示例
ValidSudoku<[
  [9, 5, 7, 8, 4, 6, 1, 3, 2],
  [2, 3, 4, 5, 9, 1, 6, 7, 8],
  [1, 8, 6, 7, 3, 2, 5, 4, 9],
  [8, 9, 1, 6, 2, 3, 4, 5, 7],
  [3, 4, 5, 9, 7, 8, 2, 6, 1],
  [6, 7, 2, 1, 5, 4, 8, 9, 3],
  [4, 6, 8, 3, 1, 9, 7, 2, 5],
  [5, 2, 3, 4, 8, 7, 9, 1, 6],
  [7, 1, 9, 2, 6, 5, 3, 8, 4],
]> // true

//实现
type RN = [number];
type R1 = [0, 1, 2, 3, 4, 5, 6, 7, 8];
type R3 = [0 | 1 | 2,  3 | 4 | 5,  6 | 7 | 8];
type Values = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;

type MapBool = {[N: number]: true; x: false};
type MapX = {[N: number]: 'x'} & Record<Values, never>;
type NotAllX<V extends number, D extends number = Values> = {[K in D]: MapX[K & V]}[D];

type Check<
  M extends number[][],
  RI extends number[],
  RJ extends number[]
> = {[I in keyof RI]: {[J in keyof RJ]: NotAllX<M[RI[I]][RJ[J]]>}[number]}[number];

type ValidSudoku<M extends number[][]> = 
	MapBool[Check<M, R1, RN> | Check<M, RN, R1> | Check<M, R3, R3>];
```

# Extreme

## GetReadonlyKeys

- 与Hard - MutableKeys对应

```tsx
 //示例
interface Todo {
  readonly title: string
  readonly description: string
  completed: boolean
}

type Keys = GetReadonlyKeys<Todo> // "title" | "description"

//实现
type GetReadonlyKeys<T> =  keyof {
  [p in keyof T as 
    Equal<Readonly<Pick<T, p>>, Pick<T, p>> extends true ? p : never
  ]: T[p]
}
```

## ParseQueryString

```tsx
 //示例
ParseQueryString<'k1&k1=v2&k1=v1&k2&k2'>
/*
{
    k1: [true, "v2", "v1"];
    k2: true;
}
*/
//实现
type AddEntries<T extends object, K extends string, V = true> = 
K extends keyof T
  ? T[K] extends any[]
    ? V extends T[K][number] 
      ? T
      : Omit<T, K> & Record<K, [...T[K], V]>
    : V extends T[K] ? T : Omit<T, K> & Record<K, [T[K], V]>
  : K extends '' ? T : T & Record<K, V>

//ParseValue用于进一步解析number、boolean，原挑战不要求
type ParseValue<V extends string> = 
V extends `0${infer L}` 
  ? L extends '' ? 0 : ParseValue<`${L}`> 
  : V extends `${infer N extends number}` 
    ? number extends N ? V : N  //数字过大超出number表现范围则返回字符串形式
    : V extends `${infer B extends boolean}` ? B : V

type EntriesToObject<S extends string, Res extends object = {}> = 
S extends `${infer K extends string}=${infer V extends string}`
  ? AddEntries<Res, K, ParseValue<V>> : AddEntries<Res, S>

type ParseQueryString<S extends string, Res extends object = {}> = 
S extends `${infer L}&${infer R}`
  ? ParseQueryString<R, EntriesToObject<L, Res>>
  : Omit<EntriesToObject<S, Res>, never>

//进一步解析的结果：
ParseQueryString<'k1=false&k3=3&k1=v2&k1=v1&k1'>
/*
{
    k1: [false, "v2", "v1", true];
    k3: 3;
}
*/
```

## Slice

- 与 Medium - Fill 对应，需处理负数情况

```tsx
 //示例
type Arr = [1, 2, 3, 4, 5]
Slice<Arr, 2, 4> // [3, 4]
Slice<Arr, -3, -1> // [3, 4]
Slice<Arr> // Arr
Slice<Arr, 2> // [3, 4, 5]

//实现
type GetSlice<
  T extends unknown[],
  Start extends number = 0,
  End extends number = T['length'],
  Res extends unknown[] = [],
  Cnt extends 1[] = [],
  AfterStart extends boolean = false
> = 
T extends [infer L,...infer R] 
  ? Cnt['length'] extends End 
    ? Res
    : Cnt['length'] extends Start
      ? GetSlice<R, Start, End, [...Res, L], [...Cnt, 1], true>
      : GetSlice<
	        R, Start, End, 
	        AfterStart extends true ? [...Res, L] : Res, 
					[...Cnt, 1], AfterStart
				>
  : Res

type Sub<
  A extends number, B extends number, 
  CntA extends 1[] = [], CntB extends 1[] = []
> = 
CntA['length'] extends A 
  ? CntB['length'] extends B
    ? CntA extends [...CntB, ...infer Res] ? Res['length'] : 0
    : Sub<A, B, CntA, [...CntB, 1]>
  : Sub<A, B, [...CntA, 1], CntB>

type FormatNegative<T extends number, Len extends number> = 
`${T}` extends `-${infer N extends number}` ? Sub<Len, N> : T

type Slice<
  T extends unknown[],
  Start extends number = 0,
  End extends number = T['length']
> = GetSlice<T, FormatNegative<Start, T['length']>, FormatNegative<End, T['length']>>
```

## InclusiveRange

- 与 Medium - Fill 对应，需特判L === H情况

```tsx
 //示例
InclusiveRange<0, 10> // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

//实现
type InclusiveRange<
  L extends number, H extends number,
  Res extends number[] = [], Cnt extends 1[] = [],
  AfterL extends boolean = false
> = 
Cnt['length'] extends H
  ? AfterL extends true 
    ? [...Res, Cnt['length']] 
    : L extends H ? [L] : Res
  : Cnt['length'] extends L
    ? InclusiveRange<L, H, [...Res, Cnt['length']], [...Cnt, 1], true>
    : InclusiveRange<L, H, AfterL extends true ? [...Res, Cnt['length']] : Res, [...Cnt, 1], AfterL>
```

## Comparator

```tsx
 //示例
Comparator<9007199254740992, 9007199254740992>
// Comparison.Equal
Comparator<-9007199254740991, -9007199254740992>
// Comparison.Greater

//实现
enum Comparison {
  Greater = 1,
  Equal = 0,
  Lower = -1,
}

//一位十进制数比较
type OppositeRes<T extends Comparison> = // 同为负数时比较结果取反
T extends Comparison.Equal 
  ? T
  : T extends Comparison.Lower ? Comparison.Greater : Comparison.Lower

type DigitCompare<A extends number, B extends number, Cnt extends 1[] = []> = 
Equal<A, B> extends true 
  ? Comparison.Equal
  : Cnt['length'] extends A
    ? Comparison.Lower
    : Cnt['length'] extends B ? Comparison.Greater : DigitCompare<A, B, [...Cnt, 1]>

type StrToNumTuple<S extends string, Res extends number[] = []> = 
S extends `${infer L extends number}${infer R}` ? StrToNumTuple<R, [...Res, L]> : Res

type StrCompare<A extends number[], B extends number[]> = 
A['length'] extends B['length'] //长度相等则进行比较
  ? [A, B] extends [[infer LA extends number, ...infer RA extends number[]], [infer LB extends number, ...infer RB extends number[]]]
    ? DigitCompare<LA, LB> extends Comparison.Equal ? StrCompare<RA, RB> : DigitCompare<LA, LB>
    : Comparison.Equal  //最后A、B都为[]，证明相等
  : StrCompare<StrToNumTuple<`${A['length']}`>, StrToNumTuple<`${B['length']}`>> 
  //长度不等直接比较长度即可，长度也是数字，递归比较

type Comparator<A extends number, B extends number> = 
`${A}` extends `-${infer SA}`
  ? `${B}` extends `-${infer SB}`
    ? OppositeRes<StrCompare<StrToNumTuple<SA>, StrToNumTuple<SB>>>
    : Comparison.Lower
  : `${B}` extends `-${infer SB}`
    ? Comparison.Greater
    : StrCompare<StrToNumTuple<`${A}`>, StrToNumTuple<`${B}`>>

// 另一种实现
type OppositeRes<T extends Comparison> = 
T extends Comparison.Equal 
  ? T
  : T extends Comparison.Lower ? Comparison.Greater : Comparison.Lower

type DigitCompare<A extends string, B extends string> =
Equal<A, B> extends true 
  ? Comparison.Equal
  : '0123456789' extends `${string}${A}${string}${B}${string}`
    ? Comparison.Lower : Comparison.Greater
    
type StrCompare<
  A extends string, B extends string, 
  Pre extends Comparison = Comparison.Equal
> = 
`${A}|${B}` extends `${infer LA}${infer RA}|${infer LB}${infer RB}`
  ? Pre extends Comparison.Equal 
    ? StrCompare<RA, RB, DigitCompare<LA, LB>> 
		: StrCompare<RA, RB, Pre>
  : A extends B
    ? Pre
    : A extends '' ? Comparison.Lower : Comparison.Greater

type Comparator<A extends number, B extends number> = 
`${A}` extends `-${infer SA}`
  ? `${B}` extends `-${infer SB}` ? OppositeRes<StrCompare<SA, SB>> : Comparison.Lower
  : `${B}` extends `-${infer SB}` ? Comparison.Greater : StrCompare<`${A}`, `${B}`>

// 支持小数版本：
type OppositeRes<T extends Comparison> = 
T extends Comparison.Equal 
  ? T
  : T extends Comparison.Lower ? Comparison.Greater : Comparison.Lower

type DigitCompare<A extends string, B extends string> =
Equal<A, B> extends true 
  ? Comparison.Equal
  : '0123456789' extends `${string}${A}${string}${B}${string}`
    ? Comparison.Lower : Comparison.Greater
    
type IntCompare<  //字典序与长度综合比较
  A extends string, B extends string, 
  Pre extends Comparison = Comparison.Equal
> = 
`${A}|${B}` extends `${infer LA}${infer RA}|${infer LB}${infer RB}`
  ? Pre extends Comparison.Equal 
    ? IntCompare<RA, RB, DigitCompare<LA, LB>> : IntCompare<RA, RB, Pre>
  : A extends B
    ? Pre
    : A extends '' ? Comparison.Lower : Comparison.Greater


type DecimalCompare<A extends string, B extends string> =   ///仅比较字典序
`${A}|${B}` extends `${infer LA}${infer RA}|${infer LB}${infer RB}`
  ? LA extends LB ? DecimalCompare<RA, RB> : DigitCompare<LA, LB>
  : A extends B
    ? Comparison.Equal
    : A extends '' ? Comparison.Lower : Comparison.Greater

type StrCompare<A extends string, B extends string> = 
A extends `${infer LA}.${infer RA}`
  ? B extends `${infer LB}.${infer RB}`
    ? LA extends LB ? DecimalCompare<RA, RB> : IntCompare<LA, LB>
    : LA extends B ? Comparison.Greater : IntCompare<LA, B>
  : B extends `${infer LB}.${infer RB}`
    ? A extends LB ? Comparison.Lower : IntCompare<A, LB>
    : A extends B ? Comparison.Equal : IntCompare<A, B>

type Comparator<A extends number, B extends number> = 
`${A}` extends `-${infer SA}`
  ? `${B}` extends `-${infer SB}` ? OppositeRes<StrCompare<SA, SB>> : Comparison.Lower
  : `${B}` extends `-${infer SB}` ? Comparison.Greater : StrCompare<`${A}`, `${B}`>

```

## **Sort**

```tsx
 //示例
const add = (a: number, b: number) => a + b
const three = add(1, 2)

const curriedAdd = Currying(add)
const five = curriedAdd(2)(3)

//实现
//归并排序, 支持大数、小数、负数, 需要借助上面实现的Comparator
enum Comparison {
  Greater = 1,
  Equal = 0,
  Lower = -1,
}

type Comparator<A extends number, B extends number> = ...

type BiDivide<T extends any[],L extends any[] = [], R extends any[] = []> = 
T extends [infer TL, ...infer M, infer TR]
  ? BiDivide<M, [...L, TL], [TR, ...R]>
  : [[...L, ...T], R]

type Merge<
  A extends any[], B extends any[], T extends number[] = [],
  Cmp extends Comparison = Comparison.Greater> = 
[A, B] extends [[infer AL extends number, ...infer AR], [infer BL extends number, ...infer BR]]
  ? Comparator<AL, BL> extends Cmp 
    ? Merge<A, BR, [...T, BL], Cmp>
    : Merge<AR, B, [...T, AL], Cmp>
  : [...T, ...A, ...B]

// 会递归过深标红
type Sort<T extends number[], Flag extends boolean = false> = 
T extends [] | [number]
  ? T
  : BiDivide<T, Flag> extends [infer A extends number[], infer B extends number[]]
    ? Merge<Sort<A, Flag>, Sort<B, Flag>, [], Flag extends true ? Comparison.Lower : Comparison.Greater>
    : never;

//另一种形式
type BiDivide<
  T extends any[], Flag extends boolean = false,
  L extends any[] = [], R extends any[] = []> = 
T extends [infer TL, ...infer M, infer TR]
  ? BiDivide<M, Flag, [...L, TL], [TR, ...R]>
  : [Sort<[...L, ...T], Flag>, Sort<R, Flag>]

type Merge<
  Arr extends [any[], any[]], T extends number[] = [],
  Cmp extends Comparison = Comparison.Greater> = 
Arr extends [[infer AL extends number, ...infer AR], [infer BL extends number, ...infer BR]]
  ? Comparator<AL, BL> extends Cmp 
    ? Merge<[Arr[0], BR], [...T, BL], Cmp>
    : Merge<[AR, Arr[1]], [...T, AL], Cmp>
  : [...T, ...Arr[0], ...Arr[1]]

//将BiDivide独立infer出P可以解决递归爆红问题
type Sort<T extends number[], Flag extends boolean = false> = 
T extends [] | [number]
  ? T
  : BiDivide<T, Flag> extends infer P extends [any[], any[]] 
    ? Merge<P, [], Flag extends true ? Comparison.Lower : Comparison.Greater>
    : never
```

## DynamicParamsCurrying

```tsx
 //示例
const add = (a: number, b: number) => a + b
const three = add(1, 2)

const curriedAdd = Currying(add)
const five = curriedAdd(2)(3)

//实现
type Curried<Args extends any[], Res> = 
Args extends []
  ? Res
  : <A extends any[]>(...args: A) => 
    Args extends [... A, ...infer R] ? Curried<R, Res> : never

declare function DynamicParamsCurrying<T extends (...args: any) => any>
(fn: T): Curried<Parameters<T>, ReturnType<T>>
```

## Sum

```tsx
 //示例
Sum<114514, 1919810> // "2034324"

//实现
// 按人类加法逻辑
namespace StrHelper {
  export type ExtractNum<S extends string> = 
    S extends `${infer N extends number}` 
      ? number extends N 
        ? S extends `${infer B extends bigint}` ? B : never
        : N
      : never

  export type ToReverseTuple<S extends string, Res extends number[] = []> = 
    S extends `${infer L extends number}${infer R}`
      ? ToReverseTuple<R, [L, ...Res]> : Res
  
  export type TupleToReverseStr<T extends any[], S extends string = ""> =
    T extends [infer L extends number, ...infer R]
      ? TupleToReverseStr<R, `${L}${S}`> : S
}

namespace Digit {
  export type NumToTupleLen<T extends number, Cnt extends 1[] = []> = 
    Cnt['length'] extends T ? Cnt : NumToTupleLen<T, [...Cnt, 1]>
  
  export type FormatResult<T extends string> = 
    `${T}` extends `${infer L extends number}${infer R extends number}` 
        ? [L, R] : [0, StrHelper.ExtractNum<T>] //[进位，本位结果]

  export type GetSum<
      A extends number, B extends number, C extends number = 0,
      T = [...NumToTupleLen<A>, ...NumToTupleLen<B>, ...NumToTupleLen<C>]
    > = `${(T extends any[] ? T : [])['length']}`

  export type Add<A, B, C extends number = 0, Res extends number[] = []> = 
    A extends [infer LA extends number, ...infer RA]
      ? B extends [infer LB extends number, ...infer RB]
        ? FormatResult<GetSum<LA, LB, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<RA, RB, NextC, [...Res, Cur]> : never
        : FormatResult<GetSum<LA, 0, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<RA, B, NextC, [...Res, Cur]> : never
      : B extends [infer LB extends number, ...infer RB]
        ? FormatResult<GetSum<0, LB, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<A, RB, NextC, [...Res, Cur]> : never
        : C extends 0 ? Res : [...Res, C]
}

type Sum<A extends string | number | bigint, B extends string | number | bigint> = 
  StrHelper.TupleToReverseStr<
    Digit.Add<
      StrHelper.ToReverseTuple<`${A}`>, StrHelper.ToReverseTuple<`${B}`>
    >
  >

//其他更优雅实现：
type Reverse<A extends string, Res extends string = ""> = 
  A extends `${infer L}${infer R}` 
    ? Reverse<R,`${L}${Res}`> : Res

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

type Add<A extends string, B extends string, Res extends string = ""> = 
  A extends `${infer AL}${infer AR}` 
  ? B extends `${infer BL}${infer BR}` 
    ? BL extends '0' ? Add<AR, BR, `${Res}${AL}`> : Add<AddOne<A>, SubOne<B>, Res> 
    : `${Res}${A}` : `${Res}${B}`

type Sum<A extends string | number | bigint, B extends string | number | bigint> = 
  Reverse<Add<Reverse<`${A}`>, Reverse<`${B}`>>>
```

## Multiply

```tsx
 //示例
Multiply<114514, 1919810> // "219845122340"

//实现
// 按人类乘法逻辑
namespace StrHelper {
  export type ExtractNum<S extends string> = 
    S extends `${infer N extends number}` 
      ? number extends N 
        ? S extends `${infer B extends bigint}` ? B : never
        : N
      : never

  export type ToReverseTuple<S extends string, Res extends number[] = []> = 
    S extends `${infer L extends number}${infer R}`
      ? ToReverseTuple<R, [L, ...Res]> : Res
  
  export type TupleToReverseStr<T extends any[], S extends string = ""> =
    T extends [infer L extends number, ...infer R]
      ? TupleToReverseStr<R, `${L}${S}`> : S
}

namespace Digit {
  export type NumToTupleLen<T extends number, Cnt extends 1[] = []> = 
    Cnt['length'] extends T ? Cnt : NumToTupleLen<T, [...Cnt, 1]>

  export type NumToBaseTupleLen<
    T extends number, Base extends 1[], 
    Cnt extends 1[] = [], Res extends 1[] = []
  > = Cnt['length'] extends T 
      ? Res : NumToBaseTupleLen<T, Base, [...Cnt, 1], [...Res, ...Base]>
  
  export type FormatResult<T extends string> = 
    `${T}` extends `${infer L extends number}${infer R extends number}` 
        ? [L, R] : [0, StrHelper.ExtractNum<T>] // [进位，本位结果]

  export type GetSum<
      A extends number, B extends number, C extends number = 0,
      T = [...NumToTupleLen<A>, ...NumToTupleLen<B>, ...NumToTupleLen<C>]
    > = `${(T extends any[] ? T : [])['length']}`

  export type Add<A, B, C extends number = 0, Res extends number[] = []> = 
    A extends [infer LA extends number, ...infer RA]
      ? B extends [infer LB extends number, ...infer RB]
        ? FormatResult<GetSum<LA, LB, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<RA, RB, NextC, [...Res, Cur]> : never
        : FormatResult<GetSum<LA, 0, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<RA, B, NextC, [...Res, Cur]> : never
      : B extends [infer LB extends number, ...infer RB]
        ? FormatResult<GetSum<0, LB, C>> extends [infer NextC extends number, infer Cur extends number]
          ? Add<A, RB, NextC, [...Res, Cur]> : never
        : C extends 0 ? Res : [...Res, C]
  
  export type GetMul<
      A extends number, B extends number, C extends number = 0,
      T = NumToBaseTupleLen<A, NumToTupleLen<B>>
    > = GetSum<(T extends any[] ? T : [])['length'], C>

  export type GetRowMul<
      Base, M extends number, C extends number = 0, 
      Res extends number[] = []
    > = Base extends [infer LB extends number, ...infer RB]
        ? FormatResult<GetMul<LB, M, C>> extends [infer NextC extends number, infer Cur extends number]
          ? GetRowMul<RB, M, NextC, [...Res, Cur]> : never
        : C extends 0 ? Res : [...Res, C]

  export type Mul<A, B, Res extends number[] = [], Cnt extends 0[] = []> = 
    A extends [infer LA extends number, ...infer RA]
      ? Mul<RA, B, Add<Res, [...Cnt, ...GetRowMul<B, LA>]>, [...Cnt, 0]> // 移位求和
      : Res extends 0[] ? [0] : Res   // 除去多余的0
}

type Multiply<A extends string | number | bigint, B extends string | number | bigint> = 
  StrHelper.TupleToReverseStr<
    Digit.Mul<StrHelper.ToReverseTuple<`${A}`>, StrHelper.ToReverseTuple<`${B}`>>
  >

//其他更优雅实现：
type Reverse<A extends string, Res extends string = ""> = 
  A extends `${infer L}${infer R}` 
    ? Reverse<R,`${L}${Res}`> : Res

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

type Add<A extends string, B extends string, Res extends string = ""> = 
  A extends `${infer AL}${infer AR}` 
  ? B extends `${infer BL}${infer BR}` 
    ? BL extends '0' ? Add<AR, BR, `${Res}${AL}`> : Add<AddOne<A>, SubOne<B>, Res> 
    : `${Res}${A}` : `${Res}${B}`

type Mul<A extends string, B extends string, R extends string = '0'> = 
  A extends '0' ? R : 
  B extends '0' ? R :
  A extends `${infer AH}${infer AT}` 
    ? AH extends '0' ? Mul<AT, `0${B}`, R> : Mul<SubOne<A>, B, Add<R, B>>
    : R

type Multiply<A extends string | number | bigint, B extends string | number | bigint> = 
  Reverse<Mul<Reverse<`${A}`>, Reverse<`${B}`>>>
```

## Subtract

```tsx
 //示例
Subtract<2, 1> // 1
Subtract<1, 2> // never

//实现
//支持number表示范围内正整数的减法（若用字符串表示则是不限范围的正整数）
type Reverse<A extends string, Res extends string = ""> = 
  A extends `${infer L}${infer R}` ? Reverse<R,`${L}${Res}`> : Res

type ToNumber<T extends string> = 
T extends `0${infer L}` 
  ? L extends '' ? 0 : ToNumber<`${L}`> 
  : T extends `${infer R extends number}` ? R : never

type DigsPrev<S = '9876543210', Res = {}> = 
  S extends `${infer L}${infer M}${infer R}`
    ? DigsPrev<`${M}${R}`, Res & Record<L, M>> : Omit<Res, never>

type SubOne<A extends string, Res extends string = ""> = 
  A extends `${infer L}${infer R}`
    ? L extends keyof DigsPrev ? `${Res}${DigsPrev[L]}${R}` : SubOne<R, `${Res}9`>
    : never

type Sub<A extends string, B extends string, Res extends string = ""> = 
  A extends `${infer AL}${infer AR}` 
  ? B extends `${infer BL}${infer BR}` 
    ? BL extends '0' ? Sub<AR, BR, `${Res}${AL}`> : Sub<SubOne<A>, SubOne<B>, Res> 
    : `${Res}${A}` : `${Res}${B}`

type Subtract<M extends number, S extends number> = 
ToNumber<Reverse<Sub<Reverse<`${M}`>, Reverse<`${S}`>>>>

// @ts-expect-error
Subtract<1000, 999>  
//这里希望结果出错，但觉得没有必要所以不予考虑
//如果想通过此用例可以采用元组长度来实现，但也算不上地狱难度了
type NumToTuple<N extends number, Res extends any[] = []> =
Res['length'] extends N ? Res : NumToTuple<N, [...Res, 1]>
type Subtract<M extends number, S extends number> = 
NumToTuple<M> extends [...NumToTuple<S>, ...infer R] ? R['length'] : never
```

## Tag

- 对象加可选属性以及 T | T & U都能与T相互赋值

```tsx
 //示例
declare let x: string
declare let y: Tag<string, 'A'>
x = y = x

type T0 = Tag<{ foo: string }, 'A'>
type T1 = Tag<T0, 'B'>
HasExactTags<T1, ['A', 'B']> //true

type T2 = Tag<number, 'C'>
Equal<GetTags<T2>, ['C']> //true

type T3 = Tag<0 | 1, 'D'>
HasTag<T3, 'D'> //true

//元组顺序应为嵌套顺序
type T4 = Tag<Tag<Tag<{}, 'A'>, 'B'>, 'C'>
HasTags<T4, ['B', 'C']> //true

type T5 = Tag<Tag<unknown, 'A'>, 'B'>
HasExactTags<T5, ['A', 'B']> //true

type T6 = Tag<{ bar: number }, 'A'>
type T7 = UnTag<T6>
HasTag<T7, 'A'> //false

//实现
declare const UNIQUE_SYMBOL: unique symbol
type US = typeof UNIQUE_SYMBOL

type TagsBag<B, T> = {
  [UNIQUE_SYMBOL]?: US | (US & [B, T])
}

type UnionToIntersection<U> = 
(U extends any ? (a: U) => any : never) extends (a: infer T) => any
  ? T : never

type _GetTags<B> = 
Equal<B, never> extends true
  ? []
  : B extends TagsBag<unknown, infer Tags extends string[]> 
    ? string[] extends Tags ? [] : Tags //Tags可能为联合类型，需要触发分布式条件类型
    : []

type GetTags<B> = 
UnionToIntersection<_GetTags<B>> extends infer Res extends string[]
  ? Equal<Res, never> extends true ? [] : Res
  : never

type FilterNullable<B, D> = 
[Equal<B, null>, Equal<B, undefined>] extends [false, false]
  ? D : B

type Tag<B, T extends string> = 
FilterNullable<B, UnTag<B> & TagsBag<UnTag<B>, [...GetTags<B>, T]>>

type UnTag<B> = FilterNullable<B, Omit<B, US>>

type Includes<A extends readonly any[], B extends readonly any[]> = 
A extends [...B, ...any[]]
  ? true
  : A extends [any, ...infer Rest] ? Includes<Rest, B> : false

type HasTag<B, T extends string> = Includes<GetTags<B>, [T]>
type HasTags<B, T extends readonly string[]> = Includes<GetTags<B>, T>
type HasExactTags<B, T extends readonly string[]> = Equal<GetTags<B>, T>
```

## DistributeUnions

```tsx
 //示例
DistributeUnions<[1 | 2, 'a' | 'b']> 
// [1, 'a'] | [2, 'a'] | [1, 'b'] | [2, 'b']
DistributeUnions<
{ type: 'a', value: number | string } | { type: 'b', value: boolean }
> 
// { type 'a', value: number }
// | { type 'a', value: string }
// | { type 'b', value: boolean }

//实现
type DistributeValues<K extends PropertyKey, V> = 
V extends V ? Record<K, V> : never

type DistributeItems<L, R extends any[], Res extends any[]> = 
L extends L ? DistributeArray<R, [...Res, L]> :never

type DistributeArray<T extends any[], Res extends any[] = []> = 
T extends [infer L, ...infer R]
  ? L extends L ? DistributeItems<DistributeUnions<L>, R, Res> : never 
  : Res

type DistributeObject<O extends object, K extends keyof O = keyof O> = 
[K] extends [never] 
  ? {}
  : K extends K 
    ? DistributeValues<K, DistributeUnions<O[K]>> & DistributeObject<Omit<O, K>>
  : never

type Merge<O> = { [K in keyof O]: O[K] }

type DistributeUnions<T> = 
T extends any[]
  ? DistributeArray<T>
  : T extends object ? Merge<DistributeObject<T>> : T
```

## assertArrayIndex

```tsx
 //示例
const matrix = [
    [3, 4],
    [5, 6],
    [7, 8],
];
assertArrayIndex(matrix, 'rows');
let sum = 0;
for (let i = 0 as Index<typeof matrix>; i < matrix.length; i += 1) {
  const columns: number[] = matrix[i];
  // @ts-expect-error: number | undefined in not assignable to number
  const x: number[] = matrix[0];
  assertArrayIndex(columns, 'columns');
  for (let j = 0 as Index<typeof columns>; j < columns.length; j += 1) {
    sum += columns[j];
    // @ts-expect-error: number | undefined in not assignable to number
    const y: number = columns[i];
    // @ts-expect-error: number | undefined in not assignable to number
    const z: number = columns[0];
    // @ts-expect-error: number[] | undefined in not assignable to number[]
    const u: number[] = matrix[j];
  }
}

//实现
type Ord = {a:1,b:2,c:3,d:4,e:5,f:6,g:7,h:8,i:9,j:10,k:11,l:12,m:13,n:14,o:15,p:16,q:17,r:18,s:19,t:20,u:21,v:22,w:23,x:24,y:25,z:26}

type Make<N, A extends 1[] = []>
    = A['length'] extends N ? A : Make<N, [1, ...A]>

type Hash<S, A extends any[] = []> = 
S extends `${infer L}${infer R}` 
  ? Hash<R, [...Make<Ord[keyof Ord & L]>, ...A]> : A['length']

function assertArrayIndex<A extends readonly any[], K extends string>(
  array: number extends A['length'] ? A : never, key: K
) : asserts array is (
  number extends A['length'] 
    ? A & {key: Hash<K>} & {[P in Hash<K>]: A[number]} 
    : never
  ) {}

type Index<Array> = Array[keyof Array & 'key']
```

## Parse

```tsx
 //示例
Parse<`
  {
    "a": "b", 
    "b": false, 
    "c": [true, false, "hello", {
      "a": "b", 
      "b": false
    }], 
    "nil": null
  }
`>
/*
{
  nil: null
  c: [true, false, 'hello', {
    a: 'b'
    b: false
  }]
  b: false
  a: 'b'
}
*/

//实现
type SpecialChar = {r: '\r', n: '\n', b: '\b', f: '\f'}

type Format<S extends string, Res extends string = ""> = 
S extends `\\${infer L extends keyof SpecialChar}${infer R}`
  ? Format<R, `${Res}${SpecialChar[L]}`>
  : S extends `${infer L}${infer R}`
    ? Format<R, `${Res}${
        L extends '}' | ']'
          ? `,${L}`
          : L extends ' ' | '\n' | '\t' ? '' : L
      }`>
    : Res

type Eval<V> = 
V extends `"${infer T}"`
  ? T 
  : V extends `${infer B extends boolean | null}`
    ? B 
    : V extends `${number}` ? never : V

type ParseObject<S extends string, Res extends object = {}> = 
S extends `${'}' | ',}'}${infer R}`
  ? [Omit<Res, never>, R extends `,${infer P}` ? P : R]
  : S extends `${infer K}:${infer R}`
    ? [Eval<K>] extends [never | boolean | null]
      ? never
      : ParseValue<R> extends [infer V, infer T extends string]
        ? ParseObject<T, Res & Record<Eval<K>, V>> 
        : never
    : never

type ParseArray<S extends string, Res extends any[] = []> = 
S extends `${']' | ',]'}${infer R}`
  ? [Res, R extends `,${infer P}` ? P : R]
  : ParseValue<S> extends [infer V, infer T extends string]
    ? ParseArray<T, [...Res, V]> 
    : never

type ParseValue<S extends string> = 
S extends `{${infer O}`
  ? ParseObject<O>
  : S extends `[${infer A}` 
    ? ParseArray<A> 
    : S extends `${infer L},${infer R}`
      ? [Eval<L>] extends [never] ? [] : [Eval<L>, R] 
      : [Eval<S>, ""]

type Parse<T extends string> = ParseValue<Format<T>>[0]

//测试用例
Parse<'{ "true": "aaa" }'> // {true: "aaa"}
Parse<'{ true: "aaa" }'> // never
Parse<'{ 1: "world" }'> // never
Parse<'["1"]'> // ["1"]
Parse<'[1]'> // never
```

## **CountReversePairs**

```tsx
 //示例
CountReversePairs<[5, 2, 6, 1]> // 4
CountReversePairs<[1, 2, 3, 4]> // 0
CountReversePairs<[-1, -1]> // 0
CountReversePairs<[-1]> // 0

//实现
type AbsReverses<X extends number, Y extends number, A extends 1[] = []> = 
A['length'] extends X
  ? 0
  : A['length'] extends Y ? 1 : AbsReverses<X, Y, [...A, 1]>;

type Reverses<A extends number, B extends number> = 
`${A}` extends `-${infer X extends number}`
  ? `${B}` extends `-${infer Y extends number}` ? AbsReverses<Y, X> : 0
  : `${B}` extends `-${number}` ? 1 : AbsReverses<A, B>;

type CountReverseWith<N extends number, A extends number[], R extends 1[] = []> = 
A extends [infer F extends number, ...infer T extends number[]]
  ? CountReverseWith<N, T, Reverses<N, F> extends 0 ? R : [...R, 1]>
  : R;

type CountReversePairs<A extends number[], R extends 1[] = []> = 
A extends [infer F extends number, ...infer T extends number[]]
  ? CountReversePairs<T, [...R, ...CountReverseWith<F, T>]>
  : R['length'];
```

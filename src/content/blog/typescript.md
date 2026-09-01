---
title: "前端工程師的知識地圖: 進階一點點的 TypeScript #2"
description: "TypeScript 筆記"
pubDate: "Sep 01 2026"
heroImage: ""
---

# Type Compatibility

TypeScript 的型別相容 (compatibility) 來自於結構的相容（Structural Typing），而非是否宣告相同型別（Nominal Typing）。

- **TypeScript 較強型別:** 在 compile 階段就檢查 (JavaScript 在 runtime 階段才檢查)
- **Soundness** = 只要型別檢查通過，就不應該因為型別問題在 runtime 出錯 → (X) 因為 <mark>TypeScript 為了兼容 JavaScript 寫法所以犧牲了型別的嚴謹度 (e.g. `any`)</mark>，所以並非 100% 健全

---

## Functions

### 參數的相容性

```typescript
let x = (a: number) => 0;
let y = (b: number, s: string) => 0;
y = x; // OK: y 需要兩個參數，x 有其中一個，所以 x 包含在 y 內
x = y; // Error: y 需要兩個參數，但 x 只有其中一個所以不能相等
```

- 接近 JavaScript 傳入參數的寫法（<mark>不需要完全符合</mark>）

```javascript
let items = [1, 2, 3];
// 本來需要傳入三個參數
items.forEach((item, index, array) => console.log(item));
// 但傳入有符合其中一個也可以
items.forEach((item) => console.log(item));
```

### 回應的相容性

```javascript
let x = () => ({ name: "Alice" });
let y = () => ({ name: "Alice", location: "Seattle" });
x = y; // OK：x 只要回傳一個參數，但 y 還多傳兩個參數，可以相符
y = x; // Error: y 應該回傳三個參數，x 只回傳一個，故不相符
```

### Bivariance

- TypeScript 基於 JavaScript 的特性，很容易可以雙向相容
- 若想要嚴格符合，則要使用 [`strictFunctionTypes`](https://www.typescriptlang.org/tsconfig#strictFunctionTypes) 更嚴格的檢查相容性

### Optional & Rest Parameter

**<mark>可選參數</mark>** 或 **<mark>剩餘參數</mark>** 皆屬於相對寬鬆的規則．

1. **optional parameter** **<mark>(?)</mark>**

```javascript
type A = (x: number) => void
type B = (x: number, y?: string) => void
```

2. **rest parameter <mark>(...)</mark>**

```javascript
(...args: number[]) => void
// 相當於遍歷生成多個可選參數
(
  arg1?: number,
  arg2?: number,
  arg3?: number,
  arg4?: number,
  ...
) => void
```

---

## Enums

Enum 的底層是數字，但不同 enum type 的列舉互不相容。

```javascript
enum Status {
  Ready,
  Waiting,
}
enum Color {
  Red,
  Blue,
  Green,
}
let status = Status.Ready;
status = Color.Green; // Error
```

---

## Class

Class 的組成可以有兩種類別:

```javascript
class Animal {
  feet: number;
  constructor(name: string, numFeet: number) {}
}
```

### Static type

```javascript
constructor(name: string, numFeet: number) {}
```

- Class 型別本身組成時就有的屬性

```javascript
class Animal {
  static category = "animal"
  feet = 4
}

Animal.category // animal
Animal.feet // Error: 因為 Animal 此類別組成本身並沒有 feet 屬性
```

### Instance type

```javascript
feet: number;
```

- new 之後的物件所帶有的屬性

```javascript
class Animal {
  static category = "animal"
  feet = 4
}

const a = new Animal()

a.feet // 4
```

### Class 和 Interface 的差別

- `Interface` : 用來定義型別(檢查)，在 <mark>runtime 不會被編譯進去。</mark>
- `Class` : 定義類別，在 <mark>runtime 會真的編譯一個 JavaScript 類別。</mark>

---

## Generics

**泛型:** 先保留型別，等到**<mark>使用到再決定是甚麼特定型別</mark>**

- **Generic 裡的 `T` 只有真的出現在型別結構中，才會影響型別相容性。**

```javascript
interface Empty<T> {} // 此時 T 不管帶入甚麼，型別都相容

interface NotEmpty<T> {
  data: T // 此時 T 帶入不同的型別，則互不相容
}
```

- 尚未使用前 (不會決定型別)，視為相容 → 皆會視為 `any` 型別，所以相容

---

## 特殊型別

### `any`

可以 **<mark>assign 給任何的型別</mark>**

```typescript
let x: any 
let y: string 

y = x  // ok
x = y  // ok
x = "hello" // ok
```

### `unknown`

**<mark>不知道型別的值</mark>**，所以**<mark>可以接收任何值</mark>**，但**<mark>不能 assign</mark>** 給其他特定型別

```typescript
let x: unknown 
let y: string 

y = x  // error
x = y  // ok
x = "hello" // ok
```

### `never`

**<mark>不可能存在的值</mark>**，所以**<mark>不能接收值</mark>**，但**<mark>可以 assign</mark>** 給其他特定型別

```typescript
let x: never 
let y: string 

y = x  // ok
x = y  // error
x = "hello" // error
```

---

# Narrowing

原本一個變數可能有很多種型別，TypeScript 根據判斷條件，把它縮小成更具體的型別。

- 外部資料在未知型別的時候，都先以 `unknown` 作為型別 (盡量不要 `any`) → 再以 narrowing 去縮窄型別檢查。

---

## Type guards `typeof`

- **<mark>`typeof` 不會回傳 `null`</mark>**  → 因為 `null` <mark>== `object`</mark>

```typescript
// 無法真正 narrow string[] 或 null
function print(value: string[] | null) {
  if (typeof value === "object") {
    // value: string[] | null 都會在這裡
  }
}

// 所以要多檢查一個 value !== null
if (value !== null && typeof value === "object") {
  // value: string[] 才會區分出 string[] 和 null
}
```

---

## Truthiness

根據某個值在 `if` 裡是「truthy 還是 falsy」，來縮小它的型別。

- **Falsy value**

```typescript
false
0
-0
0n
""
null
undefined
NaN
```

- **注意: <mark>容易吞掉合法值!</mark>**

```typescript
function show(count: number | null) {
  if (count) {
    // 這裡排除了 null，也排除了 0...
  }
}
```

---

## Equality

用如何相等來縮小型別:

```typescript
===, !==, ==, !=
```

---

## The `in` operator

用 Object 或 Prototype 中是否有該屬性，來縮小型別

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };
 
function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    return animal.swim();
  }
 
  return animal.fly();
}
```

---

## `instanceof` narrowing

沿著原型鏈找是否找得到 prototype，來縮小型別

```typescript
class Foo {}

const x = new Foo()

console.log(x instanceof Foo) // true
```

- **Prototype Chain**

```typescript
x
↓
Foo.prototype
↓
Object.prototype
↓
null
```

---

## Predicates

如果條件符合，型別縮小為 `is` Type

```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return true; // 可以是任何 Boolean 判斷式
}
```

---

## Assertion

斷言暫時當成某個型別

```typescript
const value: unknown = "hello"

const text = value as string // value 就會被當成 string 型別
```

---

## Discriminated Unions

在 Union Type 裡面使用 discriminator 來辨識其中一種型別。

<mark>→ 適合用來管控前端的 State 組態型別</mark>

```typescript
type ApiState =
  | { status: "loading" }
  | { status: "success"; data: User[] }
  | { status: "error"; message: string }

// status 就是用來辨識的 discriminator
if (state.status === "success") {
  state.data // ✅
}
```

---

## Type `never`

排除任何型別的值 (不可能存在的值)

### Exhaustiveness Checking

`never` 可以用來檢查 narrow 到最後的型別<mark>是否確實全部處理完畢</mark>

```typescript
function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2

    case "square":
      return shape.side ** 2

    default:
      const _check: never = shape // 若全部處理完畢，則為 true
      return _check
  }
}
```

---

# Generics

## Generic Type

泛型: 型別命名不重要，重要的是如何使用

### 函式裡的泛型

每次呼叫函數都可以重新決定型別

```typescript
interface GenericIdentityFn {
  <Type>(arg: Type): Type
}
```

```typescript
let fn: GenericIdentityFn

fn("hello") // Type = string
fn(123)     // Type = number
fn(true)    // Type = boolean
```

### Interface 裡的泛型

這個 interface 的 `Type` 現在已經被固定成特定型別

```typescript
interface GenericIdentityFn<Type> {
  (arg: Type): Type
}
```

```typescript
myIdentity(123) // ✅
myIdentity(456) // ✅

myIdentity("hello") // ❌
```

---

## Generic Class

1. Class 裡面可以使用泛型 T

```typescript
class Box<T> {
  value: T
}
```

2. 建立 Instance 的時候決定泛型 T 的型別

```typescript
const a = new Box<number>() // a.value = number
const b = new Box<string>() // b.value = string
```

3. 同一個 Interface 的 T 代表同一個型別

```typescript
class Box<T> {
  value: T

  setValue(value: T) {
    this.value = value
  }

  getValue(): T {
    return this.value
  }
}
```

4. 泛型 T 只能定義 instance value，不能定義 static value

```typescript
class Box<T> {
  value: T // ✅
  static value: T // ❌
}
```

- 因為 static value 是該 class 本身的屬性 → 整個 class 共用的 ⇒ <mark>不avan在 static value 使用泛型</mark>

```typescript
const a = new Box<string>
const b = new Box<number>

// 這樣會不知道 Box.value 到底是 string 還是 number
```

---

## Generic Constraint

使用泛型，卻又同時加上限制 key 要符合泛型的 Type。

```typescript
function getProperty<Type, Key extends keyof Type>(obj: Type, key: Key) {
  return obj[key];
}
 
let x = { a: 1, b: 2, c: 3, d: 4 };
 
getProperty(x, "a");
getProperty(x, "m"); // error
```

---

## Class Type in Generics

限制傳入的 Class (`Lion`) 屬於哪一個 class 類別 (`Animal`)，同時又拿到該 Class 的型別資訊 (`ZooKeeper`)

→ 適用於 `Factory`

```typescript
class ZooKeeper {
  nametag: string = "Mikle";
}
 
class Animal {
  numLegs: number = 4;
}

class Lion extends Animal {
  keeper: ZooKeeper = new ZooKeeper();
}
 
function createInstance<A extends Animal>(c: new () => A): A {
  return new c();
}
 
createInstance(Lion).keeper.nametag;
```

```json
createInstance(Lion)

Lion
↓
new () => Lion
↓
A = Lion
↓
回傳 A
↓
回傳 Lion
↓
Lion.keeper = ZooKeeper
↓
ZooKeeper.nametag = string
```

---

# Utility Type

### `Record<Keys, Type>`

定義多個 key 分別為某 Type 的型別組合

```typescript
type CatName = "miffy" | "boris" | "mordred";
 
interface CatInfo {
  age: number;
  breed: string;
}
 
const cats: Record<CatName, CatInfo> = {
  miffy: { age: 10, breed: "Persian" },
  boris: { age: 5, breed: "Maine Coon" },
  mordred: { age: 16, breed: "British Shorthair" },
};
```

### `Exclude<UnionType, ExcludedMembers>`

排除一個 UnionType 的其中幾個屬性

```typescript
type T0 = Exclude<"a" | "b" | "c", "a">;
   
type T0 = "b" | "c"
```

### `Extract<Type, Union>`

從一個 Type 中萃取可以指派給 Union 的型別，組成一個新的型別。

```typescript
type T0 = Extract<"a" | "b" | "c", "a" | "f">;
   
type T0 = "a"
```

### `Parameters<Type>`

將一個 function 的屬性提取出來組成另一個 Type

```typescript
function createUser(name: string, age: number, active: boolean) {
  // ...
}

type Params = Parameters<typeof createUser>;
```

### `ConstructorParameters<Type>`

將一個 Class 的 constructor 提取做為另一個 Type

```typescript
class User {
  constructor(
    public name: string,
    public age: number
  ) {}
}

type UserArgs = ConstructorParameters<typeof User>;
type UserArgs = [name: string, age: number];
```

### `ReturnType<Type>`

取得 function 的回傳型別

```typescript
type T1 = ReturnType<(s: string) => void>;
   
type T1 = void
```

### 基础类型

| 数据类型   | 关键字      | 描述                                                         |
| :--------- | ----------- | ------------------------------------------------------------ |
| 任意类型   | `any`       |                                                              |
| 数字类型   | `number`    |                                                              |
| 字符串类型 | `string`    | 单引号（**'**）或双引号（**"**）来表示字符串类型。反引号（**`**）来定义多行文本和内嵌表达式。 |
| 布尔类型   | `boolean`   |                                                              |
| 数组类型   |             | 1. `let arr: number = [1,2,3]`<br>2. `let arr: Array<number> = [1,2,3]`<br>3. `let arr: Array<T> = [1,2,3]`    \# T也可以用any |
| 元组类型   |             | 元组类型用来表示已知元素数量和类型的数组，各元素的类型不必相同，对应位置的类型需要相同。<br>1. `let x: [string, number];` |
| 枚举类型   | `enum`      | 枚举类型用于定义数值集合。<br>`enum Color {Red, Green, Blue};`<br> `let c: Color = Color.Blue;` |
| void       | `void`      | 用于标识方法返回值的类型，表示该方法没有返回值。             |
| null       | `null`      | 表示对象值缺失。                                             |
| undefined  | `undefined` | 用于初始化变量为一个未定义的值                               |
| never      | `never`     | never 是其它类型（包括 null 和 undefined）的子类型，代表从不会出现的值。 |


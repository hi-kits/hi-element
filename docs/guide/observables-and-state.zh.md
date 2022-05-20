

## Reactivity

模板中使用的箭头函数绑定和指令允许`hi-element`模板引擎通过只更新DOM中实际更改的部分来智能地做出反应，而不需要虚拟DOM、VDOM diffing或DOM协调算法。这种方法支持顶级初始渲染时间、业界领先的增量DOM更新和超低内存分配。

当在模板中使用绑定时，基础引擎使用一种技术来捕获在该表达式中访问哪些属性。捕获属性列表后，它会订阅属性值的更改。每当值更改时，都会在DOM更新队列上安排任务。处理队列时，所有更新都作为一个批处理运行，精确地更新已更改的DOM的各个方面。

## 观察者

要启用绑定跟踪和更改通知，属性必须用`@attr`或`@observable`装饰器。将`@attr`用于打算作为HTML属性显示在元素上的基本属性（字符串、布尔值、数字）。对所有其他属性使用`@observable`。除了观察属性外，模板系统还可以观察阵列。`repeat`指令能够有效地响应数组更改记录，根据集合中的更改更新DOM。

这些装饰器是对类上的属性进行元编程的一种手段，因此它们包括支持状态跟踪、观察和反应所需的所有实现。您可以访问模板中的任何属性，但如果该属性未使用这两个装饰器之一进行装饰，则其值在初始渲染后不会更新。

:::important
`@attr`装饰器只能在`HIElement`中使用，但`@observable`装饰器可以在任何类中使用。
:::

:::important
只有getter的属性，即作为其他可观察对象的计算属性的函数，不应使用`@attr`或`@observable`修饰。但是，根据内部逻辑，它们可能需要用`@volatile`修饰。
:::

## 可观察特征

### 访问跟踪

在模板渲染期间访问`@attr`和`@observable`修饰属性时，会跟踪它们，使引擎能够深入了解模型和视图之间的关系。这些装饰器用于为您对属性进行元编程，注入代码以启用观察系统。但是，如果您不喜欢这种方法，对于`@observable`，您总是可以手动实现通知。下面是它的样子：
**实例: 手动观察者实现**

```ts
import { Observable } from '@hi-element';

export class Person {
  private _firstName: string;
  private _lastName: string;

  get firstName() {
    Observable.track(this, 'firstName');
    return this._firstName;
  }

  set firstName(value: string) {
    this._firstName = value;
    Observable.notify(this, 'firstName');
  }

  get lastName() {
    Observable.track(this, 'lastName');
    return this._lastName;
  }

  set lastName(value: string) {
    this._lastName = value;
    Observable.notify(this, 'lastName');
  }

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

请注意，`fullName`属性不需要任何特殊代码，因为它是基于已经可以观察到的属性进行计算的。这有一个特殊的例外：如果您有一个带有分支代码路径的计算属性，例如三元运算符、if/else条件等，那么您必须告诉观察系统您的计算属性具有*volatile*依赖关系。换句话说，需要观察哪些属性可能会根据代码路径的执行情况在调用之间发生变化。
以下是您如何与装饰器一起做到这一点：
```ts
import { observable, volatile } from '@hi-element';

export class MyClass {
  @observable someBoolean = false;
  @observable valueA = 0;
  @observable valueB = 42;

  @volatile
  get computedValue() {
    return this.someBoolean ? this.valueA : this.valueB;
  }
}
```
如果没有装饰器，您可以这样做：

```ts
import { Observable, observable } from '@hi-element';

export class MyClass {
  @observable someBoolean = false;
  @observable valueA = 0;
  @observable valueB = 42;

  get computedValue() {
    Observable.trackVolatile();
    return this.someBoolean ? this.valueA : this.valueB;
  }
}
```

### 内部观察

在定义了`@attr`或`@observable`的类上，您可以选择性地实现一个 *propertyName*Changed 方法，以轻松响应您自己状态的更改。

**实例: 属性更改回调**

```ts
import { observable } from '@hi-element';

export class Person {
  @observable name: string;

  nameChanged(oldValue: string, newValue: string) {

  }
}
```

### 外部观察

可以订阅装饰的属性，以接收属性值更改的通知。模板引擎使用此功能，但您也可以直接订阅。下面是您如何订阅`Person`类的`name`属性的更改：

**实例: 订阅可观察到的**

```ts
import { Observable } from '@hi-element';

const person = new Person();
const notifier = Observable.getNotifier(person);
const handler = {
  handleChange(source: any, propertyName: string) {
    // respond to the change here
    // source will be the person instance
    // propertyName will be "name"
  }
};

notifier.subscribe(handler, 'firstName')
notifier.unsubscribe(handler, 'lastName');
```

## 观察数组
到目前为止，我们只看到了如何观察对象的属性，但也可以观察数组的变化。给定一个数组实例，可以这样观察：

**实例: 观察数组**

```ts
const arr = [];
const notifier = Observable.getNotifier(arr);
const handler = {
  handleChange(source: any, splices: Splice[]) {
    // respond to the change here
    // source will be the array instance
    // splices will be an array of change records
    // describing the mutations in the array in
    // terms of splice operations
  }
};

notifier.subscribe(handler);
```

数组观测需要注意几个重要细节：
* `hi-element`库观察数组的能力是选择性加入的，以便功能保持树的稳定性。如果您在代码中的任何位置使用`repeat`指令，您将自动选择加入。但是，如果您希望使用上述API并且不使用`repeat`，则需要通过导入和调用`enableArrayObservation()`函数来启用数组观察。
* 观测系统无法通过索引更新直接跟踪所做的更改。例如：`arr[3] = 'new value';`。这是由于JavaScript中的限制。要解决此问题，请使用等效的`splice`代码更新数组，例如`arr.splice(3, 1, 'new value');`
* 如果数组是对象的属性，则通常需要同时观察属性和数组。观察属性将允许您检测对象上的数组实例何时被完全替换，而观察数组将允许您检测数组实例本身的更改。当属性更改时，请确保取消订阅旧阵列并设置对新阵列实例的订阅。
* 观察阵列只会通知阵列本身的更改。它不会在阵列中保存的对象的属性发生更改时发出通知。需要为这些单独的财产设立单独的观察员。不过，可以根据阵列中的更改设置和拆除这些组件。

## 观察 Volatile 属性

除了查看属性和数组，您还可以查看volatile属性。

**实例: 订阅Volatile属性**

```ts
import { Observable, defaultExecutionContext } from '@hi-element';

const myObject = new MyClass();
const handler = {
  handleChange(source: any) {
    // respond to the change here
    // the source is the volatile binding itself
  }
};
const bindingObserver = Observable.binding(myObject.computedValue, handler);
bindingObserver.observe(myObject, defaultExecutionContext);

// Call this to dismantle the observer
bindingObserver.disconnect();
```

### 记录 

要检查从`BindingObserver`访问了哪些可观察对象和属性，可以从以下位置获取观察记录 `BindingObserver.records()` 观察绑定后的结果。

**实例: 获取观察记录**
```ts
const binding = (x: MyClass) => x.someBoolean ? x.valueA : x.valueB;
const bindingObserver = Observable.binding(binding);
const value = bindingObserver.observe({}, defaultExecutionContext);

for (const record of bindingObserver.records()) {
  // Do something with the binding's observable dependencies
  console.log(record.propertySource, record.propertyName)
}
```



## 基础元素

要定义自定义元素，请首先创建一个扩展`HIElement`的类，并使用`@customElement` 装饰器对其进行装饰，并提供元素名称。

**实例: 一个基础 `HIElement` 定义**

```ts
import { HIElement, customElement } from '@hi-element';

@customElement('name-tag')
export class NameTag extends HIElement {

}
```

现在，您可以在HTML中的任何位置使用带有以下标记的`name-tag`元素：
**实例: 使用Web组件**

```html
<name-tag></name-tag>
```

:::important
Web组件名称必须包含`-`，以防止将来与内置元素发生冲突，并为来自不同库的组件命名。有关Web组件基础知识的更多信息[请参阅这组文章](https://developers.google.com/web/fundamentals/web-components)。
:::

:::note
HTML有一些特殊的标记，称为“自动关闭标记”。常见的示例包括`<input>`和`<img>`。然而，大多数HTML元素和**所有**web组件必须有一个明确的结束标记。
:::

我们已经准备好了一个基本的Web组件，但它做的不多。因此，让我们添加一个属性并使其呈现一些内容。
**实例: 将属性添加到 `HIElement`**

```ts
import { HIElement, customElement, attr } from '@hi-element';

@customElement('name-tag')
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';

  // optional method 
  greetingChanged() {
    ...
  }
}
```

要向HTML元素添加属性，请创建由`@attr`装饰器修饰的属性。以这种方式定义的所有属性都将自动向平台注册，以便可以通过浏览器的本机`setAttribute`API以及属性进行更新。您可以选择将命名约定*propertyName*已更改的方法添加到类中（例如`greeting`和`greetingChanged()`），并且每当属性更改时，无论是通过属性还是通过属性API更改，都会调用此方法。

:::note
所有用`@attr`装饰器的属性也是*可观察的*。请参见[可观测项和状态](./observables-and-state.zh)，了解可观测项如何实现高效渲染的信息。
:::

默认情况下，从`HIElement`扩展的任何内容都将自动附加`ShadowRoot`，以便启用封装渲染。
要查看它的实际效果，可以使用与上面相同的HTML，或使用以下内容更改默认的`greeting`：
**实例: 使用具有属性的Web组件**

```html
<name-tag greeting="Hola"></name-tag>
```

## 自定义属性
默认情况下，使用`@attr` 创建的任何属性都不会执行显式类型强制，除非它通过`setAttribute`API将其值反映到HTML DOM中。但是，您可以将DOM属性字符串值转换为任意类型，也可以控制用于向DOM反映属性值的`mode`。通过属性配置的`mode`属性可以使用三种模式：

* `reflect` - 未指定时使用的*默认*模式。这反映了DOM的属性更改。如果提供了`converter`，它将在调用`setAttribute`DOM API之前调用该转换器。
* `boolean` - 此模式使属性使用HTML布尔属性行为运行。当属性存在于DOM中或等于其自身名称时，该值将为true。当DOM中没有该属性时，该属性的值将为false。设置属性还将通过添加/删除属性来更新DOM。
* `fromView` - 此模式跳过将属性的值反射回HTML属性的过程，但通过`setAttribute`更改时会收到更新。

除了设置`mode`，还可以通过设置属性配置的`converter`属性来提供自定义的`ValueConverter`。转换器必须实现以下接口：
```ts
interface ValueConverter {
    toView(value: any): string;
    fromView(value: string): any;
}
```

其工作原理如下：

* 当DOM属性值更改时，将调用转换器的`fromView`方法，允许自定义代码将值强制为属性所期望的正确类型
* 当属性值更改时，还将调用转换器的`fromView`方法，以确保类型正确。在此之后，将确定`mode`。如果模式设置为`reflect`，则将调用转换器的`toView`方法，以允许在使用`setAttribute`写入属性之前格式化类型。
:::important
当`mode`设置为`boolean`时，会自动使用内置的`booleanConverter`来确保类型正确性，这样在这种常见情况下就不需要手动配置转换器。
:::

**实例: 默认模式下的属性，无需特殊转换**

```ts
import { HIElement, customElement, attr } from '@hi-element';

@customElement('name-tag')
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';
}
```

**实例: 具有布尔转换的布尔模式属性**

```ts
import { HIElement, customElement, attr } from '@hi-element';

@customElement('my-checkbox')
export class MyCheckbox extends HIElement {
  @attr({ mode: 'boolean' }) disabled: boolean = false;
}
```

**实例: 具有自定义转换的反射模式中的属性**

```ts
import { HIElement, customElement, attr, ValueConverter } from '@hi-element';

const numberConverter: ValueConverter = {
  toView(value: any): string {
    // convert numbers to strings
  },

  fromView(value: string): any {
    // convert strings to numbers
  }
};

@customElement('my-counter')
export class MyCounter extends HIElement {
  @attr({ converter: numberConverter }) count: number = 0;
}
```

## 元素生命周期

所有Web组件都支持一系列生命周期事件，您可以利用这些事件在特定时间点执行自定义代码`HIElement`自动实现其中几个回调，以启用其模板引擎的功能（如[声明模板](./declaring-templates.zh)中所述）。但是，您可以重写它们以提供自己的代码。下面是一个示例，说明在将元素插入DOM时如何执行自定义代码。

**实例: 利用自定义元素生命周期**

```ts
import { HIElement, customElement, attr } from '@hi-element';

@customElement('name-tag')
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';

  greetingChanged() {
    this.shadowRoot!.innerHTML = this.greeting;
  }

  connectedCallback() {
    super.connectedCallback();
    console.log('name-tag is now connected to the DOM');
  }
}
```

可用生命周期回调的完整列表如下：
| 回调函数 | 描述 |
| ------------- |-------------|
| constructor | 创建或升级元素时运行`HIElement`此时将附加阴影DOM。 |
| connectedCallback | 将元素插入DOM时运行。在第一次连接时，`HIElement`将HTML模板水合，连接模板绑定，并添加样式。 |
| disconnectedCallback | 从DOM中删除元素时运行`HIElement`此时将删除模板绑定并清理资源。 |
| attributeChangedCallback(attrName, oldVal, newVal) | 在元素的某个自定义属性更改时运行`HIElement`使用它将属性与其属性同步。当属性更新时，如果存在模板依赖关系，渲染更新也会排队。|
| adoptedCallback | 如果元素通过调用`adoptNode(...)`，从当前`document`移动到新的`document`，则运行API。|

## 在没有装饰师的情况下工作

上面的例子和我们整个文档中的例子都利用了TypeScript，尤其是该语言的decorators特性。装饰器是计划用于未来版本JavaScript的一项即将推出的功能，但其设计尚未完成。虽然在该功能的最终版本中，decorator用法的语法不太可能改变，但我们的一些社区成员在现阶段使用该功能时可能会感到不舒服。此外，由于decorator被转换成使用助手函数的代码（在TypeScript和Babel中），编译后的输出将大于等效的非decorator代码。

虽然在完全支持语言之前使用decorator会影响大小，但它们确实提供了最具声明性和可读性的API形式，我们建议在一般项目中使用它们。为了在声明可读性和大小之间取得平衡，我们建议将TypeScript与`"importHelpers": true`编译器选项结合使用。设置此选项后，TypeScript将导入在`tslib`包中发布的一组共享助手，而不是为每个文件中的装饰程序生成助手函数。

对于那些需要尽可能小的构建的元素，通过利用类上的静态 `definition`字段，可以在Vanilla JS中完全定义HI元素，而无需使用decorators。 `definition`字段只需要呈现与`@customElement`装饰器相同的配置。下面的示例显示了使用`definition`字段以及手动调用`define`元素：

```js
import { HIElement, html, css } from '@hi-element';

const template = html`...`;
const styles = css`...`;
const converter = { ... };

export class MyElement extends HIElement {
  static definition = {
    name: 'my-element',
    template,
    styles,
    attributes: [
      'value', // same attr/prop
      { attribute: 'some-attr', property: 'someAttr' }, // different attr/prop
      { property: 'count', converter } // derive attr; add converter
    ]
  };

  value = '';
  someAttr = '';
  count = 0;
}

HIElement.define(MyElement);
```

:::note
如果需要，`definition`也可以从类中分离出来，并直接传递到`define`调用中。下面是它的样子：`HIElement.define(MyElement, myDefinition);`
:::

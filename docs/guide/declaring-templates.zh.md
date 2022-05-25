

## 基础模板

虽然可以手动在Shadow DOM中创建和更新节点，但是比较繁琐，`HIElement`为最常见的渲染场景提供了一个简化的模板系统。要为元素创建HTML模板，请导入并使用“HTML”标记的模板帮助器，并将模板传递给`@customElement`装饰器。

下面是我们如何为`name-tag`组件添加一个模板，该组件呈现一些基本结构以及`greeting`：
**实例: 将模板添加到 `HIElement`**

```ts
import { HIElement, customElement, attr, html } from '@hi-element';

const template = html<NameTag>`
  <div class="header">
    <h3>${x => x.greeting.toUpperCase()}</h3>
    <h4>my name is</h4>
  </div>

  <div class="body">TODO: Name Here</div>

  <div class="footer"></div>
`;

@customElement({
  name: 'name-tag',
  template
})
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';
}
```

在上面的例子中有几个重要的细节，所以让我们逐一分解。

首先，我们使用[tagged template literal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)标记为后面的`html`字符串提供特殊处理，创建一个模板,返回一个`ViewTemplate`实例。

在模板中，我们提供*绑定*来声明模板的*动态部分*。这些绑定是用[arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)声明的。因为模板是插入的，所以arrow函数的输入将是您在`html`标记中声明的数据模型的实例。当`html`标记处理模板时，它会识别这些动态表达式，并建立一个优化的模型，能够进行高性能渲染和高效、增量的批量更新。

最后，我们使用一种新形式的`@customElement`装饰器将模板与自定义元素关联起来，它允许我们传递更多选项。在这个配置中，我们传递一个options对象，指定`name`和`template`。

有了这个元素，我们现在有了一个`name-tag`元素，它将把它的模板呈现到Shadow DOM中，并在name tag的`greeting`属性发生变化时自动更新`h3`内容。试试看！
### 类型化模板

模板可以*插入*到它们正在渲染的数据模型中。在TypeScript中，我们将类型作为标记的一部分提供：`html<NameTag>`。对于TypeScript 3.8或更高版本，可以将`ViewTemplate`作为类型导入：
```ts
import type { ViewTemplate } from '@hi-element';

const template: ViewTemplate<NameTag> = html`
  <div>${x => x.greeting}</div>
`;
```

## 理解绑定

我们已经了解了如何使用箭头函数来声明模板的动态部分。让我们再看几个例子，看看你能得到什么。

### 内容

要绑定元素的内容，只需在元素的开始和结束标记中提供表达式。它可以是元素的唯一内容，也可以与其他元素和文本交织在一起。

**实例: 基础文本内容**

```html
<h3>${x => x.greeting.toUpperCase()}</h3>
```

**实例: 插入文本内容**

```html
<h3>${x => x.greeting}, my name is ${x => x.name}.</h3>
```

**实例: 异构内容**

```html
<h3>
  ${x => x.greeting}, my name is
  <span class="name">${x => x.name}</span>.
</h3>
```

:::note
出于安全原因，动态内容通过 `textContent` HTML属性设置。你不能这样设置HTML内容。有关设置HTML的显式选择加入机制，请参见下文。
:::

### 属性

还可以使用表达式设置HTML元素的属性值。只需将表达式放在HTML属性的值所在的位置。然后，模板引擎将使用 `setAttribute(...)`，无论何时需要更新。此外，有些属性被称为[boolean attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes#Boolean_Attributes)（例如，必填、只读、禁用）。这些属性的行为与普通属性不同，需要特殊的值处理。如果在属性名前面加上`?`，模板引擎将为您处理这个问题。

**实例: 基本属性值**

```html
<a href="${x => x.aboutLink}">About</a>
```

**实例: 插入属性值**

```html
<a href="products/${x => x.id}">
  ${x => x.name}
</a>
```

```html
<li class="list-item ${x => x.type}">
  ...
</li>
```

:::tip
当绑定到`class`时，底层引擎不会重写通过其他机制添加到元素中的类。它只添加和删除直接由绑定产生的类。这种“默认安全”的行为确实会带来轻微的性能代价。要退出此功能并通过总是覆盖所有类来挤出每一盎司的性能，请在`className`属性上使用属性绑定（见下文）。例如 `:className="list-item ${x => x.type}"`。
:::

```html
<span style="text-decoration: ${x => x.done ? 'line-through' : ''}">
  ${x => x.description}
</span>
```

**实例: ARIA 属性**

```html
<div role="progressbar"
     aria-valuenow="${x => x.value}"
     aria-valuemin="${x => x.min}"
     aria-valuemax="${x => x.max}">
</div>
```

**实例: Boolean 属性**

```html
<button type="submit" ?disabled="${x => !x.enabled}">Submit</button>
```

**实例: 空值**

在某些情况下，属性在使用时需要有值，但如果未使用则不存在。这些属性与布尔属性不同，因为它们的存在单独触发了它们的效果。在这种情况下，可能会提供一个空值（`null`或`undefined`），以便该属性在该条件下不存在。

```html
<div aria-hidden="${x => x.isViewable ? 'true' : null}"></div>
```

### 特性
特性也可以直接在HTML元素上设置。为此，请在特性名称前面加上 `:`以指示属性绑定。然后，模板引擎将使用表达式指定元素的属性值。
**实例: 基本特性值**

```html
<my-element :myCustomProperty="${x => x.mySpecialData}">
  ...
</my-element>
```

**实例: Inner HTML**

```html
<div :innerHTML="${x => x.someDangerousHTMLContent}"></div>
```

:::warning
避免需要直接设置HTML的情况，尤其是当内容来自外部源时。如果必须这样做，则应始终清理HTML。

完成HTML清理的最佳方法是配置[受信任类型策略](https://w3c.github.io/webappsec-trusted-types/dist/spec/)使用HIElement的模板引擎。HIElement确保所有HTML字符串都通过配置的策略。此外，通过利用平台的trusted types功能，您可以通过CSP头获得策略的本机强制执行。下面是一个如何配置自定义策略来清理HTML的示例：

```ts
import { DOM } from '@hi-element';

const myPolicy = trustedTypes.createPolicy('my-policy', {
  createHTML(html) {
    // TODO: invoke a sanitization library on the html before returning it
    return html;
  }
});

DOM.setHTMLPolicy(myPolicy);
```
出于安全原因，HTML策略只能设置一次。因此，它应该由应用程序开发人员而不是组件作者设置，并且应该在应用程序的启动序列中立即设置。
:::

### Events

除了呈现内容、属性和属性外，您通常还需要添加事件侦听器，并在事件触发时执行代码。为此，请在事件名称前面加上 `@`，并提供在该事件触发时要调用的表达式。在事件绑定中，您还可以访问一个特殊的*context*参数，从中可以访问`event` 对象。

**实例: Basic Events**

```html
<button @click="${x => x.remove()}">Remove</button>
```

**实例: Accessing Event Details**

```html
<input type="text"
       :value="${x => x.description}"
       @input="${(x, c) => x.handleDescriptionChange(c.event)}">
```

:::important
在上述两个示例中，在执行事件处理程序后，默认情况下会对事件对象调用`preventDefault()`。您可以从处理程序返回`true` 以选择退出此行为。
:::

第二个示例演示了模板引擎的一个重要特性：它只支持*单向数据流*（model=>view）。它不支持*双向数据绑定*（模型<=>视图）。如上所示，将数据从视图推回到模型应该通过调用模型API的显式事件来处理。

## 模板和元素生命周期

在自定义元素生命周期的`connectedCallback`阶段，`HIElement`创建模板并绑定结果视图。模板的创建仅在元素第一次连接时发生，而绑定则在元素每次连接时发生（为了对称，在`disconnectedCallback`期间解除绑定）。

:::note
在未来，我们正在计划新的优化，这将使我们能够安全地确定何时不需要解除绑定/重新绑定某些视图。
:::

在大多数情况下，`HIElement`呈现的模板由自定义元素配置的`template`属性决定。但是，也可以在名为`resolveTemplate()`的自定义元素类上实现一个方法，该方法返回一个模板实例。如果存在此方法，将在`connectedCallback`期间调用它以获取要使用的模板。这允许元素作者根据连接时元素的状态动态选择完全不同的模板。

除了在`connectedCallback`期间进行动态模板选择之外，`HIElement`的`$hiController`属性还可以通过将控制器的`template`属性设置为任何有效模板，随时动态更改模板。

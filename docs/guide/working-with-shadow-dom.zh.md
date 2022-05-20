

到目前为止，我们已经了解了如何定义元素，如何定义这些元素上的属性，以及如何通过声明性模板控制元素呈现。然而，我们还没有看到自定义元素如何与标准HTML或其他自定义元素组合在一起。

## 默认插槽

为了启用组合，`HIElement`利用了影子DOM标准。之前，我们已经看到了`HIElement`如何自动附加`ShadowRoot`，当您的元素声明模板时，它会将该模板呈现到阴影DOM中。要启用元素组合，我们需要做的就是在模板中使用标准的`<slot>`元素。

让我们回到原始的`name-tag`元素实例 看看我们如何用`slot`来组成这个人的名字。
**实例: 在 `HIElement` 中使用插槽**

```ts
import { HIElement, customElement, attr, html } from '@hi-element';

const template = html<NameTag>`
  <div class="header">
    <h3>${x => x.greeting.toUpperCase()}</h3>
    <h4>my name is</h4>
  </div>

  <div class="body">
    <slot></slot>
  </div>

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

在body`div`中，我们放置了一个`slot`元素。这被称为组件的“默认槽”，因为默认情况下，放置在元素的开始标记和结束标记之间的所有内容都将在此位置*呈现*。

为了明确这一点，让我们看看`name-tag`元素如何与内容一起使用，然后看看浏览器如何合成最终渲染的输出。

**实例: 将`name-tag`与默认插槽一起使用**

```html
<name-tag>John Doe<name-tag>
```

**实例: 具有默认插槽的`name-tag`的渲染输出**

```html
<name-tag>
  #shadow-root
    <div class="header">
      <h3>HELLO</h3>
      <h4>my name is</h4>
    </div>

    <div class="body">
      <slot>John Doe</slot>
    </div>

    <div class="footer"></div>
  #shadow-root

  John Doe
</name-tag>
```

文本"John Doe"存在于"Light DOM"中，但它被*投影*到“Shadow DOM”中的`slot`位置。

:::note
如果您发现术语 "Light DOM" 和"Shadow DOM"不直观，那么您并不孤单。另一种认为 "Light DOM" 的方式是"Semantic DOM"。它表示您的语义内容模型，而不需要考虑渲染。另一种认为"Shadow DOM"的方法是"Render DOM"。它表示元素的呈现方式，与内容或语义无关。
:::

有了可用的插槽，我们现在可以解锁HTML的完整组合模型，以便在我们自己的元素中使用。然而，插槽还可以做更多的事情。

## 命名插槽

在实例 上面，我们使用一个`slot`元素来呈现放置在`name-tag`的开始标记和结束标记之间的*所有*内容。然而，我们并不局限于只有一个默认插槽。我们还可以有*命名槽*，用于声明我们可以将内容呈现到的其他位置。为了演示这一点，让我们在`name-tag`的模板中添加一个命名槽，在这里我们可以显示此人的头像。

**实例: `name-tag`具有命名插槽**

```ts
import { HIElement, customElement, attr, html } from '@hi-element';

const template = html<NameTag>`
  <div class="header">
    <slot name="avatar"></slot>
    <h3>${x => x.greeting.toUpperCase()}</h3>
    <h4>my name is</h4>
  </div>

  <div class="body">
    <slot></slot>
  </div>

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

**实例: 将`name-tag`与命名槽一起使用**

```html
<name-tag>
  John Doe
  <img slot="avatar" src="...">
</name-tag>
```

**实例: 具有命名槽的`name-tag`的渲染输出**

```html
<name-tag>
  #shadow-root
    <div class="header">
      <slot name="avatar">
        <img slot="avatar" src="...">
      </slot>
      <h3>HELLO</h3>
      <h4>my name is</h4>
    </div>

    <div class="body">
      <slot>John Doe</slot>
    </div>

    <div class="footer"></div>
  #shadow-root

  John Doe
  <img slot="avatar" src="...">
</name-tag>
```
如果一个元素声明了命名的slot，那么它的内容就可以利用`slot`*属性*来指示它要在哪里开槽。没有`slot`属性的任何内容都将投影到默认插槽。任何具有`slot`属性的内容都将投影到其请求的插槽中。
下面是一些有关插槽的快速注释：

* 您可以将任意数量的内容节点投影到同一插槽中。
* 只能在包含元素的直接内容上放置`slot`属性。
  ```html
  <name-tag>
    <div> <!--Projected to default slot-->
      <img slot="avatar"> <!--Slot Ignored!-->
    </div>
    <img slot="avatar"> <!--Projected to "avatar" slot-->
  </name-tag>
  ```
* 如果 Light DOM 中有直接内容元素，并且没有相应的阴影DOM插槽，则不会渲染该元素。 
* 当投影到插槽时，保持顺序。因此，如果有两个元素投影到同一个插槽中，则它们将在插槽中按照在灯光DOM中显示的相同顺序进行渲染。
* 如果slot元素是模板中使用的另一个自定义元素的直接子元素，`slot`元素也可以具有`slot`属性。在这种情况下，这意味着将投影到该槽中的任何内容都将重新投影到包含元素的槽中。
  ```html
  <div class="uber-name-tag-template">
    ...
    <name-tag>
      <slot name="uber-avatar" slot="avatar">
        <!--uber-name-tag's "uber-avatar" content gets projected into name-tag's "avatar" slot-->
      </slot>
      <slot>
        <!--uber-name-tag's default content gets projected into name-tag's default slot-->
      </slot>
    </name-tag>
    ...
  </div>
  ```
* 您不需要为每个声明的插槽提供内容。在上面实例, 仅仅因为`name-tag`有一个"avatar"槽，并不意味着我们必须为该槽提供内容。如果没有为插槽提供任何内容，则不会在该位置呈现任何内容，除非插槽声明了备用内容。。。

## 备用内容

在元素中使用插槽有多种方案。到目前为止，我们已经展示了如何使用插槽进行内容投影。然而，另一个主要的用例是允许使用元素的软件替换元素渲染的各个部分。要启用此功能，您可以为任何插槽提供*备用内容*。如果元素使用者没有为该插槽提供内容，则将呈现该内容，但如果他们提供了内容，则他们自己的内容将覆盖备用内容。

**实例: 备用插槽内容**

```html
<div class="my-slider-template">
  <slot name="thumb">
    <span class="thumb"></span>
  </slot>
</div>
```

在实例 上面，`my-slider`自定义元素的作者为滑块的"thumb"提供了默认HTML，确保该元素始终能够正确呈现和运行。然而，这种设计为组件的消费者留下了一个选择，通过简单地提供HTML和分配适当的插槽名称，用他们自己的HTML替换拇指。

## 插槽API

除了到目前为止描述的使用插槽的声明性方法外，浏览器还提供了许多特定于插槽的API，您可以直接在JavaScript代码中使用。下面是对您可用内容的总结。

| API | 描述 |
| ------------- |-------------|
| `slotchange` | 通过在`slot`元素上为`slotchange`事件添加事件侦听器，您可以在特定插槽的时隙节点发生更改时随时收到通知。 |
| `assignedNodes()` | `slot`元素提供了一个`assignedNodes()`方法，可以调用该方法来获取特定slot当前呈现的所有节点的列表。如果还希望看到回退内容节点，可以传递带有`{ flatten: true }`的options对象。 |
| `assignedSlot` | `assignedSlot`属性存在于已投影到插槽的任何元素上，以便您可以确定其投影位置。 |

:::tip
请记住，您可以使用模板系统的`slotchange`事件`<slot @slotchange=${...}></slot>`。您还可以使用`ref` 指令获取对任何插槽的引用，这样可以很容易地调用`assignedNodes()`之类的API，或者手动添加/删除事件侦听器。
:::

## Events

来自阴影DOM中的事件看起来就像来自自定义元素本身一样。为了使事件从阴影DOM中传播，必须使用`composed: true`选项对其进行调度。以下是组成的内置事件列表：

* `blur`, `focus`, `focusin`, `focusout`
* `click`, `dblclick`, `mousedown`, `mouseenter`, `mousemove`, etc.
* `wheel`
* `beforeinput`, `input`
* `keydown`, `keyup`
* `compositionstart`, `compositionupdate`, `compositionend`
* `dragstart`, `drag`, `dragend`, `drop`, etc.

以下是一些不构成且仅在阴影DOM本身中可见的事件：

* `mouseenter`, `mouseleave`
* `load`, `unload`, `abort`, `error`
* `select`
* `slotchange`

要从事件对象获取完全组合的事件路径，请对事件本身调用`composedPath()`方法。这将返回一个表示事件冒泡路径的目标数组。如果自定义元素使用`closed`阴影DOM模式，则阴影DOM中的目标将不会出现在组合路径中，并且它将显示为自定义元素本身是第一个目标。

### 自定义事件

在各种情况下，自定义元素可能适合发布自己的特定于元素的事件。为此，可以在`HIElement`上使用`$emit`帮助程序。这是一种方便的方法，可以创建`CustomEvent`的实例，并在`HIElement`上使用`dispatchEvent`API和`bubbles: true`和`composed: true`选项。它还确保仅当自定义元素完全连接到DOM时才发出事件。这是一个实例:

**实例: 自定义事件调度**

```ts
customElement('my-input')
export class MyInput extends HIElement {
  @attr value: string = '';

  valueChanged() {
    this.$emit('change', this.value);
  }
}
```

:::tip
发出自定义事件时，请确保事件名称始终为小写，以便Web组件与通过DOM绑定模式附加事件的各种前端框架保持兼容（DOM不区分大小写）。
:::

## Shadow DOM 配置

在所有实例到目前为止，我们已经看到`HIElement`自动为元素创建阴影根，并以`open`模式附加它。但是，如果需要，可以指定`closed`模式或将元素渲染到灯光DOM中。这些选择可以通过在`@customElement`装饰器中使用`shadowOptions`设置来实现。

**实例: 关闭模式下的 Shadow DOM**

```ts
@customElement({
  name: 'name-tag',
  template,
  shadowOptions: { mode: 'closed' }
})
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';
}
```

:::tip
避免使用`closed`模式，因为它会影响事件传播并降低自定义元素的可检查性。
:::

**实例: 渲染到 Light DOM**

```ts
@customElement({
  name: 'name-tag',
  template,
  shadowOptions: null
})
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';
}
```

:::important
如果选择渲染到灯光DOM，则将无法合成内容、使用插槽或利用封装样式。对于可重用组件，不建议使用轻型DOM渲染。作为小型应用程序的根组件，它的使用可能有限。
:::

除了阴影DOM模式之外，`shadowOptions`还公开了可以通过标准的`attachShadow`API设置的所有选项。这意味着您还可以使用它来指定新选项，例如`delegatesFocus: true`。您只需要指定与上述默认值不同的选项。

## Shadow DOM 和元素生命周期

`HIElement`是在构造函数期间为元素附加阴影DOM的。然后，`shadowRoot`可以直接作为自定义元素的属性使用，假设该元素使用`open`模式。
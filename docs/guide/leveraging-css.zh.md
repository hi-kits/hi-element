

## 基础样式

我们组件故事的最后一部分是CSS。与HTML类似，`HIElement`提供了一个带有`css`标记的模板助手，允许创建和重用css。让我们为`name-tag`组件添加一些CSS。
**实例: 将CSS添加到 `HIElement`**

```ts
import { html, css, customElement, attr, HIElement } from "@hi-element";

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

const styles = css`
  :host {
    display: inline-block;
    contain: content;
    color: white;
    background: var(--fill-color);
    border-radius: var(--border-radius);
    min-width: 325px;
    text-align: center;
    box-shadow: 0 0 calc(var(--depth) * 1px) rgba(0,0,0,.5);
  }

  :host([hidden]) { 
    display: none;
  }

  .header {
    margin: 16px 0;
    position: relative;
  }

  h3 {
    font-weight: bold;
    font-family: 'Source Sans Pro';
    letter-spacing: 4px;
    font-size: 32px;
    margin: 0;
    padding: 0;
  }

  h4 {
    font-family: sans-serif;
    font-size: 18px;
    margin: 0;
    padding: 0;
  }

  .body {
    background: white;
    color: black;
    padding: 32px 8px;
    font-size: 42px;
    font-family: cursive;
  }

  .footer {
    height: 16px;
    background: var(--fill-color);
    border-radius: 0 0 var(--border-radius) var(--border-radius);
  }
`;

@customElement({
  name: 'name-tag',
  template,
  styles
})
export class NameTag extends HIElement {
  @attr greeting: string = 'Hello';
}
```

使用`css`助手，我们可以创建 `ElementStyles`。我们通过decorator的`styles`选项使用元素来配置它。在内部，`HIElement`将利用[可构造样式表对象](https://wicg.github.io/construct-stylesheets/)以及`ShadowRoot#adoptedStyleSheets`来高效地跨组件重用CSS。这意味着，即使我们有1k个`name-tag`组件实例，它们都将共享一个关联样式的实例，从而减少内存分配并提高性能。由于样式与`ShadowRoot`关联，因此它们也将被封装。这可以确保您的样式不会影响其他元素，其他元素样式也不会影响您的元素。

:::note
我们使用了[CSS自定义属性](https://developer.mozilla.org/en-US/docs/Web/CSS/--*)在我们的CSS以及[CSS Calc](https://developer.mozilla.org/en-US/docs/Web/CSS/calc)为了使我们的组件能够由消费者以基本的方式进行设计。此外，考虑添加[CSS阴影部分](https://developer.mozilla.org/en-US/docs/Web/CSS/::part)以实现更强大的自定义。
:::

## 撰写样式表

`ElementStyles`的一个很好的特性是它可以与其他样式组合。假设我们有一个CSS规范化，我们想在`name-tag`组件中使用它。我们可以这样将其组合到我们的风格中：
**实例: 撰写样式表**

```ts
import { normalize } from './normalize';

const styles = css`
  ${normalize}
  :host {
    display: inline-block;
    contain: content;
    color: white;
    background: var(--fill-color);
    border-radius: var(--border-radius);
    min-width: 325px;
    text-align: center;
    box-shadow: 0 0 calc(var(--depth) * 1px) rgba(0,0,0,.5);
  }

  ...
`;
```

`css`助手了解`normalize`是`ElementStyles`，而不是简单地连接CSS字符串，并且能够重复使用与使用`normalize`的任何其他组件相同的可构造样式表实例。
:::note
您还可以传递CSS`string`或[CSSStyleSheet](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet)直接引用到元素定义，甚至是`string`, `CSSStyleSheet`或`ElementStyles`的混合数组。
:::

### 部分 CSS
有时，您可能希望创建可重用*部分*CSS块，其中抽象本身不是有效的CSS，例如CSS属性组或复杂值。为此，可以使用`cssPartial`标记的模板文本：

```ts
import { css, cssPartial } from "@hi-element";

const partial = cssPartial`color: red;`;
const styles = css`:host{ ${partial} }`;
```

`cssPartial` 还可以组合 `css`可以组合的所有结构，提供更大的灵活性。

## CSS指令
`CSSDirective`允许通过`ElementStyles`将行为绑定到元素。要创建`CSSDirective`，请从`@hi-element`导入并扩展`CSSDirective`：

```ts
import { CSSDirective }  from "@hi-element"

class RandomWidth extends CSSDirective {}
```

CSS指令有两个关键方法，您可以利用它们通过CSS添加动态行为：
### 创建CSS
`CSSDirective` 具有`createCSS()`方法，该方法返回要插入到 `ElementStyles`:

```ts
class RandomWidth extends CSSDirective {
  createCSS() {
    return "width: var(--random-width);"
  }
}
```

### 创建行为
`createBehavior()`方法可用于创建绑定到使用`CSSDirective`元素的`Behavior`：

```ts
class RandomWidth extends CSSDirective {
  private property = "--random-width";
  createCSS() {
    return `width: var(${this.property});`
  }

  createBehavior() {
    return {
      bind(el) {
        el.style.setProperty(this.property, Math.random() * 100)
      }
      unbind(el) {
        el.style.removeProperty(this.property);
      }
    }
  }
}
```

### 元素样式的用法
可以在`ElementStyles`中使用`CSSDirective`，其中来自`createCSS()`的CSS字符串将插入样式表中，并且从`createBehavior()`返回的行为将使用样式表绑定到元素：
```ts
const styles = css`:host {${new RandomWidth()}}`;
```

## Shadow DOM 样式

您可能已经注意到我们在`name-tag`样式中使用的`:host`选择器。此选择器允许我们将样式直接应用于自定义元素。以下是始终为您的主机元素配置时需要考虑的一些事项：
* **display** - 默认情况下，自定义元素的`display`属性为`inline`，因此请考虑是否希望元素的默认显示行为有所不同。
* **contain** - 如果元素的绘制包含在其边界内，请考虑将CSS`contain`属性设置为 `content`。正确的包含模型可以积极地影响元素的性能。[参见MDN文档](https://developer.mozilla.org/en-US/docs/web/css/contain)有关`contain`的各种值及其作用的更多信息。 
* **hidden** - 除了默认的`display`样式之外，还要添加对`hidden`的支持，以便默认的`display`不会覆盖此状态。这可以通过`:host([hidden]) { display: none }`来完成。

## 插槽内容

除了提供宿主样式外，还可以为开槽的内容提供默认样式。例如，如果我们想设置所有插入到`name-tag`中的`img`元素的样式，我们可以这样做：
**实例: 插槽样式**

```ts
const styles = css`
  ...

  ::slotted(img) {
    border-radius: 50%;
    height: 64px;
    width: 64px;
    box-shadow: 0 0 calc(var(--depth) / 2px) rgba(0,0,0,.5);
    position: absolute;
    left: 16px;
    top: -4px;
  }

  ...
`;
```

:::note
元素可以覆盖插槽样式和主体样式。可以将这些视为您提供的*默认*样式，以便您的元素能够在开箱即用的情况下正确地显示和运行。
:::

## 样式和元素生命周期

正是在自定义元素生命周期的`connectedCallback`阶段，`HIElement`添加元素的样式。这些样式仅在第一次连接图元时添加。
在大多数情况下，`HIElement`呈现的样式由自定义元素配置的`styles` 属性决定。但是，也可以在名为`resolveStyles()`的自定义元素类上实现一个方法，该方法返回一个`ElementStyles` 实例。如果存在此方法，则将在`connectedCallback`期间调用该方法以获取要使用的样式。这允许元素作者根据连接时元素的状态动态选择完全不同的样式。

除了在`connectedCallback`期间进行动态样式选择之外，`HIElement`的`$hiController`属性还允许通过将控制器的`styles`属性设置为任何有效样式，随时动态更改样式。
### 隐藏未定义的元素

尚未[升级的自定义元素](https://developers.google.com/web/fundamentals/web-components/customelements#upgrades)如果没有附加样式，浏览器仍可以渲染这些样式，但它们可能看起来不像应该的样子。为避免*flash of un-styled content* (FOUC)，如果自定义元素尚未定义，则可以隐藏它们*：

```CSS
:not(:defined) {
  visibility: hidden;
}
```

:::important
The consuming application must apply this, as the components themselves do not.
:::

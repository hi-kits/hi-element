

除了用表达式声明模板的动态部分外，您还可以访问几个强大的*指令*，这些指令有助于常见场景。
## 结构指令

结构指令通过根据元素的状态添加和删除节点来更改DOM本身的形状。

###  `when` 指令

`when` 指令允许您有条件地呈现HTML块。当您将表达式提供给`when` 时，当表达式的计算结果为`true`时，它会将子模板渲染到DOM中，当它的计算结果为`false`时，它会删除子模板（或者如果它从来都不是`true`，则会完全跳过渲染）。
**实例: 条件渲染**

```ts
import { HIElement, customElement, observable, html, when } from '@hi-element';

const template = html<MyApp>`
  <h1>My App</h1>

  ${when(x => !x.ready, html<MyApp>`
    Loading...
  `)}
`;

@customElement({
  name: 'my-app',
  template
})
export class MyApp extends HIElement {
  @observable ready: boolean = false;
  @observable data: any = null;

  connectedCallback() {
    super.connectedCallback();
    this.loadData();
  }

  async loadData() {
    const response = await fetch('some/resource');
    const data = await response.json();
    
    this.data = data;
    this.ready = true;
  }
}
```

:::note
`@observable`装饰器创建了一个属性，模板系统可以监视该属性的更改。它与`@attr`类似，但该属性不是作为元素本身的HTML属性显示的。虽然`@attr`只能用于`HIElement`，但`@observable`可以用于任何类。
:::

除了提供有条件渲染的模板外，还可以提供计算结果为模板的表达式。这使您能够动态更改有条件渲染的内容。
**实例: 使用动态模板的条件渲染**

```ts
import { HIElement, customElement, observable, html, when } from '@hi-element';

const template = html<MyApp>`
  <h1>My App</h1>

  ${when(x => x.ready, x => x.dataTemplate)}
`;
```

### `repeat` 指令

要呈现数据列表，请使用`repeat`指令，提供要呈现的列表和用于呈现每个项目的模板。

**实例: 列表渲染**

```ts
import { HIElement, customElement, observable, html, repeat } from '@hi-element';

const template = html<FriendList>`
  <h1>Friends</h1>

  <form @submit=${x => x.addFriend()}>
    <input type="text" :value=${x => x.name} @input=${(x, c) => x.handleNameInput(c.event)}>
    <button type="submit">Add Friend</button>
  </form>
  <ul>
    ${repeat(x => x.friends, html<string>`
      <li>${x => x}</li>
    `)}
  </ul>
`;

@customElement({
  name: 'friend-list',
  template
})
export class FriendList extends HIElement {
  @observable friends: string[] = [];
  @observable name: string = '';

  addFriend() {
    if (!this.name) {
      return;
    }

    this.friends.push(this.name);
    this.name = '';
  }

  handleNameInput(event: Event) {
    this.name = (event.target! as HTMLInputElement).value;
  }
}
```

与事件处理程序类似，在`repeat`块中，您可以访问特殊的上下文对象。以下是上下文中可用的属性列表：

* `event` - 事件处理程序中的事件对象。
* `parent` - 位于`repeat`块内时的父视图模型。
* `parentContext` - 位于`repeat`块内时的父`ExecutionContext`。当重复被嵌套并且最内部的重复需要访问根视图模型时，这非常有用。
* `index` - 在`repeat`块内时当前项的索引(opt-in)。
* `length` - 在`repeat`块内的数组长度(opt-in)。
* `isEven` - 如果当前项的索引在`repeat`块内为偶数，则为True (opt-in).
* `isOdd` - 如果当前项的索引在`repeat`块内为奇数，则为True (opt-in).
* `isFirst` - 如果当前项位于`repeat`块内的数组中的第一个，则为True (opt-in).
* `isInMiddle` - 如果当前项位于`repeat`块内数组的中间位置，则为True (opt-in).
* `isLast` - 如果当前项是`repeat`块内数组中的最后一项，则为True (opt-in).

某些上下文属性是选择性加入的，因为更新它们的成本更高。因此，出于性能原因，它们在默认情况下不可用。要选择定位属性，请将选项传递给repeat指令，设置为`positioning: true`。例如，我们将如何在上面的朋友模板中使用`index`：

**实例: 使用项目索引列出呈现**

```html
<ul>
  ${repeat(x => x.friends, html<string>`
    <li>${(x, c) => c.index} ${x => x}</li>
  `, { positioning: true })}
</ul>
```
repeat指令是否重新使用项目视图可以通过`recycle`选项设置进行控制。当默认值为`recycle: true`时，repeat指令可能会重用视图，而不是从模板创建新视图。当`recycle: false`时`，以前使用的视图将始终被丢弃，并且每个项目将始终被分配一个新视图。在某些情况下，重新对齐以前使用的视图可能会提高性能，但也可能与以前显示的项目相比“脏”。

**实例: 无视图循环的列表渲染**

```html
<ul>
  ${repeat(x => x.friends, html<string>`
    <li>${(x, c) => c.index} ${x => x}</li>
  `, { positioning: true, recycle: false })}
</ul>
```

除了提供用于呈现项目的模板外，还可以提供计算结果为模板的表达式。这使您能够动态更改用于渲染项目的内容。每个项目仍将使用相同的模板进行渲染，但您可以使用下面“合成模板”中的技术根据项目本身渲染不同的模板。

### 组合模板

当在另一个模板中使用 `html`标记帮助器返回的`ViewTemplate`时，它具有特殊的处理。这样做是为了您可以创建模板并将其组合到其他模板中。

**实例: 组合模板**

```ts
import { HIElement, customElement, observable, html, repeat, when } from '@hi-element';

interface Named {
  name: string;
}

class Person {
  @observable name: string;

  constructor(name: string) {
    this.name = name;
  }
}

const nameTemplate = html<Named>`
  <span class="name">${x => x.name}</span>
`;

const template = html<FriendList>`
  <h1>Friends</h1>

  <form @submit=${x => x.addFriend()}>
    <input type="text" :value=${x => x.name} @input=${(x, c) => x.handleNameInput(c.event)}>

    ${when(x => x.name, html`
      <div>Next Name: ${nameTemplate}</div>
    `)}
    
    <div class="button-bar">
      <button type="submit">Add Friend</button>
    </div>
  </form>
  <ul>
    ${repeat(x => x.friends, html`
      <li>${nameTemplate}</li>
    `)}
  </ul>
`;

@customElement({
  name: 'friend-list',
  template
})
export class FriendList extends HIElement {
  @observable friends: Person[] = [];
  @observable name: string = '';

  addFriend() {
    if (!this.name) {
      return;
    }

    this.friends.push(new Person(this.name));
    this.name = '';
  }

  handleNameInput(event: Event) {
    this.name = (event.target! as HTMLInputElement).value;
  }
}
```

在上面的示例中，我们创建了一个独立的`nameTemplate`，然后在两个不同的地方使用它。首先在`when`模板内，然后在`repeat`模板内。

但内容组合实际上比这更强大，因为您不局限于模板的“静态组合”。还可以提供任何返回模板的表达式。因此，当表达式的`@observable`依赖项更改时，您可以动态更改为渲染选择的模板。如果不想渲染任何内容，也可以通过返回`null`或`undefined`来处理。以下是一些您可以使用内容组合的示例：

**实例: 动态合成**

```ts
const defaultTemplate = html`...`;
const templatesByType = {
  foo: html`...`,
  bar: html`...`,
  baz: html`...`
};

const template = html<MyElement>`
  <div>${x => x.selectTemplate()}</div>
`;

@customElement({
  name: 'my-element',
  template
})
export class MyElement extends HIElement {
  @observable data;

  selectTemplate() {
    return templatesByType[this.data.type] || defaultTemplate;
  }
}
```

**实例: 替代模板**

```ts
const myCustomTemplate = html`...`

@customElement({
  name: 'my-derived-element',
  template
})
export class MyDerivedElement extends MyElement {
  selectTemplate() {
    return myCustomTemplate;
  }
}
```

**实例: Complex Conditional**

```ts
const dataTemplate = html`...`;
const loadingTemplate = html`...`;

const template = html<MyElement>`
  <div>
    ${x => {
      if (x.ready) {
        return dataTemplate;
      }

      // Any logic can go here to determine which template to use.
      // Which template to use will be re-evaluated whenever @observable
      // properties from this method implementation change.

      return loadingTemplate;
    }}
  </div>
`;

@customElement({
  name: 'my-element',
  template
})
export class MyElement extends HIElement {
  @observable ready: boolean = false;
  @observable data: any = null;

  connectedCallback() {
    super.connectedCallback();
    this.loadData();
  }

  async loadData() {
    const response = await fetch('some/resource');
    const data = await response.json();
    
    this.data = data;
    this.ready = true;
  }
}
```

**实例: 每项列表类型**

```ts
const defaultTemplate = html`...`;
const templatesByType = {
  foo: html`...`,
  bar: html`...`,
  baz: html`...`
};

const template = html<MyElement>`
  <ul>
    ${repeat(x => x.items, html`
      <li>
        ${(x, c) => c.parent.selectTemplate(x)}
      </li>
    `)}
  </ul>
`;

@customElement({
  name: 'my-element',
  template
})
export class MyElement extends HIElement {
  @observable items: any[] = [];

  selectTemplate(item) {
    return templatesByType[item.type] || defaultTemplate;
  }
}
```

**实例: 自定义渲染替代**

```ts
const defaultTemplate = html`...`;
const template = html<MyElement>`
  <div>${x => x.selectTemplate()}</div>
`;

@customElement({
  name: 'my-element',
  template
})
export class MyElement extends HIElement {
  selectTemplate() {
    return defaultTemplate;
  }
}

export class MyCustomTemplate implements SyntheticViewTemplate {
  create(): SyntheticView {
    // construct your own implementation of SyntheticView
    return customView;
  }
}

const customTemplate = new MyCustomTemplate();

@customElement({
  name: 'my-derived-element',
  template
})
export class MyDerivedElement extends MyElement {
  selectTemplate() {
    return customTemplate;
  }
}
```

:::important
合成模板时，将合成的模板提取到外部变量。如果在方法、属性或表达式中内联定义模板，则每次调用时，都会创建模板的新实例，而不是重用模板。这将导致不必要的性能成本。
:::

**实例: `when` 指令**

现在我们已经解释了内容合成的工作原理，您可能会发现有趣的是，`when`实际上只是核心合成系统上的“语法糖”。让我们看看`when`本身的实现，看看它是如何工作的：
```ts
export function when(condition, templateOrTemplateExpression) {
    const getTemplate = typeof templateOrTemplateExpression === "function"
      ? templateOrTemplateExpression
      : () => templateOrTemplateExpression;

    return (source, context) => 
      condition(source, context) 
        ? getTemplate(source, context) 
        : null;
}
```
正如您所看到的，`when`所做的就是编写一个新函数来检查您的条件。如果为`true`，则调用模板提供程序函数；如果为`false`，则返回`null`，表示不应呈现任何内容。
## 引用指令

引用指令允许您在各种场景中轻松获取对DOM节点的引用。

###  `ref` 指令

有时，您需要从模板直接引用单个DOM节点。这可能是因为您希望控制`video` 元素的播放、使用`canvas`元素的绘图上下文或将元素传递给第三方库。不管是什么原因，都可以使用`ref`指令获取对DOM节点的引用。
**实例: 引用元素**

```ts
import { HIElement, customElement, attr, html, ref } from '@hi-element';

const template = html<MP4Player>`
  <video ${ref('video')}>
    <source src=${x => x.src} type="video/mp4">
  </video>
`;

@customElement({
  name: 'mp4-player',
  template
})
export class MP4Player extends HIElement {
  @attr src: string;
  video: HTMLVideoElement;

  connectedCallback() {
    super.connectedCallback();
    this.video.play();
  }
}
```

将`ref`指令放在要引用的元素上，并为其提供一个属性名称，以便为其分配引用。一旦`connectedCallback`生命周期事件运行，您的属性将设置为引用，可以使用。
:::tip
如果为HTML模板提供类型，TypeScript将对提供的属性名称进行类型检查，以确保它确实存在于元素中。
:::

### `children` 指令

除了使用`ref`引用单个DOM节点外，还可以使用`children`获取对特定元素的所有子节点的引用。

**实例: 引用子节点**

```ts
import { HIElement, customElement, html, children, repeat } from '@hi-element';

const template = html<FriendList>`
  <ul ${children('listItems')}>
    ${repeat(x => x.friends, html<string>`
      <li>${x => x}</li>
    `)}
  </ul>
`;

@customElement({
  name: 'friend-list',
  template
})
export class FriendList extends HIElement {
  @observable listItems: Node[];
  @observable friends: string[] = [];

  connectedCallback() {
    super.connectedCallback();
    console.log(this.listItems);
  }
}
```
在上面的示例中，`listItems`属性将由`ul`元素的所有子节点填充。如果`listItems`用`@observable`修饰，那么它将随着子节点的更改而动态更新。与任何可观察对象一样，您可以选择实现一个*propertyName*Changed方法，以便在节点更改时得到通知。此外，您可以向`children`指令提供`options` 对象，以指定基础的自定义配置[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)。

:::important
与`ref`类似，子节点在`connectedCallback`生命周期事件之前不可用。
:::

您还可以提供`filter`函数来控制哪些子节点与您的属性同步。为了方便起见，我们提供了一个`elements`过滤器，您可以选择指定选择器。以上述示例为例，如果我们想确保我们的`listItems`数组只包含`li`元素（而不是任何文本节点或其他潜在的子节点），我们可以这样编写模板：

**实例: 筛选子节点**

```ts
import { HIElement, customElement, html, children, repeat, elements } from '@hi-element';

const template = html<FriendList>`
  <ul ${children({ property: 'listItems', filter: elements('li') })}>
    ${repeat(x => x.friends, html<string>`
      <li>${x => x}</li>
    `)}
  </ul>
`;
```
如果对`children`使用`subtree`选项，则需要`selector`来代替`filter`。这使得在整个子树中存在潜在的大量节点的情况下，能够更有效地收集所需的节点。

### `slotted` 指令

有时，您可能希望引用指定给特定插槽的所有节点。要完成此操作，请使用`slotted`指令。（有关插槽的详细信息，请参阅[使用阴影DOM](./working-with-shadow-dom.zh)。）

```ts
import { HIElement, customElement, html, slotted } from '@hi-element';

const template = html<MyElement>`
  <div>
    <slot ${slotted('slottedNodes')}></slot>
  </div>
`;

@customElement({
  name: 'my-element',
  template
})
export class MyElement extends HIElement {
  @observable slottedNodes: Node[];

  slottedNodesChanged() {
    // respond to changes in slotted node
  }
}
```

与`children`指令类似，`slotted`指令将使用分配给插槽的节点填充 `slottedNodes`属性。如果 `slottedNodes`用`@observable`修饰，则它将随着指定节点的更改而动态更新。与任何可观察对象一样，您可以选择实现一个*propertyName*Changed方法，以便在节点更改时得到通知。此外，您还可以向`slotted`指令提供`options`对象，以指定基础[assignedNodes() API call](https://developer.mozilla.org/en-US/docs/Web/API/HTMLSlotElement/assignedNodes)或指定`filter`。

:::tip
最好为时隙节点利用更改处理程序，而不是假设节点将出现在`connectedCallback`中。
:::

## Host 指令

到目前为止，我们的绑定和指令只影响组件的影子DOM中的元素。但是，有时您希望根据属性状态影响主体元素本身。例如，进度组件可能希望根据进度状态向主机写入各种`aria`属性。为了简化类似的场景，可以使用`template`元素作为模板的根元素，它将表示宿主元素。您在`template`元素上放置的任何属性或指令都将应用于主机本身。

**实例: Host 指令模板**

```ts
const template = html<MyProgress>`
  <template (Represents my-progress element)
      role="progressbar"
      $aria-valuenow={x => x.value}
      $aria-valuemin={x => x.min}
      $aria-valuemax={x => x.max}>
    (template targeted at Shadow DOM here)
  </template>
`;
```

**实例: 具有Host指令输出的DOM**

```html
<my-progress
    min="0"              (from user)
    max="100"            (from user)
    value="50"           (from user)
    role="progressbar"   (from host directive)
    aria-valuenow="50"   (from host directive)
    aria-valuemin="0"    (from host directive)
    aria-valuemax="100"  (from host directive)>
</my-progress>
```

:::tip
在`template`元素上使用`children`指令将为您提供对自定义元素的所有Light-DOM子节点的引用，而不管它们是否时隙或时隙在哪里。
:::

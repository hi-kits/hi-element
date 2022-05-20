<div align="center">
<img src="logo.svg" alt="hi-element" width="200" >

### 一个简单的基类，用于创建快速，轻便的Web组件
# hi-element

</div>


`hi-element`库是一种轻量级的方法，可以轻松构建高性能、内存高效、符合标准的Web组件。hi-element适用于所有主流浏览器，可以与任何前端框架结合使用，甚至可以不使用框架。

## 安装

### From NPM

要安装`hi-element`库，请使用`npm`或`yarn`，如下所示：

```shell
npm install --save hi-element
```

```shell
yarn add hi-element
```

在JavaScript或TypeScript代码中，您可以像这样导入库API：

```ts
import { HIElement } from 'hi-element';
```


### From CDN

CDN上提供了一个预绑定脚本，其中包含使用FAST元素构建web组件所需的所有API。可以通过添加[`type=“module”`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)到script元素，然后从CDN导入。

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <script type="module">
          import { HIElement } from "/hi-element.min.js";

          // your code here
        </script>
    </head>
    <!-- ... -->
</html>
```

上面的标记始终引用最新版本。在部署到生产环境时，您将希望附带特定版本。下面是一个标记示例：

```html
<script type="module" src="/hi-element.min.js"></script>
```

:::note
整个文档中的示例都假设库是从NPM安装的，但您始终可以用CDN URL替换导入位置。
:::


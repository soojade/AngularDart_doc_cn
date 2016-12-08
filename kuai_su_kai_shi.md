## 快速开始


Angular 应用是由组件组成的。组件由 HTML 模板和组件类组成，组件类控制视图。下面是一个显示简单字符串的组件:

```
// lib/app_component.dart

import 'package:angular2/core.dart';

@Component(
    selector: 'my-app',
    template: '<h1>Hello {{name}}</h1>')
class AppComponent {
  var name = 'Angular';
}
```

无需任何安装就可以试试它。在另一个浏览器标签页打开并跟随 Plunker 上的[快速起步例子](http://angular-examples.github.io/quickstart/)。

每一个组件以一个`@Component`注解开始，表述了HTML 模板和组件是如何在一起工作的。

`selector`属性为 Angular 指定了在`index.html`中的自定义`<my-app>`标签里显示该组件。

```
// index.html (<body>部分)

<my-app>Loading AppComponent content here ...</my-app>
```

template属性定义了`<h1>`标题里的一条消息。 该消息以 “Hello” 开始，以 Angular 插值绑定表达式`{{name}}`结束。在运行时，Angular 用组件的name属性值替换`{{name}}`。

> 下一步
> 
> [概览]()
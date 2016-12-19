## HTTP

客户对我们的进展很满意！现在，它们想要从服务器获取英雄数据，然后让用户添加、编辑和删除英雄，并且把这些修改结果保存回服务器。

在这一章中，我们要让应用程序通过 HTTP 调用来访问远程服务器上相应的 Web API。

运行这部分的[在线例子](http://angular-examples.github.io/toh-6)。

### 我们离开的地方

在前一章中，我们学会了在仪表盘和固定的英雄列表之间导航，并编辑选定的英雄。这也就是本章的起点。

#### 保持应用的编译和运行

打开终端/控制台窗口，通过输入如下命令，启动 Dart 编译器，它会监视文件变化，并启动开发服务器：

```
pub serve
```

在我们继续构建《英雄之旅》的时候，应用保持运行并自动更新。

## 提供HTTP服务

我们将使用 Dart 中的 **http** 包的 `BrowserClient` 类来与服务器通信。

#### 更新 Pubspec

通过添加`stream_transformers`和 Dart 的 http 包来更新包依赖。

我们还需要添加 `resolved_identifiers` 项，以便通知 Angular2 转换器 我们将使用 `BrowserClient`。我们还需要从 http 中使用 `Client`，所以现在让我们一块添加上。

更新 `pubspec.yaml` ，内容如下：

```
name: angular_tour_of_heroes
# . . . 
dependencies:
  angular2: ^2.2.0
  http: ^0.11.0
  stream_transformers: ^0.3.0
  # . . . 
transformers:
- angular2:
# . . . 
    entry_points: web/main.dart
    resolved_identifiers:
        BrowserClient: 'package:http/browser_client.dart'
        Client: 'package:http/http.dart'
- dart_to_js_script_rewriter
```

#### 注册 HTTP 服务

在我们的应用开始使用 `BrowserClient` 之前，我们必须将它注册为一个服务提供者。

在应用中，我们应该能够在任何地方访问 `BrowserClient` 服务。因此我们将它注册到 `bootstrap` 调用中，那是我们启动应用和根组件 `AppComponent` 的地方。

```
import 'package:angular2/core.dart';
import 'package:angular2/platform/browser.dart';
import 'package:angular_tour_of_heroes/app_component.dart';
import 'package:http/browser_client.dart';

void main() {
  bootstrap(AppComponent, [
    provide(BrowserClient, useFactory: () => new BrowserClient(), deps: [])
  ]);
}
```

注意，我们在列表中添加了 `BrowserClient` ，作为 `bootstrap` 方法的第二个参数。这与`@Component`注解中的 `providers` 列表效果一样。

#### 模拟 Web API

我们建议在根组件 `AppComponent` 的 `providers` 中注册全应用级的服务。这里在 `main` 中进行注册有一个特殊的原因。

我们的应用处在开发的早期阶段，并且离进入产品阶段还很远。我们甚至都还没有一个用来处理英雄相关请求的 Web 服务器，在此之前，我们将不得不伪造一个 。

我们将欺骗 Http 客户端，让它从一个 Mock 服务（内存 Web API）中获取和保存数据。而应用本身并不需要知道，也不应该知道。所以我们让 内存（in-memory） Web API 悄悄的在 `AppComponent` 之上进行配置。

这个版本的 `web/main.dart` 就是用来实现这个小花招的：

```
import 'package:angular2/core.dart';
import 'package:angular2/platform/browser.dart';
import 'package:angular_tour_of_heroes/app_component.dart';
import 'package:angular_tour_of_heroes/in_memory_data_service.dart';
import 'package:http/http.dart';

void main() {
  bootstrap(AppComponent,
    [provide(Client, useClass: InMemoryDataService)]
    // Using a real back end? Import browser_client.dart and change the above to
    // [provide(Client, useFactory: () => new BrowserClient(), deps: [])]
  );
}
```

我们要替换 `BrowserClient` ，该服务会通过内存 Web API 服务，与远程服务器进行会话。我们的内存 Web API 服务如下所示，是通过 http 库的 `MockClient` 类来实现的。所有的 `http` 客户端实现都共享了一个通用的 `Client` 接口。所以我们在应用中使用了 `Clien`t 类型，以便自由地在各个实现之间进行切换。

```
// lib/in_memory_data_service.dart

import 'dart:async';
import 'dart:convert';
import 'dart:math';

import 'package:angular2/core.dart';
import 'package:http/http.dart';
import 'package:http/testing.dart';

import 'hero.dart';

@Injectable()
class InMemoryDataService extends MockClient {
  static final _initialHeroes = [
    {'id': 11, 'name': 'Mr. Nice'},
    {'id': 12, 'name': 'Narco'},
    {'id': 13, 'name': 'Bombasto'},
    {'id': 14, 'name': 'Celeritas'},
    {'id': 15, 'name': 'Magneta'},
    {'id': 16, 'name': 'RubberMan'},
    {'id': 17, 'name': 'Dynama2'},
    {'id': 18, 'name': 'Dr IQ'},
    {'id': 19, 'name': 'Magma'},
    {'id': 20, 'name': 'Tornado'}
  ];
  static final List<Hero> _heroesDb =
      _initialHeroes.map((json) => new Hero.fromJson(json)).toList();
  static int _nextId = _heroesDb.map((hero) => hero.id).fold(0, max) + 1;

  static Future<Response> _handler(Request request) async {
    var data;
    switch (request.method) {
      case 'GET':
        final id = int.parse(request.url.pathSegments.last, onError: (_) => null);
        if (id != null) {
          data = _heroesDb.firstWhere((hero) => hero.id == id); // throws if no match
        } else {
          String prefix = request.url.queryParameters['name'] ?? '';
          final regExp = new RegExp(prefix, caseSensitive: false);
          data = _heroesDb.where((hero) => hero.name.contains(regExp)).toList();
        }
        break;
      case 'POST':
        var name = JSON.decode(request.body)['name'];
        var newHero = new Hero(_nextId++, name);
        _heroesDb.add(newHero);
        data = newHero;
        break;
      case 'PUT':
        var heroChanges = new Hero.fromJson(JSON.decode(request.body));
        var targetHero = _heroesDb.firstWhere((h) => h.id == heroChanges.id);
        targetHero.name = heroChanges.name;
        data = targetHero;
        break;
      case 'DELETE':
        var id = int.parse(request.url.pathSegments.last);
        _heroesDb.removeWhere((hero) => hero.id == id);
        // No data, so leave it as null.
        break;
      default:
        throw 'Unimplemented HTTP method ${request.method}';
    }
    return new Response(JSON.encode({'data': data}), 200,
        headers: {'content-type': 'application/json'});
  }

  InMemoryDataService() : super(_handler);
}
```

该文件替换了 `mock_heroes.dart` ，它现在可以安全的删除了。

作为通用的 Web API 服务，我们的模拟内存服务将以 JSON 格式进行编解码英雄数据。现在我们给 `Hero` 类增加这些功能： 

```
class Hero {
  final int id;
  String name;

  Hero(this.id, this.name);

  factory Hero.fromJson(Map<String, dynamic> hero) =>
    new Hero(_toInt(hero['id']), hero['name']);

  Map toJson() => {'id': id, 'name': name};
}

int _toInt(id) => id is int ? id : int.parse(id);
```

### 英雄与 HTTP

来看看我们目前的`HeroService`的实现

```
Future<List<Hero>> getHeroes() async => mockHeroes;
```

我们返回了一个 Future ，它用来解析模拟英雄数据。在当时，它当时可能看起来显得有点过于复杂。不过我们预料到总有这么一天会通过 HTTP 客户端来获取英雄数据， 而且我们知道，那一定是一个异步操作。

这一天来了！我们将 `getHeroes()` 转换为使用 HTTP 。

```
  static const _heroesUrl = 'api/heroes'; // URL to web API

  final Client _http;

  HeroService(this._http);

  Future<List<Hero>> getHeroes() async {
    try {
      final response = await _http.get(_heroesUrl);
      final heroes = _extractData(response)
          .map((value) => new Hero.fromJson(value))
          .toList();
      return heroes;
    } catch (e) {
      throw _handleError(e);
    }
  }

  dynamic _extractData(Response resp) => JSON.decode(resp.body)['data'];

  Exception _handleError(dynamic e) {
    print(e); // for demo purposes only
    return new Exception('Server error; cause: $e');
  }
```

更新后的导入语句如下：

```
import 'dart:async';
import 'dart:convert';

import 'package:angular2/core.dart';
import 'package:http/http.dart';

import 'hero.dart';
```

刷新浏览器后，英雄数据就会从模拟服务器被成功读取。

#### HTTP Future

我们将仍然返回一个 Future ，但是用不同的方法来创建它。

为获取到英雄列表，我们首先异步调用 `http.get()` 。然后使用 `_extractData` 助手方法来解码响应的正文。

响应的 JSON 只有一个单一的 `data` 属性，保存了调用者真正期望的英雄列表。所以我们夺取那个列表，并将它作为解析的 Future 值来返回。

> 仔细看看这个由服务器返回的数据的形态。这个内存 Web API 的范例中所做的是返回一个带有`data`属性的对象。你的 API 也可以返回其它东西。请调整这些代码以匹配你的 Web API。

调用者并不知道这些小花招。就像之前一样，它接收一个英雄列表的 Future 。它并不知道我们是从（模拟）服务器获取英雄数据。也不必了解把 HTTP 响应转换成英雄数据时所作的这些复杂变换。看到美妙之处了吧，这正是将数据访问委托组一个服务的目的，就像 `HeroService`。

#### 错误处理

在`getHeroes()`的最后，我们`catch`了服务器的失败信息，并把它们传给了错误处理器：

```
} catch (e) {
  throw _handleError(e);
}
```

这是一个关键的步骤！我们必须预料到 HTTP 请求会失败，因为有太多我们无法控制的原因可能导致它们频繁出现各种错误。

```
Exception _handleError(dynamic e) {
  print(e); // for demo purposes only
  return new Exception('Server error; cause: $e');
}
```

在这个范例服务中，我们把错误记录到控制台中；在真实世界中，我们应该做得更好。

我们决定通过传播异常，将错误信息用一个用户友好的格式返回给调用者，以便调用者能把一个合适的错误信息显示给用户。

#### 通过 id 获取英雄

`HeroDetailComponent`请求`HeroService`获取一个单一的英雄来编辑。

`HeroService`获取所有的英雄，然后通过匹配的`id`过滤找到期望的英雄。在一个模拟的情形下是好的。但当我们只想要一个却向真实服务器请求所有的英雄是浪费的。多数的web APIs 支持这样的格式`api/hero/:id`，通过 id 获取请求。

更新`HeroService.getHero`方法，使用通过id获取请求，应用我们刚学到的来写`getHeroes`:

```
Future<Hero> getHero(int id) async {
  try {
    final response = await _http.get('$_heroesUrl/$id');
    return new Hero.fromJson(_extractData(response));
  } catch (e) {
    throw _handleError(e);
  }
}
```

这几乎和`getHeroes`一样。URL 通过编码英雄 id 到 URL 来匹配`api/hero/:id`这种模式告诉服务器应该更新哪个英雄。

我们也应该适应响应的`data`是一个单一的英雄对象而不是一个列表。

#### getHeroes API 没变

虽然我们对 `getHeroes()`和`getHeroes()` 做了一些重要的 *内部*修改，但该方法公开的函数签名并没有任何变化。从这两个方法我们返回的仍然是一个 Future，不用被迫更新任何一个调用了它们的组件。

到目前为止，我们的客户很欣赏这种富有弹性的 API 集成方式。 现在它们想增加创建和删除英雄的功能。

让我们来看看当我们试图更新英雄的详情时会发生什么。

### 更新英雄详情

我们已经可以在英雄详情中编辑英雄的名字了。来试试吧。在输入的时候，页头上的英雄名字也会随之更新。不过当我们点了Back（后退）按钮时，这些修改就丢失了。

以前是不会丢失更新的，现在是怎么回事？当该应用使用模拟出来的英雄列表时，更新直接被应用到了单一的、应用级共享的英雄列表中的英雄对象。而现在改成了从服务器获取数据。如果我们希望这些更改被持久化，我们就得把它们写回服务器。

#### 保存英雄详情

我们先来确保对英雄名字的编辑不会丢失。先在英雄详情模板的底部添加一个保存按钮，它绑定了一个`click`事件，事件绑定会调用组件中一个名叫`save`的新方法：

```
<button (click)="save()">Save</button>
```

`save`方法使用 `hero` 服务的`update`方法来保存对英雄名字的修改，然后导航回前一个视图：

```
Future<Null> save() async {
  await _heroService.update(hero);
  goBack();
}
```

#### hero 服务的 update 方法
`update`方法的大致结构与`getHeroes`类似，不过我们使用 HTTP 的 *put* 方法来把修改保存到服务端：

```
static final _headers = {'Content-Type': 'application/json'};

Future<Hero> update(Hero hero) async {
  try {
    final url = '$_heroesUrl/${hero.id}';
    final response =
        await _http.put(url, headers: _headers, body: JSON.encode(hero));
    return new Hero.fromJson(_extractData(response));
  } catch (e) {
    throw _handleError(e);
  }
}
```

我们通过一个编码在 URL 中的英雄 id 来告诉服务器应该更新哪个英雄。put 的 body 是该英雄的 JSON 字符串编码，它是通过调用`SON.encode`得到的。并且在请求头中标记出的 body 的内容类型（`application/json`）。

刷新浏览器试一下，对英雄名字的修改现在应该能保存了。

### 添加英雄

要添加一个新的英雄，我们得先知道英雄的名字。我们使用一个 `input` 元素配合一个添加按钮来实现。

把下列代码插入 heroes 组件的 HTML 中，放在标题的下面：

```
<div>
  <label>Hero name:</label> <input #heroName />
  <button (click)="add(heroName.value); heroName.value=''">
    Add
  </button>
</div>
```

当点击事件触发时，我们调用组件的点击处理器，然后清空这个输入框，以便用来输入另一个名字。

```
Future<Null> add(String name) async {
  name = name.trim();
  if (name.isEmpty) return;
  heroes.add(await _heroService.create(name));
  selectedHero = null;
}
```

当指定的名字不为空的时候，点击处理器就会委托 hero 服务来创建一个具有此名字的英雄，并把这个新的英雄添加到我们的列表中。

最后，我们在`HeroService`类中实现这个`create`方法。

```
Future<Hero> create(String name) async {
  try {
    final response = await _http.post(_heroesUrl,
        headers: _headers, body: JSON.encode({'name': name}));
    return new Hero.fromJson(_extractData(response));
  } catch (e) {
    throw _handleError(e);
  }
}
```

刷新浏览器，并创建一些新的英雄！

### 删除一个英雄

英雄太多了？ 我们在英雄列表视图中来为每个英雄添加一个删除按钮吧。

把这个按钮元素添加到英雄列表组件的 HTML 中，把它放在`<li>`标签中的英雄名的后面：

```
<button class="delete"
  (click)="delete(hero); $event.stopPropagation()">x</button>
```

`<li>`元素应该变成了这样：

```
  <li *ngFor="let hero of heroes" (click)="onSelect(hero)"
      [class.selected]="hero == selectedHero">
    <span class="badge">{ {hero.id} }</span>
    <span>{ {hero.name} }</span>
    <button class="delete"
      (click)="delete(hero); $event.stopPropagation()">x</button>
  </li>
```

除了调用组件的`delete`方法之外，这个`delete`按钮的点击处理器还应该阻止点击事件向上冒泡——我们并不希望触发`<li>`的点击事件处理器，否则它会选中我们要删除的这位英雄。

`delete`处理器的逻辑略复杂：

```
Future<Null> delete(Hero hero) async {
  await _heroService.delete(hero.id);
  heroes.remove(hero);
  if (selectedHero == hero) selectedHero = null;
}
```

当然，我们仍然把删除英雄的操作委托给了 hero 服务，不过该组件仍然负责更新显示：它从列表中移除了被删除的英雄，如果删除的是正选中的英雄，还会清空选择。

我们希望删除按钮被放在英雄条目的最右边。于是 CSS 变成了这样：

```
button.delete {
  float:right;
  margin-top: 2px;
  margin-right: .8em;
  background-color: gray !important;
  color:white;
}
```

#### hero 服务的delete方法

hero 服务的`delete`方法使用 HTTP 的 delete 方法来从服务器上移除该英雄：

```
Future<Null> delete(int id) async {
  try {
    final url = '$_heroesUrl/$id';
    await _http.delete(url, headers: _headers);
  } catch (e) {
    throw _handleError(e);
  }
}
```

刷新浏览器，并试一下这个新的删除功能。

### 数据流

回顾一下之前的内容，`HeroService.getHeroes()` 等待 `http.get()` 响应，并生成一个包含 `List<Hero>` 的 Future。当我们只对一个单一的结果感兴趣时，这很不错。

但是请求并非总是"一步到位"（one and done）的。我们可能开始一个请求，然后取消，并在服务器对第一个请求作出响应前，开始另一个请求。像这样一个*请求-取消-新请求*的序列，很难通过*Future*实现。但接下来我们会看到，它对于*Stream*却很简单。

#### 按名搜索

我们将为《英雄之旅》添加一个*英雄搜索*功能。当用户在搜索框中输入一个名字时，我们将不断发起 HTTP 请求，以获得按名字过滤的英雄。

我们先创建`HeroSearchService`服务，它会把搜索查询发送到我们服务器的 Web API。

```
import 'dart:async';
import 'dart:convert';

import 'package:angular2/core.dart';
import 'package:http/http.dart';

import 'hero.dart';

@Injectable()
class HeroSearchService {
  final Client _http;

  HeroSearchService(this._http);

  Future<List<Hero>> search(String term) async {
    try {
      final response = await _http.get('app/heroes/?name=$term');
      return _extractData(response)
          .map((json) => new Hero.fromJson(json))
          .toList();
    } catch (e) {
      throw _handleError(e);
    }
  }

  dynamic _extractData(Response resp) => JSON.decode(resp.body)['data'];

  Exception _handleError(dynamic e) {
    print(e); // for demo purposes only
    return new Exception('Server error; cause: $e');
  }
}
```
`HeroSearchService`中的`_http.get()`调用和 `HeroService` 中的类似，只是这次 URL 带了查询字符串。

#### 英雄搜索组件

我们再创建一个新的 `HeroSearchComponent` 来调用这个新的 `HeroSearchService` 。

组件模板很简单，就是一个输入框和一个相匹配的搜索结果列表。

```
<div id="search-component">
  <h4>Hero Search</h4>
  <input #searchBox id="search-box" (keyup)="search(searchBox.value)" />
  <div>
    <div *ngFor="let hero of heroes | async"
         (click)="gotoDetail(hero)" class="search-result" >
      { {hero.name} }
    </div>
  </div>
</div>
```

我们还要往这个新组件中添加样式。

```
.search-result {
  border-bottom: 1px solid gray;
  border-left: 1px solid gray;
  border-right: 1px solid gray;
  width:195px;
  height: 20px;
  padding: 5px;
  background-color: white;
  cursor: pointer;
}
#search-box {
  width: 200px;
  height: 20px;
}
```

当用户在搜索框中输入时，一个 `keyup` 事件绑定会调用该组件的`search`方法，并传入新的搜索框的值。

`*ngFor`从该组件的`heroes`属性重复获取 hero 对象。这也没啥特别的。

但是接下来我们看到，现在 `heroes` 属性是一个英雄列表的 Stream 对象，而不再只是英雄列表。 `*ngFor` 不能用 `Stream` 对象做任何事，除非我们在它后面跟一个 `async` 管道(`AsyncPipe`) 。这个 `async` 管道会订阅 `Stream` 对象，并为 `*ngFor` 生成一个英雄列表。

是时候创建 `HeroSearchComponent` 类及其元数据了。

```
import 'dart:async';

import 'package:angular2/core.dart';
import 'package:angular2/router.dart';
import 'package:stream_transformers/stream_transformers.dart';

import 'hero_search_service.dart';
import 'hero.dart';

@Component(
    selector: 'hero-search',
    templateUrl: 'hero_search_component.html',
    styleUrls: const ['hero_search_component.css'],
    providers: const [HeroSearchService])
class HeroSearchComponent implements OnInit {
  HeroSearchService _heroSearchService;
  Router _router;

  Stream<List<Hero>> heroes;
  StreamController<String> _searchTerms =
      new StreamController<String>.broadcast();

  HeroSearchComponent(this._heroSearchService, this._router) {}

  // Push a search term into the stream.
  void search(String term) => _searchTerms.add(term);

  Future<Null> ngOnInit() async {
    heroes = _searchTerms.stream
        .transform(new Debounce(new Duration(milliseconds: 300)))
        .distinct()
        .transform(new FlatMapLatest((term) => term.isEmpty
            ? new Stream<List<Hero>>.fromIterable([<Hero>[]])
            : _heroSearchService.search(term).asStream()))
        .handleError((e) {
      print(e); // for demo purposes only
    });
  }

  void gotoDetail(Hero hero) {
    var link = [
      'HeroDetail',
      {'id': hero.id.toString()}
    ];
    _router.navigate(link);
  }
}
```

#### 搜索词

仔细看下这个`_searchTerms`：

```
StreamController<String> _searchTerms =
    new StreamController<String>.broadcast();

// Push a search term into the stream.
void search(String term) => _searchTerms.add(term);
```

`StreamController`，顾名思义，是用于 Stream 的控制器，它允许我们处理基础数据流，例如添加数据。

在我们的例子中，字符串的基础数据流（`_searchTerms.stream`），表示与用户输入的搜索词匹配的英雄名称。每次调用 `search` ，都会通过控制器的 `add` 方法，将新的字符串放进数据流。

#### 初始化 heroes 属性（ngOnInit）

我们打算把搜索词的数据流转换成 `hero` 列表的数据流，并把结果赋值给 `heroes` 属性。

```
Stream<List<Hero>> heroes;

Future<Null> ngOnInit() async {
  heroes = _searchTerms.stream
      .transform(new Debounce(new Duration(milliseconds: 300)))
      .distinct()
      .transform(new FlatMapLatest((term) => term.isEmpty
          ? new Stream<List<Hero>>.fromIterable([<Hero>[]])
          : _heroSearchService.search(term).asStream()))
      .handleError((e) {
    print(e); // for demo purposes only
  });
}
```

如果我们把每一次用户按键都直接传给`HeroSearchService`，就会发起一场 HTTP 请求风暴。这可不好玩。我们不希望占用服务器资源，也不打算耗尽网络带宽。

幸运的是，数据流转换器可以帮助我们归并这些请求。我们将对 `HeroSearchService` 发起更少的调用，并且仍然及时地获得响应。做法如下：

* 在传出最终字符串之前，`transform(new Debounce(... 300)))` 会一直等待，直到搜索词暂停输入300毫秒。我们实际发起请求的间隔永远不会小于300ms。
* `distinct()` 会确保在搜索词发生变化时，我们仅发送一次请求。这样就不会重复请求同一个搜索词了。
* `transform(new FlatMapLatest(...))` 应用了一个类似映射的转换器，它会为每个已经通过去抖动（debounce）和去重复（distinct）的搜索词调用搜索服务。并且，它会丢弃之前的搜索结果，仅返回最近的搜索服务结果。
* `handleError()`处理错误。这个简单的例子中只是把错误信息打印到控制台;实际的应用应该做得更好。

#### 为仪表盘添加搜索组件

我们将英雄搜索的 HTML 元素添加到`DashboardComponent`模版的底部。

```
<h3>Top Heroes</h3>
<div class="grid grid-pad">
  <a *ngFor="let hero of heroes"  [routerLink]="['HeroDetail', {id: hero.id.toString()}]"  class="col-1-4">
    <div class="module hero">
      <h4>{ {hero.name} }</h4>
    </div>
  </a>
</div>
<hero-search></hero-search>
```

最后，从 `hero_search_component.dart` 中导入 `HeroSearchComponent` ，并将其添加到 `directives` 列表中。

```
import 'hero_search_component.dart';

@Component(
    selector: 'my-dashboard',
    templateUrl: 'dashboard_component.html',
    styleUrls: const ['dashboard_component.css'],
    directives: const [HeroSearchComponent, ROUTER_DIRECTIVES])
```

再次运行该应用，跳转到仪表盘，并在英雄下方的搜索框里输入一些文本。看起来就像这样：

![](https://webdev.dartlang.org/resources/images/devguide/toh/toh-hero-search.png)

### 应用的结构与代码

回顾一下本章在线例子中的范例代码。验证我们是否得到了如下结构：

```
angular_tour_of_heroes/
|___lib/
|    |___app_component.css
|    |___app_component.dart
|    |___dashboard_component.css
|    |___dashboard_component.dart
|    |___dashboard_component.html
|    |___hero.dart
|    |___hero_detail_component.css
|    |___hero_detail_component.dart
|    |___hero_detail_component.html
|    |___hero_search_component.css (new)
|    |___hero_search_component.dart (new)
|    |___hero_search_component.html (new)
|    |___hero_search_service.dart (new)
|    |___hero_service.dart
|    |___heroes_component.css
|    |___heroes_component.dart
|    |___heroes_component.html
|    |___in_memory_data_service.dart (new)
|___web/
|    |___main.dart
|    |___index.html
|    |___styles.css
|___pubspec.yaml
```

### 最后冲刺

旅程即将结束，不过我们已经收获颇丰。

* 我们在应用程序中添加了使用 HTTP 时必要的依赖
* 我们重构了 `HeroService` ，以通过 Web API 来加载英雄数据
* 我们扩展了 `HeroService`，以支持 Post 、 Put 和 Delete 方法
* 我们更新了组件，以允许用户添加、编辑和删除英雄
* 我们配置了一个内存（in-memory） Web API
* 我们学会了如何使用数据流

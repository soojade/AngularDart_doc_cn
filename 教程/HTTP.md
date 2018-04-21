> 版本：4.0.0+2

在本章，你会做以下改进。

* 从一个服务器获取英雄数据。
* 让用户添加、编辑和删除英雄。
* 保存改变到服务器。

你会教给应用发起到一个远程服务器的 web API 的相应的 HTTP 请求。

当你按照本章做完这一切，应用看起来应该这样——[在线示例](https://webdev.dartlang.org/examples/toh-6/) ([查看源码](https://github.com/angular-examples/toh-6/tree/4.x))。

### 你离开的地方

在[前一章](./路由.md)中，你学会了在仪表盘和固定的英雄列表之间导航，并编辑选定的英雄。这也就是本章的起点。

在继续英雄指南之前，验证你是否有如下结构。

```
angular_tour_of_heroes/
|___lib/
|   |___app_component.{css,dart}
|___src/
|   |   |___dashboard_component.{css,dart,html}
|   |   |___hero.dart
|   |   |___hero_detail_component.{css,dart,html}
|   |   |___hero_service.dart
|   |   |___heroes_component.{css,dart,html}
|   |   |___mock_heroes.dart
|___test/
|   |___app_test.dart
|   |___...
|___web/
|   |___index.html
|   |___main.dart
|   |___styles.css
|___pubspec.yaml
```

如果应用还没有运行，使用`pub serve`启动应用。当你做出改变时，通过刷新浏览器窗口，使其保持运行。

### 提供 HTTP 服务

你将使用 Dart的 [http](https://pub.dartlang.org/packages/http) 包的客户端类来与服务器通信。

#### 更新 Pubspec

通过添加Dart  [http](https://pub.dartlang.org/packages/http) 和 [stream_transform](https://pub.dartlang.org/packages/stream_transform) 包来更新包依赖。

```
// {toh-5 → toh-6}/pubspec.yaml

dependencies:            
    angular: ^4.0.0            
    angular_forms: ^1.0.0            
    angular_router: ^1.0.2            
+   http: ^0.11.0            
+   stream_transform: ^0.0.6
```

### 注册 HTTP 服务

在应用可以使用`BrowserClient`之前，你必须把它当一个服务提供器注册。

你在应用的任何地方都应该能访问`BrowserClient`服务。因此在`bootstrap` 调用中注册它，那是你启动应用和根组件`AppComponent`的地方。

```
// web/main.dart (v1)

import 'package:angular/angular.dart';
import 'package:angular_router/angular_router.dart';
import 'package:angular_tour_of_heroes/app_component.dart';
import 'package:http/browser_client.dart';
void main() {
  bootstrap(AppComponent, [
    ROUTER_PROVIDERS,
    // Remove next line in production
    provide(LocationStrategy, useClass: HashLocationStrategy),
    provide(BrowserClient, useFactory: () => new BrowserClient(), deps: [])
  ]);
}
```

注意，你在`bootstrap`方法的第二个参数列表中提供`BrowserClient`。这和在`@Component`注解中的`providers`列表中有同样的效果。

> **注意：**除非你有一个适当配置的后端服务器（或一个模拟服务器），否则，这个应用并不工作。接下来的部分展示如何与一个后端服务器模拟交互。

#### 模拟 Web API

在你有一个处理英雄数据请求的 Web 服务器之前，HTTP 客户端会从一个模拟服务器——*内存 web API* 获取并保存数据。

使用模拟服务器的版本来更新`web/main.dart`：

```
// web/main.dart (v2)

import 'package:angular/angular.dart';
import 'package:angular_router/angular_router.dart';
import 'package:angular_tour_of_heroes/app_component.dart';
import 'package:angular_tour_of_heroes/in_memory_data_service.dart';
import 'package:http/http.dart';
void main() {
  bootstrap(AppComponent, [
    ROUTER_PROVIDERS,
    // Remove next line in production
    provide(LocationStrategy, useClass: HashLocationStrategy),
    provide(Client, useClass: InMemoryDataService),
    // Using a real back end?
    // Import browser_client.dart and change the above to:
    // [provide(Client, useFactory: () => new BrowserClient(), deps: [])]
  ]);
}
```

你想要使用内存 Web API 服务代替`BrowserClient`，与远程服务器进行会话。内存 Web API 服务如下所示，是通过`http`库的`MockClient`类来实现的。所有的`http`客户端实现都共享了一个通用的`Client`接口。所以你要在应用中使用`Client` 类型，以便在各个实现之间自由地切换。

```
// lib/in_memory_data_service.dart (init)

import 'dart:async';
import 'dart:convert';
import 'dart:math';
import 'package:angular/angular.dart';
import 'package:http/http.dart';
import 'package:http/testing.dart';
import 'src/hero.dart';
@Injectable()
class InMemoryDataService extends MockClient {
  static final _initialHeroes = [
    {'id': 11, 'name': 'Mr. Nice'},
    {'id': 12, 'name': 'Narco'},
    {'id': 13, 'name': 'Bombasto'},
    {'id': 14, 'name': 'Celeritas'},
    {'id': 15, 'name': 'Magneta'},
    {'id': 16, 'name': 'RubberMan'},
    {'id': 17, 'name': 'Dynama'},
    {'id': 18, 'name': 'Dr IQ'},
    {'id': 19, 'name': 'Magma'},
    {'id': 20, 'name': 'Tornado'}
  ];
  static List<Hero> _heroesDb;
  static int _nextId;
  static Future<Response> _handler(Request request) async {
    if (_heroesDb == null) resetDb();
    var data;
    switch (request.method) {
      case 'GET':
        final id =
            int.parse(request.url.pathSegments.last, onError: (_) => null);
        if (id != null) {
          data = _heroesDb
              .firstWhere((hero) => hero.id == id); // throws if no match
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
  static resetDb() {
    _heroesDb = _initialHeroes.map((json) => new Hero.fromJson(json)).toList();
    _nextId = _heroesDb.map((hero) => hero.id).fold(0, max) + 1;
  }
  static String lookUpName(int id) =>
      _heroesDb.firstWhere((hero) => hero.id == id, orElse: null)?.name;
  InMemoryDataService() : super(_handler);
}
```

这个文件代替了`mock_heroes.dart`，它现在可以安全的删除了。

作为通用的 Web API 服务，模拟内存服务将以 JSON 格式进行编码和解码英雄，所以给`Hero`类增加这些功能： 

```
// lib/src/hero.dart

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

在当前的`HeroService`的实现中，返回一个 Future 类型的模拟英雄。

```
Future<List<Hero>> getHeroes() async => mockHeroes;
```

这是最终使用 HTTP 客户端来获取英雄的预期实现，那一定是一个异步操作。

现在将`getHeroes()`转换为使用 HTTP 的形式 。

```
// lib/src/hero_service.dart (updated getHeroes and new class members)

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

更新导入语句如下：

```
// lib/src/hero_service.dart (updated imports)

import 'dart:async';
import 'dart:convert';

import 'package:angular/angular.dart';
import 'package:http/http.dart';

import 'hero.dart';
```

刷新浏览器。英雄数据应该从模拟服务器中成功加载。

#### HTTP Future

为获取到英雄列表，首先异步调用`http.get()`。然后使用`_extractData`助手方法来解码响应的正文。

响应的 JSON 有一个单一的`data`属性，保存了调用者想要的英雄列表。所以提取这个列表，并将它作为解析后的 Future 值返回。

> 注意这个由服务器返回的数据的形态。这个特殊的内存 Web API 的实例返回一个带有`data`属性的对象。你的 API 可能返回其它东西。调整代码以匹配你的 Web API。

调用者并不知道从（模拟）服务器获取英雄。和以前一样它接收一个*英雄*的 Future。

#### 错误处理

在`getHeroes()`的最后，你`catch`了服务器的失败信息，并把它们传给了错误处理器：

```
} catch (e) {
  throw _handleError(e);
}
```

这是一个关键的步骤！你必须预料到 HTTP 请求会失败，因为有太多超出控制的原因可能导致它们频繁发生。

```
Exception _handleError(dynamic e) {
  print(e); // for demo purposes only
  return new Exception('Server error; cause: $e');
}
```

这个示例服务把错误记录到控制台中；在真实世界中，你应该在代码中处理错误。作为一个展示，它能工作就够了。

这个代码还包含一个通过传播异常给调用者的错误，以便调用者可以给用户显示恰当的错误信息。

#### 通过 id 获取英雄

当`HeroDetailComponent`请求`HeroService`来获取一个英雄时，`HeroService`当前获取所有英雄并通过匹配`id`来过滤这一个。对于一个模拟环境是很好的，但当你只想要一个时，却向真实的服务器请求所有的英雄是浪费的。多数的 web APIs 支持这样的格式`api/hero/:id`（例如`api/hero/11`）的 *get-by-id* 请求。

使用 *get-by-id* 请求更新`HeroService.getHero()`方法：

```
// lib/src/hero_service.dart (getHero)

Future<Hero> getHero(int id) async {
  try {
    final response = await _http.get('$_heroesUrl/$id');
    return new Hero.fromJson(_extractData(response));
  } catch (e) {
    throw _handleError(e);
  }
}
```

这个请求和`getHeroes()`几乎一样。在 URL中的英雄 id 标识服务器应该更新哪个英雄。

另外，响应中的`data`是一个单一的英雄对象而不是一个列表。

#### 未改变的 *getHeroes* API

尽管你对`getHeroes()`和`getHero()`做了一些重要的内部修改，但公共签名并没有改变。从这两个方法仍然返回一个 Future。你不需要更新任何调用了它们的组件。

现在是时候添加创建和删除英雄的功能了。

### 更新英雄详情

尝试在英雄详情视图编辑一个英雄的名字了。随着你的输入，英雄名字也会随之在视图的标题更新。但如果你点击了 Back（后退）按钮，这些修改就丢失了。

之前更新是不会丢失的。有什么被改变了？当应用使用模拟英雄列表时，更新直接被应用到了单一的、全应用范围共享的列表中的英雄对象。现在你从一个服务器获取数据，如果你想要保存这些更改，你必须把它们写回到服务器。

#### 添加保存英雄详情的功能

在英雄详情模板的结尾添加一个带`click`事件绑定的保存按钮，事件绑定会调用组件中一个名为`save()`的新方法。

```
// lib/src/hero_detail_component.html (save)

<button (click)="save()">Save</button>
```

添加下面的`save()`方法，它使用英雄服务的`update()`方法来保存英雄名的改变，然后导航回前一个视图。

```
// lib/src/hero_detail_component.dart (save)

Future<Null> save() async {
  await _heroService.update(hero);
  goBack();
}
```

#### 给英雄服务添加 *update()* 方法

`update()`方法的总体结构和`getHeroes()`很相似，但它使用 HTTP`put()`来把修改保存到服务器端。

```
// lib/src/hero_service.dart (update)

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

为了确定服务器应该更新哪个英雄，英雄`id`被编码进 URL 中。`putt()`body 参数是通过调用`JSON.encode`获得的英雄的 JSON 字符串编码。body 的内容类型（`application/json`）被标记在请求头中。

刷新浏览器，改变一个英雄的名字，保存你的修改，并点击浏览器的后退按钮。修改现在应该保存了。

### 增加添加英雄的功能

要添加一个英雄，应用需要这个英雄的名字。你可以使用一个`input`元素搭配一个添加按钮。

在 heroes 组件的 HTML 中，紧跟标题的后面，插入如下内容：

```
// lib/src/heroes_component.html (add)

<div>
  <label>Hero name:</label> <input #heroName />
  <button (click)="add(heroName.value); heroName.value=''">
    Add
  </button>
</div>
```

响应一个点击事件，调用组件的点击处理器，然后清空这个输入框，以便准备好输入另一个名字。

```
// lib/src/heroes_component.dart (add)

Future<Null> add(String name) async {
  name = name.trim();
  if (name.isEmpty) return;
  heroes.add(await _heroService.create(name));
  selectedHero = null;
}
```

当给定的名字不为空时，处理器委托英雄服务来创建这个命名英雄，然后把这个新英雄添加到列表中。

在`HeroService`类中实现这个`create()`方法。

```
// lib/src/hero_service.dart (create)

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

刷新浏览器，并创建一些英雄。

### 增加删除英雄的功能

在英雄视图中的每个英雄都应该有一个删除按钮。

把下面的按钮元素添加到 heroes 组件的 HTML 中，把它放在重复的`<li>`元素里英雄名的后面。

```
<button class="delete"
  (click)="delete(hero); $event.stopPropagation()">x</button>
```

`<li>`元素应该看起来像这样：

```
// lib/src/heroes_component.html (li element)

 <li *ngFor="let hero of heroes" (click)="onSelect(hero)"
    [class.selected]="hero === selectedHero">
  <span class="badge">{{hero.id}}</span>
  <span>{{hero.name}}</span>
  <button class="delete"
    (click)="delete(hero); $event.stopPropagation()">x</button>
</li>
```

除了调用组件的`delete()`方法，这个删除按钮的点击处理器代码阻止点击事件冒泡——你并不希望`<li>`的点击事件处理器被触发，因为这样做想要选择英雄时用户会删除这个英雄。

`delete()`处理器的逻辑有点棘手：

```
// lib/src/heroes_component.dart (delete)

Future<Null> delete(Hero hero) async {
  await _heroService.delete(hero.id);
  heroes.remove(hero);
  if (selectedHero == hero) selectedHero = null;
}
```

你当然委托给英雄服务来删除英雄，但该组件仍然负责更新显示：如果有必要，它从列表中移除 已删除的英雄，并重置所选英雄。

添加如下 CSS 把删除按钮放在英雄条目的最右边：

```
// 
lib/src/heroes_component.css (additions)

button.delete {
  float:right;
  margin-top: 2px;
  margin-right: .8em;
  background-color: gray !important;
  color:white;
}
```

#### 英雄服务的 *delete()* 方法

添加英雄服务的`delete()`方法，使用 HTTP 的`delete()`方法从服务器上移除英雄：

```
// lib/src/hero_service.dart (delete)

Future<Null> delete(int id) async {
  try {
    final url = '$_heroesUrl/$id';
    await _http.delete(url, headers: _headers);
  } catch (e) {
    throw _handleError(e);
  }
}
```

刷新浏览器，并试试这个新的删除功能。

### 数据流

回想一下，`HeroService.getHeroes()`等待一个`http.get()`响应，并生成一个`List<Hero>` 的 *Future*。当你只对一个单一的结果感兴趣时，这很不错。

但是请求并不总是只发起一次。你可能开始发起一个请求，然后取消，并在服务器对第一个请求作出响应前发起一个不同的请求。一个*请求-取消-新请求*的序列难以通过*Futures* 实现，但使用 *Streams* 却很容易。

#### 增加按名搜索的功能

你将为英雄指南添加一个*英雄搜索* 的特性。当用户在搜索框中输入名字时，你会重复发起根据名字过滤英雄的 HTTP 请求。 

先创建`HeroSearchService`服务，它会把搜索查询发送到服务器的 Web API。

```
// lib/src/hero_search_service.dart

import 'dart:async';
import 'dart:convert';

import 'package:angular/angular.dart';
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
`HeroSearchService`中的`_http.get()`调用和`HeroService`中的那一个类似，尽管现在这个 URL 带了查询字符串。

#### HeroSearchComponent

创建一个`HeroSearchComponent`，它调用这个新的`HeroSearchService`。

组件模板很简单，就是一个输入框和一个相匹配的搜索结果列表。

```
// lib/src/hero_search_component.html

<div id="search-component">
  <h4>Hero Search</h4>
  <input #searchBox id="search-box"
         (change)="search(searchBox.value)"
         (keyup)="search(searchBox.value)" />
  <div>
    <div *ngFor="let hero of heroes | async"
         (click)="gotoDetail(hero)" class="search-result" >
      {{hero.name}}
    </div>
  </div>
</div>
```

同时，给这个新组件添加样式。

```
// lib/src/hero_search_component.css

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

当用户在搜索框中输入时，一个 *keyup* 事件绑定调用该组件的`search()`方法，并传入新的搜索框的值。如果用户使用鼠标粘贴文本，*change* 事件绑定会被触发。

不出所料，`*ngFor`从该组件的`heroes`属性重复 hero 对象。

但你很快就会看到，现在`heroes`属性是一个英雄列表的 *Stream*，而不再只是一个英雄列表。`*ngFor` 不能使用`Stream`做任何事，除非你通过`async`管道(`AsyncPipe`)联通它。`async`管道订阅`Stream`，并为`*ngFor`生成一个英雄列表。

创建`HeroSearchComponent`类及其元数据。

```
// lib/src/hero_search_component.dart

import 'dart:async';
import 'package:angular/angular.dart';
import 'package:angular_router/angular_router.dart';
import 'package:stream_transform/stream_transform.dart';
import 'hero_search_service.dart';
import 'hero.dart';
@Component(
  selector: 'hero-search',
  templateUrl: 'hero_search_component.html',
  styleUrls: const ['hero_search_component.css'],
  directives: const [CORE_DIRECTIVES],
  providers: const [HeroSearchService],
  pipes: const [COMMON_PIPES],
)
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
        .transform(debounce(new Duration(milliseconds: 300)))
        .distinct()
        .transform(switchMap((term) => term.isEmpty
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

##### 搜索条目

聚焦`_searchTerms`：

```
StreamController<String> _searchTerms =
    new StreamController<String>.broadcast();

// Push a search term into the stream.
void search(String term) => _searchTerms.add(term);
```

 [StreamController](https://api.dartlang.org/stable/dart-async/StreamController-class.html)，顾名思义，是一个 [Stream](https://api.dartlang.org/stable/dart-async/Stream-class.html) 的控制器，比如它允许你通过给它添加数据来操作潜在的数据流。

在这个例子中，字符串的潜在的数据流（`_searchTerms.stream`），表示与用户输入的搜索词匹配的英雄名。每次调用`search()`，都会通过调用控制器的`add()`，把新的字符串放进数据流。

##### 初始化 *heroes* 属性（*ngOnInit*）

你可以把搜索条目的数据流转换成`Hero`列表的数据流，并把结果赋值给`heroes`属性。

```
Stream<List<Hero>> heroes;

Future<Null> ngOnInit() async {
  heroes = _searchTerms.stream
      .transform(debounce(new Duration(milliseconds: 300)))
      .distinct()
      .transform(switchMap((term) => term.isEmpty
          ? new Stream<List<Hero>>.fromIterable([<Hero>[]])
          : _heroSearchService.search(term).asStream()))
      .handleError((e) {
    print(e); // for demo purposes only
  });
}
```

每次用户按键都立刻传给`HeroSearchService`，就会创建海量的 HTTP 请求，浪费服务器资源并消耗大量网络流量。

相反，你可以使用链式调用`Stream`操作符，减少流向字符串`Stream`的请求。你将发起较少的`HeroSearchService`调用，并且仍然及时地获得结果。做法如下：

* `transform(debounce(... 300)))`：在传递最终的字符串之前，会一直等待，直到搜索条目暂停输入300毫秒。你发起请求的频率永远不会超过300ms。
* `distinct()`：确保只在过滤文本变化时才发送请求。
* `transform(switchMap(...))`：为每个已经通过`debounce()`和`distinct()`的搜索条目调用搜索服务。它取消并丢弃之前的搜索，仅返回最新的搜索服务流元素。
* `handleError()`：处理错误。这个简单的例子只是把错误信息打印到控制台;实际的应用应该做得更好。

#### 为仪表盘添加搜索组件

把英雄搜索的 HTML 元素添加到`DashboardComponent`模版的底部。

```
// 
lib/src/dashboard_component.html

<h3>Top Heroes</h3>
<div class="grid grid-pad">
  <a *ngFor="let hero of heroes"  [routerLink]="['HeroDetail', {id: hero.id.toString()}]"  class="col-1-4">
    <div class="module hero">
      <h4>{{hero.name}}</h4>
    </div>
  </a>
</div>
<hero-search></hero-search>
```

最后，从`hero_search_component.dart`中导入`HeroSearchComponent`，并把它添加到`directives`列表中。

```
// lib/src/dashboard_component.dart (search)

import 'hero_search_component.dart';

@Component(
  selector: 'my-dashboard',
  templateUrl: 'dashboard_component.html',
  styleUrls: const ['dashboard_component.css'],
  directives: const [CORE_DIRECTIVES, HeroSearchComponent, ROUTER_DIRECTIVES],
)
```

再次运行应用。在仪表盘中的搜索框中输入一些文字。如果你输入的字符匹配到了任何现有英雄名，你将会看到如下效果：

![](http://upload-images.jianshu.io/upload_images/892968-ca26fa5da70e70b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 应用的结构与代码

回顾本章示例的源代码 [在线示例](https://webdev.dartlang.org/examples/toh-6/) ([查看源码](https://github.com/angular-examples/toh-6/tree/4.x))。验证你是否有如下结构：

```
angular_tour_of_heroes/
|___lib/
|    |___app_component.{css,dart}
|    |___in_memory_data_service.dart (new)
|    |___src/
|    |     |___dashboard_component.{css,dart,html}
|    |     |___hero.dart
|    |     |___hero_detail_component.{css,dart,html}
|    |     |___hero_search_component.{css,dart,html} (new)
|    |     |___hero_search_service.dart (new)
|    |     |___hero_service.dart
|    |     |___heroes_component.{css,dart,html}
|___test/
|    |___app_test.dart
|    |___...
|___web/
|    |___main.dart
|    |___index.html
|    |___styles.css
|___pubspec.yaml
```

### 最后冲刺

旅程即将结束，不过你已经收获颇丰。

* 添加了在应用中使用 HTTP 时必要的依赖。
* 重构了`HeroService`，从一个 Web API 来加载英雄。
* 扩展了`HeroService`，以支持`post()`、`put()`和`delete()`方法。
* 更新了组件，以允许英雄的添加、编辑和删除。
* 配置了一个内存 Web API。
* 学会了如何使用 Streams。

### 下一步

回到[学习路径](../指南/学习Angular.md)，在哪里你可以阅读更多关于在本教程中的概念和练习。

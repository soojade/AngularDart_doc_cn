### 安装

```
pub global activate angular_cli
```

安装完成后需要把`.pub-cache/bin`添加到环境变量中：

```
export PATH = "$HOME/.pub-cache/bin:$PATH"
```

### 更新

```
pub global activate angular_cli
```

### 使用

使用`ngdart help`查看详细命令。

#### 创建新项目

```
ngdart new project_name
cd project_name
pub get
pub serve
```

浏览器打开 [http://localhost:8080](http://localhost:8080)，查看结果。

#### 生成组件

```
ngdart generate component AnotherComponent
```

在`lib/`目录下生成如下文件：

```
INFO: Saving lib/another_component.dart
INFO: Saving lib/another_component.html
```

使用`-p`参数自定义目录：

```
ngdart generate component AnotherComponent -p lib/another
```

生成结果如下：

```
INFO: Saving lib/another/another_component.dart
INFO: Saving lib/another/another_component.html
```

#### 生成测试

```
ngdart generate test lib/app_component.dart
```

会在`test`目录下生成两个文件：

```
INFO: Saving test/app_component_test.dart
INFO: Saving test/app_component_po.dart
```

运行命令：

```
pub run angular_test --test-arg=--tags=aot --test-arg=--platform=dartium  --test-arg=--reporter=expanded
```

使用 [Dartium](https://webdev.dartlang.org/tools/dartium)运行测试。

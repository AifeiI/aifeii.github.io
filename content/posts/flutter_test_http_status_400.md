+++
title = '在Flutter单元测试中遇到HTTP Status Code = 400的问题'
date = 2024-03-12T15:01:00+08:00
draft = false
+++

## 背景
在一次编写测试用例过程中偶遇的这个问题，第一次遇到时让我摸不着头脑，还是顺着日志找到了一些线索。

控制台打印的日志中，可能有以下内容出现

- 如果使用`Dio`，会出现以下内容：
```
DioException [bad response]: This exception was thrown because the response has a status code of 400 and RequestOptions.validateStatus was configured to throw for this status code.
The status code of 400 has the following meaning: "Client error - the request contains bad syntax or cannot be fulfilled"
Read more about status codes at https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
In order to resolve this exception you typically have either to verify and fix your request code or you have to fix the server code.
```

- 直接使用`HttpClient`，则有以下内容：
```
Warning: At least one test in this suite creates an HttpClient. When running a test suite that uses
TestWidgetsFlutterBinding, all HTTP requests will return status code 400, and no network request
will actually be made. Any test expecting a real network connection and status code will fail.
To test code that needs an HttpClient, provide your own HttpClient implementation to the code under
test, so that your test can consistently provide a testable response to the code under test.
```

当时编写的测试代码示例：
```dart
void main() {
  testWidgets('Widget test', (WidgetTester tester) async {
    // ...
  });

  test('Http test', () async {
    // ...
    final dio = Dio();
    final response = await dio.get('/api'); // <------- At this point, an exception will occur.
    // ...
  });
}
```

原本是考虑快速测试一个请求的响应处理，结果发现根本不能发出请求，根据线索找到了具体原因，结论是测试规范的限制，如需在Widget测试过程中与外部进行通讯的，归类为集成测试`integration tests`。

## 线索
在[Flutter Cookbook](https://docs.flutter.dev/cookbook/testing)中，分有三个篇章：[Integration](https://docs.flutter.dev/cookbook/testing/integration/introduction)、[Unit](https://docs.flutter.dev/cookbook/testing/unit/introduction)、[Widget](https://docs.flutter.dev/cookbook/testing/widget/introduction)，三个篇章都有详细的指引。


## 结论
如果需要在测试中完成一次网络请求，有两种方式：

- 作为单元测试，单独建立测试函数
  在原来的工程上，单独创建测试文件，将原本的`Http test`迁移到新的测试文件中。确保`test('', (){})`函数执行前不会执行`testWidgets('', (tester){})`

- 作为集成测试，在开发依赖`dev_dependencies`中添加`integration_test`
  具体添加方式可见：[Integration](https://docs.flutter.dev/cookbook/testing/integration/introduction)
  添加后，pubspec.yaml 文件应该可见以下内容：
  ```yaml
  # ...
  dev_dependencies:
    flutter_test:
      sdk: flutter
    integration_test:
      sdk: flutter
  # ...
  ```
  然后在主目录下新建`integration_test`目录，新建测试文件与方法，在 main() 执行前，调用`IntegrationTestWidgetsFlutterBinding.ensureInitialized()`方法
  ```dart
  void main() {
    IntegrationTestWidgetsFlutterBinding.ensureInitialized(); // <---- Add this
    testWidgets('', (WidgetTester tester) {
      // ...
    });
    test('', () async {
      // ...
      final dio = Dio();
      final response = await dio.get('/api'); // <------- An exception will not occur.
    });
  } 
  ```
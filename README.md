# Keycloak Flutter

[![pub package](https://img.shields.io/pub/v/keycloak_flutter.svg)](https://pub.dev/packages/keycloak_flutter)

> Easy Keycloak setup for Flutter applications.

## About

This library helps you to use [keycloak-js](https://www.keycloak.org/docs/latest/securing_apps/index.html#_javascript_adapter) in Flutter applications providing the following features:

- A **Keycloak Service** which wraps the `keycloak-js` methods to be used in Flutter, giving extra
  functionalities to the original functions and adding new methods to make it easier to be consumed by
  Flutter applications.
- ~~Generic **AuthGuard implementation**, so you can customize your own AuthGuard logic inheriting the authentication
  logic and the roles load.~~ (_coming soon_)
- ~~A **HttpClient interceptor** that adds the authorization header to all HttpClient requests.~~
  ~~It is also possible to disable this interceptor or exclude routes from having the authorization header.~~ (_coming
  soon_)
- ~~This documentation also assists you to configure the keycloak in your Flutter applications and with
  the client setup in the admin console of your keycloak installation.~~ (_coming soon_)

## Compatibility

The table below shows the compatibility of keycloak flutter with keycloak. Note that this table will be updated and is
not set in stone

| Keycloak_flutter version | Keycloak version |
|--------------------------|------------------|
| v0.0.3                   | v10 - v13        |
| v0.0.19                  | v17 - v19        |

## Installation

Firstly, you need to have keycloak configured. Duh!

Include [keycloak_flutter](https://pub.dev/packages/keycloak_flutter) as a dependency in the
dependencies section of your pubspec.yaml file :

```yaml
dependencies:
  flutter_web_plugins:
    sdk: flutter
  keycloak_flutter: ^latest.version
```

Next, In your `web/index.html`, you need to add a `script` with a source that references your keycloak.js file. You can
find `v10.0.2` in the example project. Your head tag should look as below.

```html
<!DOCTYPE html>
<html>
<head>
    <base href="/">
    <!-- CODE REMOVED FOR BREVITY   -->
    <link rel="manifest" href="manifest.json">
    <script src="js/keycloak.js"></script>
</head>
<body>
<script>
    if ('serviceWorker' in navigator) {
        window.addEventListener('flutter-first-frame', function () {
            navigator.serviceWorker.register('flutter_service_worker.js');
        });
    }
</script>
<script src="main.dart.js" type="application/javascript"></script>
</body>
</html>
```
#### Choosing the right keycloak-js version

The Keycloak client documentation recommends to use the same version of your Keycloak installation.

> A best practice is to load the JavaScript adapter directly from Keycloak Server as it will automatically be updated when you upgrade the server. If you copy the adapter to your web application instead, make sure you upgrade the adapter only after you have upgraded the server.

You can now use keycloak in your app.

## Note

You need to ensure you do not create multiple instances of keycloak. The example below uses a provider to ensure this.

Use the code provided below as an example and implement it's functionality in your application. In this process ensure
that the configuration you are providing matches that of your client as configured in Keycloak.

- Read more about keycloak client
  adapter [here](https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter)

# Example

## Please check and run the example code included in this repository.

A sample keycloak client is also included in the example codebase

```dart
late KeycloakService keycloakService;

void main() {
  keycloakService = KeycloakService(KeycloakConfig(
          url: 'http://localhost:8080', // Keycloak auth base url
          realm: 'sample',
          clientId: 'sample-flutter'));
  keycloakService.init(
    initOptions: KeycloakInitOptions(
      onLoad: 'check-sso',
      silentCheckSsoRedirectUri:
      '${window.location.origin}/silent-check-sso.html',
    ),
  );
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Keycloak Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: 'Flutter Keycloak demo'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key? key, this.title}) : super(key: key);

  final String? title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  KeycloakProfile? _keycloakProfile;

  void _login() {
    keycloakService.login(KeycloakLoginOptions(
      redirectUri: '${window.location.origin}',
    ));
  }

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((timeStamp) async {
      keycloakService.keycloakEventsStream.listen((event) async {
        print(event);
        if (event.type == KeycloakEventType.onAuthSuccess) {
          _keycloakProfile = await keycloakService.loadUserProfile();
        } else {
          _keycloakProfile = null;
        }
        setState(() {});
      });
      // if(keycloakService.authenticated){
      //   _keycloakProfile = await keycloakService.loadUserProfile(false);
      // }
      setState(() {});
    });
  }

  @override
  Widget build(BuildContext context) {
    // This method is rerun every time setState is called, for instance as done
    // by the _incrementCounter method above.
    //
    // The Flutter framework has been optimized to make rerunning build methods
    // fast, so that you can just rebuild anything that needs updating rather
    // than having to individually change instances of widgets.
    return Scaffold(
      appBar: AppBar(
        // Here we take the value from the MyHomePage object that was created by
        // the App.build method, and use it to set our appbar title.
        title: Text(widget.title!),
        actions: [
          IconButton(
                  icon: Icon(Icons.logout),
                  onPressed: () async {
                    await keycloakService.logout();
                  }),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'Welcome ${_keycloakProfile?.username ?? 'Guest'}',
              style: Theme
                      .of(context)
                      .textTheme
                      .headline4,
            ),
            ElevatedButton(
              onPressed: () async {
                print('refreshing token');
                await keycloakService.updateToken(1000).then((value) {
                  print(value);
                })
                        .catchError((onError) {
                  print(onError);
                });
              },
              child: Text(
                'Refresh token',
                style: Theme
                        .of(context)
                        .textTheme
                        .headline4,
              ),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _login,
        tooltip: 'Login',
        child: Icon(Icons.login),
      ),
    );
  }
}
```

In the example we have set up Keycloak to use a silent `check-sso`. With this feature enabled, your browser will not do a full redirect to the Keycloak server and back to your application, instead this action will be performed in a hidden iframe, so your application resources only need to be loaded and parsed once by the browser when the app is initialized and not again after the redirect back from Keycloak to your app.

To ensure that Keycloak can communicate through the iframe you will have to serve a static HTML asset from your application at the location provided in `silentCheckSsoRedirectUri`.

Create a file called `silent-check-sso.html` in the `assets` directory of your application and paste in the contents as seen below.

```html

<html>
<body>
<script>
  parent.postMessage(location.href, location.origin);
</script>
</body>
</html>
```

## Please check the example code included in this repository

If you want to know more about these options and various other capabilities of the Keycloak client is recommended to
read the [JavaScript Adapter documentation](https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter).

## Features and bugs

Please file feature requests and bugs at the [issue tracker][tracker].

[tracker]: https://github.com/gibahjoe/keycloak_flutter/issues

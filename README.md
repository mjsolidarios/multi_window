# multi_window

A simple example on how to have multiple windows on MacOS with Flutter and communicate between them. This is more than just an instance of the app.

## Getting Started

- Modify App Delagate

```swift
import Cocoa
import FlutterMacOS

@NSApplicationMain
class AppDelegate: FlutterAppDelegate, FlutterPlugin {
    var _controllers: [(String, FlutterViewController, NSWindow)] = []
    var controllers: [(String, FlutterViewController, NSWindow)] {
        get {
            let appDelegate = NSApp.delegate as! AppDelegate
            return appDelegate._controllers
        }
        set {
            let appDelegate = NSApp.delegate as! AppDelegate
            appDelegate._controllers = newValue
        }
    }


    override func applicationShouldTerminateAfterLastWindowClosed(_ sender: NSApplication) -> Bool {
        return true
    }

    override func applicationDidFinishLaunching(_ notification: Notification) {
        createNewWindow(key: "base", x: 300, y: 300, width: 1280, height: 720)
    }

    func createNewWindow(key: String, x: Double? = nil, y: Double? = nil, width: Int? = nil, height: Int? = nil) {
        let flutterController = FlutterViewController.init()
        let window = NSWindow()
        window.styleMask = NSWindow.StyleMask(rawValue: 0xf)
        window.backingType = .buffered
        window.contentViewController = flutterController
        if let screen = window.screen {
            let screenRect = screen.visibleFrame
            let newWidth = width ?? Int(screenRect.maxX / 2)
            let newHeight = height ?? Int(screenRect.maxY / 2)
            var newOriginX: CGFloat = (screenRect.maxX / 2) - CGFloat(Double(newWidth) / 2)
            var newOriginY: CGFloat = (screenRect.maxY / 2) - CGFloat(Double(newHeight) / 2)
            if (x != nil) { newOriginX = CGFloat(x!) }
            if (y != nil) { newOriginY = CGFloat(y!) }
            window.setFrameOrigin(NSPoint(x: newOriginX, y: newOriginY))
            window.setContentSize(NSSize(width: newWidth, height: newHeight))
        }
        RegisterGeneratedPlugins(registry: flutterController)
        controllers.append((key, flutterController, window))
        AppDelegate.register(with: flutterController.registrar(forPlugin: "AppDelegate"))
        let windowController = NSWindowController()
        windowController.contentViewController = window.contentViewController
        windowController.shouldCascadeWindows = true
        windowController.window = window
        windowController.showWindow(self)
    }

    public static func register(with registrar: FlutterPluginRegistrar) {
        let channelName = "window_controller"
        let channel = FlutterMethodChannel(name: channelName, binaryMessenger: registrar.messenger)
        let instance = AppDelegate()
        registrar.addMethodCallDelegate(instance, channel: channel)
    }

    func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
        switch call.method {
        case "createWindow":
            let args = call.arguments as? [String: Any]
            let width: Int? = args?["width"] as? Int
            let height: Int? = args?["height"] as? Int
            let x: Double? = args?["x"] as? Double
            let y: Double? = args?["y"] as? Double
            let key: String = args?["key"] as! String
            createNewWindow(
                key: key,
                x: x,
                y: y,
                width: width,
                height: height
            )
            result(true)
            break
        case "closeWindow":
            let args = call.arguments as? [String: Any]
            let key: String! = args?["key"] as? String
            result(closeWindow(_key: key))
            break
        case "windowCount":
            result(controllers.count)
            break
        case "keyIndex":
            let args = call.arguments as? [String: Any]
            let _key: String? = args?["key"] as? String
            let index = controllers.firstIndex(where: { (key, cont, win) -> Bool in
                if (key == _key) {
                    return true
                }
                return false
            })
            result(index ?? 0)
            break
        case "getWindowStats":
            let args = call.arguments as? [String: Any]
            let _key: String? = args?["key"] as? String
            let index = controllers.firstIndex(where: { (key, cont, win) -> Bool in
                if (key == _key) {
                    return true
                }
                return false
            })
            let controller = controllers[index!]
            let screen = controller.2.frame
            let origin = screen.origin
            let size = screen.size
            var _args: [String: Any?] = [:]
            _args["offsetX"] = Double(origin.x)
            _args["offsetY"] = Double(origin.y)
            _args["width"] = Double(size.width)
            _args["height"] = Double(size.height)
            result(_args)
        case "moveWindow":
            let args = call.arguments as? [String: Any]
            let _key: String? = args?["key"] as? String
            let x: Double = args?["x"] as! Double
            let y: Double = args?["y"] as! Double
            let index = controllers.firstIndex(where: { (key, cont, win) -> Bool in
                if (key == _key) {
                    return true
                }
                return false
            })
            let controller = controllers[index!]
            let screen = controller.2
            screen.setFrameOrigin(NSPoint(x: x, y: y))
            result(true)
        case "resizeWindow":
            let args = call.arguments as? [String: Any]
            let _key: String? = args?["key"] as? String
            let width: Double = args?["width"] as! Double
            let height: Double = args?["height"] as! Double
            let index = controllers.firstIndex(where: { (key, cont, win) -> Bool in
                if (key == _key) {
                    return true
                }
                return false
            })
            let controller = controllers[index!]
            let screen = controller.2
            screen.setContentSize(NSSize(width: width, height: height))
            result(true)
        case "lastWindowKey":
            let controller = controllers.last
            let _instanceKey = controller?.0;
            result(_instanceKey)
        default:
            result(FlutterMethodNotImplemented)
        }
    }


    func closeWindow(_key: String) -> Bool {
        do {
            let index = controllers.firstIndex(where: { (key, cont, win) -> Bool in
                if (key == _key) {
                    return true
                }
                return false
            })
            let controller = controllers[index ?? 0]
            controller.1.viewWillDisappear()
            controller.2.close()
            controller.1.viewDidDisappear()
            controllers.remove(at: index ?? 0)
             return true
        } catch {
             return false
        }
    }
}

```

- Modify main.dart

```dart
import 'dart:convert';
import 'dart:math';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  var _key = await WindowUtils.lastWindowKey();
  runApp(MyApp(key: ValueKey(_key)));
}

class MyApp extends StatelessWidget {
  const MyApp({ValueKey key}) : super(key: key);
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Flutter Demo',
      theme: ThemeData.light(),
      darkTheme: ThemeData.dark(),
      home: HomeScreen(windowKey: this.key),
    );
  }
}

class HomeScreen extends StatefulWidget {
  final ValueKey windowKey;

  const HomeScreen({Key key, @required this.windowKey}) : super(key: key);
  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  var _value = 0;
  List<String> _keys = [];

  @override
  void initState() {
    print("Key: ${widget.windowKey.value}");
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    final _current = widget.windowKey.value;
    return Scaffold(
      appBar: AppBar(
        title: Text('Home Screen ($_current)'),
        actions: <Widget>[
          IconButton(
            icon: Icon(Icons.timer),
            onPressed: () {
              WindowUtils.windowCount().then(
                (count) => print('Windows: $count'),
              );
              WindowUtils.keyIndex(_current).then(
                (index) => print('Index: $index'),
              );
              WindowUtils.getWindowStats(_current).then(
                (stats) => print('Stats: $stats'),
              );
            },
          ),
          IconButton(
            icon: Icon(Icons.desktop_windows),
            onPressed: () async {
              final _offset = await WindowUtils.getWindowOffset(_current);
              final _size = await WindowUtils.getWindowSize(_current);
              print("Offset: $_offset, Size: $_size");
              await WindowUtils.createWindow(
                WindowUtils.generateKey(),
                offset: (_offset.translate(_offset.dx + 50, _offset.dy + 50)),
                size: _size,
              );
              final _key = await WindowUtils.lastWindowKey();
              if (mounted)
                setState(() {
                  _keys.add(_key);
                });
            },
          ),
          IconButton(
            icon: Icon(Icons.close),
            onPressed: () => WindowUtils.closeWindow(_current),
          ),
        ],
      ),
      body: LayoutBuilder(
        builder: (context, dimens) => GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: (dimens.maxWidth / 200 ).round(),
            childAspectRatio: 9 / 16,
          ),
          itemCount: _keys.length,
          itemBuilder: (context, index) {
            final _item = _keys[index];
            return Padding(
              padding: const EdgeInsets.all(20.0),
              child: Card(
                elevation: 10,
                child: Column(
                  children: <Widget>[
                    ListTile(title: Text(_item)),
                    Row(
                      children: <Widget>[
                        Column(
                          children: <Widget>[
                            IconButton(
                              icon: Icon(Icons.add),
                              onPressed: () async {
                                final _size =
                                    await WindowUtils.getWindowSize(_item);
                                WindowUtils.resizeWindow(_item,
                                    Size(_size.width + 20, _size.height + 20));
                              },
                            ),
                            IconButton(
                              icon: Icon(Icons.remove),
                              onPressed: () async {
                                final _size =
                                    await WindowUtils.getWindowSize(_item);
                                WindowUtils.resizeWindow(_item,
                                    Size(_size.width - 20, _size.height - 20));
                              },
                            ),
                            IconButton(
                              icon: Icon(Icons.remove_circle_outline),
                              onPressed: () async {
                                WindowUtils.closeWindow(_item);
                              },
                            ),
                          ],
                        ),
                        Column(
                          children: <Widget>[
                            IconButton(
                              icon: Icon(Icons.arrow_upward),
                              onPressed: () async {
                                final _offset =
                                    await WindowUtils.getWindowOffset(_item);
                                WindowUtils.moveWindow(
                                    _item, Offset(_offset.dx, _offset.dy + 20));
                              },
                            ),
                            IconButton(
                              icon: Icon(Icons.arrow_downward),
                              onPressed: () async {
                                final _offset =
                                    await WindowUtils.getWindowOffset(_item);
                                WindowUtils.moveWindow(
                                    _item, Offset(_offset.dx, _offset.dy - 20));
                              },
                            ),
                            IconButton(
                              icon: Icon(Icons.arrow_back),
                              onPressed: () async {
                                final _offset =
                                    await WindowUtils.getWindowOffset(_item);
                                WindowUtils.moveWindow(
                                    _item, Offset(_offset.dx - 20, _offset.dy));
                              },
                            ),
                            IconButton(
                              icon: Icon(Icons.arrow_forward),
                              onPressed: () async {
                                final _offset =
                                    await WindowUtils.getWindowOffset(_item);
                                WindowUtils.moveWindow(
                                    _item, Offset(_offset.dx + 20, _offset.dy));
                              },
                            ),
                          ],
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            );
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: Center(child: Text(_value.toString())),
        onPressed: () async {
          if (mounted)
            setState(() {
              _value++;
            });
        },
      ),
    );
  }
}

class WindowUtils {
  static const _channel = const MethodChannel('window_controller');

  static final Random _random = Random.secure();

  static Future<bool> createWindow(String key,
      {Offset offset, Size size}) async {
    await _channel.invokeMethod('createWindow', {
      "key": key,
      "x": offset?.dx,
      "y": offset?.dy,
      "width": size?.width,
      "height": size?.height,
    });
    var _key = await lastWindowKey();
    return _key == key;
  }

  static Future<bool> resizeWindow(String key, Size size) async {
    return _channel.invokeMethod<bool>('resizeWindow', {
      "key": key,
      "width": size?.width,
      "height": size?.height,
    });
  }

  static Future<bool> moveWindow(String key, Offset offset) async {
    return _channel.invokeMethod<bool>('moveWindow', {
      "key": key,
      "x": offset?.dx,
      "y": offset?.dy,
    });
  }

  static Future<int> keyIndex(String key) {
    return _channel.invokeMethod<int>('keyIndex', {"key": key});
  }

  static Future<int> windowCount() {
    return _channel.invokeMethod<int>('windowCount');
  }

  static Future<bool> closeWindow(String key) {
    try {
      return _channel.invokeMethod<bool>('closeWindow', {"key": key});
    } catch (e) {
      return Future.value(false);
    }
  }

  static Future<String> lastWindowKey() {
    return _channel.invokeMethod<String>("lastWindowKey");
  }

  static Future<Map> getWindowStats(String key) {
    return _channel.invokeMethod<Map>("getWindowStats", {"key": key});
  }

  static Future<Size> getWindowSize(String key) async {
    final _stats = await getWindowStats(key);
    final w = _stats['width'] as double;
    final h = _stats['height'] as double;
    return Size(w, h);
  }

  static Future<Offset> getWindowOffset(String key) async {
    final _stats = await getWindowStats(key);
    final x = _stats['offsetX'] as double;
    final y = _stats['offsetY'] as double;
    return Offset(x, y);
  }

  static String generateKey([int length = 10]) {
    var values = List<int>.generate(length, (i) => _random.nextInt(256));
    return base64Url.encode(values);
  }
}

```
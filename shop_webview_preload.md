# SMAC&SHOP App: Preloaded Shop WebView Architecture

## 1. Architectural Overview

This document outlines the technical specification for implementing a preloaded WebView within the SMAC&SHOP application. To ensure instant access and seamless transitions when users click deep links or in-app URLs, the application utilizes an `IndexedStack` as the root navigation shell. 

This approach holds multiple widgets but only displays one at a time, keeping the state of all children intact. The Shop WebView is placed within this stack and fortified with `AutomaticKeepAliveClientMixin` to guarantee the web engine and its DOM remain active in memory while hidden behind the native application interface.

## 2. Core Components

* **Root Shell (`MainAppShell`)**: An `IndexedStack` that houses the `NativeAppTab` (Index 0) and the `ShopWebTab` (Index 1).
* **Navigation Controller (`AppNavigationController`)**: A global state manager (e.g., `ChangeNotifier`, Riverpod, or BLoC) responsible for holding the `currentIndex` and the `targetWebUrl`.
* **KeepAlive Web Tab (`ShopWebTab`)**: A Stateful widget implementing `AutomaticKeepAliveClientMixin`. It initializes the `WebViewController` once and listens for URL updates from the Navigation Controller.
* **Deep Link Interceptor**: A listener at the app entry point (e.g., `uni_links` or GoRouter deep link handling) that catches URLs destined for the Shop web interface.

## 3. Data and Interaction Flow

1. **Initialization**: Upon app launch, `MainAppShell` renders Index 0 (Native App). Index 1 (Shop) mounts silently in the background, initializing the `WebViewController` and fetching the base URL.
2. **In-App / Deep Link Trigger**: A user taps a Shop link natively or opens an external deep link.
3. **State Update**: The deep link interceptor catches the routing event and calls the global Navigation Controller with the destination URL.
4. **URL Injection**: The Navigation Controller updates `targetWebUrl` and changes `currentIndex` to 1.
5. **Surface and Navigate**: The `ShopWebTab` detects the property update (`didUpdateWidget`), commands the `WebViewController` to load the new URL, and the `IndexedStack` instantly surfaces the pre-rendered tab.

## 4. Performance & Memory Considerations

* **Memory Overhead**: Maintaining a heavy DOM in the background consumes system RAM. The application must monitor OS memory warnings (using `WidgetsBindingObserver`) and potentially call `_webViewController.clearCache()` or reload the view if memory pressure reaches a critical threshold.
* **JS Execution Constraints**: To conserve CPU cycles and battery life, JavaScript execution should be managed. Consider pausing execution via the controller when the web tab is out of focus, and resuming it immediately before switching the index back to the WebView.

## 5. Core Implementation Reference

```dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class AppNavigationController extends ChangeNotifier {
  int currentIndex = 0;
  String webUrl = '[https://shop.smac.com](https://shop.smac.com)';

  void navigateToShop(String url) {
    webUrl = url;
    currentIndex = 1;
    notifyListeners();
  }

  void navigateToNative() {
    currentIndex = 0;
    notifyListeners();
  }
}

class MainAppShell extends StatelessWidget {
  final AppNavigationController controller;

  const MainAppShell({super.key, required this.controller});

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: controller,
      builder: (context, child) {
        return IndexedStack(
          index: controller.currentIndex,
          children: [
            const NativeAppTab(),
            ShopWebTab(url: controller.webUrl),
          ],
        );
      },
    );
  }
}

class ShopWebTab extends StatefulWidget {
  final String url;
  
  const ShopWebTab({super.key, required this.url});

  @override
  State<ShopWebTab> createState() => _ShopWebTabState();
}

class _ShopWebTabState extends State<ShopWebTab> with AutomaticKeepAliveClientMixin {
  late final WebViewController _webViewController;

  @override
  bool get wantKeepAlive => true;

  @override
  void initState() {
    super.initState();
    _webViewController = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..loadRequest(Uri.parse(widget.url));
  }

  @override
  void didUpdateWidget(ShopWebTab oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.url != oldWidget.url) {
      _webViewController.loadRequest(Uri.parse(widget.url));
    }
  }

  @override
  Widget build(BuildContext context) {
    super.build(context); // Crucial for AutomaticKeepAliveClientMixin
    return Scaffold(
      body: SafeArea(
        child: WebViewWidget(controller: _webViewController),
      ),
    );
  }
}

class NativeAppTab extends StatelessWidget {
  const NativeAppTab({super.key});
  
  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Center(child: Text('Native SMAC App')),
    );
  }
}

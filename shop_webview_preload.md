# SMAC&SHOP App: Preloaded Shop WebView Architecture

## 1. Architectural Overview

This document outlines the technical specification for implementing a preloaded WebView within the SMAC&SHOP application. To ensure instant access and seamless transitions when users click deep links or in-app URLs, the application utilizes `go_router` with a `StatefulShellRoute`. 

This approach maintains separate stateful navigation branches for the native app and the Shop interface. The Shop WebView is placed within its own persistent branch, guaranteeing the web engine and its DOM remain active in memory while hidden behind the native application interface.

## 2. Core Components

* **Root Router (`GoRouter`)**: Configured with a `StatefulShellRoute.indexedStack` to manage distinct navigational branches while preserving their internal states.
* **Native Branch**: The standard native application UI tree.
* **Shop Branch (`ShopWebScreen`)**: A Stateful widget managing the `WebViewController`. It initializes the web engine once and updates the URL based on the routing state.
* **Deep Link Interceptor**: Managed inherently by `go_router`, parsing incoming URLs and routing to the correct branch seamlessly.

## 3. Data and Interaction Flow

1. **Initialization**: Upon app launch, the `StatefulShellRoute` mounts both branches. The Native branch is displayed, while the Shop branch initializes the `WebViewController` in the background.
2. **In-App / Deep Link Trigger**: A user taps a Shop link natively or opens an external deep link.
3. **Routing**: `go_router` processes the path and executes `context.go('/shop', extra: targetUrl)`.
4. **Surface and Navigate**: The router switches the active branch to the Shop interface. The `ShopWebScreen` processes the new URL, commanding the already-warmed `WebViewController` to load the destination, providing an instant visual transition.

## 4. Benefits of the Preloaded Approach

* **Eliminates Web Engine Cold Starts**: By instantiating the `WebViewController` at app launch, the significant overhead of warming up the webview is completely removed.
* **Instantaneous Transitions**: Users experience zero wait time for the base web environment to load when navigating between the native app and the Shop interface. 
* **Persistent DOM State**: Navigating away from the Shop and returning preserves the user's scroll position, session state, and DOM structure without requiring a full reload.
* **Simplified Deep Linking**: Using `go_router` centralizes deep link management, directly pushing specific URLs to the active web instance without relying on manual state managers.

## 5. Core Implementation Reference

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:webview_flutter/webview_flutter.dart';

final rootNavigatorKey = GlobalKey<NavigatorState>();
final nativeNavigatorKey = GlobalKey<NavigatorState>();
final shopNavigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: rootNavigatorKey,
  initialLocation: '/native',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        return ScaffoldWithNavBar(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          navigatorKey: nativeNavigatorKey,
          routes: [
            GoRoute(
              path: '/native',
              builder: (context, state) => const NativeAppScreen(),
            ),
          ],
        ),
        StatefulShellBranch(
          navigatorKey: shopNavigatorKey,
          routes: [
            GoRoute(
              path: '/shop',
              builder: (context, state) {
                final targetUrl = state.extra as String? ?? '[https://shop.smac.com](https://shop.smac.com)';
                return ShopWebScreen(url: targetUrl);
              },
            ),
          ],
        ),
      ],
    ),
  ],
);

class ScaffoldWithNavBar extends StatelessWidget {
  final StatefulNavigationShell navigationShell;

  const ScaffoldWithNavBar({
    super.key,
    required this.navigationShell,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: navigationShell.currentIndex,
        onTap: (index) {
          navigationShell.goBranch(
            index,
            initialLocation: index == navigationShell.currentIndex,
          );
        },
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
          BottomNavigationBarItem(icon: Icon(Icons.shopping_cart), label: 'Shop'),
        ],
      ),
    );
  }
}

class ShopWebScreen extends StatefulWidget {
  final String url;
  
  const ShopWebScreen({super.key, required this.url});

  @override
  State<ShopWebScreen> createState() => _ShopWebScreenState();
}

class _ShopWebScreenState extends State<ShopWebScreen> with AutomaticKeepAliveClientMixin {
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
  void didUpdateWidget(ShopWebScreen oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.url != oldWidget.url) {
      _webViewController.loadRequest(Uri.parse(widget.url));
    }
  }

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return Scaffold(
      body: SafeArea(
        child: WebViewWidget(controller: _webViewController),
      ),
    );
  }
}

class NativeAppScreen extends StatelessWidget {
  const NativeAppScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: ElevatedButton(
          onPressed: () {
            context.go('/shop', extra: '[https://shop.smac.com/promos](https://shop.smac.com/promos)');
          },
          child: const Text('Go to Shop Promos'),
        ),
      ),
    );
  }
}

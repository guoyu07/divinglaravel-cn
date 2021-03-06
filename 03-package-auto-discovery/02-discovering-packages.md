# 自动发现包

在查找和安装/更新不同的包时，Composer 会触发多个事件，可以订阅并运行一段代码、甚至是一个命令行执行文件。其中有一个有趣的事件叫做  `post-autoload-dump`，在 Composer 生成项目中所有自动加载的类的最终列表之后触发。然后 Laravel 就已经可以访问所有的类，并且应用程序已经准备好处理加载到其中的所有的包类。

从 Laravel 在 composer.json 文件中订阅这个事件开始发生了什么事情：

```json
"scripts": {
    "post-autoload-dump": [
        "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
        "@php artisan package:discover"
    ]
}
```

首先调用 `postAutoloadDump()` 静态方法，这个方法处理清除任何缓存服务或先前发现的包，然后运行 artisan 命令  `package:discover`，而这就是一切的开始。

## 包的发现

`Illuminate\Foundation\Console\PackageDiscoverCommand` 调用了 `Illuminate\Foundation\PackageManifest` 类的 `build()` 方法，这个类是 Laravel 发现包的地方。

在应用程序启动之前，`PackageManifest` 就被注册到容器之中，`Illuminate\Foundation\Application::registerBaseServiceProviders()` 这个方法直接创建在 Laravel 应用程序的新实例之后运行。

在 `build()` 中，Laravel 寻找 `vendor/composer/installed.json` 文件，这个文件是由 Composer 生成的，里面保存了对应所有由 Composer 安装的库的 `composer.json` 文件的完整映射。Laravel 对这个文件的内容进行映射，并搜索包含 `extra.laravel` 部分的包：

```json
"extra": {
    "laravel": {
        "providers": [
            "Barryvdh\\Debugbar\\ServiceProvider"
        ],
        "aliases": {
            "Debugbar": "Barryvdh\\Debugbar\\Facade"
        }
    }
}
```

首先它会收集这个部分的内容，然后查看 `composer.json` 文件下的 `extra.laravel.dont-discover`，再来决定是否需要自动发现哪些包或者所有的包。

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
}
```

> 你可以添加 `*` 到数组中来指示 Laravel 完全停止自动发现。

#### 所以现在 Laravel 收集有关这些包的信息

一旦获取了所有需要的信息，这将会被写入 `bootstrap/cache/packages.php` 文件中：

```php
<?php return array (
  'barryvdh/laravel-debugbar' =>
  array (
    'providers' =>
    array (
      0 => 'Barryvdh\\Debugbar\\ServiceProvider',
    ),
    'aliases' =>
    array (
      'Debugbar' => 'Barryvdh\\Debugbar\\Facade',
    ),
  ),
);
```

## 注册包

Laravel 在 HTTP 或终端内核启动时使用了两个引导程序：

* `\Illuminate\Foundation\Bootstrap\RegisterFacades`
* `\Illuminate\Foundation\Bootstrap\RegisterProviders`

首先使用 `Illuminate\Foundation\AliasLoader` 将所有的 facades 加载到应用程序中，现在发生的改变是，Laravel 会查看生成的 `packages.php` 文件并提取所有希望 Laravel 自动发现和注册的包的别名。Laravel 用了 `PackageManifest::aliases()` 方法来搜集信息。

同样，Laravel 也会在启动时注册服务提供者，`RegisterProviders` 引导程序调用 `Foundation\Application` 中的 `registerConfiguredProviders()` 方法，在这个方法中 Laravel 收集所有应该被自动注册的包的提供者并注册。


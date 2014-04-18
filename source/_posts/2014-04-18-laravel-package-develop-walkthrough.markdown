---
layout: post
title: "Laravel 插件开发攻略"
date: 2014-04-18 22:38:49 +0800
comments: true
categories: php
tags: framework,php,packages
---
最近在开发几个Laravel的包，发现开发文档中的东西说的太少，就在网上搜了很久，发现没有能用的中文版说明，于是打算自己写一个了……
看的比较不错的是这个[Laravel Workbench Walkthrough](https://github.com/orangehill/Laravel-Workbench-Walkthrough)，这个介绍有一部分是从这上面看到的
#准备
当然你得先有使用Laravel框架开发的基础了，然后就是插件的需要，Laravel是个很棒的框架，非常适合迅速快捷的开发，在不是特别庞大的系统面前我觉得非常容易而且好用，当然比较庞大的系统还没有做过，所以同样也是不了解。其次就是需要开发包的需求，有了包可以使我们的开发更加的快速，尤其是封装良好的Packages。而Laravel框架本身也提供了非常友好的包开发机制，使开发者们开发以及使用包都非常的容易，几个命令，一个包和自带的内容就非常容易的部署完成了。
#创建
作为一个包，首先就是需要作者的信息了，所以我们先打开
`app/config/workbench.php`
进行自己的名字和Email的配置，这些信息会在下面我们执行生成命令的时候自动写入到`composer.json`的文件中。
填写完自己的信息后我们就可以使用命令来自动生成一些文件，在根目录下执行下面的命令:
`php artisan workbench vendor/package --resources`
vendor是自己的厂商名或者组织或者个人的名字，package就是具体的一个包的名字，比如我就可以以`friparia/generator`来代表我生成一个叫做generator的包，这个包是friparia发布的。`--resources`这个参数是表示生成一系列资源文件夹，包括包内的controllers、migrations等等。详细内容可以到[文档](http://golaravel.com/docs/4.1/packages)了解
现在就可以看到`workbench/vendor/package`下面的文件结构和内容了，这表明我们已经创建好了自己的包，第一步完成了
#设置
既然我们创建好了自己的包，自然我们就希望很容易的去使用到自己所创建的这个包，下面来谈一谈如何设置
打开`app/config/app.php`添加一个自己的Service Provider(服务提供)
```php
'providers' => array(
    //---- Other providers
    'Vendor\Package\PackageServiceProvide',
    ),
```
这一步会在框架启动的时候去执行你的ServiceProvider去取得你的包中的内容，这样我们相当于注册了自己的服务
然后我们就可以开始编写自己包中的内容了，打开workbench/vendor/package/src/Vendor/Package/Package.php，进行自己逻辑的编写，如下所示
```php
<?php namespace Vendor\Package;
class Package {
  public static function hello(){
    return "Hello World!";
  }
}
```
然后将我们写好的类注册到IoC容器中，使得框架能够正确加载，打开workbench/vendor/package/src/Vendor/Package/PackageServiceProvider.php，在register()函数中添加以下内容
```php
public function register(){
  $this->app['package'] = $this->app->share(function($app){
      return new Package
  });
}
```
然后我们就可以在任何地方使用我们创建的包了，比如一条路由
```php
Route::get('hello', function(){
    echo \Vendor\Package\Package::hello();
});
```
访问这条路由我们就可以看到Hello World!了！
#Facade生成
觉得每次要输那么长一串再得到自己的包是不是感觉不爽，要是直接能`Package::hello()`多好，这时候我们就可以用到设计模式中的外观模式了，有关设计模式我想结合Laravel框架中的为例子再以后的blog中说，我觉得现在自己掌握的还不是特别好。下面就让我们进行Facade的创建
在`workbench/vendor/package/src/Vendor/Package`文件夹下创建一个`Facades`的文件夹
在这个`Facades`的文件夹中我们创建一个叫做`Package.php`的文件，这个文件的具体内容如下
```php
<?php namespace Vendor\Package\Facades;
use Illuminate\Support\Facades\Facade;
class Package extends Facade{
  protected static function getFacadeAccessor(){ return 'package';}
}
```
然后将写好的Facade注册到自己的包中，打开`workbench/vendor/package/src/Vendor/Package/PackageServiceProvider.php`，在register()函数中添加以下几行:
```php
$this->app->booting(function(){
    $loader = \Illuminate\Foundation\AliasLoader::getInstance();
    $loader->alias('Package', 'Vendor\Package\Facades\Package');
});
```
或者你也可以直接写在`app/config/app.php`的alias数组中，我觉得其实写在Service Provider中会更好一点
#测试
这时候我们就可以很容易的使用我们自己的包了，路由如下所示
```php
Route::get('hello', function(){
    echo Package::hello();
});
#完成
Ok,一个包就这样创建并使用了，通过包的开发方式，我们可以更容易的去复用自己的代码，减少自己的工作量。当然这才是包开发的基础，我会在下次写有关包命令开发的攻略~

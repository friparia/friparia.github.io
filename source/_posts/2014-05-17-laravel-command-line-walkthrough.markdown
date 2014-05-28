---
layout: post
title: "Laravel 命令行开发入门"
date: 2014-05-17 23:08:46 +0800
comments: true
categories: laravel,php
---
上一篇我们试过了Laravel中的包开发，我们可以创建自己的包，并且复用他们，来很快的构建我们的应用，这一次我们来简要介绍一下Laravel中另一个强大的功能，Artisan命令行，当然，文档是一切的基础[中文文档](http://www.golaravel.com/docs/4.1/commands/)，这个可能稍微有点过时，如生成使得`--bench`的参数在新版本中就已经去掉了，所以看[英文文档](https://www.laravel.com/)是一个不错的选择
<!--more-->

#准备
所有的新东西都得需要曾经的基础，首先的基础就是Laravel的开发的基础了，如果使用过命令行，比如数据库迁移命令的话会更好的理解。应用命令行我们能够更好更快的构建我们的应用，或者对我们的应用进行一些命令行的操作。

#创建
我们可能有两种不同的命令行开发，一种是在应用中的，可以简单的理解为我们的代码都是写在`app/commands`这个目录下的，另一种是在开发的包中的，也可以理解为我们所写的命令行代码是在`workbench/vendor/package/src/Vendor/Package/`文件中的，区别我们稍后会提到。

运行`php artisan command:make YourCommand`就可以创建一个命令了，默认是创建在你的应用中，即`app/commands`这个目录中，而如果我们需要创建在自己开发的包中的命令的话，我们就需要加上`--path=workbench/vendor/package/src/Vendor/Package`这个参数，这样我们创建的命令才可以随着包的发布而进行使用了。

#实现
用命令行就可以搭一个很不错的架子出来，我们可以打开生成的`workbench/vendor/package/src/commands/YourCommand.php`这个文件，来完成我们具体的命令实现。
由于`--bench`参数不再支持，所以我们手动在开始加入
```php
namespace Vendor\Package;
```

这一行，使得我们的包能够找到所对应的命令。

在`YourCommand`这个类中我们可以看到这几个方法:

 - `__consruct()`，当然这个是默认的构造函数没什么可说的
 - `fire()`， 这个函数是在我们运行命令的时候所执行的函数，即我们需要实现的具体逻辑代码就是写在这个里面的
 - `getArguments()`和`getOptions()`是得到命令的参数和选项，如果有参数设置的需要，我们可以通过`$this->argument('name')`和`$this->option('name')`来获得我们所需要的参数。
同时还有两个属性:
 - `$name`，是这个命令的名字，即在命令行中所敲的
 - `$description`，是这个命令的描述，会对这个命令进行一个简要的描述

了解了这些之后我们就可以完成我们的`fire()`函数了，我们可能用得到的输出函数`info`,`comment`,`question`,`error`等可以以一种简单的方式使用。
比如我们在刚开始询问是否生成，就可以写下以下代码:

```php
if($this->confirm('确定生成?[Y|N]){
  $this->info('生成中....');
}
```

这样我们就能够写出我们的具体的命令了，下面我们以一个压缩assets的命令例子来详细叙述整个包命令开发的过程

#例子

首先按照[包生成](http://www.friparia.com/blog/2014/04/18/laravel-package-develop-walkthrough/)来生成一个assetcompress的包，然后执行`command:make`命令在包内生成一个compress css的命令，我们执行的是`php artisan command:make CompressCssCommand --path=workbench/friparia/assetcompresser/src/Friparia/Assetcompresser`

然后打开`workbench/vendor/package/src/commands/`文件夹中刚才生成的文件，按照上述加入namespace我们来设定我们这个命令和描述
```php CompressCssCommand.php
protected $name = 'compress:css';
protected $description = 'Compress your css files to save your network traffic. No parameter means compress all css files in public/css in one file and "--file" means compress specify file';
```

这样当我们运行`php artisan`时我们就可以看到我们的命令以及说明了，然后完成`fire()`方法，我们先完成不加参数的方法(此时需要先将`getArguments()`和`getOptions()`两个方法注释掉)

```php CompressCssCommand.php
	public function fire()
	{
        if($this->confirm('Compress all css files?[Yes/No]')){
            $this->line('');
            $this->info('compressing ...');
            $cssdir = public_path().'/css';
            $compressed_filename = $cssdir.'/'.time().'_compressed.min.css';
            if(is_dir($cssdir)){
                if($dh = opendir($cssdir)){
                    $minified = "";
                    while($file = readdir($dh)){
                        if($file != '.' && $file != '..'){
                            $path_parts = pathinfo($file);
                            if($path_parts['extension'] == 'css'){
                                $minified .= Assetcompresser::minifycss($cssdir.'/'.$file);
                            }
                        }
                    }
                    file_put_contents($compressed_filename, $minified);
                    $this->line('');
                    $this->info('css has compressed! The file'.$compressed_filename.' generated');
                }else{
                    $this->error('Cannot open '.$cssdir);
                }
            }else{
                $this->error($cssdir."does not exist");
            }
        }
	}

```

我们有使用到`Assetcompresser::minifycss()`来压缩css，即我们这个包的功能，所以我们创建`Assetcompresser.php`文件，完成所需功能:

```php Assetcompresser.php
<?php namespace Friparia\Assetcompresser;
class Assetcompresser{
    /**
     * Minifies css files
     * @param  string  $path  input css file
     * @return string         compressed string
     */
    static public function minifycss($path){
        if(file_exists($path)){
            if($contents = file_get_contents($path)){
                $contents = trim($contents);
                $contents = str_replace("\r\n", "\n", $contents);
                $search = array("/\/\*[\d\D]*?\*\/|\t+/", "/\s+/", "/\}\s+/");
                $replace = array(null, " ", "}\n");
                $contents = preg_replace($search, $replace, $contents);
                $search = array("/\\;\s/", "/\s+\{\\s+/", "/\\:\s+\\#/", "/,\s+/i", "/\\:\s+\\\'/i", "/\\:\s+([0-9]+|[A-F]+)/i");
                $replace = array(";", "{", ":#", ",", ":\'", ":$1");
                $contents = preg_replace($search, $replace, $contents);
                $contents = str_replace("\n", null, $contents);
                return $contents;  
            }else{
                return "";
            }

        }else{
            return "";
        }
    }
}
```

最后别忘了在ServiceProvider里注册我们的命令，打开`AssetcompresserServiceProvider.php`，在`register()`方法中添加:
```php CompressCssCommand.php
        $this->app['command.compress.css'] = $this->app->share(function($app){
            return new CompressCssCommand($app);
        });
        $this->commands(
            'command.compress.css'
        );
```
以使得我们的命令能够正确的被执行，执行`php artisan compress:css`就可以得到我们压缩后的css文件。

到此我们的不加参数的命令行的命令开发就完成了，在下一篇博客中我会把加参数的命令开发完成。并且利用空闲时间把这整个包完成~

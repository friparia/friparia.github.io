---
layout: post
title: "Laravel 带参数的命令行开发"
date: 2014-06-16 17:27:33 +0800
comments: true
categories: laravel,php
---
上一次我们简要介绍了Laravel中命令行的不带参数或者选项的开发，今天我们要完善我们上一次的压缩css的工作，使得我们可以压缩我们指定的css文件，如以下命令所示
<!--more-->
`php artisan compress:css --file=public/css/example.css`或者`php artisan compress:css -f public/css/example.css`

这就我们需要来了解上一次被我们注释掉的`getArguments()`和`getOptions()`方法，这两个方法是由Laravel提供的，为了自定义自己的参数或者选项而设置的。

##getArguments
这个函数返回一个数组，数组中的元素定义为
```php
array($name, $mode, $description, $defaultValue)
```
`$name`是参数的名字，`$mode`是其模式，这个函数有这两个值:

 - `InputArgument::REQUIRED`，表示这个参数是必须的，如果在使用命令的时候没有输入，则会报错 
 - `InputArgument::OPTIONAL`，这个则表示是可选的，在使用命令时可以不用。
`$description`是这个参数的描述，在`--hlep`时会显示其描述，`$defaultValue`顾名思义即为这个参数的默认值

##getOptions
要想实现开头所说的效果，于是我们就着重使用getOptions这个函数，同样和`getArguments`返回相同的数组，不过数组中的元素不一样
```php
array($name, $shortcut, $mode, $description, $defaultValue)
```
多了一个`$shortcut`选项，在我们的例子中我们可以将其设置成为`-f`，而其模式`$mode`也有不同，如下所示:

 - `InputOption::VALUE_REQUIRED`，表明这个参数是必须的
 - `InputOption::VALUE_OPTIONAL`，表明这个参数是可选的
 - `InputOption::VALUE_IS_ARRAY` ，表明这个参数是一个数组，使用命令时呈现这种形式`--option=value1 --option=value2`
 - `InputOption::VALUE_NONE`，表明只是一个开关而没有具体参数，如同我们在编译安装时的`--with-xxx`

了解了我们的函数后我们就可以完成我们的方法了:
```php CompressCssCommand.php
	/**
	 * Get the console command options.
	 *
	 * @return array
	 */
	protected function getOptions()
	{
		return array(
			array('file', '-f', InputOption::VALUE_OPTIONAL, 'Compress an input css file', null),
		);
	}
```

完成这两个函数后我们可以通过这样的方法得到我们的选项或者参数值
```
$this->argument('file'); //获取$name为"file"的参数的值
$this->argument(); //获得所有参数的数组
$this->option('file'); //获取$name为"file"的选项的值
$this->option(); //获得所有选项的数组
```

然后我们完成最后的逻辑，修改我们的`fire()`函数为:
```php CompressCssCommand.php
	public function fire()
	{
        $inputfile = $this->option('file');
        if(is_null($inputfile) && $this->confirm('Compress all css files?[Yes/No]')){
          // 上一次我们书写的代码
        }else if($this->confirm('Compress this '.$inputfile.' css files?[Yes/No]')){
            $this->info('compressing ...');
            $file = $inputfile;
            $cssdir = public_path().'/css';
            $compressed_filename = $cssdir.'/'.time().'_compressed.min.css';
            $path_parts = pathinfo($file);
            if($path_parts['extension'] == 'css'){
                $minified = Assetcompresser::minifycss(base_path().'/'.$file);
            }
            file_put_contents($compressed_filename, $minified);
            $this->line('');
            $this->info('css has compressed! The file'.$compressed_filename.' generated');
        }
	}
```

#总结
这个asset压缩的包我会上传到我的github和packagist上去，并且将其功能完善得更好，欢迎大家拍砖……

这样我们的Laravel的命令行攻略也算介绍完了，个人觉得主要是因为文档中没有一个确定的例子来带着做有可能在第一次使用到的时候遇到麻烦，所以写了这几篇，也算是练练手，非常感谢[@耗子飞飞飞_BYR](http://weibo.com/byrwdjwxh)给予的指导~接下来可能会准备深入了解一下Laravel的框架，然后在更深的层面上写一点东西。

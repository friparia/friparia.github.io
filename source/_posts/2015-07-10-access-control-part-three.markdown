---
layout: post
title: "信息系统中的权限控制(三)：基于资源的权限控制"
date: 2015-07-10 13:37:46 +0800
comments: true
categories: web, framework
---

前两篇文章介绍了基于用户的权限控制，这种权限的控制是将用户进行划分，对系统中的操作进行一定的权限控制。但是在日常应用中，我们可能遇到这样一种场景：不同的用户分管不同的系统资源，比如Linux中的文件系统、比如淘宝店铺中的商品只能归所拥有的店主管理。在这种情形下就无法针对用户进行权限控制，而是应当具体到每个实体资源上去。

<!--more-->

#基于资源的权限控制

文章中对基于用户的权限控制的具体实现涉及到的比较多，而这篇可能就是提出一种思路或者想法，并没有一种特别好的实现方式。尤其是在Web应用下具体的某个资源非常难以精确抽象，不像Linux系统中我们可以很容易的抽象出目录和文件这两个资源，并进行权限控制)。

因为Web应用可以做为资源的实体会非常多，而对资源的控制要求又比较强。当然，最简单的就是把他当做产品的具体业务逻辑，根据具体的情形书写具体的代码，这个方法如果管理比较复杂的权限控制的话就会产生大量烦杂的代码，对整个系统不很友好。

如果在系统中使用了ORM，并且运用了一定的框架，那么就可以将具体的资源抽象到每个Model上面去，然后在Model的基类来完成具体的权限控制代码。一般情形是对查找出来的结果集进行过滤，然后将过滤后的结果集返回。或者在查询的时候添加一定的条件。下面给一个phalcon框架的filter例子:

```php
<?php
public function afterFetch(){
    if(!Auth::canOperate(get_called_class(), $this->id)){
        die("你没有权限操作该对象!");
    }
}

public static function find($parameters = null){
    $where = Auth::resourceWhere(get_called_class());
    if(!$where){
        return parent::find($parameters);
    }
    if(is_array($parameters)){
        if(isset($parameters['conditions'])){
            if($parameters['conditions'] == ""){
                $parameters['conditions'] = $where;
            }else{
                $parameters['conditions'] .= " and ".$where;
            }
        }else{
            if($parameters[0] == ""){
                $parameters[0] = $where;
            }else{
                $parameters[0] .= " and ".$where;
            }
        }
    }else{
        if($parameters == ""){
            $parameters = $where;
        }else{
            $parameters .= " and ".$where;
        }
    }
    return parent::find($parameters);
}
```

如果没有使用ORM，只是用纯SQL语句取结果集的话，可能就需要针对具体的SQL进行拼接，拼接出权限控制后的SQL语句。

不管什么样的方式，基于资源的权限控制都不是一件很好处理的事情，尤其是当需求不断的叠加时，会导致权限控制更加复杂，更加麻烦。

#总结

只要有信息的地方，就会有权限控制，权限控制也是安全中的一项非常重要的一环，因为权限而导致的漏洞数不胜数，而且每个漏洞的危害都巨大无比，这里有个微信的资源控制的[漏洞](http://www.wooyun.org/bugs/wooyun-2010-090898)，赤裸裸的抢钱啊。权限控制说起来容易，但是做起来，如何去设计，如何去兼容到整个系统中，如何将对其他模块的影响降到最低，是一件非常棘手的事情。希望我的这几篇博客给大家带来一点有关权限控制的思路。

#P.S.

有关权限的东西也写的差不多了，现在做的系统的权限控制的地方也很稳定了(除了基于资源的权限)，刚开始觉得自己还有挺多东西要说的，写的时候才感觉到捉襟见肘，还是要多写写多练练啊。。

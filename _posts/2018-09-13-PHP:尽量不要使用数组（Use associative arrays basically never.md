---
layout: post
title: Use associative arrays basically never
---

# PHP:尽量不要使用数组（Use associative arrays basically never
对于一些简单的一次性内部数据结构，大多数情况下我们会使用一个PHP关联数组来处理。少数人可能会在备注块里备注一下，但大部分人都不会这么做。那么问题来了，为什么大部分都不通过定义一个类来处理这种情况呢？是因为使用对象实际上比使用数组更难，抑或是花费更多的时间呢？接下来我们一起来测试一下，到底使用数组和使用对象哪个更好。

测试环境：
* I5-7200u (2.71 GHz Intel Core i5)
* 8 GB 2400 MHz DDR4
* 停用xdebug
* PHP 7.1.16
* 测试结果可能因为机器不同而有所波动，但是对比结果应该是比较接近的

## 基础数据
基础的测试脚本代码如下
```
<?php
declare(strict_types=1);

error_reporting(E_ALL | E_STRICT);

const TEST_SIZE = 1000000;

$list = [];
$start = $stop = 0;

$start = microtime(true);

for ($i = 0; $i < TEST_SIZE; ++$i) {
  $list[$i] = [
    'a' => random_int(1, 500),
    'b' => base64_encode(random_bytes(16)),
  ];
}

ksort($list);

usort($list, function($first, $second) {
  return [$first['a'], $first['b']] <=> [$second['a'], $second['b']];
});

$stop = microtime(true);
$memory = memory_get_peak_usage();
printf("Runtime: %s\nMemory: %s\n", $stop - $start, $memory);
```

脚本里面，我们创建了一百万个数组，每个数组都是一个关联数组，包含一个整形和短字符。这种数据结构我们在日常开发中应该经常遇到，一般用于一个类中的私有属性中，并且只能在类中访问。（但是一般没有这种说法，想怎么操作就怎么操作，这种不好的 code style 确实需要注意）

这个测试的目的主要是测试处理这种数据的时间，所以脚本中对数据排序了两次，一次是根据数据的 key 来排序，还有一次根据数组的 value 来排序

另外，还有一个针对序列化后存储大小的测试，序列化是比较常见的手段，而且通常是存储到数据库的（只知道操作数据花费的时间好像没什么用，毕竟处理数据的时间少，但是占的空间大的话实用性不大），所以知道序列化后的大小也是很有用的，脚本如下：

```
<?php
declare(strict_types=1);

error_reporting(E_ALL | E_STRICT);

const TEST_SIZE = 1000000;

$list = [];
$start = $stop = 0;

$start = microtime(true);

for ($i = 0; $i < TEST_SIZE; ++$i) {
  $list[$i] = [
    'a' => random_int(1, 500),
    'b' => base64_encode(random_bytes(16)),
  ];
}

$ser = serialize($list);
unserialize($ser);

$stop = microtime(true);
$memory = memory_get_peak_usage();
printf("Runtime: %s\nMemory: %s\nSize: %s\n", $stop - $start, $memory, strlen($ser));
```

为了减少误差，建议多跑几次脚本，然后求平均数来作为数据的样本。
下面是跑了三次的记录：

数组（排序）

| Run   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-: |
| 1 | 14.376363992691   | 541412760 |
| 2 | 14.834184885025   | 541412760 |
| 3 | 15.238895893097   | 541412760 |
| Avg | 14.816481590271 | 541412760 | 

数组（序列化）

| Run   |      Runtime (s)      |    Memory (bytes)   | Size |
| :-: | :-: | :-: | :-: |
| 1 | 6.0441019535065   | 1164346656 | 68672256 |
| 2 | 6.1000030040741   | 1164346656 | 68673829 |
| 3 | 6.0260410308838   | 1164346656 | 68672328 |
| Avg | 6.056715329488133 | 1164346656 | 68672804 |

## stdClass
接下来我们试试 stdClass 对象。 以下是测试脚本
```
# ...
for ($i = 0; $i < TEST_SIZE; ++$i) {
  $o = new stdclass();
  $o->a = random_int(1, 500);
  $o->b = base64_encode(random_bytes(16));
  $list[$i] = $o;
}

ksort($list);

usort($list, function($first, $second) {
	return [$first->a, $first->b] <=> [$second->a, $second->b];
});
# ...
```

stdClass（排序）

| Run   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-: |
| 1 | 17.793854236603   | 589793512 |
| 2 | 17.036823987961   | 589793512 |
| 3 | 17.020701169968   | 589793512 |
| Avg | 17.283793131510667 | 589793512 | 

stdClass（序列化）

| Run   |      Runtime (s)      |    Memory (bytes)   | Size |
| :-: | :-: | :-: | :-: |
| 1 | 7.754655122757   | 1274116784 | 81672617 |
| 2 | 7.7843279838562   | 1274116784 | 81673200 |
| 3 | 7.7281250953674   | 1274116784 | 81673173 |
| Avg | 7.755702733993533 | 1274116784 | 81672997 |

相比数组，stdclass 各方面都都有所提升，但是在这里提升不是一件好事。显然我们希望的是更低内存和更快的运行时间和占用更小的空间，所以还是放弃使用 stdClass 吧。

## 公共属性的对象
看到上面使用对象竟然比数组更差，是不是觉得平时用的没有错？ 这你就错了，下面我们开始真正的测试。这次我们使用对象的公共属性来存储数据。
```
class Item
{
  public $a;
  public $b;
}

for ($i = 0; $i < TEST_SIZE; ++$i) {
  $o = new Item();
  $o->a = random_int(1, 500);
  $o->b = base64_encode(random_bytes(16));
  $list[$i] = $o;
}

ksort($list);

usort($list, function($first, $second) {
  return [$first->a, $first->b] <=> [$second->a, $second->b];
});
```

对象的公共属性（排序）

| Run   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-:|
| 1 | 13.319245100021   | 253794048 |
| 2 | 13.293720006943   | 253794048 |
| 3 | 13.34242606163   | 253794048 |
| Avg | 13.318463722864667 | 253794048 | 

对象的公共属性（序列化）

| Run   |      Runtime (s)      |    Memory (bytes)   | Size |
| :-:| :-: | :-: | :-: |
| 1 | 7.9552869796753   | 1326117128 | 77673389 |
| 2 | 7.8424470424652   | 1326117128 | 77672440 |
| 3 | 7.983806848526   | 1326117128 | 77672658 |
| Avg | 7.927180290222167 | 1326117128 | 77672829 |

瞬间爆炸，从这次的测试中，我们可以看到运行时间有所下降，但是内存减少了将近一倍。但是在序列化中和 stdClass 表现基本相同。

## 私有属性的对象
很多人其实不喜欢使用公共属性而是使用私有受保护的方法或属性。让我们一起看看它会有怎样的表现
```
class Item
{
  protected $a;
  protected $b;

  public function __construct(int $a, string $b)
  {
    $this->a = $a;
    $this->b = $b;
  }

  public function a() : int { return $this->a; }
  public function b() : string { return $this->b; }
}

for ($i = 0; $i < TEST_SIZE; ++$i) {
  $list[$i] = new Item(random_int(1, 500), base64_encode(random_bytes(16)));
}

ksort($list);

usort($list, function(Item $first, Item $second) {
  return [$first->a(), $first->b()] <=> [$second->a(), $second->b()];
});
```

对象的私有属性（排序）

| Run   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-: |
| 1 | 15.906311988831   | 253795464 |
| 2 | 16.583369016647   | 253795464 |
| 3 | 16.628217935562   | 253795464 |
| Avg | 16.372632980346667 | 253795464 | 

对象的私有属性（序列化）

| Run   |      Runtime (s)      |    Memory (bytes)   | Size |
| :-: | :-: | :-: | :-: |
| 1 | 8.0769779682159   | 1332114912 | 83672798 |
| 2 | 8.0331327915192   | 1332114912 | 83672633 |
| 3 | 8.2951149940491   | 1332114912 | 83672663 |
| Avg | 8.1350752512614 | 1332114912 | 83672698 |

显而易见，在类中添加方法确实会使运行时间减少一点。内存方面和公共属性的时候相差不大。不知道什么原因导致序列化的时候变慢了一点，而且占得空间变大了，但是不显著。（我觉得是因为 “private” 相比 “public” 变长了导致的

## 匿名类
很多人对定义类依然很抗拒，他们觉得定义类会使程序变慢，并且要花费更多的时间。或许他们是从文件数量变多的觉度考虑的，所以接下来我们使用匿名类来“消除”这部分人的疑惑。这里我们只做公共属性的测试了，因为上面已经对比过所以这里就不重复了。

```
for ($i = 0; $i < TEST_SIZE; ++$i) {
  $o = new class(random_int(1, 500), base64_encode(random_bytes(16))) {
    public $a;
    public $b;

    public function __construct(int $a, string $b)
    {
      $this->a = $a;
      $this->b = $b;
    }
  };
  $list[$i] = $o;
}
```

匿名类（排序）

| Run   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-: |
| 1 | 13.534195899963   | 253794728 | 
| 2 | 13.446998119354   | 253794728 | 
| 3 | 13.605525970459   | 253794728 | 
| Avg | 13.528906663258667 | 253794728 | 

匿名类和类的数据比较相似，由于匿名类无法序列化，因此这里只做排序的比较。明显，你可以不用定义一个类，也能做到优化的效果。需要的可能是多敲一点代码，但明显不是每个人都愿意和我一样多敲那些代码。

## 总结
这里是所有数据的总结，用百分比来表示各个测试相对样本（数组）的差值，在这里要注意的是，➖才是好的。

### 排序
| Technique   |      Runtime (s)      |    Memory (bytes)   |
| :-: | :-: | :-: |
| 数组 |  14.816   |  541412760 |
| stdClass |  17.283  |  589793512 |
| 公有属性 |  13.318   |  253794048 |
| 私有属性 | 16.372 |  253795464 | 
| 匿名类 | 13.528 |  253794728 | 

### 序列化
| Technique   |      Runtime (s)      |    Memory (bytes)   | Size |
| :-: | :-:| :-:| :-: |
| 数组 |  6.05   |  1164346656 | 68672804 |
| stdClass |  7.75   |  1274116784 | 81672997  |
| 公有属性 |  7.92   |  1326117128 | 77672829  |
| 私有属性 | 8.13 |  1332114912 | 83672698  |  

首先，我们处理的是一百万为基数的样本，数据量少了，对比可能就没那么明显了。但是每当写程序的时候，都要避免陷入这些陷阱中。因为谁知道以后这些程序的使用量呢？

第一样总结出的是，如果你考虑的是序列化的话，使用数据可能是最好的选择。

第二就是，无论如何，避免使用 stdClass ，因为它的表现在各方面都比较差。

在我能想到的几乎所有其他情况中，使用类更好。 类的内存使用量是数组的一半。 PHP引擎在预先知道数据结构将会是多大的时候可以做相应的优化，并在内存消耗方面带来巨大的回报。 使用类的速度也快了10％。 唯一的缺点是当时间，内存和存储大小增加时尝试序列化的时候，需要增加相当大的内存或空间成本。 

对于使用共有属性或者私有属性，我的理解是：他们都提供了一个关系更清晰，可阅读性更高，更灵活的方法，但同时确实比使用数组时占更大的 CPU。
但我们在日常使用中，不需要因为这点而多使用共有属性而不使用私有属性，在使用量低的时候其实是没什么所谓的。这叫看情况，只要能对你的程序有帮助的话，可以大胆使用私有属性并定义相关的方法。

作为另一个考虑因素，现在大型框架基于插件信息生成代码并将其存储在磁盘上而不是作为序列化字符串存储，而是作为生成的PHP类存储，然后可以像其他任何一样加载。 （想想依赖注入容器，事件调度程序，可以注册模板插件的主题系统等）。在这种情况下，序列化点没有实际意义，你绝对没有理由不使用类。 在编译代码中生成一个大的嵌套关联数组是不可原谅的浪费，不要那样做。

尽管这次测试是在PHP 7.1 上测试的，但是相信对于PHP7.0 及以后的版本来说，结果应该是差不多的。但是PHP5.0 的话就不好说了。

Author: [crell](https://steemit.com/@crell)
原文：https://steemit.com/php/@crell/php-use-associative-arrays-basically-never


#php

# 自定义进程

在webman中你可以像workerman那样自定义监听或者进程。

> **注意**
> windows用户需要使用 `php windows.php` 启动webman才能启动自定义进程。

## 自定义监听例子

新建 `process/Pusher.php`
```php
<?php
namespace process;

use Workerman\Connection\TcpConnection;

class Pusher
{
    public function onConnect(TcpConnection $connection)
    {
        echo "onConnect\n";
    }

    public function onWebSocketConnect(TcpConnection $connection, $http_buffer)
    {
        echo "onWebSocketConnect\n";
    }

    public function onMessage(TcpConnection $connection, $data)
    {
        $connection->send($data);
    }

    public function onClose(TcpConnection $connection)
    {
        echo "onClose\n";
    }
}
```
> 注意：所有onXXX属性均为public

在`config/process.php`中添加如下配置
```php
return [
    // ... 其它进程配置省略
    
    // websocket_test 为进程名称
    'websocket_test' => [
        // 这里指定进程类，就是上面定义的Pusher类
        'handler' => process\Pusher::class,
        'listen'  => 'websocket://0.0.0.0:8888',
        'count'   => 1,
    ],
];
```

## 自定义非监听进程例子
新建 `process/TaskTest.php`
```php
<?php
namespace process;

use Workerman\Timer;
use support\Db;

class TaskTest
{
  
    public function onWorkerStart()
    {
        // 每隔10秒检查一次数据库是否有新用户注册
        Timer::add(10, function(){
            Db::table('users')->where('regist_timestamp', '>', time()-10)->get();
        });
    }
    
}
```
在`config/process.php`中添加如下配置
```php
return [
    // ... 其它进程配置省略
    
    'task' => [
        'handler'  => process\TaskTest::class
    ],
];
```

> 注意：listen省略则不监听任何端口，count省略则进程数默认为1。

## 配置文件说明

一个进程完整的配置定义如下：
```php
return [
    // ... 
    
    // websocket_test 为进程名称
    'websocket_test' => [
        // 这里指定进程类
        'handler' => process\Pusher::class,
        // 监听的协议 ip 及端口 （可选）
        'listen'  => 'websocket://0.0.0.0:8888',
        // 进程数 （可选，默认1）
        'count'   => 2,
        // 进程运行用户 （可选，默认当前用户）
        'user'    => '',
        // 进程运行用户组 （可选，默认当前用户组）
        'group'   => '',
        // 当前进程是否支持reload （可选，默认true）
        'reloadable' => true,
        // 是否开启reusePort （可选，此选项需要php>=7.0，默认为true）
        'reusePort'  => true,
        // transport (可选，当需要开启ssl时设置为ssl，默认为tcp)
        'transport'  => 'tcp',
        // context （可选，当transport为是ssl时，需要传递证书路径）
        'context'    => [], 
        // 进程类构造函数参数，这里为 process\Pusher::class 类的构造函数参数 （可选）
        'constructor' => [],
    ],
];
```

## 总结
webman的自定义进程实际上就是workerman的一个简单封装，它将配置与业务分离，并且将workerman的`onXXX`回调通过类的方法来实现，其它用法与workerman完全相同。

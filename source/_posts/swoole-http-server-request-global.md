---
title: swoole_http_server中request的全局对象
date: 2017-11-12 19:16:46
tags: [swoole,php]
---
```php
<?php
/**
 * http.php
 *
 * @author      shengyayun<719048774@qq.com>
 * @since       2017/11/12 18:39
 */
$http = new swoole_http_server("127.0.0.1", 9501);
$http->set(
    [
        'worker_num' => 2,//开启2个worker进程
        'dispatch_mode' => 1,//request轮询分配给每个worker进程
    ]
);
$http->on('request', function ($request, $response) {
    App::init($request, $response);//必须放在最前面，不然下一个request可能取到上一个request的gpc
    $result = (new A())->run();//调用实现类
    $response->end("子进程(" . getmypid() . ")完成请求的处理：" . $result);//request请求是在worker进程执行的
});
$http->start();
echo "主进程(" . getmypid() . ")已启动";

/**
 * 存放全局对象，进程内唯一，进程内全局
 */
class App
{
    /**
     * @var array 进程内的全局对象GPC
     */
    public static $GPC;

    public static function init($request, $response)
    {
        foreach (['get', 'post', 'cookie', 'files'] as $attr) {
            self::$GPC[$attr] = property_exists($request, $attr) ? $request->$attr : [];
        }
        $response->header("Content-Type", "text/html; charset=UTF-8");
    }
}

/**
 * 在具体业务实现中取得全局对象
 */
class A
{
    public function run()
    {
        //获取全局gpc对象
        return var_export(App::$GPC, true);
    }
}
```
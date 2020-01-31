### RPC-Server

##### AutoLoader.php

```php
/**
* @return array
*/
public function beans(): array
{
    return [
        'rpcServer' => [
            'class' => ServiceServer::class,
            'on'    => [
                // 监听connect close receive事件
                // 其中connect和close事件主要干了一件事，对外暴漏事件监听的钩子，允许用户在Connect和close时做相应的逻辑处理
                SwooleEvent::CONNECT => bean(ConnectListener::class),
                SwooleEvent::CLOSE   => bean(CloseListener::class),
                SwooleEvent::RECEIVE => bean(ReceiveListener::class),
            ]
        ],
        'rpcServerPacket' => [
            // 对接收到的信息进行解包和封包处理，和rpc-client使用的协议一样
            'class' => Packet::class
        ]
    ];
}
```

##### receive事件

```php
/**
* @param Server $server
* @param int    $fd
* @param int    $reactorId
* @param string $data
*
* @throws RpcException
*/
public function onReceive(Server $server, int $fd, int $reactorId, string $data): void
{
```

​			// 实例化一个request对象,将传递过来的data信息解析,并保存到reqeust对象
​			$request  = Request::new($server, $fd, $reactorId, $data); [go](#newRequest) <a name="newRequestReturn"></a>

```php
    // 实例化一个response对象
    $response = Response::new($server, $fd, $reactorId);

    // 实例化一个RPC服务调用实例
    // 之前再framework源码提到过,再初始化bean实例时,如果该实例有init方法,会先执行init方法
    // 再serviceDispatcher实例中init方法会初始化中间件
    // 默认会添加 Swoft\Rpc\Server\Middleware\UserMiddleware::class中间件和Swoft\Rpc\Server\Middleware\DefaultMiddleware中间件
    /* @var ServiceDispatcher $dispatcher */ 
```

​			$dispatcher = BeanFactory::getSingleton('serviceDispatcher'); // [go](#serviceDispatcher) <a name="serviceDispatcherReturn"></a>

​			$dispatcher->dispatch($request, $response); // [go](#dispatch) <a name="dispatchReturn"></a>

```php
 }
```

##### 实例化request对象 <a name="newRequest"></a> [back](#newRequestReturn)

```php
/**
* @param Server $server
* @param int    $fd
* @param int    $reactorId
* @param string $data
*
* @return Request
* @throws RpcException
*/
public static function new(
    Server $server = null,
    int $fd = null,
    int $reactorId = null,
    string $data = null
): self {
    $instance = self::__instance();

    /* @var Packet $packet */
    $packet   = \bean('rpcServerPacket');
    // 利用jsonPacket解析传递过来的数据包
    // 返回相关的protocol实例，具体的解包可以查看rpc组件关于packet解析
    $protocol = $packet->decode($data);

    // 将解析数据包的结果通过protocol实例传给当前request实例
    // 数据包中存放了版本号\接口名\方法名\参数\后缀
    // 并将当前swoft传入的信息一并传递给request实例,并返回
    $instance->version     = $protocol->getVersion();
    $instance->interface   = $protocol->getInterface();
    $instance->method      = $protocol->getMethod();
    $instance->params      = $protocol->getParams();
    $instance->ext         = $protocol->getExt();
    $instance->data        = $data;
    $instance->server      = $server;
    $instance->reactorId   = $reactorId;
    $instance->fd          = $fd;
    $instance->requestTime = microtime(true);

    return $instance;
}
```

##### serviceDispatcher <a name="serviceDispatcher"></a> [back](#serviceDispatcherReturn)

```php
/**
* 这里默认初始化了一些中间件
* Init
*/
public function init(): void
{
    $this->preMiddlewares     = array_merge($this->preMiddleware(), $this->preMiddlewares);
    $this->afterMiddlewares   = array_merge($this->afterMiddleware(), $this->afterMiddlewares);
    $this->requestMiddlewares = array_merge($this->preMiddlewares, $this->middlewares, $this->afterMiddlewares);
}


/** 经过初始化后，默认追加一个UserMiddleware中间件
* @return array
*/
public function afterMiddleware(): array
{
    return [
        UserMiddleware::class
    ];
}
```

##### 路由调度dispatch <a name="dispatch"></a> [back](#dispatchReturn)

```php
/**
* @param array $params
*
*/
public function dispatch(...$params)
{
    /**
         * @var Request  $request
         * @var Response $response
         */
    [$request, $response] = $params;

    try {
        // 触发 swoft.rpc.server.receive.before 事件
        Swoft::trigger(ServiceServerEvent::BEFORE_RECEIVE, null, $request, $response);

        // serviceHandler实例用来管理中间件
        // handler管理请求前中间件 请求中中间件 请求后中间件 还有最后响应前执行的默认中间件(用来输出响应信息)
       
```

​				$handler  = ServiceHandler::new($this->requestMiddleware(), $this->defaultMiddleware); // [go](#serviceHandler) <a name="serviceHandlerReturn"></a>

​				// 执行中间件,最后通过defaultMiddleware返回处理的响应结果
​      		  $response = $handler->handle($request); // [go](#handle) <a name="handleReturn"></a>

```php
 
        
    } catch (Throwable $e) {
        /** @var RpcErrorDispatcher $errDispatcher */
        $errDispatcher = BeanFactory::getSingleton(RpcErrorDispatcher::class);

        // Handle request error
        $response = $errDispatcher->run($e, $response);
    }

    Swoft::trigger(ServiceServerEvent::AFTER_RECEIVE, null, $response);
}
```

##### ServiceHandler <a name="serviceHandler"></a> [back](#serviceHandlerReturn)

```php
/**
* @param array  $middlewares
* @param string $defaultMiddleware
*
* @return self
*
*/
public static function new(array $middlewares, string $defaultMiddleware): self
{
    $instance = self::__instance();

    // 设置中间件从第0个开始
    $instance->offset = 0;

    $instance->middlewares       = $middlewares;
    $instance->defaultMiddleware = $defaultMiddleware;

    return $instance;
}
```

##### 中间件调度 <a name="handle"></a> [back](#handleReturn)

```php
/**
* @param RequestInterface $request
*
* @return ResponseInterface
*/
public function handle(RequestInterface $request): ResponseInterface
{
    // Default middleware to handle request route
    $middleware = $this->middlewares[$this->offset] ?? $this->defaultMiddleware;

    // 根据offst获取相关的中间件实例
    /* @var MiddlewareInterface $bean */
    $bean = BeanFactory::getBean($middleware);

    // Next middleware
    $this->offset++;

    // 调用中间件的process方法执行相关逻辑
    return $bean->process($request, $this);
}
```

##### UserMiddleware

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Server\Middleware;


use ReflectionException;
use Swoft\Bean\Annotation\Mapping\Bean;
use Swoft\Bean\BeanFactory;
use Swoft\Bean\Exception\ContainerException;
use Swoft\Rpc\Server\Contract\MiddlewareInterface;
use Swoft\Rpc\Server\Contract\RequestHandlerInterface;
use Swoft\Rpc\Server\Contract\RequestInterface;
use Swoft\Rpc\Server\Contract\ResponseInterface;
use Swoft\Rpc\Server\Exception\RpcServerException;
use Swoft\Rpc\Server\Request;
use Swoft\Rpc\Server\Router\Router;
use Swoft\Rpc\Server\ServiceHandler;

/**
 * Class UserMiddleware
 *
 * @since 2.0
 *
 * @Bean()
 */
class UserMiddleware implements MiddlewareInterface
{
    /**
     * @param RequestInterface        $request
     * @param RequestHandlerInterface $requestHandler
     *
     * @return ResponseInterface
     * @throws RpcServerException
     */
    public function process(RequestInterface $request, RequestHandlerInterface $requestHandler): ResponseInterface
    {
        // 从request实例获取packet中相关信息
        $version   = $request->getVersion();
        $interface = $request->getInterface();
        $method    = $request->getMethod();

        // 获取router对象,用来解析路由
        /* @var Router $router */
        $router = BeanFactory::getBean('serviceRouter');  
```

​				    // 根据rpc-client传递过来的接口名和版本号匹配路由信息，返回rpc-server对应的接口匹配路由的相关信息handler
​				    $handler = $router->match($version, $interface); // [go](#match) <a name="matchReturn"></a>

```php
     	$request->setAttribute(Request::ROUTER_ATTRIBUTE, $handler);
		// status 表示是否匹配到接口  className表示匹配的类名
        [$status, $className] = $handler;
		
        if ($status != Router::FOUND) {
            // 如果没有匹配到，调用下一个中间件处理
            return $requestHandler->handle($request);
        }

		// 获取当前实现类是否存在中间件
        // 获取接口处理的中间件,并将接口的中间件插入到当前执行的中间件之后
        $middlewares = MiddlewareRegister::getMiddlewares($className, $method);
        if (!empty($middlewares) && $requestHandler instanceof ServiceHandler) {
            $requestHandler->insertMiddlewares($middlewares);
        }

       
```

​					# 最后执行默认的中间件 [go](#defaultMiddleware) <a name="defaultMiddlewareReturn"></a>

```php
 		// 继续执行中间件
        return $requestHandler->handle($request);
    }
}
/**
* @param RequestInterface $request
*
* @return ResponseInterface
*/
public function handle(RequestInterface $request): ResponseInterface
{
    // Default middleware to handle request route
    $middleware = $this->middlewares[$this->offset] ?? $this->defaultMiddleware;

    // 根据offst获取相关的中间件实例
    /* @var MiddlewareInterface $bean */
    $bean = BeanFactory::getBean($middleware);

    // Next middleware
    $this->offset++;

    // 调用中间件的process方法执行相关逻辑
    return $bean->process($request, $this);
}
```

##### 路由匹配 <a name="match"></a> [back](#matchReturn)

```php
/**
* @param string $version
* @param string $interface
*
* @return array
*/
public function match(string $version, string $interface): array
{
    // 获取访问路由,默认是 route = interface@version
    $route = $this->getRoute($interface, $version);

    // 通过interface@version获取具体的实现类
    /**
    * @var array
    *
    * @example
    * [
    *    'interface@version' => $className
    * ]
    */
```

​			# 判断路由是否存在，这些路由是什么是否存在routes属性中的 [go](#routes) <a name="routesReturn"></a>

```php
	if (isset($this->routes[$route])) {
        // 如果接口有对应的实现类，那么返回具体的实现类名
        return [self::FOUND, $this->routes[$route]];
    }

    return [self::NOT_FOUND, ''];
}
```

##### 服务路由注册 <a name="routes"></a> [back](#routesReturn)

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Server\Listener;


use ReflectionException;
use Swoft\Bean\BeanFactory;
use Swoft\Bean\Exception\ContainerException;
use Swoft\Event\Annotation\Mapping\Listener;
use Swoft\Event\EventHandlerInterface;
use Swoft\Event\EventInterface;
use Swoft\Rpc\Server\Middleware\MiddlewareRegister;
use Swoft\Rpc\Server\Router\Router;
use Swoft\Rpc\Server\Router\RouteRegister;
use Swoft\SwoftEvent;

/**
 * Class AppInitCompleteListener
 *
 * @since 2.0
 * 监听RPC-Service的 APP_INIT_COMPLETE 事件，启动完成后将服务绑定到路由中
 * @Listener(event=SwoftEvent::APP_INIT_COMPLETE)
 */
class AppInitCompleteListener implements EventHandlerInterface
{
    /**
     * @param EventInterface $event
     *
     * @throws \Swoft\Rpc\Server\Exception\RpcServerException
     */
    public function handle(EventInterface $event): void
    {
        /* @var Router $router */
        $router = BeanFactory::getBean('serviceRouter');

        // Register router
        RouteRegister::registerRoutes($router);

        // Register middleware
        MiddlewareRegister::register();
    }
}

/** 注册路由
* @param Router $router
*/
public static function registerRoutes(Router $router): void
{
    // 将@Service注解搜集的服务循环绑定到routes属性中
    foreach (self::$services as $interface => $service) {
        foreach ($service as $version => $className) {
            $router->addRoute($interface, $version, $className);
        }
    }
}
/**
* @param string $interface
* @param string $version
* @param string $className
*/
public function addRoute(string $interface, string $version, string $className): void
{
    // 获取访问路由,默认是 route = interface@version
    $route = $this->getRoute($interface, $version);

    $this->routes[$route] = $className;
}
```

##### 默认中间件DefaultMiddlewareReturn <a name="defaultMiddleware"></a> [back](#defaultMiddlewareReturn)

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Server\Middleware;


use function context;
use ReflectionException;
use Swoft\Bean\Annotation\Mapping\Bean;
use Swoft\Bean\BeanFactory;
use Swoft\Bean\Exception\ContainerException;
use Swoft\Rpc\Code;
use Swoft\Rpc\Server\Contract\MiddlewareInterface;
use Swoft\Rpc\Server\Contract\RequestHandlerInterface;
use Swoft\Rpc\Server\Contract\RequestInterface;
use Swoft\Rpc\Server\Contract\ResponseInterface;
use Swoft\Rpc\Server\Exception\RpcServerException;
use Swoft\Rpc\Server\Request;
use Swoft\Rpc\Server\Router\Router;
use Swoft\Stdlib\Helper\PhpHelper;

/**
 * Class DefaultMiddleware
 *
 * @since 2.0
 *
 * @Bean()
 */
class DefaultMiddleware implements MiddlewareInterface
{
    /**
     * @param RequestInterface        $request
     * @param RequestHandlerInterface $requestHandler
     *
     * @return ResponseInterface
     * @throws RpcServerException
     * @throws \Swoft\Exception\SwoftException
     */
    public function process(RequestInterface $request, RequestHandlerInterface $requestHandler): ResponseInterface
    {
        return $this->handler($request);
    }

    /**
     * @param RequestInterface $request
     *
     * @return ResponseInterface
     * @throws RpcServerException
     * @throws \Swoft\Exception\SwoftException
     */
    private function handler(RequestInterface $request): ResponseInterface
    {
        // 从request请求中获取解析packet的相关参数
        $version   = $request->getVersion();
        $interface = $request->getInterface();
        $method    = $request->getMethod();
        $params    = $request->getParams();

        // 通过\Swoft\Rpc\Server\Middleware\UserMiddleware中间件获取绑定在Request::ROUTER_ATTRIBUTE属性上的内容
        // router对象以 interferce@version作为路由的key,通过key找到对应的实现类
        // status 对应版本的接口路由是否存在
        // className 对应版本接口的实现类
        // 通过router对象
        [$status, $className] = $request->getAttribute(Request::ROUTER_ATTRIBUTE);

        if ($status != Router::FOUND) {
            throw new RpcServerException(
                sprintf('Route(%s-%s) is not founded!', $version, $interface),
                Code::INVALID_REQUEST
            );
        }

        // 获取实现类的实例
        $object = BeanFactory::getBean($className);
        if (!$object instanceof $interface) {
            throw new RpcServerException(
                sprintf('Object is not instanceof %s', $interface),
                Code::INVALID_REQUEST
            );
        }

        if (!method_exists($object, $method)) {
            throw new RpcServerException(
                sprintf('Method(%s::%s) is not founded!', $interface, $method),
                Code::METHOD_NOT_FOUND
            );
        }

        // 调用对应实例的方法并将参数传递过去
        $data = PhpHelper::call([$object, $method], ...$params);

        // 执行结果返回
        $response = context()->getResponse();
        return $response->setData($data);
    }
}
```




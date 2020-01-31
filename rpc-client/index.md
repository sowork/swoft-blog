### RPC-Client

##### AutoLoader.php

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Client;


use ReflectionException;
use Swoft\Bean\Exception\ContainerException;
use Swoft\Rpc\Packet;
use Swoft\Rpc\Packet\SwoftPacketV1;
use Swoft\SwoftComponent;

/**
 * Class AutoLoader
 *
 * @since 2.0
 */
class AutoLoader extends SwoftComponent
{
    /**
     * @return array
     */
    public function getPrefixDirs(): array
    {
        return [
            __NAMESPACE__ => __DIR__,
        ];
    }

    /**
     * @return array
     */
    public function metadata(): array
    {
        return [];
    }

    /**
     * @return array
     */
    public function beans(): array
    {
        return [
            'rpcClientPacket'        => [
                // packet 在RPC包中介绍了，该对象用来对数据包进行解包和封包，以及响应
                'class' => Packet::class 
             	],
            'rpcClientSwoftPacketV1' => [
                // rpcClientSwoftPacketV1 兼容swoftV1的RPC，这里就不细看了
                'class'   => Packet::class,
                'packets' => [
                    'swoftV1' => bean(SwoftPacketV1::class)
                ],
                'type'    => 'swoftV1',
                'packageEof' => "\r\n",
            ]
        ];
    }
}
```

##### RPC-Client配置

```php
return [
    // 代表 会实例一个userBean服务
    'user'       => [
        // user bean实例是 ServiceClient的实例
        'class'   => ServiceClient::class,
        // 注入host属性的值为120.0.0.1
        'host'    => '127.0.0.1',
        // 注入prot属性的值为18307
        'port'    => 18307,
        // 注入setting属性的值
        'setting' => [
            'timeout'         => 0.5,
            'connect_timeout' => 1.0,
            'write_timeout'   => 10.0,
            'read_timeout'    => 0.5,
        ],
        // 注入packet属性的值为 rpcClientPacket实例，该实例在AutoLoader.php被声明加载
        'packet'  => bean('rpcClientPacket')
    ],
    // 设置user服务的连接池配置
    'user.pool'  => [
        // user.pool的实例时一个ServicePool实例
        'class'  => ServicePool::class,
        // 默认注入一个client 是上面定义的userBean服务
        'client' => bean('user')
    ],
];
```

##### RPC-Client如何通信

* 首先我们来看一段swoft提供的示例

```php
<?php declare(strict_types=1);

namespace App\Http\Controller;

use App\Rpc\Lib\UserInterface;
use Exception;
use Swoft\Co;
use Swoft\Http\Server\Annotation\Mapping\Controller;
use Swoft\Http\Server\Annotation\Mapping\RequestMapping;
use Swoft\Rpc\Client\Annotation\Mapping\Reference;

/**
 * Class RpcController
 *
 * @since 2.0
 *
 * @Controller()
 */
class RpcController
{
    /**
     * @Reference(pool="user.pool")
     *
     * @var UserInterface
     */
    private $userService;

    /**
     * @Reference(pool="user.pool", version="1.2")
     *
     * @var UserInterface
     */
    private $userService2;

    /**
     * @RequestMapping("getList")
     *
     * @return array
     */
    public function getList(): array
    {
        $result  = $this->userService->getList(12, 'type');
        $result2 = $this->userService2->getList(12, 'type');

        return [$result, $result2];
    }
}
```

* 我们看到其实就是一个普通的Controller，特别的是注入的属性增加了一个@Reference注解并传递给该注解相应的参数，我们还需要声明注入属性相应的接口名，也就是声明的@var注解

* 简单的几个设置就可以访问远程的服务了，现在我们来揭秘下在swoft中是如何访问远程服务的

##### @Reference注解

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Client\Annotation\Mapping;


use Doctrine\Common\Annotations\Annotation\Attribute;
use Doctrine\Common\Annotations\Annotation\Attributes;
use Doctrine\Common\Annotations\Annotation\Required;
use Doctrine\Common\Annotations\Annotation\Target;
use Swoft\Rpc\Protocol;

/**
 * 注解有两个参数，第一个参数代表服务的连接池对象，第二个参数代表访问服务的版本
 * Class Reference
 *
 * @since 2.0
 *
 * @Annotation
 * @Target("PROPERTY")
 * @Attributes({
 *     @Attribute("event", type="string"),
 * })
 */
class Reference
{
    /**
     * @var string
     *
     * @Required()
     */
    private $pool;

    /**
     * @var string
     */
    private $version = Protocol::DEFAULT_VERSION; // 默认是1.0

    /**
     * Reference constructor.
     * 将该注解传入的参数绑定到对应的属性上
     * @param array $values
     */
    public function __construct(array $values)
    {
        if (isset($values['value'])) {
            $this->pool = $values['value'];
        } elseif (isset($values['pool'])) {
            $this->pool = $values['pool'];
        }

        if (isset($values['version'])) {
            $this->version = $values['version'];
        }
    }

    /** 
     * @return string
     */
    public function getVersion(): string
    {
        return $this->version;
    }

    /**
     * @return string
     */
    public function getPool(): string
    {
        return $this->pool;
    }
}
```

##### @ReferenceParser解析器

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Client\Annotation\Parser;


use PhpDocReader\AnnotationException;
use PhpDocReader\PhpDocReader;
use ReflectionException;
use ReflectionProperty;
use Swoft\Annotation\Annotation\Mapping\AnnotationParser;
use Swoft\Annotation\Annotation\Parser\Parser;
use Swoft\Proxy\Exception\ProxyException;
use Swoft\Rpc\Client\Annotation\Mapping\Reference;
use Swoft\Rpc\Client\Exception\RpcClientException;
use Swoft\Rpc\Client\Proxy;
use Swoft\Rpc\Client\ReferenceRegister;

/** 每一个类的注解都会有一个对应的parser解析器，@reference注解对应的注解解析器是referenceParser
 * Class ReferenceParser
 *
 * @since 2.0
 *
 * @AnnotationParser(Reference::class)
 */
class ReferenceParser extends Parser
{
    /**
     * @param int       $type
     * @param Reference $annotationObject
     *
     * @return array
     * @throws RpcClientException
     * @throws AnnotationException
     * @throws ReflectionException
     * @throws ProxyException
     */
    public function parse(int $type, $annotationObject): array
    {
        // Parse php document
        // 这里用的是一个三方的扩展 php-di，通过该扩展的phpDocReader对象
        // 我们能轻松的获取到属性上内容
        $phpReader       = new PhpDocReader();
        // this->className可以当作上面 RpcController， propertyName可以当作上面的 userService
        $reflectProperty = new ReflectionProperty($this->className, $this->propertyName);
        // 解析类里面属性所有属性注释中@var注解的内容
        $propClassType   = $phpReader->getPropertyClass($reflectProperty);

        // 如果定义了@Reference注解没有定义属性的@var注解，抛出指定的异常
        if (empty($propClassType)) {
            throw new RpcClientException(
                sprintf('`@Reference`(%s->%s) must to define `@var xxx`', $this->className, $this->propertyName)
            );
        }        
        // 1. 生成 interface UserService 接口 的动态代理类 class UserServicexxxxxxx
        // 2. 对动态代理的接口相关的方法进行实现，默认调用 __proxyCall方法
        // 3. 加载（require）生成的代理类
        // 4. 返回代理类的名称
```

​		        $className = Proxy::newClassName($propClassType); // [go](#newClassName) <a name="newClassNameReturn"></a>

```php
        // 将该代理类添加到parser对象的definitions属性中
        // parser的definitions属性会默认和AnnotationObjParser实例的definitions属性进行合并，最终绑定到container的definitions属性上
        // 在controller中使用@Refence注解时，框架启动时会搜集注解同时将该definition进行实例化bean对象
        $this->definitions[$className] = [
            'class' => $className,
        ];

        // 该属性注入的服务className，相关的连接池，版本号一并绑定到referenceRegister的references属性上
        /**
         * references属性保存相关服务的配置
         * @var array
         *
         * @example
         * [
         *     'className' => [
         *         'pool' => 'poolName',
         *         'version' => 'version',
         *     ]
         * ]
         */
        ReferenceRegister::register($className, $annotationObject->getPool(), $annotationObject->getVersion());
        // 返回解析后的结果
        // className作为bean的name，
        // true 标识是否是一个引用变量，如果为true，根据value值会从config文件中引用对应的值
        return [$className, true];
    }
}
```

##### 动态代理注入接口类 <a name="newClassName"></a> [back](#newClassNameReturn)

```php
/**
* @param string $className
*
* @return string
* @throws RpcClientException
* @throws ProxyException
*/
public static function newClassName(string $className): string
{
    // 在使用@Reference注解时,@var注解必须是一个接口
    if (!interface_exists($className)) {
        throw new RpcClientException(
            sprintf('`@var` for `@Reference` must be exist interface!')
        );
    }

    // 生成代理文件类名后缀，类名称中出现_IGNORE_，后续该类将不会再被代理
    $proxyId   = sprintf('IGNORE_%s', Str::getUniqid());
   
```

​			$visitor   = new ProxyVisitor($proxyId); // [go](#ProxyVisitor) <a name="ProxyVisitorReturn"></a>

```php
 
    // 生成代理类文件存放到临时目录下，require该代理类，并返回当前代理类的名称（逻辑和framework中代理AOP是一样的）
    $className = BaseProxy::newClassName($className, $visitor);
    return $className;
}
```

##### ProxyVisitor <a name="ProxyVisitor"></a> [back](#ProxyVisitorReturn)

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Client\Proxy\Ast;


use function array_unshift;
use function sprintf;
use Swoft\Proxy\Ast\Visitor\Visitor;
use Swoft\Rpc\Client\Concern\ServiceTrait;
use PhpParser\Node;
use PhpParser\Node\Stmt\ClassMethod;
use PhpParser\NodeFinder;
use PhpParser\NodeTraverser;
use function uniqid;

/**
 * Class ProxyVisitor
 *
 * @since 2.0
 */
class ProxyVisitor extends Visitor
{
    /**
     * Namespace
     *
     * @var string
     */
    private $namespace = '';

    /**
     * New class name
     *
     * @var string
     */
    private $proxyId;

    /**
     * Origin class name
     *
     * @var string
     */
    private $originalInterfaceName = '';

    /**
     * Proxy class name without namespace
     *
     * @var string
     */
    private $proxyName = '';

    /**
     * Aop class name
     *
     * @var string
     */
    private $serviceTrait;

    /**
     * ProxyVisitor constructor.
     *
     * @param string $proxyId
     * @param string $TraitClassName
     */
    public function __construct(string $proxyId = '', string $TraitClassName = ServiceTrait::class) // 注意这里的ServiceTrait实现了动态代理方法proxyCall的逻辑
    {
        $this->serviceTrait = $TraitClassName;
        $this->proxyId      = $proxyId ?: uniqid();
    }

    /**
     * Enter node
     *
     * @param Node $node
     *
     * @return int|Node|null]
     */
    public function enterNode(Node $node)
    {
        // Namespace for proxy class name
        if ($node instanceof Node\Stmt\Namespace_) {

            $this->namespace = $node->name->toString();
            return null;
        }

        // Origin interface node
        if ($node instanceof Node\Stmt\Interface_) {
            $name                        = $node->name->toString();
            $this->proxyName             = sprintf('%s_%s', $name, $this->proxyId);
            $this->originalInterfaceName = sprintf('%s\\%s', $this->namespace, $name);

            return null;
        }

        return null;
    }

    /**
     * Leave node
     *
     * @param Node $node
     *
     * @return int|Node|Node[]|null
     */
    public function leaveNode(Node $node)
    {
        // Parse new interface node
        // 如果节点是接口，那么替换该节点的接口名称，但实际实例还是之前类实例
        if ($node instanceof Node\Stmt\Interface_) {
            // 构造一个该接口的实现类
            $newClassNodes = [
                'flags'      => 0,
                'stmts'      => $node->stmts,
                'implements' => [
                    new Node\Name('\\' . $this->originalInterfaceName)
                ],
            ];

            // 构造class，类名为this->proxyName
            return new Node\Stmt\Class_($this->proxyName, $newClassNodes);
        }

        // Parse class method and rewrite public and protected
        if ($node instanceof ClassMethod) {
            // 移除静态方法和私有方法
            if ($node->isPrivate() || $node->isStatic()) {
                return NodeTraverser::REMOVE_NODE;
            }

            // 代理方法
            return $this->proxyMethod($node);
        }

        return $node;
    }

    /**
     * After traverse
     *
     * @param array $nodes
     *
     * @return array|Node[]|null
     */
    public function afterTraverse(array $nodes)
    {
        $nodeFinder = new NodeFinder();

        // 在遍历结束后，找到该类将默认的trait添加进去，该trait实现了 __proxyCall 方法
        /** @var Node\Stmt\Class_ $classNode */
        $classNode = $nodeFinder->findFirstInstanceOf($nodes, Node\Stmt\Class_::class);

        // 将传递进来的trait和额外添加的this->getOriginalClassName方法追加到代理类中
        $traitNode          = $this->getTraitNode();
        $originalMethodNode = $this->getOriginalClassNameMethodNode();

        array_unshift($classNode->stmts, $traitNode, $originalMethodNode);
        return $nodes;
    }

    /**
     * Proxy method
     *
     * @param ClassMethod $node
     *
     * @return ClassMethod
     */
    private function proxyMethod(ClassMethod $node): ClassMethod
    {
        $methodName = $node->name->toString();

        // TODO Origin method params
        $params = [];
        foreach ($node->params as $key => $param) {
            $params[] = $param;
        }

        // Proxy method params
        // 给代理的方法传递参数
        // 原始的接口名称，原始的方法名，以及原有方法的参数
        $newParams = [
            new Node\Scalar\String_($this->originalInterfaceName),
            new Node\Scalar\String_($methodName),
            new Node\Expr\FuncCall(new Node\Name('func_get_args')),
        ];

        // Proxy method call
        // 代理接口中的方法
        // 相当于在接口中编写 this->__proxyCall() 这样一个表达式，并将上面的参数传递进去
        // $this->__proxyCall('\App\Rpc\Lib\UserInterface::class', 'getList', func_get_args());
        // 在该ProxyVisitor的构造方法中传入ServiceTrait，里面实现了__proxyCall()方法
```

​					// this->__proxyCall()  [go](#proxyCall) <a name="proxyCallReturn"></a>

```php
		$proxyCall = new Node\Expr\MethodCall(
            new Node\Expr\Variable('this'),
            '__proxyCall',
            $newParams
        );

        // New method stmts
        // 获取方法的返回类型
        $type = $node->returnType;
        // 编写 表达式添加return语句
        $stmt = new Node\Stmt\Return_($proxyCall);
        if ($type && $type instanceof Node\Identifier && $type->name === 'void') {
            // 如果返回值是void，那么表达式是不需要return的
            $stmt = new Node\Stmt\Expression($proxyCall);
        }

        // Return `self` to return `originalClassName`
        $returnType = $node->returnType;
        // 返回的类型如果是自己本身，修改该方法的返回值类型为 this->originalInterfaceName
        if ($returnType instanceof Node\Name && $returnType->toString() === 'self') {
            $returnType->parts = [
                sprintf('\\%s', $this->originalInterfaceName)
            ];
        }

        // New method nodes
        $newMethodNodes = [
            'flags'      => $node->flags,
            'byRef'      => $node->byRef,
            'name'       => $node->name,
            'params'     => $node->params,
            'returnType' => $returnType,
            'stmts'      => [
                $stmt
            ],
        ];

        // 返回代理后的方法
        // 相当于自动生成一个实现 this->originalInterfaceName 该接口的实现类，将接口中的方法做了实现
        // 每个方法会自动调用该实现类的 __proxyCall 方法
        return new ClassMethod($methodName, $newMethodNodes);
    }

    /**
     * Get proxy class name
     *
     * @return string
     */
    public function getProxyClassName(): string
    {
        return sprintf('%s\\%s', $this->namespace, $this->proxyName);
    }

    /**
     * Get proxy class name
     *
     * @return string
     */
    public function getOriginalInterfaceName(): string
    {
        return $this->originalInterfaceName;
    }

    /**
     * @return string
     */
    public function getProxyName(): string
    {
        return $this->proxyName;
    }

    /**
     * Get aop trait node
     *
     * @return Node\Stmt\TraitUse
     */
    private function getTraitNode(): Node\Stmt\TraitUse
    {
        return new Node\Stmt\TraitUse([
            new Node\Name('\\' . $this->serviceTrait)
        ]);
    }

    /**
     * Get original class method node
     *
     * @return ClassMethod
     */
    private function getOriginalClassNameMethodNode(): ClassMethod
    {
        // Add getOriginalClassName method
        return new ClassMethod('getOriginalClassName', [
            'flags'      => Node\Stmt\Class_::MODIFIER_PUBLIC,
            'returnType' => 'string',
            'stmts'      => [
                new Node\Stmt\Return_(new Node\Scalar\String_($this->getOriginalInterfaceName()))
            ],
        ]);
    }
}
```

##### 动态代理方法实现之proxyCall <a name="proxyCall"></a> [back](#proxyCallReturn)

```php
/**
* @param string $interfaceClass
* @param string $methodName
* @param array  $params
*
* @return mixed
* @throws ConnectionPoolException
* @throws ContainerException
* @throws ReflectionException
* @throws RpcClientException
* @throws RpcResponseException
*/
protected function __proxyCall(string $interfaceClass, string $methodName, array $params)
{  
```

​			// 获取指定服务连接池类名，这里用UserService举例，得到的beanName=user.pool
​			$poolName = ReferenceRegister::getPool(`__CLASS__`); [go](#getPool) <a name="getPoolReturn"></a>
​			// 获取调用服务版本号
​			$version  = ReferenceRegister::getVersion(`__CLASS__`); [go](#getVersion) <a name="getVersionReturn"></a>

```php
    // 通过容器获取RPC连接池对象
    /* @var Pool $pool */
    $pool = BeanFactory::getBean($poolName);
    
```

​			// 通过channel获取RPC-Client连接
​			// 调用Swoft\Rpc\Client\Pool的createConnection（）方法生成一个swoole TPC-client对象
​			// 也就是说connection是一个包装了client的实例
​			/* @var Connection $connection */
​			$connection = $pool->getConnection(); [go](#getConnection) <a name="getConnectionReturn"></a>

```php
	// 标记该链接可以被回收
    $connection->setRelease(true);
    // 获取connection对应的packet，该packet由RPC-client配置时默认注入,下面是RPC-Client示例
    // 'user'       => [
    //        'class'   => ServiceClient::class,
    //        'host'    => '127.0.0.1',
    //        'port'    => 18307,
    //        'setting' => [
    //            'timeout'         => 0.5,
    //            'connect_timeout' => 1.0,
    //            'write_timeout'   => 10.0,
    //            'read_timeout'    => 0.5,
    //        ],
    //        'packet'  => bean('rpcClientPacket')
    //    ],
    //    'user.pool'  => [
    //        'class'  => ServicePool::class,
    //        'client' => bean('user')
    //    ],
    // 该packet主要用来对发送的packet进行编码和解码
    $packet = $connection->getPacket();

    // Ext data
    // 扩充的数据，允许我们传递自定义数据
    $ext = $connection->getClient()->getExtender()->getExt();
    // protocol对象封装了发送的数据，包括访问哪个类，哪个方法，参数是什么，版本号，ext 等数据
    $protocol = Protocol::new($version, $interfaceClass, $methodName, $params, $ext);
    // 对发送的数据进行编码，具体encode操作可以查看rpc组件关于packet的介绍
    $data     = $packet->encode($protocol);
	// 定义rpc通信失败的错误提示
    $message  = sprintf(
        'Rpc call failed.interface=%s method=%s pool=%s version=%s',
        $interfaceClass, $methodName, $poolName, $version
    );
```

​			// 通过链接发送和接受数据，拿到远程service返回的服务
​		    $result = $this->sendAndRecv($connection, $data, $message); [go](#sendAndRecv) <a name="sendAndRecvReturn"></a>
​			// 判断该连接是否超过channel的最大限制，如果没有，存放到channel里
​			// 同时标记该链接被回收
​			$connection->release(); [go](#release) <a name="releaseReturn"></a>

```php
    // 通过packet解析并返回响应信息
    $response = $packet->decodeResponse($result);
    // 判断RPC请求响应的错误信息
    if ($response->getError() !== null) {
        $code      = $response->getError()->getCode();
        $message   = $response->getError()->getMessage();
        $errorData = $response->getError()->getData();

        // Record rpc error message
        $errorMsg = sprintf(
            'Rpc call error!code=%d message=%s data=%s pool=%s version=%s',
            $code, $message, JsonHelper::encode($errorData), $poolName, $version
        );

        Error::log($errorMsg);

        // Only to throw message and code
        throw new RpcResponseException($message, $code);
    }

    // 否则返回结果
    return $response->getResult();

}
```

##### RPC发送和接受数据包 <a name="sendAndRecv"></a> [back](#sendAndRecvReturn)

```php
/**
* @param Connection $connection
* @param string     $data
* @param string     $message
* @param bool       $reconnect
*
* @return string
* @throws RpcClientException
* @throws ReflectionException
* @throws ContainerException
*/
private function sendAndRecv(Connection $connection, string $data, string $message, bool $reconnect = false): string
{
    // Reconnect
    if ($reconnect) {
        $connection->reconnect();
    }

    // 如果发送失败，那么继续重连发送一次，如果还是失败，那么抛出异常
    // 调用swoole client自带的send方法发送数据
    if (!$connection->send($data)) {
        if ($reconnect) {
            throw new RpcClientException($message);
        }

        return $this->sendAndRecv($connection, $data, $message, true);
    }

    // 接受返回的数据
    $result = $connection->recv();
    if ($result === false || empty($result)) {
        if ($reconnect) {
            throw new RpcClientException($message);
        }

        return $this->sendAndRecv($connection, $data, $message, true);
    }

    return $result;
}
```

##### 获取服务连接池对象 <a name="getPool"></a> [back](#getPoolReturn)

```php
/**
* @param string $className
*
* @return string
* @throws RpcClientException
*/
public static function getPool(string $className): string
{
    // self::references 这个在ReferenceParser解析器parser()方法中会默认注册
    $pool = self::$references[$className]['pool'] ?? '';
    if (empty($pool)) {
        throw new RpcClientException(
            sprintf('`@Reference` pool (%s) is not exist!', $className)
        );
    }

    return $pool;
}
```

##### 获取服务版本号 <a name="getVersion"></a> [back](#getVersionReturn)

```php
/**
* @param string $className
*
* @return string
* @throws RpcClientException
*/
public static function getVersion(string $className): string
{
    // self::references 这个在ReferenceParser解析器parser()方法中会默认注册
    $version = self::$references[$className]['version'] ?? '';
    if ($version == '') {
        throw new RpcClientException(
            sprintf('`@Reference` version(%s) is not exist!', $className)
        );
    }

    return $version;
}
```

##### 连接池获取RPC服务远程连接 <a name="getConnection"></a> [back](#getConnectionReturn)

```php
/**
* @return ConnectionInterface
* @throws ConnectionPoolException
*/
public function getConnection(): ConnectionInterface
{
   
```

​			return $this->getConnectionByChannel(); [go](#getConnectionByChannel) <a name="getConnectionByChannelReturn"></a>

```php
	
}
```

##### 连接池通过管道获取connection连接 <a name="getConnectionByChannel"></a> [back](#getConnectionByChannelReturn)

```php
/**
* Get connection by channel
*
* @return ConnectionInterface
* @throws ConnectionPoolException
*/
private function getConnectionByChannel(): ConnectionInterface
{
    // Create channel
    // 创建一个channle，channel和go的channel很像
    // channel 底层自动实现了协程的切换和调度
    // channel 是一个类似队列的东西，push存，pop取
    // 当存满了（=maxActive）,生产者会自动阻塞，等待消费者去消费
    // 当channel取完后，消费者又会自动阻塞，等待生产者去生产
    // 下面默认创建一个channel，最多存放maxActive个rpcClient连接
    if ($this->channel === null) {
        $this->channel = new Channel($this->maxActive);
    }

    // To reach `minActive` number
    // 默认至少创建minActive个rpcClient连接
    if ($this->count < $this->minActive) {
       
```

​					return $this->create(); [go](#create) <a name="createReturn"></a>

```php
    }

    // Pop connection
    $connection = null;
	// 管道队列部位空直接从里面取出一个
    if (!$this->channel->isEmpty()) {
        $connection = $this->popByChannel();
    }

    // Pop connection is not null
    if ($connection !== null) {
        // Update last time
        $connection->updateLastTime();
        return $connection;
    }

    // Channel is empty or  not reach `maxActive` number
	// 如果管道暂时没有但连接数小于最大连接设置，那么继续创建一个
    if ($this->count < $this->maxActive) {

        return $this->create();
    }

    // Out of `maxWait` number
    $stats = $this->channel->stats();
	// 如果设置了最大连接池数量，并且等待的消费者数量超过了最大连接池数量会触发连接池异常
    if ($this->maxWait > 0 && $stats['consumer_num'] >= $this->maxWait) {
        throw new ConnectionPoolException(
            sprintf('Channel consumer is full, maxActive=%d, maxWait=%d, currentCount=%d',
                    $this->maxActive, $this->maxWaitTime, $this->count)
        );
    }

    /* @var ConnectionInterface $connection */
    // Sleep coroutine and resume coroutine after `maxWaitTime`, Return false is waiting timeout
	// 等待的消费者数量没有超过最大的连接池数量设置，在等待this->maxWaitTime时没有返回可用的connection会抛出异常
    $connection = $this->channel->pop($this->maxWaitTime);
    if ($connection === false) {
        throw new ConnectionPoolException(
            sprintf('Channel pop timeout by %fs', $this->maxWaitTime)
        );
    }

    // Update last time
    $connection->updateLastTime();

    return $connection;
}
```

##### 调用连接池创建连接方法 <a name="create"></a> [back](#createReturn)

```php
/**
* @return ConnectionInterface
*
* @throws ConnectionPoolException
*/
private function create(): ConnectionInterface
{
    // Count before to fix more connection bug
    $this->count++;

    try {   
```

​				    // 连接池创建连接
​				    $connection = $this->createConnection(); [go](#createConnection) <a name="createConnectionReturn"></a>

```php
 	} catch (Throwable $e) {
        // Create error to reset count
        $this->count--;

        throw new ConnectionPoolException(
            sprintf('Create connection error(%s) file(%s) line (%d)',
                    $e->getMessage(),
                    $e->getFile(),
                    $e->getLine())
        );
    }

    return $connection;
}
```

##### 创建远程服务连接 <a name="createConnection"></a> [back](#createConnectionReturn)

```php

/**
* @return ConnectionInterface
* @throws RpcClientException
*/
public function createConnection(): ConnectionInterface
{
    if (empty($this->client)) {
        throw new RpcClientException(
            sprintf('Pool(%s) client can not be null!', __CLASS__)
        );
    }
```

​		return $this->client->createConnection($this); // [go](#createConnection1) <a name="createConnection1Return"></a>

```php
}
```

##### RPC-Client创建远程服务连接 <a name="createConnection1"></a> [back](#createConnection1Return)

```php
/**
* @param Pool $pool
*
* @return Connection
* @throws RpcClientException
*/
public function createConnection(Pool $pool): Connection
{
    $connection = Connection::new($this, $pool);
    $connection->create();

    return $connection;
}

/**
* @param \Swoft\Rpc\Client\Client $client
* @param Pool                     $pool
*
* @return Connection
*/
public static function new(RpcClient $client, Pool $pool): Connection
{
    // 实例一个client的connection链接实例
    $instance = self::__instance();
    // 将pool和client保存到该链接对象中
    $instance->client = $client;
    $instance->pool   = $pool;

    $instance->lastTime = time();

    return $instance;
}

/**
* @throws RpcClientException
*/
public function create(): void
{
    // 实例化一个swoole tcp client
    $connection = new Client(SWOOLE_SOCK_TCP);

    // 获取prot和host
    [$host, $port] = $this->getHostPort();
    // 获取RPC-CLient配置信息，默认读取
    // 'open_eof_check' => true,
    // 'open_eof_split' => true,
    // 'package_eof'    => "\r\n\r\n",
    // open_eof_check=true
    // 打开EOF检测，此选项将检测客户端连接发来的数据，当数据包结尾是指定的字符串时才会投递给Worker进程。
    // 否则会一直拼接数据包，直到超过缓存区或者超时才会中止。当出错时底层会认为是恶意连接，丢弃数据并强制关闭连接。
    // 1.7.15版本增加了open_eof_split配置项，支持从数据中查找EOF，并切分数据
    // packge_eof 与 open_eof_check 或者 open_eof_split 配合使用，设置EOF字符串
    $setting = $this->client->getSetting();

    // 给TCPClient设置初始化配置
    if (!empty($setting)) {
        $connection->set($setting);
    }

    // 创建连接
    if (!$connection->connect($host, (int)$port)) {
        throw new RpcClientException(
            sprintf('Connect failed host=%s port=%d', $host, $port)
        );
    }

    $this->connection = $connection;
}
```

##### 回收连接池连接 <a name="release"></a> [back](#releaseReturn)

```php
/**
* @param ConnectionInterface $connection
*/
public function release(ConnectionInterface $connection): void
{
    $this->releaseToChannel($connection);
}

/**
* Release to channel
*
* @param ConnectionInterface $connection
*/
private function releaseToChannel(ConnectionInterface $connection)
{
    // 获取管道当前状态
    // 返回一个数组，缓冲通道将包括4项信息，无缓冲通道返回2项信息
    //consumer_num 消费者数量，表示当前通道为空，有N个协程正在等待其他协程调用push方法生产数据
    //producer_num 生产者数量，表示当前通道已满，有N个协程正在等待其他协程调用pop方法消费数据
    //queue_num 通道中的元素数量
    // array(
    //  "consumer_num" => 0,
    //  "producer_num" => 1,
    //  "queue_num" => 10
    //);
    $stats = $this->channel->stats();
    // 如果管道元素容量不满足最大限制，那么将当前链接放入到管道中
    if ($stats['queue_num'] < $this->maxActive) {
        $this->channel->push($connection);
    }
}
```




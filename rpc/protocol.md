### Protocol

> 是请求和响应的代理对象，当发出请求时，会将protocol对象转换成字符串传输，当接收到响应时会将接收到的字符串再转换成protocol对象



##### protocol对象的封装

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc;

use ReflectionException;
use Swoft\Bean\Annotation\Mapping\Bean;
use Swoft\Bean\Concern\PrototypeTrait;
use Swoft\Bean\Exception\ContainerException;


/**
 * Class Protocol
 *
 * @since 2.0
 *
 * @Bean(scope=Bean::PROTOTYPE)
 */
class Protocol
{
    /**
     * Default version
     */
    const DEFAULT_VERSION = '1.0';

    use PrototypeTrait;

    /**
     * @var string
     */
    private $interface = '';

    /**
     * @var string
     */
    private $method = '';

    /**
     * @var array
     */
    private $params = [];

    /**
     * @var array
     */
    private $ext = [];

    /**
     * @var string
     */
    private $version = self::DEFAULT_VERSION;

    /**
     * Replace constructor
     *
     * @param string $version
     * @param string $interface
     * @param string $method
     * @param array  $params
     * @param array  $ext
     *
     * @return Protocol
     */
    public static function new(string $version, string $interface, string $method, array $params, array $ext)
    {
        // 实例化当前类对象
        $instance = self::__instance();

        // RPC请求会访问远程服务对应版本接口实现类的方法，因此protocol封装了访问远程函数需要传递的信息
        // 比如 版本号 、 接口名 、 方法名和参数等等
        $instance->version   = $version;
        $instance->interface = $interface;
        $instance->method    = $method;
        $instance->params    = $params;
        $instance->ext       = $ext;

        return $instance;
    }

    /**
     * @return string
     */
    public function getInterface(): string
    {
        return $this->interface;
    }

    /**
     * @return string
     */
    public function getMethod(): string
    {
        return $this->method;
    }

    /**
     * @return array
     */
    public function getParams(): array
    {
        return $this->params;
    }

    /**
     * @return array
     */
    public function getExt(): array
    {
        return $this->ext;
    }

    /**
     * @return string
     */
    public function getVersion(): string
    {
        return $this->version;
    }
}
```


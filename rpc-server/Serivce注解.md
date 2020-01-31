### Service注解

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Server\Annotation\Mapping;


use Doctrine\Common\Annotations\Annotation\Attribute;
use Doctrine\Common\Annotations\Annotation\Attributes;
use Doctrine\Common\Annotations\Annotation\Target;
use Swoft\Rpc\Protocol;

/** 类注解 有一个属性value，用来设置服务的版本号
 * Class Service
 *
 * @since 2.0
 *
 * @Annotation
 * @Target({"CLASS"})
 * @Attributes({
 *     @Attribute("version", type="string"),
 * })
 */
class Service
{
    /**
     * @var string
     */
    private $version = Protocol::DEFAULT_VERSION;

    /**
     * Service constructor.
     *
     * @param array $values
     */
    public function __construct(array $values)
    {
        if (isset($values['value'])) {
            $this->version = $values['value'];
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
}
```

##### ServiceParser解析器

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc\Server\Annotation\Parser;


use ReflectionClass;
use ReflectionException;
use Swoft\Annotation\Annotation\Mapping\AnnotationParser;
use Swoft\Annotation\Annotation\Parser\Parser;
use Swoft\Bean\Annotation\Mapping\Bean;
use Swoft\Rpc\Server\Annotation\Mapping\Service;
use Swoft\Rpc\Server\Router\RouteRegister;

/**
 * Class ServiceParser
 *
 * @since 2.0
 *
 * @AnnotationParser(annotation=Service::class)
 */
class ServiceParser extends Parser
{
    /**
     * @param int     $type
     * @param Service $annotationObject
     *
     * @return array
     * @throws ReflectionException
     */
    public function parse(int $type, $annotationObject): array
    {
        $reflectionClass = new ReflectionClass($this->className);
        // 获取当前服务实现的接口名称
        $interfaces      = $reflectionClass->getInterfaceNames();
		
        // 将接口注册到路由实力上，当rpc-client访问时通过接口来找到对应的实现类
        foreach ($interfaces as $interface) {
            

```

​							RouteRegister::register($interface, $annotationObject->getVersion(), $this->className); // [go](#register) <a name="registerReturn"></a>

        	}
    
        	return [$this->className, $this->className, Bean::SINGLETON, ''];
    	}
    }
##### 注册服务的路由 <a name="register"></a> [back](#registerReturn)

```php
/**
* @param string $interface
* @param string $version
* @param string $className
*/
public static function register(string $interface, string $version, string $className): void
{
    self::$services[$interface][$version] = $className;

    // Record classNames
    self::$serviceClassNames[$className] = $interface;
}
```




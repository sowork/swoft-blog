### RPC

> RPC组件主要封装对传输协议的封包和解包，RPC-Client和RPC-Server基于该组件实现数据包的传输，进而实现相互调用

##### RPC对Packet封装

```php
<?php declare(strict_types=1);


namespace Swoft\Rpc;


use function bean;
use ReflectionException;
use Swoft\Bean\Exception\ContainerException;
use Swoft\Rpc\Contract\PacketInterface;
use Swoft\Rpc\Exception\RpcException;
use Swoft\Rpc\Packet\AbstractPacket;
use Swoft\Rpc\Packet\JsonPacket;
use Swoft\Stdlib\Helper\Arr;

/**
 * Class Packet
 *
 * @since 2.0
 */
class Packet implements PacketInterface
{
    /**
     * Json packet
     */
    const JSON = 'JSON';

    /**
     * Packet type
     * 默认的packet只支持json
     * @var string
     */
    private $type = self::JSON;

    /**
     * Packet
     */
    private $packets = [];

    /**
     * @var bool
     */
    private $openEofCheck = true;

    /**
     * @var string
     */
    private $packageEof = "\r\n\r\n";

    /**
     * @var bool
     */
    private $openEofSplit = false;

    /**
     * @var AbstractPacket
     */
    private $packet;

    /**
     * @param Protocol $protocol
     *
     * @return string
     * @throws RpcException
     */
    public function encode(Protocol $protocol): string
    {
        // 获取packet实例，对传输的数据进行编码
        $packet = $this->getPacket();      
```

​					// 调用jsonPacket的encode方法对传入的protocol对象进行编码，返回编码后的内容
​					return $packet->encode($protocol);  // [go](#encode) <a name="encodeReturn"></a>

```php
  
    }

    /**
     * @param string $string
     *
     * @return Protocol
     * @throws RpcException
     */
    public function decode(string $string): Protocol
    {
        // 获取数据包处理对象
        // 当一个packet数据包传过来时,我们会使用对应的packet对象去解析这个数据包,默认支持jsonPacket
        $packet = $this->getPacket();
        // 对数据包解析,返回一个protocol实例

```

​					return $packet->decode($string); // [go](#decode) <a name="decode"></a>

```php
	}

    /**
     * @param mixed    $result
     * @param int|null $code
     * @param string   $message
     * @param null     $data
     *
     * @return string
     * @throws RpcException
     */
    public function encodeResponse($result, int $code = null, string $message = '', $data = null): string
    {
        $packet = $this->getPacket();
        return $packet->encodeResponse($result, $code, $message, $data);
    }

    /**
     * @param string $string
     *
     * @return Response
     * @throws RpcException
     */
    public function decodeResponse(string $string): Response
    {
        // 获取packet对象，
        $packet = $this->getPacket();
        return $packet->decodeResponse($string);
    }

    /**
     * @return array
     */
    public function defaultPackets(): array
    {
        return [
            self::JSON => bean(JsonPacket::class)
        ];
    }

    /**
     * @return bool
     */
    public function isOpenEofCheck(): bool
    {
        return $this->openEofCheck;
    }

    /**
     * @return string
     */
    public function getPackageEof(): string
    {
        return $this->packageEof;
    }

    /**
     * @return bool
     */
    public function isOpenEofSplit(): bool
    {
        return $this->openEofSplit;
    }

    /**
     * @return PacketInterface
     * @throws RpcException
     */
    private function getPacket(): PacketInterface
    {
        // 如果packet属性已经有具体的packet解析对象，那么直接返回
        if (!empty($this->packet)) {
            return $this->packet;
        }

        // 获取默认的数据包类型
        $packets = Arr::merge($this->defaultPackets(), $this->packets);
        // 从支持的数据包数组中选择当前处理的数据包类型,默认只支持jsonPacket
        $packet  = $packets[$this->type] ?? null;
        if (empty($packet)) {
            throw new RpcException(
                sprintf('Packet type(%s) is not supported!', $this->type)
            );
        }

        if (!$packet instanceof AbstractPacket) {
            throw new RpcException(
                sprintf('Packet type(%s) is not instanceof PacketInterface!', $this->type)
            );
        }

        // $packet = jsonPacket
        // 相当于执行了$packet->packet = this
        $packet->initialize($this);
        // 将jsonPacket同时绑定到当前数据包对象上
        $this->packet = $packet;

        return $packet;
    }
}
```



##### JsonPacket encode

```php
/**
* @param Protocol $protocol
*
* @return string
*/
public function encode(Protocol $protocol): string
{
    // protocol 对象主要规定了在客服端和服务端在通信时两者传输数据的格式
    // 通过protocol，jsonpacket对象能够按照规则去解析传入protocol里面的数据，进而返回相应的数据
    // 
    $version    = $protocol->getVersion();
    $interface  = $protocol->getInterface();
    $methodName = $protocol->getMethod();

    // self::DELIMITER = '::';
    // 将版本号、接口名、方法名按照定界符拼接，当传给RPC-Server的时候，server会知道访问哪个版本的接口以及调用接口对应的方法
    $method = sprintf('%s%s%s%s%s', $version, self::DELIMITER, $interface, self::DELIMITER, $methodName);
    // 组装发送的数据
    $data   = [
        'jsonrpc' => self::VERSION, // 现在是V2版本
        'method'  => $method, // 远程调用的方法
        'params'  => $protocol->getParams(), // 方法传入的参数
        'id'      => '',
        'ext'     => $protocol->getExt()
    ];

    // 将数据调用json_encode进行编码
    $string = JsonHelper::encode($data, JSON_UNESCAPED_UNICODE);
    
```

​			// 每一个数据包发送的时候会在结尾根据swoole是否开启EOF检测，如果开启那么会拼接指定的界定符，这里默认的界定符是 "\r\n\r\n"
​		    $string = $this->addPackageEof($string); // [go](#addPackageEof) <a name="addPackageEofReturn"></a>
​		    return $string;

```php

}
```

##### 对packet追加eof <a name="addPackageEof"></a> [back](#addPackageEofReturn)

```php
/**
* @param string $string
*
* @return string
*/
protected function addPackageEof(string $string): string
{
    // Fix mock server null
    if (empty($this->packet)) {
        return $string;
    }

    if ($this->packet->isOpenEofCheck() || $this->packet->isOpenEofSplit()) {
        // 这里调用 getPackageEof 返回 界定符为 "\r\n\r\n"
        $string .= $this->packet->getPackageEof();
    }

    return $string;
}
```

##### JsonPacket decode

```php
/**
* @param string $string
*
* @return Protocol
* @throws RpcException
*/
public function decode(string $string): Protocol
{
    // 使用json_decode解析数据包
    // swoft对json_decode方法进行了封装,主要是解析失败返回自己自定义的异常
    $data  = JsonHelper::decode($string, true);
    // 获取json解析最后一次的错误
    $error = json_last_error();
    // 如果json解析有错误抛出异常
    if ($error != JSON_ERROR_NONE) {
        throw new RpcException(
            sprintf('Data(%s) is not json format!', $string)
        );
    }

    // 获取数据包中相关属性
    $method = $data['method'] ?? '';
    $params = $data['params'] ?? [];
    $ext    = $data['ext'] ?? [];

    if (empty($method)) {
        throw new RpcException(
            sprintf('Method(%s) cant not be empty!', $string)
        );
    }

    $methodAry = explode(self::DELIMITER, $method);
    if (count($methodAry) < 3) {
        throw new RpcException(
            sprintf('Method(%s) is bad format!', $method)
        );
    }

    [$version, $interfaceClass, $methodName] = $methodAry;

    if (empty($interfaceClass) || empty($methodName)) {
        throw new RpcException(
            sprintf('Interface(%s) or Method(%s) can not be empty!', $interfaceClass, $method)
        );
    }

    // 根据解析的json结果,生成一个protocol相关实例
    return Protocol::new($version, $interfaceClass, $methodName, $params, $ext);
}
```


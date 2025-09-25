# 在 Laravel 中使用代理

[![Bright Data 促销](https://github.com/bright-cn/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn/)

本指南解释如何在 Laravel 项目中配置与实现代理，用于不被封锁的网页抓取与地理访问控制。

- [什么是 Laravel 代理？](#什么是-Laravel-代理)
- [Laravel 中代理的使用场景](#Laravel-中代理的使用场景)
- [在 Laravel 中使用代理：分步指南](#在-Laravel-中使用代理-分步指南)
- [高级用法](#高级用法)
- [在 Laravel 中使用 Bright Data 代理](#在-Laravel-中使用-Bright-Data-代理)
- [额外：Symfony 的 HttpClient 代理集成](#额外-Symfony-的-HttpClient-代理集成)

## 什么是 Laravel 代理？

Laravel 代理在你的 Laravel 应用与外部服务器之间充当中间人。它允许你以编程方式将服务器流量[通过代理服务器](https://www.bright.cn/blog/proxy-101/what-is-proxy-server)转发，以隐藏你的 IP 地址。

在 Laravel 中，代理的工作方式如下：

1. Laravel 使用配置了代理的 HTTP 客户端库发起 HTTP 请求。
2. 请求通过代理服务器传输。
3. 代理将请求转发到目标服务器。
4. 目标服务器将响应返回给代理。
5. 代理将响应转发给 Laravel。

因此，目标服务器看到的请求来源是代理的 IP，而不是你的 Laravel 服务器地址。该机制可用于绕过地理限制、增强匿名性以及处理速率限制。

## Laravel 中代理的使用场景

Laravel 中的代理用途广泛，以下三种最为常见：

- 网页抓取：在创建网页抓取 API 时使用代理，避免 IP 封禁、绕过限流或其他限制。更多信息可阅读我们的[使用 Laravel 进行网页抓取教程](https://www.bright.cn/blog/web-data/web-scraping-with-laravel)。
- 绕过第三方 API 的限流：在代理 IP 间轮换，从而保持在 API 配额内并避免被限速。
- 访问地域受限内容：选择特定位置的代理服务器，以使用仅在某些国家可用的服务。

更多示例请参阅我们的[网络数据与代理使用场景指南](https://www.bright.cn/use-cases)。

## 在 Laravel 中使用代理：分步指南

本节我们将演示如何使用[默认 HTTP 客户端](https://laravel.com/docs/master/http-client)在 Laravel 中接入代理。稍后也会介绍使用 [Symfony `HttpClient`](https://symfony.com/doc/current/http_client.html) 库的代理集成。

> 注意：
>
> Laravel 的 HTTP 客户端构建于 Guzzle 之上，你可能也想查看我们的 [Guzzle 代理集成指南](https://www.bright.cn/blog/how-tos/proxy-with-guzzle)。

作为示例，我们将创建一个 `GET` `/api/v1/get-ip` 端点，其将：

1. 使用已配置的代理向 [`https://httpbin.io/ip`](https://httpbin.io/ip) 发送 `GET` 请求；
2. 从响应中提取出口 IP 地址；
3. 将该 IP 返回给调用该 Laravel 端点的客户端。

如果配置正确，API 返回的 IP 将与代理的 IP 地址一致。

### 第一步：项目初始化

如果你已有配置好的 Laravel 应用，可直接跳到第二步。

否则，按照以下步骤创建一个新的 Laravel 项目。在终端执行以下 Composer [`create-project`](https://getcomposer.org/doc/03-cli.md#create-project) 命令初始化一个全新的 Laravel 项目：

```sh
composer create-project laravel/laravel laravel-proxies-application
```

该命令会在名为 `laravel-proxies-application` 的目录中创建一个新的 Laravel 项目。使用你喜欢的 PHP IDE 打开此文件夹。

此时，目录应包含标准的 Laravel 项目结构：

![Laravel 项目结构](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/The-Laravel-projext-structure.png)

### 第二步：定义测试 API 端点

在项目目录中，执行如下 [Artisan 命令](https://laravel.com/docs/11.x/artisan)以生成一个新的控制器：

```sh
php artisan make:controller IPController
```

这会在 `/app/Http/Controllers` 目录下创建名为 `IPController.php` 的文件，默认内容如下：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class IPController extends Controller
{
//
}
```

现在，将下面的 `getIP()` 方法添加到 `IPController.php` 中：

```php
public function getIP(): JsonResponse
{
// 通过 "/ip" 端点发起 GET 请求以获取服务器的 IP
$response = Http::get('https://httpbin.io/ip');

// 获取响应数据
$responseData = $response->json();

// 返回响应数据
return response()->json($responseData);
}
```

该方法使用 Laravel 的 `Http` 客户端从 `https://httpbin.io/ip` 获取你的 IP 地址，并以 JSON 返回。

记得引入以下两个类：

```php
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;
```

如果你希望 Laravel 应用提供无状态 API，使用 [`install:api`](https://laravel.com/docs/master/routing#api-routes) Artisan 命令启用 API 路由：

```sh
php artisan install:api
```

为了通过 API 端点暴露该方法，在 `routes/api.php` 文件中添加以下路由：

```php
use App\Http\Controllers\IPController;

Route::get('/v1/get-ip', [IPController::class, 'getIP']);
```

你的新 API 端点地址为：

```php
/api/v1/get-ip
```

注意：默认情况下，所有 Laravel API 都在 `/api` 路径下。

现在来测试 `/api/v1/get-ip` 端点！

运行以下命令启动 Laravel 开发服务器：

```sh
php artisan serve
```

你的服务器现在应在本地监听 `8000` 端口。

使用 cURL 调用 `/api/v1/get-ip` 端点：

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip'
```

注意：在 Windows 上，将 `curl` 替换为 `curl.exe`。更多内容参见我们关于[如何用 cURL 发送 GET 请求](https://www.bright.cn/faqs/curl/curl-get-requests)的指南。

你应收到类似如下的响应：

```json
{
  "origin": "45.89.222.18"
}
```

该响应与 HttpBin 的 `/ip` 端点输出完全一致，证实你的 Laravel API 工作正常。具体而言，显示的 IP 地址是你机器的公网 IP。

### 第三步：获取代理

在 Laravel 应用中使用代理，首先需要一个可用的代理服务器。

典型的代理 URL 格式如下：

```
<protocol>://<host>:<port>
```

其中：

- `protocol` 是连接代理服务器所需的协议（如 `http`、`https`、`socks5`）
- `host` 是代理服务器的 IP 地址或域名
- `port` 是转发流量所使用的端口

在本示例中，假设你的代理 URL 为：

```
http://66.29.154.103:3128
```

将其保存在 `getIP()` 方法中的变量里：

```php
$proxyUrl = 'http://66.29.154.103:3128';
```

### 第四步：在 `Http` 中集成代理

使用 `Http` 客户端在 Laravel 中集成代理，只需极少配置：

```php
$proxyUrl = 'http://66.29.154.103:3128';

$response = Http::withOptions([
proxy' => $proxyUrl
])->get('https://httpbin.io/ip');

$responseData = $response->json();
```

你需要通过 [`withOptions()`](https://laravel.com/docs/11.x/http-client#guzzle-options) 方法传入代理 URL。这样即指示 Laravel 的 HTTP 客户端使用 [Guzzle 的 `proxy` 选项](https://docs.guzzlephp.org/en/stable/request-options.html#proxy)将请求经由指定代理服务器转发。

### 第五步：整合

集成代理后的 Laravel API 最终逻辑应如下所示：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

class IPController extends Controller
{

    public function getIP(): JsonResponse
    {
        // the URL of the proxy server
        $proxyUrl = 'http://66.29.154.103:3128'; // replace with your proxy URL

        // make a GET request to /ip endpoint through the proxy server
        $response = Http::withOptions([
            'proxy' => $proxyUrl
        ])->get('https://httpbin.io/ip');

        // retrieve the response data
        $responseData = $response->json();

        // return the response data
        return response()->json($responseData);
    }
}
```

通过本地运行 Laravel 进行测试：

```sh
php artisan serve
```

然后再次访问 `/api/v1/get-ip` 端点：

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip'
```

这一次，输出应类似：

```json
{
  "origin": "66.29.154.103"
}
```

`"origin"` 字段显示的是代理服务器的 IP 地址，表明你的真实 IP 已被代理隐藏。

> 警告：
>
> 免费代理服务器通常不稳定或寿命较短。当你尝试时，示例代理可能已不可用。如有需要，在测试前用当前可用的代理替换 `$proxyUrl`。

如果在发起请求时遇到 SSL 错误，请参考下文高级用法部分提供的故障排除建议。

## 高级用法

你已经掌握了在 Laravel 中使用代理的基础，下面再介绍一些高级场景。

### 代理认证

高品质（付费）代理通常要求认证，以确保只有授权用户能访问。若缺少正确凭据，你将会遇到如下错误：

```
cURL error 56: CONNECT tunnel failed, response 407
```

带认证的代理 URL 通常使用以下格式：

```
<protocol>://<username>:<password>@<host>:<port>
```

其中 `username` 和 `password` 为认证凭据。

Laravel 的 `Http` 类（底层使用 Guzzle）完全支持认证代理，只需做很少修改——将认证信息直接写入代理 URL 即可：

```php
$proxyUrl = '<protocol>://<username>:<password>@<host>:<port>';
```

例如：

```php
// 使用带用户名和密码的认证代理
$proxyUrl = 'http://<username>:<password>@<host>:<port>';

$response = Http::withOptions([
proxy' => $proxyUrl
])->get('https://httpbin.io/ip');
```

用有效的认证代理 URL 替换 `$proxyUrl` 的值。

`Http` 现在会将流量定向到你配置的认证代理服务器！

### 避免 SSL 证书问题

在使用 Laravel 的 `Http` 客户端配置代理时，请求可能因 SSL 证书验证错误而失败，例如：

```
cURL error 60: SSL certificate problem: self-signed certificate in certificate chain
```

这通常发生在代理服务器使用了自签名的[SSL 证书](https://www.cloudflare.com/learning/ssl/what-is-an-ssl-certificate/)。

如果你信任该代理服务器，并且只是在本地或安全环境中测试，你可以临时禁用 SSL 验证：

```php
$response = Http::withOptions([
proxy' => $proxyUrl,
verify' => false, // 禁用 SSL 证书验证
])->get('https://httpbin.io/ip');
```

> 警告：
>
> 禁用 SSL 验证会让你容易遭受中间人攻击。仅在可信环境中使用此选项。

或者，如果你拥有[代理服务器的证书文件](https://docs.www.bright.cn/general/account/ssl-certificate)（例如 `proxy-ca.crt`），可以使用它进行 SSL 验证：

```php
$response = Http::withOptions([
proxy' => $proxyUrl,
verify' => storage_path('certs/proxy-ca.crt'), // CA 包路径
])->get('https://httpbin.io/ip');
```

确保 `proxy-ca.crt` 文件存储在安全且可访问的目录（如 `storage/certs/`），并且 Laravel 具有读取权限。

采用上述任一方式后，由代理引起的 SSL 验证错误应能得到解决。

### 代理轮换

如果你反复使用同一代理服务器，目标网站最终很可能会识别并封禁该代理 IP。为避免这种情况，你可以[轮换代理服务器](https://www.bright.cn/blog/how-tos/how-to-rotate-an-ip-address)——每次请求使用不同的代理。

在 Laravel 中轮换代理的步骤如下：

1. 创建一个包含多个代理 URL 的数组；
2. 在每次请求前随机选择一个；
3. 在 HTTP 客户端配置中设置所选代理。

实现代码如下：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

function getRandomProxyUrl(): string
{
// 有效代理 URL 列表（替换为你的代理列表）
$proxies = [
<protocol_1>://<proxy_host_1>:<port_1>',
// ...
<protocol_n>://<proxy_host_n>:<port_n>',
];

// 从列表中随机返回一个代理 URL
return $proxies[array_rand($proxies)];
}

class IPController extends Controller
{
public function getIP(): JsonResponse
{
// 代理服务器的 URL
$proxyUrl = getRandomProxyUrl();
// 通过代理服务器向 /ip 端点发起 GET 请求
$response = Http::withOptions([
proxy' => $proxyUrl
])->get('https://httpbin.io/ip');

// 获取响应数据
$responseData = $response->json();

// 返回响应数据
return response()->json($responseData);
}
}
```

上述代码演示了如何从列表中随机选择代理以实现代理轮换。但此方法有以下局限：

1. 你需要维护一组可靠的代理服务器，这通常会产生成本；
2. 若要有效轮换，代理池必须足够大；否则同一代理会被频繁使用，易被识别并封禁。

为克服这些挑战，可考虑使用 [Bright Data 的轮换代理网络](https://www.bright.cn/solutions/rotating-proxies)。

## 在 Laravel 中使用 Bright Data 代理

按照以下步骤在 Laravel 中使用 Bright Data 的住宅代理。

如果你还没有账号，请[注册 Bright Data](https://www.bright.cn/cp/start)。如果已有账号，请登录访问你的用户仪表盘：

![Bright Data 仪表盘](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/The-Bright-Data-dashboard.png)

登录后，点击“Get proxy products”按钮：

![点击 “Get proxy products” 按钮](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/Clicking-the-Get-proxy-products-button.png)

你将被引导至 “Proxies & Scraping Infrastructure” 页面：

![“Proxies & Scraping Infrastructure” 页面](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/The-Proxies-Scraping-Infrastructure-page-1.png)

在表格中找到 “Residential” 行并点击：

![点击 “residential” 行](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/Clicking-the-residential-row.png)

进入住宅代理页面：

![“residential” 页面](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/The-residential-page.png)

首次使用时，请根据引导向导按需配置代理服务。如需额外帮助，欢迎[联系其 24/7 支持团队](https://www.bright.cn/contact)。

在 “Overview” 选项卡中，找到你的代理主机、端口、用户名和密码：

![代理凭据](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/The-proxy-credentials.png)

使用这些信息构造你的代理 URL：

```php
$proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';
```

将占位符（`<brightdata_proxy_username>`、`<brightdata_proxy_password>`、`<brightdata_proxy_host>`、`<brightdata_proxy_port>`）替换为你的实际代理凭据。

确保将“Off”开关切换为“On”以启用代理产品，并完成其余设置步骤：

![点击启用开关](https://github.com/bright-cn/laravel-with-proxy/blob/main/images/Clicking-the-activation-toggle.png)

配置好代理 URL 后，即可通过 `Http` 客户端将其集成到 Laravel 中。以下示例展示如何通过 Bright Data 的轮换住宅代理在 Laravel 中发送请求：

```php
public function getIp()
{
// TODO: 将占位符替换为你的 Bright Data 代理信息
$proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';

// 通过代理向 "/ip" 发起 GET 请求
$response = Http::withOptions([
proxy' => $proxyUrl,
])->get('https://httpbin.org/ip');

// 获取响应数据
$responseData = $response->json();

return response()->json($responseData);
}
```

每次执行该脚本，你都会看到不同的出口 IP。

## [额外] Symfony 的 `HttpClient` 代理集成

如果你更喜欢使用 Symfony 的 `HttpClient` 组件而非 Laravel 默认的 `Http` 客户端，可按以下步骤在 Laravel 中集成 `HttpClient` 的代理。

首先，通过 Composer 安装 Symfony HTTP 客户端包：

```sh
composer require symfony/http-client
```

接着，可按如下方式在 `HttpClient` 中使用代理：

```php
<?php

namespace App\Http\Controllers;

use Symfony\Contracts\HttpClient\HttpClientInterface;
use Illuminate\Http\JsonResponse;

class IpController extends Controller
{
// 用于存放 HttpClient 实例
private $client;

public function __construct(HttpClientInterface $client)
{
// 初始化 HttpClient 私有实例
$this->client = $client;
}

public function getIp(): JsonResponse
{
// 你的代理 URL
$proxyUrl = 'http://66.29.154.103:3128'; // 将其替换为你的代理 URL

// 通过代理向 "/ip" 端点发起 GET 请求
$response = $this->client->request('GET', 'https://httpbin.io/ip', [
proxy' => $proxyUrl,
]);

// 解析响应 JSON 并返回
$responseData = $response->toArray();
return response()->json($responseData);
}
}
```

通过上述配置，你即可使用 Symfony 的 `HttpClient` 通过代理发送请求。

## 总结

免费代理服务通常不可靠且可能存在风险。若要获得一致的性能、安全性与可扩展性，你需要值得信赖的代理提供商。选择行业领先的代理提供商 Bright Data，可节省时间与精力。

立即创建账号，免费开始测试我们的代理！

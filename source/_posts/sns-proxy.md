---
title: 针对第三方注册使用代理(google,facebook,twitter)
date: 2019-05-31 11:41:28
tags: 
- php
- proxy
- sns
categories: php相关
---
## 前言
前段时间在测试服上修改第三方(google, facebook, twitter)注册代码的时候,遇到了问题，因为我们的测试服在国内，因为没法翻墙，导致每次请求都超时。
所以就针对这三个 SNS 的使用都加上了代理，当然前提是这台服务器上已经配置了代理端口了，比如 127.0.0.1:8102 类似的。 我们用的是 php 的 yii2 框架。
## google
google 很简单，SDK 有支持，直接去环境变量去取，所以在原来的代码的基础上加上这个就行了：
```html
// 测试环境: google 的请求使用代理
if (env("YII_ENV") == 'dev' && yiicfg('tpProxy')) {
    putenv('HTTPS_PROXY='.yiicfg('tpProxy'));
}
```
<!--more-->
## twitter
Twitter 也比较简单，SDK 也有支持，加个 curl_proxy 配置就行了：
```html
$config = [
    'consumer_key' => Yii::$app->params['twitterConsumerKey'],
    'consumer_secret' => Yii::$app->params['twitterConsumerSecret'],
];

// 测试环境: twitter 的请求使用代理
if (env("YII_ENV") == 'dev' && yiicfg('tpProxy')) {
    $config['curl_proxy'] = yiicfg('tpProxy');
}

$this->tmhOAuth = new \tmhOAuth($config);
```
## facebook
facebook 也是加一个配置，不过加的不是一个 代理的地址，而是一个 代理的 client 对象：
```html
public static function getFacebookClient()
{
    $config = [
        'app_id' => yiicfg('facebookClientId'),
        'app_secret' => yiicfg('facebookClientSecret'),
        'default_graph_version' => 'v2.8',
        //'default_access_token' => '{access-token}', // optional
    ];

    // 测试环境: facebook 的请求使用代理
    if (env("YII_ENV") == 'dev' && yiicfg('tpProxy')) {
        LogService::logError("************dev");
        $config['http_client_handler'] = (new FacebookProxyHttpClient());
    }

    $fb = new Facebook($config);
    return $fb;
}
```
所以我们要自己实现这个代理客户端： FacebookProxyHttpClient
```html
<?php

namespace api\components;

use Facebook\HttpClients\FacebookHttpClientInterface;
use api\services\LogService;

/**
* 注意:
* 1. 除了 send 方法的 CURLOPT 中新增了 proxy 选项，其余操作严格参照 \Facebook\HttpClients\FacebookCurlHttpClient
* 2. 仅限测试环境使用
*
* Class FacebookProxyHttpClient
*/
class FacebookProxyHttpClient implements FacebookHttpClientInterface
{
    /**
     * FacebookProxyHttpClient constructor.
     */
    public function __construct()
    {
        LogService::logError("====Custom facebook proxy http client====");
    }

    /**
     * @param string $url
     * @param string $method
     * @param string $body
     * @param array $headers
     * @param int $timeOut
     * @return \Facebook\Http\GraphRawResponse
     * @throws \Facebook\Exceptions\FacebookSDKException
     */
    public function send($url, $method, $body, array $headers, $timeOut)
    {
        // 1. 初始化 curl 资源并设置参数
        $ch = curl_init();
        $options = [
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_URL => $url,
            CURLOPT_HTTPHEADER => $this->compileRequestHeaders($headers),
            CURLOPT_CONNECTTIMEOUT => 10,
            CURLOPT_TIMEOUT => $timeOut,
            CURLOPT_RETURNTRANSFER => true, // Follow 301 redirects
            CURLOPT_HEADER => true, // Enable header processing
            CURLOPT_SSL_VERIFYHOST => 2,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_CAINFO => __DIR__ . '/certs/DigiCertHighAssuranceEVRootCA.pem',
            CURLOPT_PROXY => yiicfg('tpProxy')
        ];

        LogService::logError("***********" . __DIR__ . '/certs/DigiCertHighAssuranceEVRootCA.pem');
        if ($method !== "GET") {
            $options[CURLOPT_POSTFIELDS] = $body;
        }
        curl_setopt_array($ch, $options);

        // 2. 执行
        $rawResponse = curl_exec($ch);

        // 3. 检测并抛出 curl 异常
        $curlErrorCode = curl_errno($ch);
        $curlError = curl_error($ch);
        if ($curlErrorCode) {
            throw new \Facebook\Exceptions\FacebookSDKException($curlErrorCode, $curlErrorCode);
        }

        // 4. 从原生 response 中提取 header 和 body
        list($rawHeaders, $rawBody) = $this->extractResponseHeadersAndBody($rawResponse);

        // 5. 关闭 curl
        curl_close($ch);

        // 6. 返回 response
        return new \Facebook\Http\GraphRawResponse($rawHeaders, $rawBody);
    }


    /**
     * Compiles the request headers into a curl-friendly format.
     *
     * @param array $headers The request headers.
     *
     * @return array
     */
    public function compileRequestHeaders(array $headers)
    {
        $return = [];

        foreach ($headers as $key => $value) {
            $return[] = $key . ': ' . $value;
        }

        return $return;
    }


    /**
     * Extracts the headers and the body into a two-part array
     *
     * @return array
     */
    public function extractResponseHeadersAndBody($rawResponse)
    {
        $parts = explode("\r\n\r\n", $rawResponse);
        $rawBody = array_pop($parts);
        $rawHeaders = implode("\r\n\r\n", $parts);

        return [trim($rawHeaders), trim($rawBody)];
    }
}
```
这样就可以了。






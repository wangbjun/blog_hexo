---
title: Token 和 JWT Token
date: 2019-01-03 21:05:45
tags: JWT
category: 编程开发
---

## 1.Token的用途
在很多计算机系统里面都少不了用户认证这一步骤,最常见的认证就是账号密码认证,也就是注册、登录这一流程。

在现实生活中,人也需要认证,大家应该都有个 **身份证**,回想一下这个身份证是从哪里来的呢? 
办过身份证的应该都知道,一般情况下,身份证需要本人带着 **户口本** 去 **公安局** (不知道现在改了木有?)办理,工作人员在核对了相关信息,确认无误的情况下会给你颁发一个身份证, **有效期** 一般是10-20年,在一些需要认证的时候,你就可以拿出身份证 **校验** 核对身份,比如买火车票,出国,或者办理其它证件.

<!--more-->

很多Web系统里面token就类似于身份证,账号密码就相当于咱的户口本和本人,需要核对账号密码后获取,拿到token之后就可以使用一些需要认证的服务,而且token也有有效期，和身份证一样,理论上token必须是唯一。

## 2.常见的Web认证方式

### 1.HTTP Basic Auth
这种方式在早期一些Web系统比较常见，就是那种在浏览器弹出一个框让你输账号密码那种，简单易用，但是缺点一个不安全，其账号密码其实是明文（base64encode）传输的，而且每次都得带上。另外就是太丑了。。。

![](https://ww1.sinaimg.cn/large/5f6e3e27ly1fz041egnonj20ci08sgmf.jpg)


### 2.Cookies\Session
这种认证方式其实就是类似我们最开始说的身份证这种，只需要输入一次账号密码，认证成功后，系统会将用户信息存入**session**，session是服务器的本地存储功能，然后系统根据session生成一个唯一的 **sessionid** 以cookies的形式发送给浏览器。

**cookies**是浏览器本地存储，在这套机制里面的作用是用来存储sessionid，你也可以不使用cookies存储，早期有些网站在一些不支持cookies的浏览器上面会把sessionid追加到url上面。

cookies里面存储的sessionid其实就是相当于身份证编号，每次访问网站里面我们带着这个编号，服务器拿着编号就可以找到对应的session里面存储的信息，一般情况下里面会存储一些用户信息，比如uid。

![](https://ww1.sinaimg.cn/large/5f6e3e27ly1fz04jhi9wij20lr03v74q.jpg)

讲道理这套机制其实问题并不大，大部分时候都管用，但是cookies有一个毛病就是无法跨域，很多大公司有很多网站，这些网站域名可能还不一样。而且cookies对现在的手机APP支持不好，原生并不支持cookies。最后，就是服务器存储session也需要一些开销，特别是用户特别多的情况下。还有其它缺点这里就不列出来了，很多文章都有写到。

但是其实我想说这套机制大部分情况下是够用的，特别是对于一些中小型网站来说，简单易用，快速开发。


### 3.JWT
一般说到JWT都会提到token，在我的理解里面token其实就是一个字符串，它可以是jwt token，也可以是sessionid token，token就是是一个携带认证信息的字符串。

网上关于介绍JWT的文章特别多，大同小异，我们这里也懒的再说一遍了，贴一个大神的教程，我觉得讲的挺清晰了，[JSON Web Token 入门教程](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)。

简单的说，JWT本质上是一种解决方案标准，该方案下一个token应该有3部分组成: **Header、Payload、Signature**, 其中前2部分差不多就是明文的，都是**json** 对象，里面存了一些信息，使用 **base64urlencode** 编码成一个字符串。最后的 **Signature** 是前面2个元素和**secret**一起加密之后的结果,加密算法默认是 **SHA256**, 这个**secret**应该只有服务器知道，解密的时候需要用到。

最后生成的token是一个比较长的字符串，当用户登录成功之后可以把这个串返回给浏览器，浏览器下次请求的时候带着这个串就行了，问题来了，怎么带？很多文章说放到cookies里面，讲道理放到cookies里面那和sessionid有啥区别？ 标准做法是放到HTTP请求的头信息Authorization字段里面。

服务器拿到这个串，首先把前面2段的Header和Payload使用 **base64urldecode** 解码出来，然后使用刚才使用的加密算法和secret校验一下是否和第3段的signature一样，如果不一样，则说明这个Token是伪造的，如果一样，就可以相信Payload里面的信息了，一般Payload里面会存放一些用户信息，比如uid，如果Payload里面需要存放一些敏感信息，比如手机号，建议先加密Payload。

### PHP实战
下面我将使用PHP构建一个简单的例子：

#### JWT类：
```php
<?php

namespace App;

class Jwt
{
    private $alg = 'sha256';

    private $secret = "123456";

    /**
     * alg属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256）；typ属性表示这个令牌（token）的类型（type），JWT 令牌统一写为JWT
     */
    public function getHeader()
    {
        $header = [
            'alg' => $this->alg,
            'typ' => 'JWT'
        ];

        return $this->base64urlEncode(json_encode($header, JSON_UNESCAPED_UNICODE));
    }

    /**
     * Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用，这里可以存放私有信息，比如uid
     * @param $uid int 用户id
     * @return mixed
     */
    public function getPayload($uid)
    {
        $payload = [
            'iss' => 'admin', //签发人
            'exp' => time() + 600, //过期时间
            'sub' => 'test', //主题
            'aud' => 'every', //受众
            'nbf' => time(), //生效时间
            'iat' => time(), //签发时间
            'jti' => 10001, //编号
            'uid' => $uid, //私有信息，uid
        ];

        return $this->base64urlEncode(json_encode($payload, JSON_UNESCAPED_UNICODE));
    }

    /**
     * 生成token,假设现在payload里面只存一个uid
     * @param $uid int
     * @return string
     */
    public function genToken($uid)
    {
        $header  = $this->getHeader();
        $payload = $this->getPayload($uid);

        $raw   = $header . '.' . $payload;
        $token = $raw . '.' . hash_hmac($this->alg, $raw, $this->secret);

        return $token;
    }


    /**
     * 解密校验token,成功的话返回uid
     * @param $token
     * @return mixed
     */
    public function verifyToken($token)
    {
        if (!$token) {
            return false;
        }
        $tokenArr = explode('.', $token);
        if (count($tokenArr) != 3) {
            return false;
        }
        $header    = $tokenArr[0];
        $payload   = $tokenArr[1];
        $signature = $tokenArr[2];

        $payloadArr = json_decode($this->base64urlDecode($payload), true);

        if (!$payloadArr) {
            return false;
        }

        //已过期
        if (isset($payloadArr['exp']) && $payloadArr['exp'] < time()) {
            return false;
        }

        $expected = hash_hmac($this->alg, $header . '.' . $payload, $this->secret);

        //签名不对
        if ($expected !== $signature) {
            return false;
        }

        return $payloadArr['uid'];
    }

    /**
     * 安全的base64 url编码
     * @param $data
     * @return string
     */
    private function base64urlEncode($data)
    {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }

    /**
     * 安全的base64 url解码
     * @param $data
     * @return bool|string
     */
    private function base64urlDecode($data)
    {
        return base64_decode(str_pad(strtr($data, '-_', '+/'), strlen($data) % 4, '=', STR_PAD_RIGHT));
    }
}
```

#### 测试：
```php
<?php
$jwt = new \App\Jwt();

//获取token
$token = $jwt->genToken(1);

//解密token
$uid = $jwt->verifyToken($token);

var_dump($uid);
```
以上代码仅供参考，实际应用的话最好找个现成的库，不推荐重复造轮子，jwt的思想是通用的，不分语言，github上面有很多。。。这里贴一个PHP的库: [firebase/php-jwt](https://github.com/firebase/php-jwt)。

最后再说说session和jwt的选择问题，网上随便搜搜就可以看到很多文章比较这2者优劣，总结就是各有利弊，实际上很多公司既不是session，也不是jwt，可能就是自己搞的类似jwt token这样的一个字符串，然后放在cookies里面，只要这个串能够代表一个用户都可以。

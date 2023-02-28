# cdn-edge-script
CDN, Cloud edge script example

如何利用海外CDN或Cloud的Edge脚本功能代理一个网站

非常适合用来下载和传播某个网站的可下载文件，或视频音频等

## 关于Edge
CDN所采用Web服务器可以执行一些简单的脚本，这样不太复杂的任务就不需要转发给另一台运行php的中央服务器，消除了网络瓶颈。因为分布式的CDN服务器更靠近用户，所以这种计算方式称作Edge边缘计算。

很多知名的Cloud/CDN运营商都支持Edge边缘计算，语法等可能略有差异，下面仅以cloudflare举例，其它平台大家可以举一反三，参考他们平台的文档和范例。

## 增加 mhdownload 服务

* 注册一个cloudflare账户，点左边菜单Workers->Create Service, 第一次运行会要求你创建一个xxx.workers.dev的二级域名，取一个不太敏感的二级域名，不要用默认值，默认值是你的账号邮箱。

* cloudflare会生成一个随机的serivce名，可以修改成好记的，也可以用它随机生成的。这里取名为mhdownload。需要选择脚本模板，随便选一个，点create service按钮

* 在出现的代码框里把原来的代码删掉，把下面代码复制到代码框：
```
const SITE = 'minghui.org';
const USER = 'kuqixinzhi';
const PASS = 'ymdfgckdcllsbskxxzngggddcccdsmbkyqjkqrhhcdsskcssft';

export default {
  async fetch(request, env) {
    async function noAccess() {
        return new Response('You need to login.', {
          status: 401,
          headers: {
            'WWW-Authenticate': 'Basic realm="my scope", charset="UTF-8"',
          },
        });
    }

    var access = false;
    async function basicAuthentication(request) {
      const Authorization = request.headers.get('Authorization');
      if(!Authorization) return false;
      const [scheme, encoded] = Authorization.split(' ');
      if (!encoded || scheme !== 'Basic') return false;

      const buffer = Uint8Array.from(atob(encoded), (character) =>
        character.charCodeAt(0)
      );
      const decoded = new TextDecoder().decode(buffer).normalize();
      const index = decoded.indexOf(':');
      if (index === -1 || /[\0-\x1F\x7F]/.test(decoded)) return false;
      let vuser = decoded.substring(0, index).trim();
      let vpass = decoded.substring(index + 1);
      if((vuser==USER) && (vpass==PASS)) access = true;
    }

    await  basicAuthentication(request);
    if(!access) return noAccess();

    let url = new URL(request.url);
    let service = url.host.split('.')[0]+'.';
    let vhost = url.host.replace(service,'');
    let turl = request.url.replace(vhost, SITE);
    
    let target = new Request(turl, {
      body: request.body,
      headers: request.headers,
      method: request.method,
      redirect: request.redirect
    });
    return fetch(target);
  }
}
```
* 修改代码里的SITE, USER, PASS 的值为你想要访问的网站，访问用户名和密码， 点下方的“Save and Deploy"按钮

* 点"Preview"标签查看效果，输入你设置的用户名和密码即可访问。但workers.dev一般情况下是封锁的，所以还需要映射成一个其他网址


## 设置下载链接

比如我们需要分享明慧最新文章[为什么会有人类](https://www.minghui.org/mh/articles/2023/1/20/%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E6%9C%89%E4%BA%BA%E7%B1%BB-455562.html)的PDF文件，文件链接是https://package.minghui.org/jw/2023/1/20/jw20230120-letter.zip

* 如果你已经在cloudflare上有一个叫 example.com 的域名，则需要增加一条dns记录，如: *.example.com

* 设置新增dns记录 Name: * , Type: CNAME , Target: workers.dev

* 再在examples.com网站上增加 Workers Route， 选 Workers Route -> Http Route -> Add Route

* Route 填：\*.example.com\/\* ,  Service 填：dnsquery

原始文件链接是:  https://package.minghui.com/jw/2023/1/20/jw20230120-letter.zip

新的下载链接是:  https://package.example.com/jw/2023/1/20/jw20230120-letter.zip

其他下载链接也以此类推


## 使用限制

虽然理论上它可以代理一个完整网站，但如果这个网站有链接其它被封锁网站的资源，浏览器会直接访问链接的资源，访问会失败且这种访问也不大安全，不建议在敏感网站上这样访问。

可选则注释掉代码中验证用户密码的部分，但会增加域名example.com被暴露的风险。这两行修改为：
```
    //await  basicAuthentication(request);
    //if(!access) return noAccess();
```

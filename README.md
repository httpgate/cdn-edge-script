# edge-worker-script
CDN, Cloud edge worker example
如何利用海外CDN或Cloud的Edge脚本功能代理一个网站

## 关于Edge
CDN所采用Web服务器可以执行一些简单的脚本，这样不太复杂的任务就不需要转发给另一台运行php的服务器，消除了网络瓶颈。
很多知名的Cloud/CDN运营商都支持Edge边缘计算，语法等可能略有差异，下面仅以cloudflare举例，其它平台大家可以举一反三，参考他们平台的文档和范例。

## 使用
* 注册一个cloudflare账户，点左边菜单Workers->Create Service, 第一次运行会要求你创建一个xxx.worker.dev的二级域名，取一个不太敏感的二级域名，不要用默认值，默认值是你的账号邮箱。

* cloudflare会生成一个随机的serivce名，可以修改成好记的，也可以用它随机生成的。需要选择脚本模板，随便选一个，点create service按钮

* 在出现的代码框里把原来的代码删掉，把下面代码复制到代码框：
```
const SITE = 'www.minghui.org';
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
    let turl = request.url.replace(url.host, SITE);
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

* 点"Previe"标签查看效果，输入你设置的用户名和密码即可访问。 "Previe"标签下的网址可以分享给朋友访问。

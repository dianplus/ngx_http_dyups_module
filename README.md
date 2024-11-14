# Nginx ngx_http_dyups_module

## Description

This module can be used to update your upstream-list without reloadding Nginx.

## Example

file: `conf/nginx.conf`

`ATTENTION`: You MUST use nginx variable to do proxy_pass

```nginx
    daemon off;
    error_log logs/error.log debug;

    events {
    }

    http {

        include conf/upstream.conf;

        server {
            listen   8080;

            location / {
                # The upstream here must be a nginx variable
                proxy_pass http://$dyups_host;
            }
        }

        server {
            listen 8088;
            location / {
                return 200 "8088";
            }
        }

        server {
            listen 8089;
            location / {
                return 200 "8089";
            }
        }

        server {
            listen 8081;
            location / {
                dyups_interface;
            }
        }
    }
```

If your original config looks like this:

```nginx
    proxy_pass http://upstream_name;
```

please replace it with:

```nginx
    set $ups upstream_name;
    proxy_pass http://$ups;
```

`$ups` can be any valid nginx variable.

file: conf/upstream.conf

```nginx
    upstream host1 {
        server 127.0.0.1:8088;
    }

    upstream host2 {
        server 127.0.0.1:8089;
    }
```

## Installation

* Only install dyups module

```bash
git clone git://github.com/yzprofile/ngx_http_dyups_module.git

# to compile as a static module
./configure --add-module=./ngx_http_dyups_module

# to compile as a dynamic module
./configure --add-dynamic-module=./ngx_http_dyups_module
```

* Install dyups module with lua-nginx-module and upstream check module
    * upstream check module: To make upstream check module work well with dyups module, you should use [patched upstream check module](https://github.com/yzprofile/nginx_upstream_check_module).
    * lua-nginx-module: To enable [dyups LUA API](#lua-api-example), you MUST put `--add-module=./lua-nginx-module` in front of `--add-module=./ngx_http_dyups_module` in the `./configure` command.

```bash
git clone git://github.com/yzprofile/ngx_http_dyups_module.git
git clone git@github.com:yzprofile/nginx_upstream_check_module.git
git clone git@github.com:openresty/lua-nginx-module.git

# to compile as a static module
./configure \
  --add-module=./nginx_upstream_check_module \
  --add-module=./lua-nginx-module \
  --add-module=./ngx_http_dyups_module
```

## Directives

### dyups_interface

- Syntax: **dyups_interface**
- Default: `none`
- Context: `loc`

This directive set the interface location where you can add or delete the upstream list. See the section of Interface for detail.

### dyups_read_msg_timeout

- Syntax: **dyups_read_msg_timeout** `time`
- Default: `1s`
- Context: `main`

This directive set the interval of workers readding the commands from share memory.

### dyups_shm_zone_size

- Syntax: **dyups_shm_zone_size** `size`
- Default: `2MB`
- Context: `main`

This directive set the size of share memory which used to store the commands.

### dyups_upstream_conf

- Syntax: **dyups_upstream_conf** `path`
- Default: `none`
- Context: `main`

This directive has been deprecated.

### dyups_trylock

- Syntax: **dyups_trylock** `on | off`
- Default: `off`
- Context: `main`

You will get a better prefomance but it maybe not stable, and you will get a '409' when the update request conflicts with others.

### dyups_read_msg_log

- Syntax: **dyups_read_msg_log** `on | off`
- Default: `off`
- Context: `main`

You can enable / disable log of workers readding the commands from share memory. The log looks like:

> 2017/02/28 15:37:53 [info] 56806#0: [dyups] has 0 upstreams, 1 static, 0 deleted, all 1

## Restful interfaces

### GET

- `/detail`         get all upstreams and their servers
- `/list`           get the list of upstreams
- `/upstream/name`  find the upstream by it's name

### POST

- `/upstream/name`  update one upstream
- `body` commands;
- `body` server ip:port;

### DELETE

- `/upstream/name`  delete one upstream

Call the interface, when you get the return code is `HTTP_INTERNAL_SERVER_ERROR 500`, you need to reload nginx to make the Nginx work at a good state.

If you got `HTTP_CONFLICT 409`, you need resend the same commands again latter.

The /list and /detail interface will return `HTTP_NO_CONTENT 204` when there is no upstream.

Other code means you should modify your commands and call the interface again.

`ATTENTION`: You also need a `third-party` to generate the new config and dump it to Nginx'conf directory.

### Sample

```bash
curl -H "host: dyhost" 127.0.0.1:8080
```

> <html>
> <head><title>502 Bad Gateway</title></head>
> <body bgcolor="white">
> <center><h1>502 Bad Gateway</h1></center>
> <hr><center>nginx/1.3.13</center>
> </body>
> </html>

```bash
curl -d "server 127.0.0.1:8089;server 127.0.0.1:8088;" 127.0.0.1:8081/upstream/dyhost
```

> success

```bash
curl -H "host: dyhost" 127.0.0.1:8080
```

> 8089

```bash
curl -H "host: dyhost" 127.0.0.1:8080
```

> 8088

```bash
curl 127.0.0.1:8081/detail
```

> host1
> server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
>
> host2
> server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
>
> dyhost
> server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
> server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

```bash
curl -i -X DELETE 127.0.0.1:8081/upstream/dyhost
```

> success

```bash
curl 127.0.0.1:8081/detail
```

> host1
> server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
>
> host2
> server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

## C API

```c
extern ngx_flag_t ngx_http_dyups_api_enable;
ngx_int_t ngx_dyups_update_upstream(ngx_str_t *name, ngx_buf_t *buf, ngx_str_t *rv);
ngx_int_t ngx_dyups_delete_upstream(ngx_str_t *name, ngx_str_t *rv);

extern ngx_dyups_add_upstream_filter_pt ngx_dyups_add_upstream_top_filter;
extern ngx_dyups_del_upstream_filter_pt ngx_dyups_del_upstream_top_filter;
```

## Lua API Example

NOTICE:

You should add the directive `dyups_interface` into your config file to active this feature

```lua
content_by_lua '
    local dyups = require "ngx.dyups"

    local status, rv = dyups.update("test", [[server 127.0.0.1:8088;]]);
    ngx.print(status, rv)
    if status ~= ngx.HTTP_OK then
        ngx.print(status, rv)
        return
    end
    ngx.print("update success")

    status, rv = dyups.delete("test")
    if status ~= ngx.HTTP_OK then
        ngx.print(status, rv)
        return
    end
    ngx.print("delete success")
';
```

## Change Log

### RELEASE v0.3.0

- Fix: memory leak of ssl session reuse.
- Each processes starts read_msg_timer separately at random timeout.
- Fix: unlocking behavior.
- Fix: compilation error without upstream check module.
- Fix: if a domain name contains multiple IP addresses, call them.
- When dyups and health check module together use, ngx_shmtx_lock block too long time and cpu full load, cause health check timeout, 502.
- Fix: build error when compiled with a higher version of OpenSSL.
- etc.

### RELEASE V0.2.9

Feature: Added add/del upstream filter to make other modules operate upstream easily after upstream changed

### RELEASE V0.2.8

Bugfixes: upstream connect failed caused coredump

### RELEASE V0.2.7

Supported: C API and Lua API

### RELEASE V0.2.6

Bugfixes: Supported sandbox before updatting

### RELEASE V0.2.5

1. Bugfixes: wrong string comparison for string "upstream", @chobits
2. Bugfixes: that response of /detail uri has no Content-Length header, @chobits
3. Feature: if you use this [branch of tengine](https://github.com/alibaba/tengine/tree/jst), update upstream rbtree, @SarahWang
4. Feature: simplify upstream parsing methods via ngx_conf_parse api, @chobits

### RELEASE V0.2.4

1. Bugfixes: client timed out cause a coredumped while adding an exist upstream
2. Bugfixes: when proxy_pass to a no-variable address dyups will coredump

### RELEASE V0.2.2

Bugfixed: upstream will be deleted in the process of finding upstream.

### RELEASE V0.2.0

1. check every command to make sure they are all ok before update upstream. `done`
2. support ip_hash and keepalive or other upstream module `done`
3. support `weight`,`max_fails`,`fail_timeout`,`backup` `done`
4. support health check module, you should use [this branch of Tengine](https://github.com/yaoweibin/tengine/tree/dynamic_upstream_check) or wait for it's release. `done`
5. restore upstream configuration in `init process` handler. `done`

## Compatibility

### Tengine Compatibility

This module has been merged into Tengine.

* [Tengine dyups module documentation](http://tengine.taobao.org/document/http_dyups.html)

### Module Compatibility

* [lua-upstream-nginx-module](https://github.com/agentzh/lua-upstream-nginx-module): You can use `lua-upstream-nginx-module` to get more detail infomation of upstream.
* [yzprofile/upstream check module](https://github.com/yzprofile/nginx_upstream_check_module): To make upstream check module work well with dyups module, you should use [patched upstream check module](https://github.com/yzprofile/nginx_upstream_check_module).

## Run Tests

```bash
hg clone http://hg.nginx.org/nginx-tests/ ..
# Or use git as below if you prefer the GitHub mirror
git clone git@github.com:nginx/nginx-tests/ ..
TEST_NGINX_BINARY=/path/to/your/nginx/dir/sbin/nginx prove -I ../nginx-tests/lib ./t/dyups.t
```

To make the tests pass, you should also install lua-nginx-module and patched upstream check module.

## Author

* yzprofile (袁茁) <yzprofile@gmail.com>
* chobits (王笑臣) <wangxiaochen0@gmail.com>, Alibaba Inc.

## Copyright & License

> These codes are licenced under the BSD license.
>
> Copyright (C) 2012-2020 by Zhuo Yuan (yzprofile) <yzprofiles@gmail.com>
>
> All rights reserved.
>
> Redistribution and use in source and binary forms, with or without
> modification, are permitted provided that the following conditions
> are met:
>
> * Redistributions of source code must retain the above copyright notice,
>   this list of conditions and the following disclaimer.
> * Redistributions in binary form must reproduce the above copyright notice,
>   this list of conditions and the following disclaimer in the documentation
>   and/or other materials provided with the distribution.
>
> THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
> "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
> LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
> A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
> HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
> SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
> TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
> PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
> LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
> NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
> SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

# binhelu-peoms
binhelu-peoms 冰河路诗歌资料 

ngx_http_lua_module - Embed the power of Lua into Nginx HTTP Servers.

*This module is not distributed with the Nginx source.* See [the installation instructions](#installation).

Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Version](#version)
* [Synopsis](#synopsis)
* [Description](#description)
* [Typical Uses](#typical-uses)
* [Nginx Compatibility](#nginx-compatibility)
* [Installation](#installation)
    * [Building as a dynamic module](#building-as-a-dynamic-module)
    * [C Macro Configurations](#c-macro-configurations)
    * [Installation on Ubuntu 11.10](#installation-on-ubuntu-1110)
* [Community](#community)
    * [English Mailing List](#english-mailing-list)
    * [Chinese Mailing List](#chinese-mailing-list)
* [Code Repository](#code-repository)
* [Bugs and Patches](#bugs-and-patches)
* [Lua/LuaJIT bytecode support](#lualuajit-bytecode-support)
* [System Environment Variable Support](#system-environment-variable-support)
* [HTTP 1.0 support](#http-10-support)
* [Statically Linking Pure Lua Modules](#statically-linking-pure-lua-modules)
* [Data Sharing within an Nginx Worker](#data-sharing-within-an-nginx-worker)
* [Known Issues](#known-issues)
    * [TCP socket connect operation issues](#tcp-socket-connect-operation-issues)
    * [Lua Coroutine Yielding/Resuming](#lua-coroutine-yieldingresuming)
    * [Lua Variable Scope](#lua-variable-scope)
    * [Locations Configured by Subrequest Directives of Other Modules](#locations-configured-by-subrequest-directives-of-other-modules)
    * [Cosockets Not Available Everywhere](#cosockets-not-available-everywhere)
    * [Special Escaping Sequences](#special-escaping-sequences)
    * [Mixing with SSI Not Supported](#mixing-with-ssi-not-supported)
    * [SPDY Mode Not Fully Supported](#spdy-mode-not-fully-supported)
    * [Missing data on short circuited requests](#missing-data-on-short-circuited-requests)
* [TODO](#todo)
* [Changes](#changes)
* [Test Suite](#test-suite)
* [Copyright and License](#copyright-and-license)
* [See Also](#see-also)
* [Directives](#directives)
* [Nginx API for Lua](#nginx-api-for-lua)
* [Obsolete Sections](#obsolete-sections)
    * [Special PCRE Sequences](#special-pcre-sequences)

Status
======

Production ready.

Version
=======

This document describes ngx_lua [v0.10.7](https://github.com/openresty/lua-nginx-module/tags) released on 4 November 2016.

Synopsis
========
```nginx

 # set search paths for pure Lua external libraries (';;' is the default path):
 lua_package_path '/foo/bar/?.lua;/blah/?.lua;;';

 # set search paths for Lua external libraries written in C (can also use ';;'):
 lua_package_cpath '/bar/baz/?.so;/blah/blah/?.so;;';

 server {
     location /lua_content {
         # MIME type determined by default_type:
         default_type 'text/plain';

         content_by_lua_block {
             ngx.say('Hello,world!')
         }
     }

     location /nginx_var {
         # MIME type determined by default_type:
         default_type 'text/plain';

         # try access /nginx_var?a=hello,world
         content_by_lua_block {
             ngx.say(ngx.var.arg_a)
         }
     }

     location = /request_body {
         client_max_body_size 50k;
         client_body_buffer_size 50k;

         content_by_lua_block {
             ngx.req.read_body()  -- explicitly read the req body
             local data = ngx.req.get_body_data()
             if data then
                 ngx.say("body data:")
                 ngx.print(data)
                 return
             end

             -- body may get buffered in a temp file:
             local file = ngx.req.get_body_file()
             if file then
                 ngx.say("body is in file ", file)
             else
                 ngx.say("no body found")
             end
         }
     }

     # transparent non-blocking I/O in Lua via subrequests
     # (well, a better way is to use cosockets)
     location = /lua {
         # MIME type determined by default_type:
         default_type 'text/plain';

         content_by_lua_block {
             local res = ngx.location.capture("/some_other_location")
             if res then
                 ngx.say("status: ", res.status)
                 ngx.say("body:")
                 ngx.print(res.body)
             end
         }
     }

     location = /foo {
         rewrite_by_lua_block {
             res = ngx.location.capture("/memc",
                 { args = { cmd = "incr", key = ngx.var.uri } }
             )
         }

         proxy_pass http://blah.blah.com;
     }

     location = /mixed {
         rewrite_by_lua_file /path/to/rewrite.lua;
         access_by_lua_file /path/to/access.lua;
         content_by_lua_file /path/to/content.lua;
     }

     # use nginx var in code path
     # WARNING: contents in nginx var must be carefully filtered,
     # otherwise there'll be great security risk!
     location ~ ^/app/([-_a-zA-Z0-9/]+) {
         set $path $1;
         content_by_lua_file /path/to/lua/app/root/$path.lua;
     }

     location / {
        client_max_body_size 100k;
        client_body_buffer_size 100k;

        access_by_lua_block {
            -- check the client IP address is in our black list
            if ngx.var.remote_addr == "132.5.72.3" then
                ngx.exit(ngx.HTTP_FORBIDDEN)
            end

            -- check if the URI contains bad words
            if ngx.var.uri and
                   string.match(ngx.var.request_body, "evil")
            then
                return ngx.redirect("/terms_of_use.html")
            end

            -- tests passed
        }

        # proxy_pass/fastcgi_pass/etc settings
     }
 }
```

[Back to TOC](#table-of-contents)

Description
===========

This module embeds Lua, via the standard Lua 5.1 interpreter or [LuaJIT 2.0/2.1](http://luajit.org/luajit.html), into Nginx and by leveraging Nginx's subrequests, allows the integration of the powerful Lua threads (Lua coroutines) into the Nginx event model.

Unlike [Apache's mod_lua](https://httpd.apache.org/docs/trunk/mod/mod_lua.html) and [Lighttpd's mod_magnet](http://redmine.lighttpd.net/wiki/1/Docs:ModMagnet), Lua code executed using this module can be *100% non-blocking* on network traffic as long as the [Nginx API for Lua](#nginx-api-for-lua) provided by this module is used to handle
requests to upstream services such as MySQL, PostgreSQL, Memcached, Redis, or upstream HTTP web services.

At least the following Lua libraries and Nginx modules can be used with this ngx_lua module:

* [lua-resty-memcached](https://github.com/openresty/lua-resty-memcached)
* [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql)
* [lua-resty-redis](https://github.com/openresty/lua-resty-redis)
* [lua-resty-dns](https://github.com/openresty/lua-resty-dns)
* [lua-resty-upload](https://github.com/openresty/lua-resty-upload)
* [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)
* [lua-resty-lock](https://github.com/openresty/lua-resty-lock)
* [lua-resty-logger-socket](https://github.com/cloudflare/lua-resty-logger-socket)
* [lua-resty-lrucache](https://github.com/openresty/lua-resty-lrucache)
* [lua-resty-string](https://github.com/openresty/lua-resty-string)
* [ngx_memc](http://github.com/openresty/memc-nginx-module)
* [ngx_postgres](https://github.com/FRiCKLE/ngx_postgres)
* [ngx_redis2](http://github.com/openresty/redis2-nginx-module)
* [ngx_redis](http://wiki.nginx.org/HttpRedisModule)
* [ngx_proxy](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)
* [ngx_fastcgi](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)

Almost all the Nginx modules can be used with this ngx_lua module by means of [ngx.location.capture](#ngxlocationcapture) or [ngx.location.capture_multi](#ngxlocationcapture_multi) but it is recommended to use those `lua-resty-*` libraries instead of creating subrequests to access the Nginx upstream modules because the former is usually much more flexible and memory-efficient.

The Lua interpreter or LuaJIT instance is shared across all the requests in a single nginx worker process but request contexts are segregated using lightweight Lua coroutines.

Loaded Lua modules persist in the nginx worker process level resulting in a small memory footprint in Lua even when under heavy loads.

This module is plugged into NGINX's "http" subsystem so it can only speaks downstream communication protocols in the HTTP family (HTTP 0.9/1.0/1.1/2.0, WebSockets, and etc).
If you want to do generic TCP communications with the downstream clients, then you should use the [ngx_stream_lua](https://github.com/openresty/stream-lua-nginx-module#readme) module instead
which has a compatible Lua API.

[Back to TOC](#table-of-contents)

Typical Uses
============

Just to name a few:

* Mashup'ing and processing outputs of various nginx upstream outputs (proxy, drizzle, postgres, redis, memcached, and etc) in Lua,
* doing arbitrarily complex access control and security checks in Lua before requests actually reach the upstream backends,
* manipulating response headers in an arbitrary way (by Lua)
* fetching backend information from external storage backends (like redis, memcached, mysql, postgresql) and use that information to choose which upstream backend to access on-the-fly,
* coding up arbitrarily complex web applications in a content handler using synchronous but still non-blocking access to the database backends and other storage,
* doing very complex URL dispatch in Lua at rewrite phase,
* using Lua to implement advanced caching mechanism for Nginx's subrequests and arbitrary locations.

The possibilities are unlimited as the module allows bringing together various elements within Nginx as well as exposing the power of the Lua language to the user. The module provides the full flexibility of scripting while offering performance levels comparable with native C language programs both in terms of CPU time as well as memory footprint. This is particularly the case when LuaJIT 2.x is enabled.

Other scripting language implementations typically struggle to match this performance level.

The Lua state (Lua VM instance) is shared across all the requests handled by a single nginx worker process to minimize memory use.

[Back to TOC](#table-of-contents)

Nginx Compatibility
===================

The latest version of this module is compatible with the following versions of Nginx:

* 1.11.x  (last tested: 1.11.2)
* 1.10.x
* 1.9.x (last tested: 1.9.15)
* 1.8.x
* 1.7.x (last tested: 1.7.10)
* 1.6.x

Nginx cores older than 1.6.0 (exclusive) are *not* supported.

[Back to TOC](#table-of-contents)

Installation
============

It is highly recommended to use the [OpenResty bundle](http://openresty.org) that bundles Nginx, ngx_lua,  LuaJIT 2.0/2.1 (or the optional standard Lua 5.1 interpreter), as well as a package of powerful companion Nginx modules. The basic installation step is a simple command: `./configure --with-luajit && make && make install`.

Alternatively, ngx_lua can be manually compiled into Nginx:

1. Install LuaJIT 2.0 or 2.1 (recommended) or Lua 5.1 (Lua 5.2 is *not* supported yet). LuaJIT can be downloaded from the [LuaJIT project website](http://luajit.org/download.html) and Lua 5.1, from the [Lua project website](http://www.lua.org/).  Some distribution package managers also distribute LuaJIT and/or Lua.
1. Download the latest version of the ngx_devel_kit (NDK) module [HERE](https://github.com/simpl/ngx_devel_kit/tags).
1. Download the latest version of ngx_lua [HERE](https://github.com/openresty/lua-nginx-module/tags).
1. Download the latest version of Nginx [HERE](http://nginx.org/) (See [Nginx Compatibility](#nginx-compatibility))

Build the source with this module:

```bash

 wget 'http://nginx.org/download/nginx-1.11.2.tar.gz'
 tar -xzvf nginx-1.11.2.tar.gz
 cd nginx-1.11.2/

 # tell nginx's build system where to find LuaJIT 2.0:
 export LUAJIT_LIB=/path/to/luajit/lib
 export LUAJIT_INC=/path/to/luajit/include/luajit-2.0

 # tell nginx's build system where to find LuaJIT 2.1:
 export LUAJIT_LIB=/path/to/luajit/lib
 export LUAJIT_INC=/path/to/luajit/include/luajit-2.1

 # or tell where to find Lua if using Lua instead:
 #export LUA_LIB=/path/to/lua/lib
 #export LUA_INC=/path/to/lua/include

 # Here we assume Nginx is to be installed under /opt/nginx/.
 ./configure --prefix=/opt/nginx \
         --with-ld-opt="-Wl,-rpath,/path/to/luajit-or-lua/lib" \
         --add-module=/path/to/ngx_devel_kit \
         --add-module=/path/to/lua-nginx-module

 make -j2
 make install
```

[Back to TOC](#table-of-contents)

Building as a dynamic module
----------------------------

Starting from NGINX 1.9.11, you can also compile this module as a dynamic module, by using the `--add-dynamic-module=PATH` option instead of `--add-module=PATH` on the
`./configure` command line above. And then you can explicitly load the module in your `nginx.conf` via the [load_module](http://nginx.org/en/docs/ngx_core_module.html#load_module)
directive, for example,

```nginx
load_module /path/to/modules/ndk_http_module.so;  # assuming NDK is built as a dynamic module too
load_module /path/to/modules/ngx_http_lua_module.so;
```

[Back to TOC](#table-of-contents)

C Macro Configurations
----------------------

While building this module either via OpenResty or with the NGINX core, you can define the following C macros via the C compiler options:

* `NGX_LUA_USE_ASSERT`
	When defined, will enable assertions in the ngx_lua C code base. Recommended for debugging or testing builds. It can introduce some (small) runtime overhead when enabled. This macro was first introduced in the `v0.9.10` release.
* `NGX_LUA_ABORT_AT_PANIC`
	When the Lua/LuaJIT VM panics, ngx_lua will instruct the current nginx worker process to quit gracefully by default. By specifying this C macro, ngx_lua will abort the current nginx worker process (which usually result in a core dump file) immediately. This option is useful for debugging VM panics. This option was first introduced in the `v0.9.8` release.
* `NGX_LUA_NO_FFI_API`
	Excludes pure C API functions for FFI-based Lua API for NGINX (as required by [lua-resty-core](https://github.com/openresty/lua-resty-core#readme), for example). Enabling this macro can make the resulting binary code size smaller.

To enable one or more of these macros, just pass extra C compiler options to the `./configure` script of either NGINX or OpenResty. For instance,


    ./configure --with-cc-opt="-DNGX_LUA_USE_ASSERT -DNGX_LUA_ABORT_AT_PANIC"


[Back to TOC](#table-of-contents)

Installation on Ubuntu 11.10
----------------------------

Note that it is recommended to use LuaJIT 2.0 or LuaJIT 2.1 instead of the standard Lua 5.1 interpreter wherever possible.

If the standard Lua 5.1 interpreter is required however, run the following command to install it from the Ubuntu repository:

```bash

 apt-get install -y lua5.1 liblua5.1-0 liblua5.1-0-dev
```

Everything should be installed correctly, except for one small tweak.

Library name `liblua.so` has been changed in liblua5.1 package, it only comes with `liblua5.1.so`, which needs to be symlinked to `/usr/lib` so it could be found during the configuration process.

```bash

 ln -s /usr/lib/x86_64-linux-gnu/liblua5.1.so /usr/lib/liblua.so
```

[Back to TOC](#table-of-contents)

Community
=========

[Back to TOC](#table-of-contents)

English Mailing List
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

[Back to TOC](#table-of-contents)

Chinese Mailing List
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

[Back to TOC](#table-of-contents)

Code Repository
===============

The code repository of this project is hosted on github at [openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module).

[Back to TOC](#table-of-contents)

Bugs and Patches
================

Please submit bug reports, wishlists, or patches by

1. creating a ticket on the [GitHub Issue Tracker](https://github.com/openresty/lua-nginx-module/issues),
1. or posting to the [OpenResty community](#community).

[Back to TOC](#table-of-contents)

Lua/LuaJIT bytecode support
===========================

As from the `v0.5.0rc32` release, all `*_by_lua_file` configure directives (such as [content_by_lua_file](#content_by_lua_file)) support loading Lua 5.1 and LuaJIT 2.0/2.1 raw bytecode files directly.

Please note that the bytecode format used by LuaJIT 2.0/2.1 is not compatible with that used by the standard Lua 5.1 interpreter. So if using LuaJIT 2.0/2.1 with ngx_lua, LuaJIT compatible bytecode files must be generated as shown:

```bash

 /path/to/luajit/bin/luajit -b /path/to/input_file.lua /path/to/output_file.luac
```

The `-bg` option can be used to include debug information in the LuaJIT bytecode file:

```bash

 /path/to/luajit/bin/luajit -bg /path/to/input_file.lua /path/to/output_file.luac
```

Please refer to the official LuaJIT documentation on the `-b` option for more details:

<http://luajit.org/running.html#opt_b>

Also, the bytecode files generated by LuaJIT 2.1 is *not* compatible with LuaJIT 2.0, and vice versa. The support for LuaJIT 2.1 bytecode was first added in ngx_lua v0.9.3.

Similarly, if using the standard Lua 5.1 interpreter with ngx_lua, Lua compatible bytecode files must be generated using the `luac` commandline utility as shown:

```bash

 luac -o /path/to/output_file.luac /path/to/input_file.lua
```

Unlike as with LuaJIT, debug information is included in standard Lua 5.1 bytecode files by default. This can be striped out by specifying the `-s` option as shown:

```bash

 luac -s -o /path/to/output_file.luac /path/to/input_file.lua
```

Attempts to load standard Lua 5.1 bytecode files into ngx_lua instances linked to LuaJIT 2.0/2.1 or vice versa, will result in an error message, such as that below, being logged into the Nginx `error.log` file:


    [error] 13909#0: *1 failed to load Lua inlined code: bad byte-code header in /path/to/test_file.luac


Loading bytecode files via the Lua primitives like `require` and `dofile` should always work as expected.

[Back to TOC](#table-of-contents)

System Environment Variable Support
===================================

If you want to access the system environment variable, say, `foo`, in Lua via the standard Lua API [os.getenv](http://www.lua.org/manual/5.1/manual.html#pdf-os.getenv), then you should also list this environment variable name in your `nginx.conf` file via the [env directive](http://nginx.org/en/docs/ngx_core_module.html#env). For example,

```nginx

 env foo;
```

[Back to TOC](#table-of-contents)

HTTP 1.0 support
================

The HTTP 1.0 protocol does not support chunked output and requires an explicit `Content-Length` header when the response body is not empty in order to support the HTTP 1.0 keep-alive.
So when a HTTP 1.0 request is made and the [lua_http10_buffering](#lua_http10_buffering) directive is turned `on`, ngx_lua will buffer the
output of [ngx.say](#ngxsay) and [ngx.print](#ngxprint) calls and also postpone sending response headers until all the response body output is received.
At that time ngx_lua can calculate the total length of the body and construct a proper `Content-Length` header to return to the HTTP 1.0 client.
If the `Content-Length` response header is set in the running Lua code, however, this buffering will be disabled even if the [lua_http10_buffering](#lua_http10_buffering) directive is turned `on`.

For large streaming output responses, it is important to disable the [lua_http10_buffering](#lua_http10_buffering) directive to minimise memory usage.

Note that common HTTP benchmark tools such as `ab` and `http_load` issue HTTP 1.0 requests by default.
To force `curl` to send HTTP 1.0 requests, use the `-0` option.

[Back to TOC](#table-of-contents)

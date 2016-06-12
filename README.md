# Background

One of my projects uses `mitmproxy` (version 0.16)  as a transparent proxy server to capture all 

HTTP/HTTPS traffic.  And i followed [mitmproxy docs steps](http://docs.mitmproxy.org/en/latest/transparent/linux.html)  and it works. 

In my case, because i don't know which port the http(s) request send to, so i redirect all tcp port traffic to the mitmproxy port like this:

```
iptables -t  nat -I PREROUTING -i eth0 -p tcp -s 172.16.1.88  -j REDIRECT --to-port 8888
```

Capture works well, but non-HTTP protocol traffic was blocked, like use `ssh`,  `nc` command. 

According [--raw-tcp Add cmdline argument to dump raw binary conversation to file](https://github.com/mitmproxy/mitmproxy/issues/866)

> As of mitmproxy 0.14, we try to guess the content type by looking at the first bytes of a connection. For example, if the client starts sending `\x16\x03\x01`, we know it's a TLS handshake and we start TLS interception. If the first three bytes are `GET`, we are pretty sure it's HTTP. `--raw-tcp` instructs mitmproxy to use our TCP protocol layer if we can't understand the first few bytes (at the moment, that means it's neither TLS nor HTTP). In the TCP layer, we just redirect tcp messages as-is (without trying to parse any protocol - note that we still do TLS MITM though). `--tcp` forces this layer for a certain host, even if it the traffic looks like valid HTTP/TLS.

`--raw-tcp` works for non-TLS tcp traffic, like `nc`.  But TLS (non-HTTP) still block, like `ssh` failed:

`ssh_exchange_identification: Connection closed by remote host`

# Solution

After reading the source code, i add some code to the `RootContext` class,  TLS (non-HTTP)  works.

python file:`[python_home]/lib/python2.7/site-packages/libmproxy/proxy/root_context.py`

function: `_next_layer`

line number: 115-126

following is what i added:

```
        # --------------- add non-HTTP TLS support --------------
        # 7. Check for http fingerprint
        http_method_list = ["GET", "HEAD", "POST", "PUT",
                            "DELETE", "CONNECT", "OPTIONS", "TRACE"]
        http_flag = False
        for method in http_method_list:
            if d in method:
                http_flag = True
                break
        if not http_flag:
            return RawTCPLayer(top_layer, logging=False)
        # --------------- add non-HTTP TLS support --------------
```

# How	 to use

install mitmproxy 0.16

`pip install mitmproxy==0.16`

replace 

`[python_home]/lib/python2.7/site-packages/libmproxy/proxy/root_context.py`

with the repository `root_context.py` file

or modify by yourself

enjoying~

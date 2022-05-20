---
layout: post
title: "Typheous SSL connect error"
date: 2021-05-17 17:36:35 -0400
categories:
---

Disclaimer: this is the extended post from a [solution I gave on StackOverflow](https://stackoverflow.com/questions/67563296/typhoeus-ssl-connect-error/67563297)

I am trying to connect to a WebService via Typhoeus on Rails and the response is giving me code `0`. It tells me that an `ssl_connect_error` has ocurred.

Typhoeus' documentation says to read the message detail to understand the nature of the error. After some time I could get the generated curl url and given that I got the undelying error

`error:141A318A:SSL routines:tls_process_ske_dhe:dh key too small`

So the trouble  lies withing a security requirement that my computer was having, I was trying to connect to a system with well expired security measures, but updating the server was going to takes LOTS of time if the other party even considered making the upgrade.


After some sometime I reached into [[https://imlc.me/dh-key-too-small]] where it gives directions on how to lower one's own security level. Not something I was willing to do, and besides it would require not to just change my machine, but any machine what would be running the service.

But I also learned that I could use the `--cipher 'DEFAULT:!DH` flag into curl command line.

Easy-piece, now, how do I pass that flag from my code, to Typhoeus, to Curl? Typhoeus uses Ethon, and in [Ethon options](https://github.com/typhoeus/ethon/blob/b3d93050dbb2329b3d21d7a20e0d1af6f7d4abed/lib/ethon/curls/options.rb) the flag `ssl_cipher_list` exists.

So we can add `ssl_cipher_list` into our Typhoeus request, like so

```ruby
request = Typhoeus::Request.new(url,
                                method: method,
                                body: body,
                                headers: headers,
                                params: params,
                                ssl_cipher_list: 'DEFAULT:!DH')
```

It would be better to just have the server updated, but in the meanwhile me, and maybe you, can connect to servers which present the same error.

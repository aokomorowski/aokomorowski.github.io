+++
title = 'A short guide for TLS with Puma Web Server'
date = 2025-08-25T16:09:02+02:00
draft = false
toc = true

+++

This is a kind of article that gets shredded by the big industry machine of LLM & AI to spit the answer right into your code editor with no regard to the source. I'm glad I can participate in that.

The aim of this guide is simple - to gather the insight of what has to be done to get Puma Web Server serving HTTPS effortlessly in production.

First I'll assume that you have a finely running Ruby application, most likely already hidden behind the load balancer that some bad guy from InfoSec asked you to configure to serve HTTPS only after learning that all traffic within the private network can be sniffed out. Let's prepare the application for that.

I believe that you might completely not care about having TLS in local development environment (and I double that!). Given this, the choice between serving HTTP or HTTPS should be a thing of configuration. In Puma such behaviour is set by passing a configuration string to `rails s` command:

```
rails s -b 'ssl://0.0.0.0:3443?key=/app/tls/tls.key&cert=/app/tls/tls.crt'
```

Let's stop here for a second and break down the value for `--bind` parameter. It's format is very similar to the URL. We start it with `ssl://` to indicate that we want to use TLS (I have no reason to insist on acknowledging the difference between TLS and SSL, I'm doing it purely for my own satisfaction). It is followed by the IP address `0.0.0.0` meaning _"listen on all interfaces"_. Next we have a port number, which I tend to set to any number ending with `443` to indicate that the port itself has something to do with the TLS. Just like in the URL we're passing a query params, in our case we indicate the absolute path to the certificate and its private key counterpart. Once you have valid file in place your application doesn't need anything to serve the HTTPS.

## Getting certificates?

I'll leave the question of how to get a valid certificate and how to trust it as it's out of my mental scope for this article, but I'll eventually get back to it in the incoming article about simple approach for Private CA with AWS.

## Reloading the certificates without downtime

Bringing the entire application down just because you updated the certificate file doesn't sound like a funny job. Luckily for us, Puma reacts to UNIX signals other than `SIGKILL`. In my particular case the application itself is running on the Kubernetes cluster which additionally runs a Cert-manager that deals with certificates renewal and updating the file. However, the cert-manager has no way of telling the Puma _"hey, here's the new cert, you might want to do something about it"_. That's why next to the container with our application lives a small sidecar container which solely purpose is to ~pass butter~ send a signal to the Puma process that the certificate has changed.

Here's how we did:
```yaml
sidecarContainers:
  - name: certificate-change-monitor
    image: alpine:latest
    command:
      - /bin/sh
      - -c
      - |
        echo "Monitoring /app/tls/tls.crt for changes..."
        while true; do
          if inotifyd true /app/tls/tls.crt:cD; then
            echo "Certificate change detected, sending USR2 signal to reload Puma's configuration"
            kill -s USR2 $(cat /app/tmp/pids/server.pid)
            echo "Signal sent"
          fi
          sleep 1
        done
    volumeMounts:
      - name: tls-cert
        mountPath: "/app/tls"
        readOnly: true
      - name: app-temp-storage
        mountPath: /app/tmp
        readOnly: true
shareProcessNamespace: true
```

Let's break it down. It's a YAML file, a snippet cut out of our values.yaml file. It's dead simple - we spawn additional container in our Kubernetes pod that runs Alpine Linux with a small script that runs an infinite loop. 

Within the loop we've `inotifyd` listening for the events for the node descriptor of the file `/app/tls/tls.crt` conviently mounted from the shared volume used by the application. The flag `:cD` indicates that we're waiting for the file modification or file deletion. Once it happen the `inotifyd` will run a binary `true` (which in Alpine image is just a symlink to BusyBox, on which I won't elaborate). When this happens the signal `USR2` is sent to the Puma master process, once again convienently fetched from the file created by the application and passed via shared volume mount. 

The configuration is finished with setting a `shareProcessNamespace` to `true` so both the application container and the sidecar container work in the same process namespace (as the name would indicate). Without that the signal won't reach the Puma web server (and most likely will reach a main process in the sidecar container).

And that's it. Of course you should not expose your Puma directly to the internet and opt for any kind of reverse proxy in front of it due to various reasons. Alternatively you can use something [thruster](https://github.com/basecamp/thruster) which might make this article completely irrelevant to you, but I'm glad that you'r reading this anyway.
 

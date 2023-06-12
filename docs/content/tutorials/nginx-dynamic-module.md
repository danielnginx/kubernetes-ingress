---
title: NGINX Ingress Controller with dynamic module
description: |
  Use NGINX Ingress Controller with dynamic module
weight: 1800
doctypes: ["concept"]
toc: true
docs: "DOCS-1231"
---

## How to use a NGINX dynamic module with NGINX Ingress controller    

NGINX has several dnynamic modules that can be used to add additional features and capabilities. NGINX Ingress controller can also use these same NGINX modules with NGINX Ingress controller.     
You will need to modify your NGINX Ingress controller build to ensure it is built into the image, and then load it into NGINX Ingress controller.
This document will walk you throw how to accomplish this task.


You can find more NGINX dynamic modules on the NGINX Plus website:    
[NGINX Dynamic modules](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/dynamic-modules/)

Here are the basic steps that you will need to perform.    

- Update the Dockerfile and add your dynamic module you want to add to your NGINX Ingress controller image
- Build the image with updated Dockerfile, reflecting the image you are goign to add to the image build.    
- Load the module using a `configmap` into NGINX Ingress controller

In order to build a custom NGINX Ingress image with specific modules, the first step you need to take is to modify the `Dockerfile` located in the `./build` directory of the NGINXINC repo. 

Clone the NGINX Ingress controller repo:    

```
git clone git@github.com:nginxinc/kubernetes-ingress.git
```

Once you have the image cloned, edit the `Dockerfile` located in the `build` directory of the cloned repo.


In our example, we are going to add the `Headers-more` dynamic module to our NGINX Ingress controller image.    

Locate your preferred OS version that you want to modify in the DOckerfile (debian, alpine etc.).  
Here is a snippet of the `Dockerfile` for `debian-plus`

```
FROM debian:${DEBIAN_VERSION} AS debian-plus
ARG IC_VERSION
ARG NGINX_PLUS_VERSION
ARG BUILD_OS

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN --mount=type=secret,id=nginx-repo.crt,dst=/etc/ssl/nginx/nginx-repo.crt,mode=0644 \
	--mount=type=secret,id=nginx-repo.key,dst=/etc/ssl/nginx/nginx-repo.key,mode=0644 \
	apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y ca-certificates gnupg curl apt-transport-https libcap2-bin \
	&& curl -fsSL https://cs.nginx.com/static/keys/nginx_signing.key | gpg --dearmor > /etc/apt/trusted.gpg.d/nginx_signing.gpg \
	&& curl -fsSL -o /etc/apt/apt.conf.d/90pkgs-nginx https://cs.nginx.com/static/files/90pkgs-nginx \
	&& DEBIAN_VERSION=$(awk -F '=' '/^VERSION_CODENAME=/ {print $2}' /etc/os-release) \
	&& printf "%s\n" "Acquire::https::pkgs.nginx.com::User-Agent \"k8s-ic-$IC_VERSION${BUILD_OS##debian-plus}-apt\";" >> /etc/apt/apt.conf.d/90pkgs-nginx \
	&& printf "%s\n" "deb https://pkgs.nginx.com/plus/${NGINX_PLUS_VERSION^^}/debian ${DEBIAN_VERSION} nginx-plus" > /etc/apt/sources.list.d/nginx-plus.list \
	&& apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y nginx-plus nginx-plus-module-njs \
	&& apt-get purge --auto-remove -y apt-transport-https gnupg curl \
	&& rm -rf /var/lib/apt/lists/*
```

Look for a line similar to the following line in the `Dockerfile`:

```
apt-get install --no-install-recommends --no-install-suggests -y nginx-plus nginx-plus-module-njs
```

This is the line you will want to modify/add the module you want to have loaded into NGINX Ingress controller.   
We are going to add the `headers-more` module. The updated line would look like this:

```
apt-get install --no-install-recommends --no-install-suggests -y nginx-plus nginx-plus-module-njs nginx-plus-module-headers-more
```

In the above example, I added a single module line:
```
nginx-plus-module-headers-more
```

After the new NGINX Ingress module image has been built successfully, the next step is to load the module into your NGINX Ingress controller when it will be deployed into your Kubernetes cluster.

For this to work, we will need to edit and update your `configmap` and load the module into the `main` context.   
Here is a simple example of updating of the `nginx-cofng.yaml` file, that is used when deploying via manifest (helm is also supported. Just updated the correct entries line appropriately.)

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
  main-snippets: |
    load_module modules/ngx_http_headers_more_filter_module.so;
```

With the above `configmap` configured to load in our `ngx_http_headers_more` module, NGINX Ingress controller will not load that module.   
You can verify this be executing `nginx -T` in the NGINX Ingress controller pod:

If you are using `helm`, you will need to add a setting like the following in your `values.yaml` file:    

```
config:
  name: nginx-ingress
  entries:
    main-snippets: load_module modules/ngx_http_headers_more_filder_module.so;
    http-snippets: underscores_in_headers on;
    lb-method: "least_time last_byte"
```

kubectl exec -it -n nginx-ingress <nginx_ingress_pod> -- nginx -T

You should see in the `nginx -T` full output, that your module is now loaded into NGINX Ingress controller.

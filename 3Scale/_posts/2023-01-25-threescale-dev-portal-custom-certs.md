---
layout: post_with_sidebar
title:  "How to setup custom certificates for S3 in 3Scale Developer Portal"
date:   2023-01-23 12:04:04 +0100
category: 3Scale
subcategory: Developer Portal
toc: true
tags: Developer Portal
---




While using 3scale with a S3 storage having custom or self-signed certificates, the Developer portal returns a 404 error. Trying to initialize seed data on the portal following leads to the following error:

~~~
[Aws::S3::Client 0 2.135508 3 retries] put_object(content_type:"image/jpeg",acl:"private",bucket:"3scale-bucket-42140d8e-7f92-4342-85e9-00bfc66f4e8a",key:"provider-name/2022/10/13/desk-a58e4d189064acce.jpg",body:#<File:/tmp/b506e098cf253ce028b8ba0a5fc7d47320221013-80-c733i6.jpg (220488 bytes)>) Seahorse::Client::NetworkingError SSL_connect returned=1 errno=0 state=error: certificate verify failed (self signed certificate in certificate chain)
~~~

## How to configure custom certificates and trust store

The correct certificate can be retrieved with the following command:

~~~ bash     
$ echo -n | openssl s_client -connect <S3 SERVER FQDN>:<S3 PORT> -servername -servername <S3 SERVER FQDN> --showcerts | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > S3Certs.pem
~~~

{%- capture note -%}
some versions of <b>openssl</b>  accept <b>-showcerts</b> instead of <b>\-\-showcerts</b>
{%- endcapture -%}

{%- include note.html content=note -%}


Copy the file located in `/etc/pki/tls/cert.pem` from the `system-sidekiq` container on the `system-sidekiq` pod to the local working directory:

~~~ bash     
$ oc rsh -c system-sidekiq dc/system-sidekiq cat /etc/pki/tls/cert.pem > SidekiqCerts.pem 2>&1
~~~

Append the contents of the custom CA certificate file to `SidekiqCerts.pem` as in the following example:

~~~ bash     
$ cat S3Certs.pem >> SidekiqCerts.pem
~~~
    
Attach the file to a new `ConfigMap` object:

~~~ bash     
$ oc create configmap sidekiqcerts --from-file=./SidekiqCerts.pem
~~~

{%- capture note -%}
The above assumes that the file <b>SidekiqCerts.pem</b> is located inside the current working directory.
{%- endcapture -%}

{%- include note.html content=note -%}

Mount a new volume on the location `/etc/pki/tls/custom/SidekiqCerts.pem` containing the `ConfigMap` from the step above:

~~~ bash     
$ oc set volume dc/system-sidekiq --add --overwrite --name=customcerts --mount-path=/etc/pki/tls/custom/SidekiqCerts.pem --sub-path=SidekiqCerts.pem --source='{"configMap":{"name":"sidekiqcerts"}}'
~~~

Adjust the Environment Variable `SSL_CERT_FILE` to refer to the file:

~~~ bash     
$ oc set env dc/system-sidekiq SSL_CERT_FILE=/etc/pki/tls/custom/SidekiqCerts.pem
~~~

Copy the file located in `/etc/pki/tls/cert.pem` from the `system-developer` container on the `system-app` pod to the local working directory:

~~~ bash     
$ oc rsh -c system-developer dc/system-app cat /etc/pki/tls/cert.pem > SystemAppCerts.pem 2>&1
~~~

Append the contents of the custom CA certificate file to `SystemAppCerts.pem` as in the following example:

~~~ bash     
$ cat S3Certs.pem >> SystemAppCerts.pem
~~~
    
Attach the file to a new `ConfigMap` object:

~~~ bash     
$ oc create configmap systemappcerts --from-file=./SystemAppCerts.pem
~~~
{%- capture note -%}
The above assumes that the file <b>SystemAppCerts.pem</b> is located inside the current working directory.
{%- endcapture -%}

{%- include note.html content=note -%}   

Mount a new volume on the location `/etc/pki/tls/custom/SystemAppCerts.pem` containing the `ConfigMap` from the step above:

~~~ bash     
$ oc set volume dc/system-app --add --overwrite --name=customcerts --mount-path=/etc/pki/tls/custom/SystemAppCerts.pem --sub-path=SystemAppCerts.pem --source='{"configMap":{"name":"systemappcerts"}}'
~~~

Adjust the Environment Variable `SSL_CERT_FILE` to refer to the file:

~~~ bash     
$ oc set env dc/system-app SSL_CERT_FILE=/etc/pki/tls/custom/SystemAppCerts.pem
~~~

After the pods redeploy, the developer portal should be available.

{%- capture note -%}
if the Dev Portal is still not available, perform the following steps to repopulate the Dev Portal CMS content to the standard files - be aware that the steps below will overwrite all CMS customization:
{%- endcapture -%}

{%- include note.html content=note -%}

Create a debug pod for system-app

~~~ bash     
$ oc debug dc/system-app --as-root -n <3scale_namespace>
~~~

Login to Rails console

~~~ bash      
$ bundle exec rails console
~~~

Run the command to re-populate the developer portal pages. Please note that this command does work only when the debug pod is run as root (*--as-root*). It is intended the command to fail so we can diagnose the issue. You should replace the *TENANT_ID* in the command below with the tenant id which is having developer portal page issues.  

~~~ ruby     
CMSResetService.new.call(Account.find(Account.find('<ADMIN_PORTAL_HOST>').tenant_id))
~~~
    
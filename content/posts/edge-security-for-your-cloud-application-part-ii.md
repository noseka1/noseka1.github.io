+++
title = "Edge Security for Your Cloud Application Part Ii"
date = "2018-01-12"
slug = "2018/01/12/edge-security-for-your-cloud-application-part-ii"
Categories = []
+++

In this article, we're going to create a proof-of-concept deployment featuring a non-TLS client connecting to our cloud application. We are going to leverage the architecture approach discussed in the [previous blog post](/blog/2018/01/10/edge-security-for-your-cloud-application-part-i). A secure communication channel is going to be established between the client and the cloud application including mutual authentication.

<!-- more -->

## Deployment overview

Before we get our hands dirty, let's gain a better understanding of what we are trying to achieve. The diagram depicting our test deployment looks as follows:

{% img /images/posts/edge_security_for_your_cloud_application_poc_arch.svg 1000 Architecture Overview %}

We're going to spin up two virtual instances. The edge instance will host our edge service. The client instance will host the client that will be accessing our edge service. For the sake of POC, the edge service is going to be an Apache web server and we're going to use the curl command-line utility in place of the client. The battle-proven HAProxy is going to play the role of the client-side as well as the server-side proxy, securing the client-server communication.

## Getting started

You can start off with creating two CentOS 7 instances in AWS. Choose a minimalist t2.micro instance type which is sufficient for our proof of concept. In the security groups settings, make sure that in addition to the SSH port you have also enabled access to port 443 (HTTPS) from anywhere.

After the instances booted up, install HAProxy on both instances:

{% codeblock lang:sh %}
$ yum install -y haproxy
{% endcodeblock %}

And create a directory that will hold the keys and certificates required by HAProxy:

{% codeblock lang:sh %}
$ mkdir /etc/haproxy/ssl
{% endcodeblock %}

Make sure that you run the above command on both instances, too.

## PKI keys and certificates

We are going to leverage the TLS protocol to establish a secure, mutually authenticated connection between the two proxies. TLS relies on PKI keys and certificates that we'll need to generate. The PKI setup for our company consists of a root CA, a layer of subordinate CAs and three end-entity certificates.

{% img /images/posts/edge_security_for_your_cloud_application_pki.svg 500 600 PKI %}

It is a common practice to sign the end-entity certificates by one or more subordinate CAs as it prevents the necessity of revoking a root certificate in the case that an end-entity certificate is incorrectly issued or compromised.

In total, we are going to generate three end-entity certificates. `Edge Service Certificate` is going to be used by the reverse proxy running on the edge instance in order to authenticate itself to the clients. `Customer1 Client Certificate` and `Customer2 Client Certificate` are certificates that our company securely distributes to the tenants (customers). Each tenant uses her certificate and the associated private key to authenticate herself when accessing the cloud application.

A PKI certificate can be created in three steps:

1. Generate a private key.
2. Using the private key, generate a Certificate Signing Request (CSR).
3. Using a CA certificate along with the respective private key and the CSR, generate the certificate.

An exeption from this three-step procedure is the root certificate which is self-signed.

In the following, we are going to generate seven certificates. All the commands are to be issued on the edge instance. Note that you can populate the certificate fields with pretty arbitrary values with one exception: the `Common Name` field of the end-entity certificates. `Common Name` of the `Edge Service Certificate` must match the DNS name of the edge instance. `Common Name` of the `Customer1 Client Certificate` and the `Customer2 Client Certificate` must match the HAProxy configuration. By inspecting the `Common Name` field, HAProxy is able to recognize which client is trying to access the cloud application and it is able to route the client request to the appropriate backend service.

### Company RootCA certificate

Let's generate the company's greatest secret - the private key of the Company's RootCA:

{% codeblock lang:sh %}
$ openssl genrsa -out rootca.company.example.key.pem 4096
{% endcodeblock %}

And create a self-signed RootCA certificate. Below you can see the sample input data:

{% codeblock lang:sh %}
$ openssl req -key rootca.company.example.key.pem -new -x509 -extensions v3_ca -out rootca.company.example.crt.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CA
Locality Name (eg, city) [Default City]:San Diego
Organization Name (eg, company) [Default Company Ltd]:Company RootCA
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:rootca.company.example
Email Address []:
{% endcodeblock %}

For the sake of conciseness, I shortened the console output a bit. Note that we are adding the command-line option `-extensions v3_ca` to denote this certificate as a CA certificate. Otherwise, an end-entity certificate would have been generated by default.

### Company SubCA certificate

Next, issue the command to generate the private key of the Company's SubCA:

{% codeblock lang:sh %}
$ openssl genrsa -out subca.company.example.key.pem 4096
{% endcodeblock %}

Using the private key, let's create a Certificate Signing Request:

{% codeblock lang:sh %}
$ openssl req -new -key subca.company.example.key.pem -out subca.company.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CA
Locality Name (eg, city) [Default City]:San Diego
Organization Name (eg, company) [Default Company Ltd]:Company SubCA
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:subca.company.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}

And finally, generate the certificate of the Company's SubCA:

{% codeblock lang:sh %}
$ openssl x509 -req -in subca.company.example.csr.pem -CA rootca.company.example.crt.pem -CAkey rootca.company.example.key.pem -CAcreateserial -extfile /etc/pki/tls/openssl.cnf -extensions v3_ca -out subca.company.example.crt.pem
{% endcodeblock %}

### Customer1 SubCA certificate

Steps to generate the Customer1 SubCA certificate should be quite clear now:

{% codeblock lang:sh %}
$ openssl genrsa -out subca.customer1.example.key.pem 4096
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl req -new -key subca.customer1.example.key.pem -out subca.customer1.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CA
Locality Name (eg, city) [Default City]:San Diego
Organization Name (eg, company) [Default Company Ltd]:Customer1 SubCA
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:subca.customer1.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl x509 -req -in subca.customer1.example.csr.pem -CA rootca.company.example.crt.pem -CAkey rootca.company.example.key.pem -CAserial rootca.srl -extfile /etc/pki/tls/openssl.cnf -extensions v3_ca -out subca.customer1.example.crt.pem
{% endcodeblock %}

### Customer2 SubCA certificate

Analogicaly,  we are going to generate the Customer2 SubCA certificate:

{% codeblock lang:sh %}
$ openssl genrsa -out subca.customer2.example.key.pem 4096
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl req -new -key subca.customer2.example.key.pem -out subca.customer2.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CA
Locality Name (eg, city) [Default City]:San Diego
Organization Name (eg, company) [Default Company Ltd]:Customer2 SubCA
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:subca.customer2.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}


{% codeblock lang:sh %}
$ openssl x509 -req -in subca.customer2.example.csr.pem -CA rootca.company.example.crt.pem -CAkey rootca.company.example.key.pem -CAserial rootca.srl -extfile /etc/pki/tls/openssl.cnf -extensions v3_ca -out subca.customer2.example.crt.pem
{% endcodeblock %}

### Edge service certificate

This is the first out of the three end-entity certificates. First, we will generate the private key:

{% codeblock lang:sh %}
$ openssl genrsa -out app.company.example.key.pem 2048
{% endcodeblock %}

Next, we are going to create the CSR. Make sure you populate the `Common Name` field exactly as you can see below:

{% codeblock lang:sh %}
$ openssl req -new -key app.company.example.key.pem -out app.company.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:CA
Locality Name (eg, city) [Default City]:San Diego
Organization Name (eg, company) [Default Company Ltd]:Company
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:app.company.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}

And finally, type this command to generate the certificate:

{% codeblock lang:sh %}
$ openssl x509 -req -in app.company.example.csr.pem -CA subca.company.example.crt.pem -CAkey subca.company.example.key.pem -CAcreateserial -out app.company.example.crt.pem
{% endcodeblock %}

Note that we didn't include the `-extensions v3_ca` option as we wanted to create an end-entity certificate.

### Customer1 client certificate

Now we are going to create a certificate for our first customer.

{% codeblock lang:sh %}
$ openssl genrsa -out customer1.example.key.pem 2048
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl req -new -key customer1.example.key.pem -out customer1.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:MA
Locality Name (eg, city) [Default City]:Boston
Organization Name (eg, company) [Default Company Ltd]:Customer1
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:customer1.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl x509 -req -in customer1.example.csr.pem -CA subca.customer1.example.crt.pem -CAkey subca.customer1.example.key.pem -CAcreateserial -out customer1.example.crt.pem
{% endcodeblock %}

### Customer2 client certificate

And finally, we are going to create a certificate for our second customer.

{% codeblock lang:sh %}
$ openssl genrsa -out customer2.example.key.pem 2048
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl req -new -key customer2.example.key.pem -out customer2.example.csr.pem
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:TX
Locality Name (eg, city) [Default City]:Austin
Organization Name (eg, company) [Default Company Ltd]:Customer2
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:customer2.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
{% endcodeblock %}

{% codeblock lang:sh %}
$ openssl x509 -req -in customer2.example.csr.pem -CA subca.customer2.example.crt.pem -CAkey subca.customer2.example.key.pem -CAcreateserial -out customer2.example.crt.pem
{% endcodeblock %}

If everything went well, your working directory should contain a collection of PKI files similar to this list:

{% codeblock lang:sh %}
$ ls -1
app.company.example.crt.pem
app.company.example.csr.pem
app.company.example.key.pem
customer1.example.crt.pem
customer1.example.csr.pem
customer1.example.key.pem
customer2.example.crt.pem
customer2.example.csr.pem
customer2.example.key.pem
rootca.company.example.crt.pem
rootca.company.example.key.pem
rootca.srl
subca.company.example.crt.pem
subca.company.example.csr.pem
subca.company.example.key.pem
subca.customer1.example.crt.pem
subca.customer1.example.csr.pem
subca.customer1.example.key.pem
subca.customer2.example.crt.pem
subca.customer2.example.csr.pem
subca.customer2.example.key.pem
subca.srl
{% endcodeblock %}

## Installing the edge service

In our POC project, the role of the edge service will be played by the Apache server. To install the Apache server, issue the following command on the edge instance:

{% codeblock lang:sh %}
$ yum install -y httpd
{% endcodeblock %}

You can start the Apache server by typing:

{% codeblock lang:sh %}
$ systemctl start httpd
{% endcodeblock %}

The default static web page served by Apache is for our purposes a bit too long. Let's replace it with a simple, one line message:

{% codeblock lang:sh %}
$ echo 'It works!' > /var/www/html/index.html
{% endcodeblock %}

To verify that Apache was installed properly and is running, issue the command:

{% codeblock lang:sh %}
$ curl localhost
It works!
{% endcodeblock %}

## Configuring the reverse proxy on the edge instance

In this section, we are going to set up the reverse proxy on the edge instance. First, let's prepare two files which will be referred to from the HAProxy configuration. The file `app.crt` will include the edge service certificate along with the CA chain and the respective private key. It is used by HAProxy to authenticate itself to the clients. In the following sections, we will configure the client-side HAProxy to trust these certificates and hence verify that it is connecting to the correct service and not for example to a service of an attacker. To create the `app.crt` file, you can type:

{% codeblock lang:sh %}
$ cat \
app.company.example.crt.pem \
subca.company.example.crt.pem \
rootca.company.example.crt.pem \
app.company.example.key.pem \
> /etc/haproxy/ssl/app.crt
{% endcodeblock %}

The important task of the server-side HAProxy is to authenticate the incoming client connections. In our project, we are looking at two customers that should be allowed to access our application. For that, we're going to include the CA certificate chains of the two customers into the `customer-ca.crt` file:

{% codeblock lang:sh %}
$ cat \
subca.customer1.example.crt.pem \
subca.customer2.example.crt.pem \
rootca.company.example.crt.pem \
> /etc/haproxy/ssl/customer-ca.crt
{% endcodeblock %}

As the last step in this section, we are going to configure HAProxy. You can open the HAProxy configuration file `/etc/haproxy/haproxy.cfg` in your favorite editor and replace its content with:

{% codeblock lang:sh %}
global
  tune.ssl.default-dh-param 1024

defaults
  timeout client 30s
  timeout server 30s
  timeout connect 5s

frontend proxy
  bind 0.0.0.0:443 ssl crt /etc/haproxy/ssl/app.crt ca-file /etc/haproxy/ssl/customer-ca.crt verify required
  mode http

  use_backend edge_customer1 if { ssl_c_s_dn(cn) -i customer1.example }
  use_backend edge_customer2 if { ssl_c_s_dn(cn) -i customer2.example }

backend edge_customer1
  mode http
  server customer1 127.0.0.1:80 check

backend edge_customer2
  mode http
  server customer2 127.0.0.1:80 check
{% endcodeblock %}

This is a minimalist configuration file, good enough for our proof of concept. HAProxy is going to listen on port 443 for incoming TLS connections. It will present the edge service certificate to the clients. At the same time, it will only accept connections from the clients sending the Customer1 or Customer2 certificate. Based on the `Common Name` field of the client's certificate, HAProxy will forward the request to the respective backend. In our case, both customers will be served by the same service listening on 127.0.0.1:80, however, you can imagine that in the real scenario there would be two separate edge services perhaps running on two different machines.

You can start the HAProxy service using the following command:

{% codeblock lang:sh %}
$ systemctl restart haproxy
{% endcodeblock %}

Before moving on, check the HAProxy logs. If everything worked well, there should be no errors or warnings:

{% codeblock lang:sh %}
$ journalctl -u haproxy -e
{% endcodeblock %}

## Configuring the proxy on the client instance

In this section, we're going to turn our attention to the client instance. First, we'll need to copy some of the PKI keys and certificates from the edge instance to the client instance. I configured SSH between the two instances, so that I can issue the following command on the edge instance to copy the files to the client instance:

{% codeblock lang:sh %}
$ scp \
customer1.example.crt.pem \
customer1.example.key.pem \
customer2.example.crt.pem \
customer2.example.key.pem \
subca.company.example.crt.pem \
rootca.company.example.crt.pem \
ip-172-31-33-109.us-west-2.compute.internal:
{% endcodeblock %}

In the above command, make sure that you replace the target host name `ip-172-31-33-109.us-west-2.compute.internal` with the DNS name of your client instance.

On the client instance, we are going to add a line to the `/etc/hosts` file which will make the DNS name `app.company.example` resolve to the IP address of the edge instance. In the following command, replace the IP address with the IP address of your edge instance:

{% codeblock lang:sh %}
$ echo 172.31.44.105 app.company.example >> /etc/hosts
{% endcodeblock %}

Let's check that the TLS client is able to connect to the edge service. On the client instance, type:

{% codeblock lang:sh %}
$ curl \
--cacert rootca.company.example.crt.pem \
--cert ./customer1.example.crt.pem \
--key ./customer1.example.key.pem \
https://app.company.example
It works!
{% endcodeblock %}

Excellent! We've just verified that the client with the built-in TLS support is able to successfully connect to our edge service and authenticate itself as Customer1.

The ultimate goal of this tutorial was to allow a client without TLS support to access the edge service, too. In order to achieve this goal, we're going to set up a client-side HAProxy. First, let's prepare two files that will be needed by HAProxy. The `customer1.crt` file enables HAProxy to authenticate itself to the edge service as Customer1. You can create this file by issuing the command:

{% codeblock lang:sh %}
$ cat \
customer1.example.crt.pem \
customer1.example.key.pem \
> /etc/haproxy/ssl/customer1.crt
{% endcodeblock %}

Second file allows HAProxy to verify the authenticity of the edge service. It includes a chain of CA certificates, against which the certificate presented by the edge service will be verified. You can create the `company-ca.crt` file using the following command:

{% codeblock lang:sh %}
$ cat \
subca.company.example.crt.pem \
rootca.company.example.crt.pem \
> /etc/haproxy/ssl/company-ca.crt
{% endcodeblock %}

And finally, we are going to configure the client-side HAProxy. On the client instance, open the file `/etc/haproxy/haproxy.cfg`  in your favorite editor and replace its content with the following configuration:

{% codeblock lang:sh %}
global
  tune.ssl.default-dh-param 1024

defaults
  timeout client 30s
  timeout server 30s
  timeout connect 5s

frontend proxy
  bind 0.0.0.0:80
  mode http

  use_backend app

backend app
  mode http
  server app.company.example app.company.example:443 check ssl verify required ca-file /etc/haproxy/ssl/company-ca.crt crt /etc/haproxy/ssl/customer1.crt
{% endcodeblock %}

HAProxy will listen on port 80 for the incoming HTTP connections. For each HTTP connection it will open a secure HTTPS connection to the edge service and it will pass the data back and forth between the two connections. You can start the HAProxy on the client instance by typing:

{% codeblock lang:sh %}
$ systemctl restart haproxy
{% endcodeblock %}

Double-check that there are no warnings or errors in the HAProxy logs:

{% codeblock lang:sh %}
$ journalctl -u haproxy -e
{% endcodeblock %}

If everything went well, you should be able to connect to the edge service using a non-TLS client. Here we go:

{% codeblock lang:sh %}
$ curl localhost
It works!
{% endcodeblock %}

To verify that our client is recognized by the cloud application as Customer1, you can comment out the line `use_backend edge_customer1 if ...` in the `/etc/haproxy/haproxy.cfg` file on the edge instance. Remember to restart HAProxy after you modified the configuration. The repeated test from the client instance proves that indeed there's no backend for the Customer1 available anymore:

{% codeblock lang:sh %}
$ curl localhost
<html><body><h1>503 Service Unavailable</h1>
No server is available to handle this request.
</body></html>
{% endcodeblock %}

## Conclusion and final remarks

In this blog post, we established a secure communication channel between the client and our application running in the cloud. As demonstrated, our approach is also suitable for client applications that don't support TLS.

In our example, the client certificate authentication was carried out by HAProxy and the edge service sitting behind the proxy didn't get any information about the client. To improve the design, HAProxy could be configured to forward the attributes of the certificate presented by the client to the backend, by setting them as HTTP request headers. HAProxy even allows to insert the entire client certificate into a request header for the backend.

Our server presented a certificate that was signed by our own CA. In practice, we would deploy a certificate signed by a trusted third-party CA. AWS provides [AWS Certificate Manager](https://aws.amazon.com/certificate-manager) (ACM) service to generate certificates for ELBs, API Gateway and other AWS services. Unfortunately, we cannot utilize this service for our use case as it doesn't provide access to the private key. Instead, we can purchase a certificate from one of the trusted certificate authorities or leverage the free of charge [Let's Encrypt](https://letsencrypt.org/) certificate authority.

This concludes our miniseries about edge security for cloud applications. We are still evaluating the proposed approach. What do you think about it? If you have any feedback, please, feel free to add your comments in the comment section below.

---
layout: post
title: "Generating a non-self signed SSL Certificate for Kubernetes"
date: "2020-04-28 16:47:03 -0700"
---

We had an issue with our Kubernetes cluster running Astronomer where we had a SSL certificate for our cluster - but it had a 90 day expiration. The configuration was through Terraform, but due to some version skew, as well as complicated dependency trees we didn't want to address, we needed to generate a valid SSL certificate - and update a secret that containers in EKS were using. Here's how we did it.

# 1 - Generate a valid SSL cert.
Since we're using Kubernetes, we can use a Docker container to handle this for us.

Follow the instructions here: https://www.linode.com/docs/security/ssl/install-lets-encrypt-to-create-ssl-certificates/

This meant I ran the following command:

## MacOS/Linux:
```
docker run -it --rm --name letsencrypt -v /Users/nickbethem/letsencrypt1:/etc/letsencrypt -v /Users/nickbethem/letsencrypt2:/var/lib/letsencrypt certbot/certbot:latest certonly -d "*.mywebsite.com" --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
```

Follow the instructions on screen -

![Docker instructions](/assets/img/generating-a-nonself-signed-ssl-certificate-for-kubernetes/docker-instructions.png)

Following the instructions - I placed the txt into my domain name service (I use Route53) - ![Inserting records into Route53](/assets/img/generating-a-nonself-signed-ssl-certificate-for-kubernetes/inserting-records-into-route53.png)

This gave me some wonderful new certs.

![My new certs](/assets/img/generating-a-nonself-signed-ssl-certificate-for-kubernetes/my-new-certs.png)

The `README` is very helpful here to explain what each of these are:

````
`[cert name]/privkey.pem`  : the private key for your certificate.
`[cert name]/fullchain.pem`: the certificate file used in most server software.
`[cert name]/chain.pem`    : used for OCSP stapling in Nginx >=1.3.7.
`[cert name]/cert.pem`     : will break many server configurations, and should not be used
                 without reading further documentation (see link below).
````

Thus - we want to use the `privkey.pem` and `fullchain.pem` - other options are valid, but these two are the best default choice I think.

Finally - we can create the secret inside of Kubernetes -
```kubectl create secret tls my-tls-secret  --cert=fullchain.pem --key=privkey.pem```

I had to create it in a namespace, so that became:

```kubectl create secret tls my-tls-secret  --cert=fullchain.pem --key=privkey.pem --namespace mynamespace```

Now - if I want to use an `ngix` container - I can do so easily by adding it as an argument or secret.

```
  containers:
  - args:
    - /nginx-ingress-controller
    - --default-ssl-certificate=mynamespace/my-tls-secret

```
Last step (if you're updating a SSL cert) - just restart  containers by deleting everything (assuming you have replication set up!)

```
kubectl delete pods --all
```

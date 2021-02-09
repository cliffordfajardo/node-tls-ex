# Secure Server Example

## Summary
This example covers various topics like
- TLS (transport security layer), the thing that puts the `S`(secure) in `HTTPS`
  - TLS encrypts HTTP traffic so during transmission of the requests, if someone is attempting to snoop the traffic it will look like jumlbed text.
  - TLS supplants the older SSL (Secure Socket Layer) protocol; use TLS!
- _Much like gzip, TLS is a CPU-intensive operation and should also be performed by an external process such as a Reverse Proxy._

## TS - How TLS Works
- TLS encrypts HTTP traffic so during transmission of the requests, if someone is attempting to snoop the traffic it will look like jumlbed text.
- TLS works using certificates, specifically these 2 types of certificates, which are inherently paired:
  1. Public key: you can share this with anyone
    - you/anyone can encrypt a message using this public key
  2. Private key: should rename a secret;
    - only a person with the public keys corresponding private key can decode/decrypt an encrpyted message.

## Where do I store my public and private keys?
- With HTTP, a server will provide a client its public key, so that a client can later send encrypted messages beack to the server.
- When the client first connects with the server, it also generates a large random number, essentially a password for the session.
  - this large random number/session is encrypted using the public key and sent to the server. This random number is whats used to encrypt the TLS session


## Preface before 'Hands On TLS Setup'
FOr a long time generating certificates was a long, cumbersome process. 
Now there is a tool called 'Lets Encrypt' that automates the process and it also free.
  - caveat you'll need to expose your server to be publically exposed on the internet to verify
    DNS ownership of the domain. **This makes is difficult to encrpy internal services, even though
    it is the clear winnder for public services**

## Hands On TLS Setup
The easiest way to get an HTTPS server running locally is to:
- generate a self-signed certificate
- have a client make a request to the server **without performing certificate validation**

To generate your own certificate, run the command following command inside this folder:

```sh
mkdir -p ./{recipe-api,shared}/tls
openssl req -nodes -new -x509 \
-keyout recipe-api/tls/basic-private-key.key \
-out shared/tls/basic-certificate.cert

# When prompted for values to type into the command line prompt, feel free to use any values BUT
# when asked for a common name put: localhost

# country name: us
# state or provice name: california
# locality: hawyward
# organization name: .
# organizational unit name: .
# common name: localhost
# email address: .
#
```
This command creates two files:
- basic-private-key.key (the private key)
- basic-certificate.cert (the public key).

Once you've created the server file, run the server and then make a request to it. You can do this by running the following commands:

```sh
# terminal 1
node recipe-api/producer-https-basic.js

# terminal 2
curl --insecure https://localhost:4000/recipes/42
```
The `--insecure` flag probably caught your attention. If you were to open the URL directly in a web browser,
you would get a warning that there is a problem with the certificate. This is what happens when a certificate is self-signed.

If you were to make a request to this service using a Node.js application, the request would also fail.
The inner Node.js `http` and `https` modules accept an options argument, and most higher-level HTTP libraries in npm accept those same options in some manner.
One such way to avoid these errors is to provide the `rejectUnauthorized: false` flag.
Unfortunately, this isnt much more secure than using plain HTTP and should be avoided.


## Why does correctly signing certicates event matter?
The reason all this matters is that it;s not necessarily safe to trust
just any old certificate encountered on the internet. Instead, it's important
to know that a certificate is valid. This is usually done by having one certificate "sign" another certificate.
This is a way of saying that one certificate is vouching for the other. 

## Example 1: Why does correctly signing certicates event matter?
As an example of this, the certificate for (1) `thomashunter.name` has been signed for by
another certificate called (2)`Let's Encrypt Authority X3`.
That certificate has been signed by another one called (3)`IdenTrust DST Root CA X3`.
The three certificates form a _chain of trust_.

```
          Root cert
    (IdenTrust DST Root CA X3)
              ↓
       Intermediate cert
    (Let's Encrypt Authority X3)
             ↓
        Identity cert
      (thomashunter.name)

```
The highest point in the chain is called the root certificate. This certificate is trusted
by much of the world; in fact, its public key is included in modern browsers and operating systems.

### Approach to Working with Self Signed Certificates
A better approach to working with self-signed certificates is to actually give the
client a copy of the trusted self-signed certificate (as opposed to what ????), in this case the `basic-certificate.cert` file generated previously.

This certificate can then be passed along by using the `ca: certContent` options flag.
An example of this can be seen in this folder at `web-api/consumer-https-basic.js`

Now run the `web-api` service and make an HTTP request tp it by running the following commands

```sh
# terminal 1
node web-api/consumer-https-basic.js 

# terminal 2
# attempting to make a call to our server via HTTP. What do you think will happen?
curl http://localhost:3000
```

`curl`ing with `http` is going to give us back an error. Each HTTPS server needs access to both
the public and private keys in order to recieve requests. At this point in time, when we attempted
to connect with `curl http://localhost:3000` we failed to connect to the HTTPS server, which was supposed
to give us back a public key for future communication.

_Also recall that a private key should never fall into the hands of an adversary. So, having a single pair of public and private keys for all services within a company is dangerous. **If just one of the projects leaks its private key**, then all projects are affected!_

## How to Be your own ceriticate authority: an Approach for secured communication between internal services
One approach is to generate a new key for every single running service. Unfortunately, a copy of every server's public key would need to be distributed to every client that might want to communicate with it, like in Example 2-8 (`web-api/consumer-https-basic.js`).
**This would be a maintenance nightmare**. Instead, the approach used by non-self-signed certificates can be emulated:
- generate a single internal root certificate
- keep the private key for that secure
- but use it to sign each service's set of keys

Run the commands in Example 2-9 below to do this:

```sh
# -----Happens once for the CA (certificate authority)-----
#1. CSR: Generate a private key ca-private-key.key for the Certificate Authority. You'll be prompted for a password.
openssl genrsa -des3 -out ca-private-key.key 2048
# passhphrase: password



#2. CSR: Generate a root cert shared/tls/ca-certificate.cert (this will be provided to clients). 
#   You'll get asked a lot of questions, but they don’t matter for this example.
openssl req -x509 -new -nodes -key ca-private-key.key \
  -sha256 -days 365 -out shared/tls/ca-certificate.cert
#passphrase: password
#commong name: localhost




# -----Happens for each new certificate-----
#3. APP: Generate a private key producer-private-key.key for a particular service
openssl genrsa -out recipe-api/tls/producer-private-key.key 2048

#4. APP: Create a CSR producer.csr for that same service. Be sure to answer `localhost` for the Common Name question,
#   but other questions don't matter as much.
openssl req -new -key recipe-api/tls/producer-private-key.key \
  -out recipe-api/tls/producer.csr
# commonname: localhost
# challenge password: (i put nothing)



#5. CSR: Generate a service certificate producer-certificate.cert signed by the CA
openssl x509 -req -in recipe-api/tls/producer.csr \
  -CA shared/tls/ca-certificate.cert \
  -CAkey ca-private-key.key -CAcreateserial \
  -out shared/tls/producer-certificate.cert -days 365 -sha256
```

Now modify the code in `web-api/consumer-https-basic.js` to load the `ca-certificate.cert` file.
Also modify `recipe-api/producer-https-basic.js` to load both the `producer-private-key.key`
and `producer-certificate.cert` files. Restart both servers and run the following command again

```sh
# Unfortunately getting "{"statusCode":500,"code":"DEPTH_ZERO_SELF_SIGNED_CERT","error":"Internal Server Error","message":"request to https://localhost:4000/recipes/42 failed, reason: self signed certificate"}% "
# this is a work around until I find a solution. This shouldnt be used in production as we lose security right now: https://github.com/nodejs/node/blob/42dbaed4605f44c393a057aad75a31cac1d0e5f5/lib/_tls_wrap.js#L1307
export NODE_TLS_REJECT_UNAUTHORIZED=0 && node web-api/consumer-https-basic.js 
export NODE_TLS_REJECT_UNAUTHORIZED=0 && node recipe-api/consumer-https-basic.js 

curl http://localhost:3000
```

You should get a successful response, even though web-api wasn't aware of the
recipe-api service's exact certificate; it gains its trust from the root `ca-certificate.cert`
certificate instead.






## Resources
- tebs lab: https://www.youtube.com/channel/UC9NYtw4fjx2FzBTOu9RlTgw/videos
- https://blog.bradfieldcs.com/the-secret-life-of-your-login-credentials-6a254bad52ce




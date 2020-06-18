This example shows how to use [verifyCertificateSpki](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings) configuration in istio SDS gateway support.  I tested the steps with Istio 1.6.1. 

1. Follow the SDS gateway example, especially this [section](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-mutual-tls-ingress-gateway).

2. Get the base64-encoded SHA-256 hashes of the Subject Public Key Information (SPKI) for your authorized httpbin client certificate. 
I used instruction from envoy's [doc](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#extensions-transport-sockets-tls-v3-certificatevalidationcontext).

```
$ openssl x509 -in client.example.com.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
IQnLhNPH/LTI1HLbPBtX3TxAQgdTLkjYQpp8WkxQseE=
```

Note: SKPI is recommended over certificate hash because it is tied to tied to a private key, so it doesn’t change when the certificate is renewed using the same private key.

3. Update the httpbin gateway with your SKPI, and apply it:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: mygateway
spec:
 selector:
   istio: ingressgateway # use istio default ingress gateway
 servers:
 - port:
     number: 443
     name: https
     protocol: HTTPS
   tls:
     mode: MUTUAL
     credentialName: httpbin-credential # must be the same as secret
     verifyCertificateSpki:
     - IQnLhNPH/LTI1HLbPBtX3TxAQgdTLkjYQpp8WkxQseE=
   hosts:
   - httpbin.example.com
EOF
```

4. Access httpbin using curl and your client certificate, key and your CA cert. You should get 418 teapot.
```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt --cert client.example.com.crt --key client.example.com.key \
"https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

...
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 418 
< server: istio-envoy
< date: Thu, 18 Jun 2020 13:40:42 GMT
< x-more-info: http://tools.ietf.org/html/rfc2324
< access-control-allow-origin: *
< access-control-allow-credentials: true
< content-length: 135
< x-envoy-upstream-service-time: 27
< 

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`

```

5. Update the httpbin gateway with an invalid SKPI, and apply it:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: mygateway
spec:
 selector:
   istio: ingressgateway # use istio default ingress gateway
 servers:
 - port:
     number: 443
     name: https
     protocol: HTTPS
   tls:
     mode: MUTUAL
     credentialName: httpbin-credential # must be the same as secret
     verifyCertificateSpki:
     - abnLhNPH/LTI1HLbPBtX3TxAQgdTLkjYQpp8WkxQseE=
   hosts:
   - httpbin.example.com
EOF
```

6. Access httpbin using curl and your client certificate, key and your CA cert. You should get a certificate validation failure.

```
$ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
--cacert example.com.crt --cert client.example.com.crt --key client.example.com.key \
"https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

* Added httpbin.example.com:443:169.60.74.107 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 169.60.74.107...
* TCP_NODELAY set
* Connected to httpbin.example.com (169.60.74.107) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: example.com.crt
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS alert, certificate unknown (558):
* error:1401E416:SSL routines:CONNECT_CR_FINISHED:sslv3 alert certificate unknown
* Closing connection 0
curl: (35) error:1401E416:SSL routines:CONNECT_CR_FINISHED:sslv3 alert certificate unknown
```

7. Access your istio ingress gateway to review your configuration:

```
istioctl dashboard envoy istio-ingressgateway-6dd66b7c95-cd99x.istio-system
```

Visit the envoy configuration at http://localhost:58657/config_dump, you should see your verify_certificate_spki within the default validation context:

```
         "transport_socket": {
          "name": "envoy.transport_sockets.tls",
          "typed_config": {
           "@type": "type.googleapis.com/envoy.api.v2.auth.DownstreamTlsContext",
           "common_tls_context": {
            "alpn_protocols": [
             "h2",
             "http/1.1"
            ],
            "tls_certificate_sds_secret_configs": [
             {
              "name": "httpbin-credential",
              "sds_config": {
               "api_config_source": {
                "api_type": "GRPC",
                "grpc_services": [
                 {
                  "google_grpc": {
                   "target_uri": "unix:/var/run/ingress_gateway/sds",
                   "stat_prefix": "sdsstat"
                  }
                 }
                ]
               }
              }
             }
            ],
            "combined_validation_context": {
             "default_validation_context": {
              "verify_certificate_spki": [
               "abnLhNPH/LTI1HLbPBtX3TxAQgdTLkjYQpp8WkxQseE="
              ]
             },
             "validation_context_sds_secret_config": {
              "name": "httpbin-credential-cacert",
              "sds_config": {
               "api_config_source": {
                "api_type": "GRPC",
                "grpc_services": [
                 {
                  "google_grpc": {
                   "target_uri": "unix:/var/run/ingress_gateway/sds",
                   "stat_prefix": "sdsstat"
                  }
                 }
                ]
               }
              }
             }
            }
           },
           "require_client_certificate": true
          }
         }
 ```
That is it!  You can simply leverage the verifyCertificateSpki, or subjectAltNames or other fields within the [ServerTLSSettings](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings) of your ingress gateway resource for additional certificate validation.




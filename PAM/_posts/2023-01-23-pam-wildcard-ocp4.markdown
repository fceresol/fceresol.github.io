---
layout: post_with_sidebar
title: "How to use Openshift Wildcard Certificates with PAM before 7.11"
date: 2023-01-23 12:04:04 +0100
category: PAM
subcategory: OpenShift
tags: [PAM, OpenShift, HowTo]
 
---

When using RHPAM versions older than 7.11, the default Openshift Routes are created in Passtrough mode, using the default built-in certificates will lead in some certificates trust problems since those are self signed.
The following document will explain how to inject OpenShift wildcard certificates into the pods in order to use them.


In order to use OCP wildcard certificates from RH-PAM follow those steps:

1. Extract existing certificates from the openshift-ingress namespace
1.1 Extract the openshift secret
1.2 Create tls.crt and tls.key from the extracetd secret
1. Create the PKCS12 keystore with openssl
2. Convert the keystore in jks format 
3. Add the CA Certificate to the cacerts trustore (optional)
4. Provide the keystore to kie-server and business central
5.1 create the secret needed by the CR
5.2 configure the PAM Custom Resources to use the new keystore
5.3 configure the PAM Custom Resources to use the new trustore (optional)

the following tools will be needed:
- oc client
- openssl
- jq
- java keytool


## 1. Extract existing certificates from the openshift-ingress namespace ## 
extract the secret named `router-certs-default` from the `openshift-ingress` namespace: the secret contains two files: `tls.key` the tls encription key and `tls.crt` which contains the certificate chain.
### 1.1 Extract the openshift secret ### 
the secret can be exported to a file with the command

~~~ bash
$ oc get secret router-certs-default -n openshift-ingress -o json > router-certs-default-secret.json
~~~
{%- capture note -%}
the secret name may be different if you are using your own certificates, the one exported here is the default one.
{%- endcapture -%}

{% include note.html content=note %}

the extracted file content should look like the following:

~~~JSON
{
    "apiVersion": "v1",
    "data": {
        "tls.crt": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURXekNDQWtPZ0F3SUJBZ0lJTFY0NzhNby9ucXN3RFFZSktvWklodmNOQVFFTEJRQXdKakVrTUNJR0ExVUUKQXd3YmFXNW5jbVZ6Y3kxdmNHVnlZWFJ2Y2tBeE5qRXlNalEyT0RZd01CNFhEVEl4TURJd01qQTJNakV3TkZvWApEVEl6TURJd01qQTJNakV3TlZvd0hURWJNQmtHQTFVRUF3d1NLaTVoY0hCekxXTnlZeTUwWlhOMGFXNW5NSUlCCklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF5aW9HaEFvQTc1V2lKelBOcFByWHRNY1QKL3BHVHlOb2ZKZDI1Tmt3YTMrdkYvMTA4MHl0bW84S2Njcnh0dnBEQWJSQSs0dmdZZ2ljZW5hbGxSYXFnYXVkRgo4Y0tmTUdqV2pmSkVjN2VHZVR1R1hITTRDKzF4M3F0aEJzL2VQeGx2N3RKTVRNWkd6bFZSSzRGbXNlenVLbnFSCnJKUXZ3OGxkZ1ZVUHJuNjJlb0hTVGlrSzlwOHlHSFhiMitrS1JreW5JdlYramNQU2lVZ0swSFJwa1A2dGNRUmsKSGEwWjVRMnprZzRWU3lObXY0L01XQ1BoQTJDRmErb1MxSHdMYXNOaHkvNE9wTW1SOTJiS29xdjRSV0sxZzFlWgpMK2NUcGlQeGRjQnlzL1JSZllWVmtlTFQ3WE9NZ294VGhrQk1UNk5yUXZNNHBYdWhXWkt6dW11WERjZjFwd0lECkFRQUJvNEdWTUlHU01BNEdBMVVkRHdFQi93UUVBd0lGb0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREFUQU0KQmdOVkhSTUJBZjhFQWpBQU1CMEdBMVVkRGdRV0JCUUJHWkFFeFZQS016Rko4Sm02RkhYbmhYM3lSVEFmQmdOVgpIU01FR0RBV2dCUlpnSUxqdEFKYTBmV0MwZGswb3piT2RGKzRBekFkQmdOVkhSRUVGakFVZ2hJcUxtRndjSE10ClkzSmpMblJsYzNScGJtY3dEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSFZhbGxoTEpYUzJla3BhM1MzbVp4S3oKTFZzbjFKQ2lZZmloendLeExGK2hneHcrRUp6a2lmcHNjdkVJY2lYSWJSbllRM1VJcW5KRHJPK1FaZWdNVGZKQwpGZElRWjNtT3dFTGsrYThEM1ZxditQL0YyN2I2ZGIwejBtQ1lLajMwTHNKV0YyNW95RXpNNWlkbnRaVnFCTGdICmtaY25LbXdIcEVhMXZycE5hNC9zVXFQRmZITE81V1VCMFE0ZE9BK0dFOUd2L3dtM25JWGRqTm0raDJIOXhGNEMKdVlRbEx5Tm0rb1RUTEdta2h1eXE3WXNpcTZuYXFsbGU3d0RWVGxYeEM4R0ljOXJTZGdqMWhxLzlpZVNrZnVVZwpKY3BvWGJyVnVtb2dmOGllaU9jNmJtQllUdUVZTFYvVUJkbUtVK0dTTVplV3dkUjNKSDBpQnRoTnJyTExXTms9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0KLS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURERENDQWZTZ0F3SUJBZ0lCQVRBTkJna3Foa2lHOXcwQkFRc0ZBREFtTVNRd0lnWURWUVFEREJ0cGJtZHkKWlhOekxXOXdaWEpoZEc5eVFERTJNVEl5TkRZNE5qQXdIaGNOTWpFd01qQXlNRFl5TURVNVdoY05Nak13TWpBeQpNRFl5TVRBd1dqQW1NU1F3SWdZRFZRUUREQnRwYm1keVpYTnpMVzl3WlhKaGRHOXlRREUyTVRJeU5EWTROakF3CmdnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUURVV0pwVW5tN3RtRko1VXFKVWFBVEQKbzBvaXRKYXAvN2xzTlhidy8zT2Jva2hINFVNUmlRdlZ6aVA2bytSdjdyZlBQNjFoUkFwNmVRVkN3NXlRK1JhUApRMXMvQlpmSCs4MGNCL2FTbkpWZkh1TERJNXE4alZicmdUWUNmTy9lTjFiV05PWGFDVXhJL05tbW1zanlRMUlMCnhETG5kSC9NK0tRckNhQll4SFFCaVkrMWNHWUR3b3ZmSFdQOUVoTGdnK1MvK2hHYmpaTDJhUlV3YmhEdEFzN2QKSnUyYzRiekV2SUUyaHJHY2k4T1h3ZUdmL2xZcWdaZVk1SFJjblRRMUtrUnh0WklqdnFRR0RGVlRJOWNJQ0hCcQpMZk53TWFlRFNoUlRnWE9SZVdFV1dzbjlUaXpSS0JIUWxNWmJKblVybW5hdjFtTVhwRFFxVllHTmV0dTZQZEdKCkFnTUJBQUdqUlRCRE1BNEdBMVVkRHdFQi93UUVBd0lDcERBU0JnTlZIUk1CQWY4RUNEQUdBUUgvQWdFQU1CMEcKQTFVZERnUVdCQlJaZ0lManRBSmEwZldDMGRrMG96Yk9kRis0QXpBTkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQQpnY2Y0UUd5ZWtLMUx0bVVSU3I4TFlMcG9LRWpvL2h2VzVWenY2bjJxbThtUnZnZnc0VHk5UGgxdEcxVkpqWVFlCnFVbURlSVFEL3RWVlIxOWpOVlpJa0hYSzR2NUFvUFIvZ2RLSWM4ZnlPQm9kcWV1cUZmdTJ1TDNrRC9YSFhuOWgKRFFJTnVwWFlpRmt3T25FRU5ycWJsRVhvZHRDV28xeVRvR2RuK3N1MTF6ZFlka3V5ekJLWkJKV0NhZWp6V1ZMNwo4djZ0Wkk4Ui84YThDa01mV1B4VE5SODgvemhKVk4vTmpuOXc5RUpUNjRzYXdhTjNEYnRNMi91SEtiZE9xMFFLClRDdXppcHY3ajljS2V1d2FnVHBPNzFQY21EZWZCelRnbnZ6ZkZuTWNLd1h0cGpmczBHNWx4UmtDUVhJd0FGRnIKZ1lNTjdFY09Pb2JTTVloWWJEU3Mydz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K",
        "tls.key": "LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBeWlvR2hBb0E3NVdpSnpQTnBQclh0TWNUL3BHVHlOb2ZKZDI1Tmt3YTMrdkYvMTA4CjB5dG1vOEtjY3J4dHZwREFiUkErNHZnWWdpY2VuYWxsUmFxZ2F1ZEY4Y0tmTUdqV2pmSkVjN2VHZVR1R1hITTQKQysxeDNxdGhCcy9lUHhsdjd0Sk1UTVpHemxWUks0Rm1zZXp1S25xUnJKUXZ3OGxkZ1ZVUHJuNjJlb0hTVGlrSwo5cDh5R0hYYjIra0tSa3luSXZWK2pjUFNpVWdLMEhScGtQNnRjUVJrSGEwWjVRMnprZzRWU3lObXY0L01XQ1BoCkEyQ0ZhK29TMUh3TGFzTmh5LzRPcE1tUjkyYktvcXY0UldLMWcxZVpMK2NUcGlQeGRjQnlzL1JSZllWVmtlTFQKN1hPTWdveFRoa0JNVDZOclF2TTRwWHVoV1pLenVtdVhEY2YxcHdJREFRQUJBb0lCQUN3SUR6Yysvb2t3TEFzaAp5MDU5bS9HeDBuY0Z1Z3hyQlpHM3d4bENaakFUS0NMQWFma01ZT1NXQklFdzdTNHVWTnJzU09ZaVp5UWg1UmN0CngvTHVnTllIM1VJVXc1dEZta1Y4V05CalRwU2xGRlNhZThDTlROblV0ZU5IN3Y0TFNrZlg0ZXB1M1FrZnAvZ3oKek93LzBIZk1EbUpxUENVR2ZLa29uNnUveVhyUTNLTGlNNVdPYXRHNHpjc2ZOUFhNY1FWQ3EybkxINXZkdEJvQwo1cHhGTW1xY3hwd0RZOUI2VXZCRmgwNlZlbVJHQ0k1dG9DMnVqNEtvYW91dThmL3pIcWtEc29OSU9vMlQxdlR3Ckk5OFIrWXRiSE1rK0EwRkxBNzRiTjFCVEY0OWova2ZvczdSblNaS3lKcFRQa3IxRjRDcmYwdzhldlZydHpGZkIKT29sUWY3RUNnWUVBMys3YlVLTU9QTEpEZHcvUVJZUG45ejRITThEZnhqRzVSUXF4THJWdjZyZlArSDdTSU0wZApydVorSHJ1SWdaUEg4ekNHSFRyWjBvYkZ0SnZIY21kOGtKUUF0cUZOSjdmMitVR1lyd0hPeGcrZmlvdjN3SThECkhKNmx4anpnQ3V3clhGUXBENDJGdUluMmNlVHQrT3N3ZG5FK2Q4eUpnd1p5S3NKRENRaUZRcTBDZ1lFQTV4MGwKT1E2eWF3TTBPMVZqRFh3SE80b0xGVStOcjNWWGRtWUxpQkJBMXdkcUlZQUxKaE44dnBJV01hdjJ0Qkg3b1BtWApzU1RFMy84NGJ1OVRaYlFYdnVNVWowNkM4anR5ditXWDJGTkttM1UydGFSWjJRb0g5SkNJS0ZTWjlvalNrb3g2CkxYUElhcWlqTTY2Z1dZdjNVeElPVWdlbXY1Qk10QTU2RktvaU9DTUNnWUVBelhiM3ZCRWdLd2pWWmhVWVgvQWIKa250VFdHVUw3V29LT0JNTFozUUtjQzZmbjcyZFI0TnNUT0lucmtNYmlPanplV3Q0WXJGdzB2M2R3VTE5dnJhOQpVRnE4SE5YN1dRb3VqWjFtWG8wbUVBeWRzaDJqQVFjM0w3ZFJHNGNYZW00Zml1T2RtU3VkR2lsYitqeTNMTUYvCkFlMytCeVdndHB2ZmZPUXBaY3h2bVRFQ2dZQlJZVlRyRzM2OTZkbnBqcTZiWC9JWUNBclJEVHRCN2xyRzZUWGsKU256YWV0VG5TUFFrQ3phZzBFWWFaWWd3YmlpaHpXR1owZTIxUm1Senc3Z2xGdDVKckNKZ04vQXFKYjdKVGFwRApWVWp2SnI0R0JnSlJSNVAzalRFMHFsMndqd3MrNlZKWVVPM2dpTk0yN3FXdUFuZ3JleThwdVdJQkVHbkIrVnNKCmpjTVE2d0tCZ0dUMkpJZlg5RHFpTkw0MmpoSjlneHJnTllvV2FhQkdsdU1oTGV6MGU1eVlwdEpCZGV1bi9sb3UKNVVKTE1lVlVURk11L2hIeXNDWldDWGpla05CVi9pdEJQSisyRmI2YXJlMEI0dnMwM1A0TGVmQ1ZoUCtnSksyNQpuaXRLNzhMK1huVnhzcDdXdW1Na2RZVEZ0eFd4d3dFaW9CQlZQUjVyeTd6TlFYeHNwMmg2Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg=="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2021-02-02T06:21:05Z",
        "managedFields": [
            {
                "apiVersion": "v1",
                "fieldsType": "FieldsV1",
                "fieldsV1": {
                    "f:data": {
                        ".": {},
                        "f:tls.crt": {},
                        "f:tls.key": {}
                    },
                    "f:metadata": {
                        "f:ownerReferences": {
                            ".": {},
                            "k:{\"uid\":\"ff4cc088-fa88-4a7a-9bf8-3807b06b1bd8\"}": {
                                ".": {},
                                "f:apiVersion": {},
                                "f:controller": {},
                                "f:kind": {},
                                "f:name": {},
                                "f:uid": {}
                            }
                        }
                    },
                    "f:type": {}
                },
                "manager": "ingress-operator",
                "operation": "Update",
                "time": "2021-02-02T06:21:05Z"
            }
        ],
        "name": "router-certs-default",
        "namespace": "openshift-ingress",
        "ownerReferences": [
            {
                "apiVersion": "apps/v1",
                "controller": true,
                "kind": "Deployment",
                "name": "router-default",
                "uid": "ff4cc088-fa88-4a7a-9bf8-3807b06b1bd8"
            }
        ],
        "resourceVersion": "6523",
        "selfLink": "/api/v1/namespaces/openshift-ingress/secrets/router-certs-default",
        "uid": "03121ea7-161c-49dd-9e05-e4325e1ae6cc"
    },
    "type": "kubernetes.io/tls"
}
~~~

the two lines in the `data:` section contains our files.

### 1.2 Create tls.crt and tls.key from the extracted secret ### 

~~~ bash
$ cat router-certs-default-secret.json | jq -r '.data."tls.key"' | base64 -d > tls.key
~~~

the generated file content will be similar tho the following one:

~~~
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAyioGhAoA75WiJzPNpPrXtMcT/pGTyNofJd25Nkwa3+vF/108
0ytmo8KccrxtvpDAbRA+4vgYgicenallRaqgaudF8cKfMGjWjfJEc7eGeTuGXHM4
C+1x3qthBs/ePxlv7tJMTMZGzlVRK4FmsezuKnqRrJQvw8ldgVUPrn62eoHSTikK
9p8yGHXb2+kKRkynIvV+jcPSiUgK0HRpkP6tcQRkHa0Z5Q2zkg4VSyNmv4/MWCPh
A2CFa+oS1HwLasNhy/4OpMmR92bKoqv4RWK1g1eZL+cTpiPxdcBys/RRfYVVkeLT
7XOMgoxThkBMT6NrQvM4pXuhWZKzumuXDcf1pwIDAQABAoIBACwIDzc+/okwLAsh
y059m/Gx0ncFugxrBZG3wxlCZjATKCLAafkMYOSWBIEw7S4uVNrsSOYiZyQh5Rct
x/LugNYH3UIUw5tFmkV8WNBjTpSlFFSae8CNTNnUteNH7v4LSkfX4epu3Qkfp/gz
zOw/0HfMDmJqPCUGfKkon6u/yXrQ3KLiM5WOatG4zcsfNPXMcQVCq2nLH5vdtBoC
5pxFMmqcxpwDY9B6UvBFh06VemRGCI5toC2uj4Koaouu8f/zHqkDsoNIOo2T1vTw
I98R+YtbHMk+A0FLA74bN1BTF49j/kfos7RnSZKyJpTPkr1F4Crf0w8evVrtzFfB
OolQf7ECgYEA3+7bUKMOPLJDdw/QRYPn9z4HM8DfxjG5RQqxLrVv6rfP+H7SIM0d
ruZ+HruIgZPH8zCGHTrZ0obFtJvHcmd8kJQAtqFNJ7f2+UGYrwHOxg+fiov3wI8D
HJ6lxjzgCuwrXFQpD42FuIn2ceTt+OswdnE+d8yJgwZyKsJDCQiFQq0CgYEA5x0l
OQ6yawM0O1VjDXwHO4oLFU+Nr3VXdmYLiBBA1wdqIYALJhN8vpIWMav2tBH7oPmX
sSTE3/84bu9TZbQXvuMUj06C8jtyv+WX2FNKm3U2taRZ2QoH9JCIKFSZ9ojSkox6
LXPIaqijM66gWYv3UxIOUgemv5BMtA56FKoiOCMCgYEAzXb3vBEgKwjVZhUYX/Ab
kntTWGUL7WoKOBMLZ3QKcC6fn72dR4NsTOInrkMbiOjzeWt4YrFw0v3dwU19vra9
UFq8HNX7WQoujZ1mXo0mEAydsh2jAQc3L7dRG4cXem4fiuOdmSudGilb+jy3LMF/
Ae3+ByWgtpvffOQpZcxvmTECgYBRYVTrG3696dnpjq6bX/IYCArRDTtB7lrG6TXk
SnzaetTnSPQkCzag0EYaZYgwbiihzWGZ0e21RmRzw7glFt5JrCJgN/AqJb7JTapD
VUjvJr4GBgJRR5P3jTE0ql2wjws+6VJYUO3giNM27qWuAngrey8puWIBEGnB+VsJ
jcMQ6wKBgGT2JIfX9DqiNL42jhJ9gxrgNYoWaaBGluMhLez0e5yYptJBdeun/lou
5UJLMeVUTFMu/hHysCZWCXjekNBV/itBPJ+2Fb6are0B4vs03P4LefCVhP+gJK25
nitK78L+XnVxsp7WumMkdYTFtxWxwwEioBBVPR5ry7zNQXxsp2h6
-----END RSA PRIVATE KEY-----
~~~

~~~ bash
$ cat router-certs-default-secret.json | jq -r '.data."tls.crt"' | base64 -d > tls.crt
~~~

~~~
-----BEGIN CERTIFICATE-----
MIIDWzCCAkOgAwIBAgIILV478Mo/nqswDQYJKoZIhvcNAQELBQAwJjEkMCIGA1UE
AwwbaW5ncmVzcy1vcGVyYXRvckAxNjEyMjQ2ODYwMB4XDTIxMDIwMjA2MjEwNFoX
DTIzMDIwMjA2MjEwNVowHTEbMBkGA1UEAwwSKi5hcHBzLWNyYy50ZXN0aW5nMIIB
IjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyioGhAoA75WiJzPNpPrXtMcT
/pGTyNofJd25Nkwa3+vF/1080ytmo8KccrxtvpDAbRA+4vgYgicenallRaqgaudF
8cKfMGjWjfJEc7eGeTuGXHM4C+1x3qthBs/ePxlv7tJMTMZGzlVRK4FmsezuKnqR
rJQvw8ldgVUPrn62eoHSTikK9p8yGHXb2+kKRkynIvV+jcPSiUgK0HRpkP6tcQRk
Ha0Z5Q2zkg4VSyNmv4/MWCPhA2CFa+oS1HwLasNhy/4OpMmR92bKoqv4RWK1g1eZ
L+cTpiPxdcBys/RRfYVVkeLT7XOMgoxThkBMT6NrQvM4pXuhWZKzumuXDcf1pwID
AQABo4GVMIGSMA4GA1UdDwEB/wQEAwIFoDATBgNVHSUEDDAKBggrBgEFBQcDATAM
BgNVHRMBAf8EAjAAMB0GA1UdDgQWBBQBGZAExVPKMzFJ8Jm6FHXnhX3yRTAfBgNV
HSMEGDAWgBRZgILjtAJa0fWC0dk0ozbOdF+4AzAdBgNVHREEFjAUghIqLmFwcHMt
Y3JjLnRlc3RpbmcwDQYJKoZIhvcNAQELBQADggEBAHVallhLJXS2ekpa3S3mZxKz
LVsn1JCiYfihzwKxLF+hgxw+EJzkifpscvEIciXIbRnYQ3UIqnJDrO+QZegMTfJC
FdIQZ3mOwELk+a8D3Vqv+P/F27b6db0z0mCYKj30LsJWF25oyEzM5idntZVqBLgH
kZcnKmwHpEa1vrpNa4/sUqPFfHLO5WUB0Q4dOA+GE9Gv/wm3nIXdjNm+h2H9xF4C
uYQlLyNm+oTTLGmkhuyq7Ysiq6naqlle7wDVTlXxC8GIc9rSdgj1hq/9ieSkfuUg
JcpoXbrVumogf8ieiOc6bmBYTuEYLV/UBdmKU+GSMZeWwdR3JH0iBthNrrLLWNk=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDDDCCAfSgAwIBAgIBATANBgkqhkiG9w0BAQsFADAmMSQwIgYDVQQDDBtpbmdy
ZXNzLW9wZXJhdG9yQDE2MTIyNDY4NjAwHhcNMjEwMjAyMDYyMDU5WhcNMjMwMjAy
MDYyMTAwWjAmMSQwIgYDVQQDDBtpbmdyZXNzLW9wZXJhdG9yQDE2MTIyNDY4NjAw
ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDUWJpUnm7tmFJ5UqJUaATD
o0oitJap/7lsNXbw/3ObokhH4UMRiQvVziP6o+Rv7rfPP61hRAp6eQVCw5yQ+RaP
Q1s/BZfH+80cB/aSnJVfHuLDI5q8jVbrgTYCfO/eN1bWNOXaCUxI/NmmmsjyQ1IL
xDLndH/M+KQrCaBYxHQBiY+1cGYDwovfHWP9EhLgg+S/+hGbjZL2aRUwbhDtAs7d
Ju2c4bzEvIE2hrGci8OXweGf/lYqgZeY5HRcnTQ1KkRxtZIjvqQGDFVTI9cICHBq
LfNwMaeDShRTgXOReWEWWsn9TizRKBHQlMZbJnUrmnav1mMXpDQqVYGNetu6PdGJ
AgMBAAGjRTBDMA4GA1UdDwEB/wQEAwICpDASBgNVHRMBAf8ECDAGAQH/AgEAMB0G
A1UdDgQWBBRZgILjtAJa0fWC0dk0ozbOdF+4AzANBgkqhkiG9w0BAQsFAAOCAQEA
gcf4QGyekK1LtmURSr8LYLpoKEjo/hvW5Vzv6n2qm8mRvgfw4Ty9Ph1tG1VJjYQe
qUmDeIQD/tVVR19jNVZIkHXK4v5AoPR/gdKIc8fyOBodqeuqFfu2uL3kD/XHXn9h
DQINupXYiFkwOnEENrqblEXodtCWo1yToGdn+su11zdYdkuyzBKZBJWCaejzWVL7
8v6tZI8R/8a8CkMfWPxTNR88/zhJVN/Njn9w9EJT64sawaN3DbtM2/uHKbdOq0QK
TCuzipv7j9cKeuwagTpO71PcmDefBzTgnvzfFnMcKwXtpjfs0G5lxRkCQXIwAFFr
gYMN7EcOOobSMYhYbDSs2w==
-----END CERTIFICATE-----
~~~

## 2. Create the PKCS12 keystore with openssl ## 

the following command will create a PKCS12 keystore containing the tls key and certificate using **jboss** as alias

~~~ bash
$ openssl pkcs12 -export -in tls.crt -inkey tls.key -out wildcard.p12 -name jboss
~~~
the user will be prompted for a password twice

## 3. Convert the keystore in jks format ## 

~~~ bash
$ keytool -importkeystore \
        -deststorepass [changeit] \
        -destkeystore keystore.jks \
        -deststoretype jks \
        -srckeystore wildcard.p12 \
        -srcstoretype PKCS12 \
        -srcstorepass [the password entered in the step above] \
        -trustcacerts
~~~
The command output will look like this:

~~~ 
Importing keystore wildcard.p12 to keystore.jks...
Entry for alias wildcard successfully imported.
Import command completed:  1 entries successfully imported, 0 entries failed or cancelled

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore keystore.jks -destkeystore keystore.jks -deststoretype pkcs12".
~~~

the keystore content could be verified with the following command:

~~~ bash

$ keytool -list -v -keystore keystore.jks -storepass [changeit]

~~~
The output should resemble: 
~~~

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

Alias name: jboss
Creation date: Feb 25, 2021
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=*.apps-crc.testing
Issuer: CN=ingress-operator@1612246860
Serial number: 2d5e3bf0ca3f9eab
Valid from: Tue Feb 02 07:21:04 CET 2021 until: Thu Feb 02 07:21:05 CET 2023
Certificate fingerprints:
	 SHA1: 8E:C2:9F:BD:17:8E:B8:1F:5D:65:6C:45:90:A4:19:9B:7D:B9:6F:0D
	 SHA256: 42:6C:0D:13:CD:91:3C:E2:06:3C:87:BA:2D:1A:07:22:AA:82:91:92:76:C4:E5:26:3D:E5:EB:E3:AC:84:01:27
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 59 80 82 E3 B4 02 5A D1   F5 82 D1 D9 34 A3 36 CE  Y.....Z.....4.6.
0010: 74 5F B8 03                                        t_..
]
]

#2: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:false
  PathLen: undefined
]

#3: ObjectId: 2.5.29.37 Criticality=false
ExtendedKeyUsages [
  serverAuth
]

#4: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Key_Encipherment
]

#5: ObjectId: 2.5.29.17 Criticality=false
SubjectAlternativeName [
  DNSName: *.apps-crc.testing
]

#6: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 01 19 90 04 C5 53 CA 33   31 49 F0 99 BA 14 75 E7  .....S.31I....u.
0010: 85 7D F2 45                                        ...E
]
]

Certificate[2]:
Owner: CN=ingress-operator@1612246860
Issuer: CN=ingress-operator@1612246860
Serial number: 1
Valid from: Tue Feb 02 07:20:59 CET 2021 until: Thu Feb 02 07:21:00 CET 2023
Certificate fingerprints:
	 SHA1: A1:1B:42:CD:6C:62:2E:19:C9:21:60:D2:38:C1:B4:27:2C:45:A9:31
	 SHA256: C8:AB:A5:C9:E2:37:CB:A7:B2:3E:DC:F6:B6:6F:DB:26:73:17:DF:63:CC:01:2B:75:AA:77:82:DC:72:48:34:C5
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:true
  PathLen:0
]

#2: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Key_Encipherment
  Key_CertSign
]

#3: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 59 80 82 E3 B4 02 5A D1   F5 82 D1 D9 34 A3 36 CE  Y.....Z.....4.6.
0010: 74 5F B8 03                                        t_..
]
]



*******************************************
*******************************************



Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore keystore.jks -destkeystore keystore.jks -deststoretype pkcs12".
~~~

## 4. Add the CA Certificate to the cacerts trustore (optional)

locate the default java trust store **cacerts** usually under ***&lt;JAVA_HOME&gt;/jre/lib/security/cacerts*** and copy it in a local folder

import the **tls.crt** cert file in the default trust store with

~~~ bash

$ keytool -import \
          -trustcacerts \
          -alias wildcard \
          -file tls.crt \
          -keystore cacerts
~~~
the command will ask for the keystore password, the default one is ***changeit***.
after that the certificate to get imported will be printed and the user will be asked to trust it.
reply yes.

here's a sample output

~~~
$ keytool -import -trustcacerts -alias wildcard -file tls.crt -keystore cacerts
Enter keystore password:  
Owner: CN=*.apps-crc.testing
Issuer: CN=ingress-operator@1612246860
Serial number: 2d5e3bf0ca3f9eab
Valid from: Tue Feb 02 07:21:04 CET 2021 until: Thu Feb 02 07:21:05 CET 2023
Certificate fingerprints:
	 SHA1: 8E:C2:9F:BD:17:8E:B8:1F:5D:65:6C:45:90:A4:19:9B:7D:B9:6F:0D
	 SHA256: 42:6C:0D:13:CD:91:3C:E2:06:3C:87:BA:2D:1A:07:22:AA:82:91:92:76:C4:E5:26:3D:E5:EB:E3:AC:84:01:27
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 59 80 82 E3 B4 02 5A D1   F5 82 D1 D9 34 A3 36 CE  Y.....Z.....4.6.
0010: 74 5F B8 03                                        t_..
]
]

#2: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:false
  PathLen: undefined
]

#3: ObjectId: 2.5.29.37 Criticality=false
ExtendedKeyUsages [
  serverAuth
]

#4: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Key_Encipherment
]

#5: ObjectId: 2.5.29.17 Criticality=false
SubjectAlternativeName [
  DNSName: *.apps-crc.testing
]

#6: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 01 19 90 04 C5 53 CA 33   31 49 F0 99 BA 14 75 E7  .....S.31I....u.
0010: 85 7D F2 45                                        ...E
]
]

Trust this certificate? [no]:  yes
Certificate was added to keystore

~~~

## 5. Provide the keystore to kie-server and business central ## 

Once the jks keystore has been created it must be provided to PAM components in order to be used. For use it a secret containing the file will be provided and the related configurations will be performed in the Custom Resource owned by the Operator

### 5.1 create the secret needed by the components ### 

the command for creating the new secret on OCP is

~~~ bash

$ oc create secret generic rhpam-wildcard-keystore-secret --from-file keystore.jks --from-file cacerts

~~~

the second **--from-file** could be omitted if the cacerts file has not been created in **step 4**

### 5.2 configure the PAM Custom Resources to use the new keystore ### 

after the secret creation, the last step is to configure the CR in order to use it

there are some distinct configurations to be applied to the PAM Custom Resource:

under spec: commonConfig: a property called keyStorePassword has to be configured to provide the key store password

~~~ yaml
...
spec:
  commonConfig:
    keyStorePassword: [changeit]
...
~~~

under `spec: objects: console:` **`keystoreSecret`** must be specified containing the secret name to be used by Business Central (in this case the secret for BC and Kie-Server will be the same)

~~~ yaml
...
spec:
...
  objects:
    console:
      keystoreSecret: rhpam-wildcard-keystore-secret
...
~~~

under `spec: objects: servers:`  **`keystoreSecret`** must be specified containing the secret name to be used by Business Central (in this case are the same for BC and Kie-Server)

~~~ yaml
spec:
...
  objects:
...
    servers:
      - ...
        keystoreSecret: rhpam-wildcard-keystore-secret
~~~

{%- capture warning -%}
the secret must be specified for every server of the configuration
{% endcapture %}

{%- include warning.html content=warning -%}

when the changes are applied the Custom Resource will be similar to this:
~~~ yaml
apiVersion: app.kiegroup.org/v2
kind: KieApp
metadata:
  creationTimestamp: '2021-02-16T14:20:20Z'
  generation: 10
  managedFields:
    - apiVersion: app.kiegroup.org/v2
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          .: {}
          'f:commonConfig':
            .: {}
            'f:keyStorePassword': {}
          'f:environment': {}
          'f:objects':
            .: {}
            'f:console':
              .: {}
              'f:env': {}
              'f:keystoreSecret': {}
            'f:servers': {}
      manager: Mozilla
      operation: Update
      time: '2021-02-24T14:47:06Z'
    - apiVersion: app.kiegroup.org/v2
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:app.kiegroup.org': {}
        'f:spec':
          'f:upgrades': {}
        'f:status':
          .: {}
          'f:applied':
            .: {}
            'f:commonConfig':
              .: {}
              'f:adminPassword': {}
              'f:adminUser': {}
              'f:amqClusterPassword': {}
              'f:amqPassword': {}
              'f:applicationName': {}
              'f:dbPassword': {}
              'f:keyStorePassword': {}
            'f:environment': {}
            'f:objects':
              .: {}
              'f:console':
                .: {}
                'f:env': {}
                'f:jvm':
                  .: {}
                  'f:javaInitialMemRatio': {}
                  'f:javaMaxMemRatio': {}
                'f:keystoreSecret': {}
                'f:replicas': {}
                'f:resources':
                  .: {}
                  'f:limits':
                    .: {}
                    'f:cpu': {}
                    'f:memory': {}
                  'f:requests':
                    .: {}
                    'f:cpu': {}
                    'f:memory': {}
              'f:servers': {}
            'f:upgrades': {}
            'f:version': {}
          'f:conditions': {}
          'f:consoleHost': {}
          'f:deployments':
            .: {}
            'f:ready': {}
          'f:phase': {}
          'f:version': {}
      manager: kie-cloud-operator
      operation: Update
      time: '2021-02-25T10:56:55Z'
  name: rhpam-trial
  namespace: rhpam
  resourceVersion: '635668'
  selfLink: /apis/app.kiegroup.org/v2/namespaces/unicredit/rhpam/rhpam-trial
  uid: 493f5b3a-12b5-4a8d-8a66-8557f0cd1364
spec:
  commonConfig:
    keyStorePassword: [changeit]
  environment: rhpam-authoring
  objects:
    console:
      keystoreSecret: rhpam-wildcard-keystore-secret
    servers:
      - keystoreSecret: rhpam-wildcard-keystore-secret
status:
...
~~~

Once saved the Operator will take care on restarting the servers.

### 5.3 configure the PAM Custom Resources to use the new truststore (optional) ### 

if a new trust store has been created in **step 4** there are some additional properties to be set in the KieApp CR

~~~yaml

apiVersion: app.kiegroup.org/v2
kind: KieApp
metadata:
  creationTimestamp: '2021-02-16T14:20:20Z'
  generation: 10
  managedFields:
    - apiVersion: app.kiegroup.org/v2
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          .: {}
          'f:commonConfig':
            .: {}
            'f:keyStorePassword': {}
          'f:environment': {}
          'f:objects':
            .: {}
            'f:console':
              .: {}
              'f:env': {}
              'f:keystoreSecret': {}
            'f:servers': {}
      manager: Mozilla
      operation: Update
      time: '2021-02-24T14:47:06Z'
    - apiVersion: app.kiegroup.org/v2
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:app.kiegroup.org': {}
        'f:spec':
          'f:upgrades': {}
        'f:status':
          .: {}
          'f:applied':
            .: {}
            'f:commonConfig':
              .: {}
              'f:adminPassword': {}
              'f:adminUser': {}
              'f:amqClusterPassword': {}
              'f:amqPassword': {}
              'f:applicationName': {}
              'f:dbPassword': {}
              'f:keyStorePassword': {}
            'f:environment': {}
            'f:objects':
              .: {}
              'f:console':
                .: {}
                'f:env': {}
                'f:jvm':
                  .: {}
                  'f:javaInitialMemRatio': {}
                  'f:javaMaxMemRatio': {}
                'f:keystoreSecret': {}
                'f:replicas': {}
                'f:resources':
                  .: {}
                  'f:limits':
                    .: {}
                    'f:cpu': {}
                    'f:memory': {}
                  'f:requests':
                    .: {}
                    'f:cpu': {}
                    'f:memory': {}
              'f:servers': {}
            'f:upgrades': {}
            'f:version': {}
          'f:conditions': {}
          'f:consoleHost': {}
          'f:deployments':
            .: {}
            'f:ready': {}
          'f:phase': {}
          'f:version': {}
      manager: kie-cloud-operator
      operation: Update
      time: '2021-02-25T10:56:55Z'
  name: rhpam-trial
  namespace: rhpam
  resourceVersion: '635668'
  selfLink: /apis/app.kiegroup.org/v2/namespaces/unicredit/rhpam/rhpam-trial
  uid: 493f5b3a-12b5-4a8d-8a66-8557f0cd1364
spec:
  commonConfig:
    keyStorePassword: [changeit]
  environment: rhpam-authoring
  objects:
    console:
      jvm:
        javaOptsAppend: -Djavax.net.ssl.trustStore=/etc/businesscentral-secret-volume/cacerts
      keystoreSecret: rhpam-wildcard-keystore-secret
    servers:
    - jvm:
        javaOptsAppend: -Djavax.net.ssl.trustStore=/etc/kieserver-secret-volume/cacerts
      keystoreSecret: rhpam-wildcard-keystore-secret
status:
...
~~~

{%- capture note -%}
the <b>javaOptsAppend</b> property will be used to specify the truststore location with
  <b>-Djavax.net.ssl.trustStore=&lt;TrustStorePath&gt;</b><br/>
if the trustsore password has changed from the default an additional property must be added:
  <b>-Djavax.net.ssl.trustStorePassword=&lt;trustStorePassword&gt;</b>
{% endcapture %}

{% include note.html content=note %}
---
title: "Authenticating Servers with Their Existing Certificates via Traefik mTLS"
description: "Why the servers I manage authenticate to services in Kubernetes with the certificate their Puppet agent already holds: the Traefik TLSOption and CA Secret, RequireAndVerifyClientCert versus VerifyClientCertIfGiven, the passTLSClientCert header the app reads, and the seven months a stale CA bundle rejected every server without anyone noticing."
dateString: October 2026
draft: false
tags: ["Kubernetes", "Traefik", "mTLS", "Security", "Puppet", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/traefik-mtls-existing-certificates/traefik-mtls-existing-certificates.png"
---

Every Linux server I manage ships metrics to a Prometheus in a Kubernetes cluster and calls a Go API in the same cluster. Both endpoints sit behind Traefik on a public hostname. The usual answer to how a server proves who it is to endpoints like these is a token per host, written into a config file by configuration management, stored somewhere central, rotated when someone remembers. The token is a second identity for a machine that already has one.

It already has one because every one of those servers runs a Puppet (OpenVox) agent, and the Puppet CA signed a certificate for each host the day it enrolled. The subject is the host's certname. The key never leaves the machine. Issuing that certificate, putting it on the disk and keeping it current is already part of how the fleet is managed, for reasons that have nothing to do with my API. The hard parts of mTLS already belong to something else.

So the servers authenticate with that certificate. Traefik trusts the Puppet CA, the Ingresses that only servers should reach require a client certificate, and a middleware hands the certificate's CN to the backend as a header. Most of what follows is about the one decision inside that, which `clientAuthType` goes where, and about the way it failed on me: a CA bundle that was a hand-made snapshot, and a Puppet CA that moved on without it.

Puppet is the CA here because it is what I run. Anything that already issues a certificate per host fits the same shape: step-ca, Vault PKI, SPIFFE, Teleport host certificates, a corporate PKI. If a key pair and a CA-signed certificate are already on every machine for some other reason, this post applies.

## The certificate is already on the disk

A Puppet agent keeps its credentials under `/etc/puppetlabs/puppet/ssl`. Three settings name the files, and `puppet config print` resolves them:

```
$ puppet config print hostcert hostprivkey localcacert
hostcert = /etc/puppetlabs/puppet/ssl/certs/web01.acme.pem
hostprivkey = /etc/puppetlabs/puppet/ssl/private_keys/web01.acme.pem
localcacert = /etc/puppetlabs/puppet/ssl/certs/ca.pem
```

The certificate itself:

```
$ openssl x509 -in /etc/puppetlabs/puppet/ssl/certs/web01.acme.pem -noout -subject -issuer -dates
subject=CN = web01.acme
issuer=CN = "Puppet CA generated on puppet at 2025-12-11 12:31:33 +0000"
notBefore=Dec 23 06:47:41 2025 GMT
notAfter=Dec 23 06:47:41 2030 GMT
```

Two things in that output carry the whole design. The CN is the certname, and in my environments the certname is deliberately two labels, `<hostname>.<customer-id>`, so the certificate says not just which machine is calling but which tenant it belongs to. And the issuer is a CA I control, whose CN is a timestamp of when it was generated. That timestamp matters later.

The certificates are five-year. In one environment the CA is a two-certificate chain, a signing CA under a self-signed `Puppet Root CA: <id>`. Custom facts `hostcert`, `hostprivkey` and `localcacert` return the corresponding `Puppet[:setting]`, so a manifest that configures a client never hard-codes a path.

## Traefik: one Secret and one TLSOption

Traefik reads the CA from a Secret in the same namespace as the TLSOption that references it. The CRD reference says the Secret "must contain a certificate under either a `tls.ca` or a `ca.crt` key", and the provider code checks `tls.ca` first, then `ca.crt`.

```bash
kubectl -n traefik create secret generic puppetca-cert --from-file=ca.crt=ca.pem
```

I keep that Secret in git as a SealedSecret, but there is nothing in it a plain Secret cannot hold; a CA certificate is public.

The TLSOption is what turns a CA into a handshake policy:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: prometheus-client-cert
  namespace: traefik
spec:
  minVersion: VersionTLS12
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
      - puppetca-cert
```

`clientAuthType` takes the five values of Go's `tls.ClientAuthType`, under Go's names. Two of them authenticate: `RequireAndVerifyClientCert` requires a certificate and verifies it against the CAs, `VerifyClientCertIfGiven` verifies one if it is sent and proceeds without one otherwise. `NoClientCert`, `RequestClientCert` and `RequireAnyClientCert` either ignore the certificate or accept it unverified, for applications that want to inspect it themselves.

I run two options from the same CA list, and the difference is that one word:

```yaml
tlsOptions:
  api-client-cert:
    minVersion: VersionTLS12
    maxVersion: VersionTLS13
    clientAuth:
      clientAuthType: VerifyClientCertIfGiven
      secretNames:
        - puppetca-cert
        - puppetca-cert-2025-12
        - customer-a-puppetca-cert
        - customer-b-puppetca-cert
  prometheus-client-cert:
    minVersion: VersionTLS12
    maxVersion: VersionTLS13
    clientAuth:
      clientAuthType: RequireAndVerifyClientCert
      secretNames:
        - puppetca-cert
        - puppetca-cert-2025-12
        - customer-a-puppetca-cert
        - customer-b-puppetca-cert
```

That is the `tlsOptions` value of the `traefik/traefik` Helm chart, 41.6.0 at the time of writing (Traefik v3.7.13). The chart renders one TLSOption per key, named after the key, in the release namespace, which is why the Secrets live in `traefik` too.

The Prometheus remote-write endpoint is only ever called by servers. A connection without a certificate from one of those CAs has no business there, and `RequireAndVerifyClientCert` refuses it during the handshake, before a single HTTP byte. The API also serves a browser UI on the same hostname, where people log in and get a JWT, and a browser has no Puppet certificate. So the API gets `VerifyClientCertIfGiven`: a server's certificate is verified, a browser's missing certificate is tolerated, and the application decides per route whether a verified certificate was required. Both options verify a certificate when one is presented. Only the first refuses a client that presents none.

The `secretNames` list is longer than one entry on purpose. Each customer environment has its own Puppet server and so its own CA, one Secret each. Two of the entries are my own CA twice over in time: in December 2025 a storage failure destroyed my Puppet server's CA volume, a new CA was generated, and a new Secret went in next to the old one so that, in the words of the commit, agents not yet re-issued keep working. Two months later the same thing happened on another cluster. The list is the union of every CA any live server might present. Removing an entry is a deliberate step once no client uses it, and nothing in Traefik will tell you when that is.

## Attaching it: an annotation or a field

On a plain Ingress, the option is one annotation. CRD objects are referenced from an Ingress as `<namespace>-<name>@kubernetescrd`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.options: traefik-prometheus-client-cert@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: prometheus.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus
                port:
                  name: web
  tls:
    - hosts:
        - prometheus.example.com
      secretName: prometheus-example-com-tls
```

On an IngressRoute it is a field, `spec.tls.options` with `name` and `namespace`. When the namespace is omitted Traefik looks in the IngressRoute's own namespace, and when it is a different one the CRD provider needs `allowCrossNamespace: true`, which is off by default. I run both forms.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: prometheus
  namespace: monitoring
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`prometheus.example.com`)
      kind: Rule
      services:
        - name: prometheus
          port: web
  tls:
    secretName: prometheus-example-com-tls
    options:
      name: prometheus-client-cert
      namespace: traefik
```

Now the constraint. A TLS option is declared on a router but applied during the handshake, when Traefik knows nothing about the request except the SNI server name. So the option is not attached to the route; it is attached to every hostname in the router's `Host` rule, and the Ingress docs say so: options are mapped to the host names of the rules. Two routers on the same entry point serving the same hostname with different options is a conflict, and the documented behaviour is that both mappings are discarded and the hostname falls back to the TLS option named `default`. A `default` with no `clientAuth` has no client certificate requirement, so the hostname silently stops asking for one. `core.strictTLSOptions` makes the conflict a hard failure instead, the routers are not built, which is louder and safer. Either way: every router on a hostname references the same option, and `default` is at least as strict as anything that might fall back to it.

This is why the API hostname has one option, `VerifyClientCertIfGiven`, rather than a strict option on the server-facing paths and a lenient one on the UI. That split is the obvious first design, and it cannot be expressed in TLS, because the paths share a hostname, so the handshake accepts both kinds of client and the server-only routes are enforced in the application. The alternative, a second hostname for the server-facing API with its own strict option, is the cleaner design if you are starting fresh.

## Getting the certname to the app

A verified handshake tells Traefik who the client is. The backend still has to be told, and the `passTLSClientCert` middleware does it with headers. With `pem: true` it sends the whole certificate in `X-Forwarded-Tls-Client-Cert`, delimiters and newlines removed, which the docs warn can exceed a backend's header size limit. With `info` you pick fields and get `X-Forwarded-Tls-Client-Cert-Info`, `Key="value"` pairs separated by `;` and escaped to be valid in a URL query. I only want the CN:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cn-header
spec:
  chain:
    middlewares:
      - name: clear-cn-headers
      - name: add-cn-header
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: clear-cn-headers
spec:
  headers:
    customRequestHeaders:
      X-Ssl-Client-Cn: ""
      X-Forwarded-Tls-Client-Cert-Info: ""
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: add-cn-header
spec:
  passTLSClientCert:
    info:
      subject:
        commonName: true
```

These live in the application's namespace and ship with its chart; the Ingress attaches the chain with `traefik.ingress.kubernetes.io/router.middlewares: <namespace>-cn-header@kubernetescrd`. Middlewares run in declared order, so the chain clears first and sets second.

Read the middleware's code and two facts fall out. When a certificate is present it calls `req.Header.Set`, which replaces any value the client sent. When no certificate is present it does nothing: it logs at debug and leaves the request alone, headers included. With `RequireAndVerifyClientCert` that second case never reaches the backend. With `VerifyClientCertIfGiven` it is every browser request, and a client without a certificate could send its own `X-Forwarded-Tls-Client-Cert-Info: Subject%3D%22CN%3Dweb01.acme%22` and have it arrive untouched. The `headers` middleware with an empty value strips a request header, so the chain blanks it on every request and `passTLSClientCert` sets it again only from a verified certificate. `X-Ssl-Client-Cn` is a header an older setup used; it costs nothing to keep blanking it.

There is a second spoofing route the chain does not cover. Go treats `X_Forwarded_Tls_Client_Cert_Info` as a different header from the dashed form, so neither middleware touches it, but a backend that normalises header names to environment-variable style reads both as the same thing. Traefik v3.7.12 added an entry point option for this, `aliasHeadersStrategy`. Its default is `keep`; `delete` drops any request header whose name contains anything but letters, digits and dashes, and the docs name `PassTLSClientCert` among the middlewares that depend on it. A Go backend does not normalise, so the chain alone closes the hole there. A backend that reads headers as CGI variables, PHP-FPM, WSGI, Rack, sees both spellings as `HTTP_X_FORWARDED_TLS_CLIENT_CERT_INFO`; for those, set it to `delete`. An nginx in front of them drops underscored header names by default (`underscores_in_headers off`), which hides the problem until someone turns that on for a legacy header.

With only `subject.commonName` enabled the decoded header is exactly `Subject="CN=web01.acme"`. The Go API parses it like this:

```go
func certnameFromHeader(v string) (string, error) {
	decoded, err := url.QueryUnescape(v)
	if err != nil {
		return "", err
	}
	parts := strings.Split(decoded, "=")
	if len(parts) < 3 {
		return "", fmt.Errorf("unexpected client cert info header: %q", v)
	}
	return strings.Trim(parts[2], `"`), nil
}
```

Splitting on `=` gives `Subject`, `"CN`, `web01.acme"`, and the third part with the quotes trimmed is the certname. That only works because exactly one field is enabled; turn on `issuer.commonName` too and the third part becomes `web01.acme";Issuer`. The header is a format you chose by what you enabled, and the parser belongs next to the middleware definition, not in another repository.

From there the API lower-cases the certname, splits it into hostname and customer id, and compares it with the certname the request is about, taken from the path or the query. Header missing, unparsable or not matching: 401. A caller already authenticated with an admin JWT skips the check, which is how the browser UI works on the same hostname. One exception is encoded in the names: a certname beginning with `puppetserver-` is a Puppet server and may act for any host of its own customer, because it is what enrols them. The comparison is where the two-label certname earns its keep: a server from customer A, presenting a perfectly valid certificate from customer A's CA and asking about a host of customer B, is refused by the second label of its own CN.

## Three clients that present it

The central Prometheus has remote-write receive enabled. Each server runs Prometheus with the `--agent` flag, which the docs describe as disabling querying, alerting and local storage in favour of a WAL that forwards to `remote_write`. The Puppet manifest points the TLS client at the host certificate through the facts:

```puppet
remote_write_configs => [{
  url        => "https://${prometheus_server}/api/v1/write",
  tls_config => {
    cert_file => $facts['hostcert'],
    key_file  => $facts['hostprivkey'],
  },
}],
```

That renders to `remote_write[].tls_config.cert_file` and `key_file`, the fields Prometheus documents as the certificate and key for client certificate authentication to the server. The agent presents the bare leaf, no chain, because that is what the hostcert file contains.

The small Go CLI on each host does the same with two environment variables Puppet writes into `/etc/default/<cli>`:

```go
cert, err := tls.LoadX509KeyPair(os.Getenv("PUPPETCERT"), os.Getenv("PUPPETPRIVKEY"))
if err != nil {
	return nil, err
}
client := &http.Client{
	Transport: &http.Transport{
		TLSClientConfig: &tls.Config{Certificates: []tls.Certificate{cert}},
	},
}
```

`LoadX509KeyPair` reads PEM files and will take intermediates after the leaf if the file has them; Puppet's does not, so this client also sends the leaf alone.

Not every client is a host. The central Alertmanager posts to a webhook behind the same kind of TLS option, with a certificate the same Puppet CA issued to a certname of its own, mounted from a Kubernetes Secret as `tls.crt` and `tls.key` and wired in through `webhook_configs[].http_config.tls_config.cert_file` and `key_file`, which Alertmanager documents in the same words as Prometheus. An identity for a service, issued by the CA that identifies the servers, is just another certname, and the tenant check treats it like one.


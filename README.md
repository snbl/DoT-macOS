# DoT-macOS
DNS over TLS Configuration for macOS via knot-resolver
Tested on Catalina 10.15.4

## Install via homebrew
Update homebrew and upgrade all installed packages.

```
brew update && brew upgrade
brew install knot-resolver
```

## Adding missing symlinks/paths
The knot-resolver installs by default to `/usr/local/etc/knot-resolver/` and is using `/usr/local/etc/knot-resolver/kresd.config` as configuration file. The knot-resolver daemon needs `/usr/local/etc/kresd/` and the default config path `/usr/local/etc/kresd/config`. By default a `var`-directory is missing to. Creating symlinks for the three sources is sufficient.

```
ln -s /usr/local/etc/knot-resolver /usr/local/etc/kresd
ln -s /usr/local/etc/knot-resolver/kresd.conf /usr/local/etc/knot-resolver/config
mkdir /usr/local/var/kresd
```

## Configure knot-resolver for TLS queries (cloudflare & google)
Open config file and change Network Interface Configuration by disabling net.listen statements for local dns-over-tls queries and ipv6 if applicable.

`vim /usr/local/etc/knot-resolver/config`

```
-- Network interface configuration
net.listen('127.0.0.1', 53, { kind = 'dns' })
-- net.listen('127.0.0.1', 853, { kind = 'tls' })
-- net.listen('::1', 53, { kind = 'dns', freebind = true })
-- net.listen('::1', 853, { kind = 'tls', freebind = true })
```

Get cloudflare & google TLS certificate chain from nameservers

`openssl s_client -showcerts -connect cloudflare-dns.com:443`

Search for "Certificate chain" and store the following ascii armored base64 certificates (-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----) in the following pem file (complete chain):

`/usr/local/etc/kresd/DigiCertGlobalRootCA.pem`

Same for the google dns servers:

`openssl s_client -showcerts -connect dns.google:443`
`/usr/local/etc/kresd/GlobalSignRootCA.pem`

### Adding knot-resolver configuration
To forward the local queries via DoT a `TLS_FORWARD` policy is necessary. The full docs are available here: https://knot-resolver.readthedocs.io/en/stable/modules-policy.html#forwarding-over-tls-protocol-dns-over-tls. Add the policy statement at the end of the config file as below.

```
-- cloudlfare server
policy.add(policy.all(policy.TLS_FORWARD({
{'1.1.1.1', hostname='cloudflare-dns.com', ca_file='/usr/local/etc/kresd/DigiCertGlobalRootCA.pem'}, 
{'1.0.0.1', hostname='cloudflare-dns.com', ca_file='/usr/local/etc/kresd/DigiCertGlobalRootCA.pem'}, 
{'2606:4700:4700::1111', hostname='cloudflare-dns.com', ca_file='/usr/local/etc/kresd/DigiCertGlobalRootCA.pem'}, 
{'2606:4700:4700::1001', hostname='cloudflare-dns.com', ca_file='/usr/local/etc/kresd/DigiCertGlobalRootCA.pem'}})))

-- google server
-- policy.add(policy.all(policy.TLS_FORWARD({
-- {'8.8.8.8', hostname='dns.google', ca_file='/usr/local/etc/kresd/GlobalSignRootCA.pem'}, 
-- {'8.8.4.4', hostname='dns.google', ca_file='/usr/local/etc/kresd/GlobalSignRootCA.pem'}, 
-- {'2001:4860:4860::8888', hostname='dns.google', ca_file='/usr/local/etc/kresd/GlobalSignRootCA.pem'}, 
-- {'2001:4860:4860::8844', hostname='dns.google', ca_file='/usr/local/etc/kresd/GlobalSignRootCA.pem'}})))
```

## knot-resolver daemon startup

Start the dameon via `brew services start|stop|restart <name>`.

`sudo brew services restart knot-resolver`

Verify output of the knot-resolver daemon (output of the log file should something like below).

`cat /usr/local/var/log/kresd.log`

```
deprecation WARNING: use --noninteractive instead of --forks=1
[system] failed to get or set file-descriptor limit: Invalid argument
```

Eventually change the dns server in Network Settings to one single entry: `127.0.0.1` and check the dns functionality. Response should looke like the following.

```
$ kdig google.com
...
;; From 127.0.0.1@53(UDP) in 7.2 ms
``

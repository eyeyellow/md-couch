## Stack deployment

Because couch re-writes `/opt/couchdb/etc/local.ini` we cannot use config for the ini files.  Have also tried using config in `local.d`, etc.

For this reason putting the `/opt/couchdb/etc/local.d` contents in a volume and attaching that.

To initialize the volume, create a `00-local.ini` with the desired config (copy of `local.ini` with updated admin password) and pre-seed the volume with the config file.

```
$ docker volume create couch_locald
$ docker container run --rm -v $(pwd):/home -v couch_locald:/locald alpine sh -c 'cp /home/00-local.ini /locald'
```

When we did this on test we did also update the ownership and permissions but I don't _think_ that's necessary since couch appears to run as root and update permissions on config at boot time.

E.g.

```
$ docker container run --rm --it -v couch_locald:/locald alpine sh
$ cd /locald
$ chown 5984:5984 00-local.ini
$ chmod 660 00-local.ini
```

For CORs to actually work where the application (`mdEditor`) is hosted on a different domain requires that login cookies have `;Secure; SameSite=None`.  You cannot explicitly configure couch to add the `Secure` flag to the cookie if couch itself is not configured to use SSL (disregards the common scenario where a reverse proxy is performing SSL termination for couch).  Because of this couch also must be configured to use SSL.

This required an additional volume be created and initialized containing the SSL specific files.

E.g.

```
$ mkdir cert
$ cd cert
$ openssl genrsa > privkey.pem
$ openssl req -new -x509 -key privkey.pem -out couchdb.pem -days 1095
$ docker volume create couch_certs
$ docker container run --rm -v $(pwd):/local -v couch_certs:/certs alpine sh -c '/local/* /certs'
```

See [3.5.2. HTTPS (SSL/TLS) Options](https://docs.couchdb.org/en/stable/config/http.html#https-ssl-tls-options).  The generation of `couchdb.pem` specifies `-days 1095` which only covers 3 years.  Assuming SSL will be terminated using a reverse proxy that will use an acceptable certificate and this configuration is just to secure the hop between the proxy and couch (and only to get secure cookies) then that number should probably be increased to avoid any obscure failures 3 years after initial deployment due to expired certificates (assuming the proxy cares).

The ownership/permissions of the cert files:

```
$ docker container run -it --rm -v $(pwd):/local -v couch_certs:/certs alpine sh
/ # ls -lh /certs/
total 8K     
-rw-------    1 5984     5984        1.4K Mar 14 18:55 couchdb.pem
-rw-------    1 5984     5984        1.6K Mar 14 18:55 privkey.pem
/ #
```

__Note:__ `5984` is the `uid/gid` of the `couchdb` user when run within the couch container.

See below for the contents of `/opt/couchdb/etc/local.d/00-local.ini` that contains all the needed configuration (except a strong password within the config).

With this configuration couch is now listening for both HTTP (`5984`) and HTTPS (`6984`) requests.  The `stack.yml` is currently exposing `5984` out of the container and probably in the end should not.  HTTP could also be disabled entirely, though not necessary.  With a reverse proxy out front it should not have a need to expose any ports.

```
; CouchDB Configuration Settings

; Custom settings should be made in this file. They will override settings
; in default.ini, but unlike changes made to default.ini, this file won't be
; overwritten on server upgrade.

[couchdb]
;max_document_size = 4294967296 ; bytes
;os_process_timeout = 5000

[couch_peruser]
; If enabled, couch_peruser ensures that a private per-user database
; exists for each document in _users. These databases are writable only
; by the corresponding user. Databases are in the following form:
; userdb-{hex encoded username}
;enable = true

; If set to true and a user is deleted, the respective database gets
; deleted as well.
;delete_dbs = true

; Set a default q value for peruser-created databases that is different from
; cluster / q
;q = 1

[chttpd]
enable_cors = true
;port = 5984
;bind_address = 127.0.0.1

; Options for the MochiWeb HTTP server.
;server_options = [{backlog, 128}, {acceptor_pool_size, 16}]

; For more socket options, consult Erlang's module 'inet' man page.
;socket_options = [{sndbuf, 262144}, {nodelay, true}]

[httpd]
; NOTE that this only configures the "backend" node-local port, not the
; "frontend" clustered port. You probably don't want to change anything in
; this section.
; Uncomment next line to trigger basic-auth popup on unauthorized requests.
;WWW-Authenticate = Basic realm="administrator"

; Uncomment next line to set the configuration modification whitelist. Only
; whitelisted values may be changed via the /_config URLs. To allow the admin
; to change this value over HTTP, remember to include {httpd,config_whitelist}
; itself. Excluding it from the list would require editing this file to update
; the whitelist.
;config_whitelist = [{httpd,config_whitelist}, {log,level}, {etc,etc}]

[ssl]
port = 6984
enable = true
cert_file = /etc/couchdb/cert/couchdb.pem
key_file = /etc/couchdb/cert/privkey.pem
;password = somepassword

; set to true to validate peer certificates
;verify_ssl_certificates = false

; Set to true to fail if the client does not send a certificate. Only used if verify_ssl_certificates is true.
;fail_if_no_peer_cert = false

; Path to file containing PEM encoded CA certificates (trusted
; certificates used for verifying a peer certificate). May be omitted if
; you do not want to verify the peer.
;cacert_file = /full/path/to/cacertf

; The verification fun (optional) if not specified, the default
; verification fun will be used.
;verify_fun = {Module, VerifyFun}

; maximum peer certificate depth
;ssl_certificate_max_depth = 1

; Reject renegotiations that do not live up to RFC 5746.
;secure_renegotiate = true

; The cipher suites that should be supported.
; Can be specified in erlang format "{ecdhe_ecdsa,aes_128_cbc,sha256}"
; or in OpenSSL format "ECDHE-ECDSA-AES128-SHA256".
;ciphers = ["ECDHE-ECDSA-AES128-SHA256", "ECDHE-ECDSA-AES128-SHA"]

; The SSL/TLS versions to support
;tls_versions = [tlsv1, 'tlsv1.1', 'tlsv1.2']

; To enable Virtual Hosts in CouchDB, add a vhost = path directive. All requests to
; the Virtual Host will be redirected to the path. In the example below all requests
; to http://example.com/ are redirected to /database.
; If you run CouchDB on a specific port, include the port number in the vhost:
; example.com:5984 = /database
[vhosts]
;example.com = /database/

; To create an admin account uncomment the '[admins]' section below and add a
; line in the format 'username = password'. When you next start CouchDB, it
; will change the password to a hash (so that your passwords don't linger
; around in plain-text files). You can add more admin accounts with more
; 'username = password' lines. Don't forget to restart CouchDB after
; changing this.
[admins]
admin = super_strong_password_goes_here

[chttpd_auth]
same_site = none


[cors]
origins = *
credentials = true
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE

[couch_httpd_auth]
cookie_domain = couch.djcase.com
```
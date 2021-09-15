Apache with LDAP example
===

This project represents a simple example of how to expose a service on Apache using LDAP.

These instructions were tested on MacOS. The app behind Apache is just a demo server that dumps all HTTP headers on the console. The app-specific configuration is contained in fil `httpd-ssl.conf`:

```
<Location /app>
  ProxyPass "http://host.docker.internal:9176"
  ProxyPassReverse "http://host.docker.internal:9176"
</Location>
```

You can replace it by pointing at your app.

## Creating the server private key and SSL certificate

Create the private key `localhost.key` and the self-signed SSL certificate `localhost.crt` using
[mkcert](https://github.com/FiloSottile/mkcert):
```
mkcert --install
mkcert localhost
openssl x509 -outform der -in localhost.pem -out localhost.crt
openssl pkey -in localhost-key.pem -out localhost.key
```

## Installing the LDAP server

A user has been configured in file `foo.ldif` with credentials "foo" / "bar". The password has been generated as follows:

```
$ slappasswd -h {SSHA} -s bar
{SSHA}49K6gZdMQQ5vqz7ScDtwZ50/CKG2Eeug
```

Create the LDAP server as follows:

```
docker rm -f ldap
docker run --name ldap -p 389:389 -p 636:636 --detach osixia/openldap:1.5.0 
```

Finally, add the user:
```
docker cp foo.ldif ldap:/container/service/slapd/assets/test/
docker exec ldap ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin -f /container/service/slapd/assets/test/foo.ldif -H ldap://localhost -ZZ
```

This command should return the "foo" user:

```
docker exec ldap ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin "(cn=foo)"
```

## Setting up Apache with HTTPS

```
docker rm -f apache-ldap 
docker run -dit --name apache-ldap -p 443:443 httpd:2.4
docker cp httpd.conf apache-ldap:/usr/local/apache2/conf
docker cp httpd-ssl.conf apache-ldap:/usr/local/apache2/conf/extra
docker cp localhost.pem apache-ldap:/usr/local/apache2/conf/server.crt
docker cp localhost.key apache-ldap:/usr/local/apache2/conf/server.key
docker exec apache-ldap apachectl restart
```

## Installing the dummy app

Issue the following command:

```
nc -l -p 9176
```

Or this is you have the BSD version of Netcat:

```
nc -l 9176
```

If you now navigate to the app at <https://localhost/app> the page will prompt for credentials, enter "foo" / "bar".
The page will then stay there hanging, but on the console you can see all HTTP headers coming through in clear text
(i.e., unencrypted), which means that the Apache HTTPS proxy is working and that LDAP authentication was successful:

```
GET / HTTP/1.1
Host: host.docker.internal:9176
Cache-Control: max-age=0
Authorization: Basic Zm9vOmJhcg==
sec-ch-ua: "Google Chrome";v="93", " Not;A Brand";v="99", "Chromium";v="93"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "macOS"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.82 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9,it;q=0.8,pl;q=0.7
username: foo
X-Forwarded-For: 172.17.0.1
X-Forwarded-Host: localhost
X-Forwarded-Server: www.example.com
Connection: Keep-Alive
```

Most importantly:

* remote_user: foo
* uid: foo
* mail: foo@example.org
* gid: 14564100

## References

* <https://devcenter.heroku.com/articles/ssl-certificate-self>
* <https://hub.docker.com/_/httpd>
* <https://docs.docker.com/docker-for-mac/networking/> (`host.docker.internal`)
* <https://github.com/osixia/docker-openldap>
* <https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html>
* <https://ldapwiki.com/wiki/Apache%20Web%20Server%20and%20LDAP>
* <https://www.openldap.org/faq/data/cache/347.html>

(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$ openssl genrsa -out ca-key.pem 4096
(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$ openssl req -new -x509 -days 365 -key ca-key.pem -out ca-cert.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

---

Country Name (2 letter code) [AU]:NG
State or Province Name (full name) [Some-State]:.
Locality Name (eg, city) []:.
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Webhook
Organizational Unit Name (eg, section) []:section
Common Name (e.g. server FQDN or YOUR name) []:Nockk
Email Address []:ob.ogheneochuko@gmail.com
(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$ openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem -out server.csr
openssl x509 -req -days 365 -in server.csr \
 -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
 -out server-cert.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

---

Country Name (2 letter code) [AU]:NG
State or Province Name (full name) [Some-State]:.
Locality Name (eg, city) []:.
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Webhook
Organizational Unit Name (eg, section) []:section
Common Name (e.g. server FQDN or YOUR name) []:Nockk
Email Address []:ob.ogheneochuko@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:NOLIMITS4ME
An optional company name []:Webhook
Certificate request self-signature ok
subject=C=NG, O=Webhook, OU=section, CN=Nockk, emailAddress=ob.ogheneochuko@gmail.com
(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$ openssl genrsa -out client-key.pem 4096
openssl req -new -key client-key.pem -out client.csr
openssl x509 -req -days 365 -in client.csr \
 -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
 -out client-cert.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

---

Country Name (2 letter code) [AU]:NG
State or Province Name (full name) [Some-State]:.
Locality Name (eg, city) []:.
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Webhook
Organizational Unit Name (eg, section) []:section
Common Name (e.g. server FQDN or YOUR name) []:Nockk
Email Address []:ob.ogheneochuko@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:NOLIMITS4ME
An optional company name []:Webhook
Certificate request self-signature ok
subject=C=NG, O=Webhook, OU=section, CN=Nockk, emailAddress=ob.ogheneochuko@gmail.com
(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$ ^C
(backend) nockk@DESKTOP-8B1UC26:~/portfolio/webhook$

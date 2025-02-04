# Kickstart, encryption and or tokens as secure as possible ?

A recent discussion from some colleagues of mine brought up the following scenario. Imagine to have a fully working Kickstart installation process. At the end of that process, you want to integrate your system into your Ansible Automation Platform. To achieve the event of kicking of the automation, one needs to have API access. That token is the main focus of the discussion on this article.

We did understand that there might be even more suiteable ways to handle that situation, we focused on least inversive change to the process and came up with three steps that can be taken.


## Scenario 1 moving a shared file system plain-text token out of the game
As infrastructures grow, the currently implementation was utilizing a shared file system with a file containing the token to execute API actions in the AAP controller. We all did something similar in the past so please no finger pointing here.

The least disruptive way and considered as only the first step on the journey is to get rid of plain-text sensitive data. 

With the kickstart boot process itself, we are limited in what can be done easily without modifying too much of the process itself. Luckily we are in the **post** section of the kickstart process which also provides functionality like calling scripts or other binaries.

[TANG](https://github.com/latchset/tang) a service for binding data to network presence, is often used in the context of [LUKS and automatic disk decryption](https://github.com/latchset/clevis?tab=readme-ov-file#clevis-configuration) as long as the device is in the segment it should be.

And as TANG and clevis can do on-the-fly en/decryption it can also be utilized for protecting sensitive data like in our scenario. That said, the goal of Scenario 1 is to replace the shared file system file content with an encrypted content. That content is retrieved and piped through the clevis tool for being decrypted without the need to move a preshared secret through the network.

## Scenario 2 extending scenario 1 to restrict who can retrieve the encrypted content 
With Scenario 1 in place, we already improved the initial process by removing plain-text from the game. Now we add restrictions on who can fetch the secret by moving the content from a shared file system into a mTLS (mutual TLS) service meaning, a client needs to present a valid certificate to authenticate to the service for retrieving the secret.

## Scenario 3 extending scenario 2 by restricting clevis communication to be mTLS protected too
Since clevis and jose cannot handle mTLS out-of-the-box (RFE in progress)  we will be utilizing stunnel in addition which will also make secrets scenario aware as the TANG url is included in the secret meaning, you cannot decrypt it if it was encrypted through localhost and stunnel.



# Hands on now 
The POC expects that you can modify one System/VM with privileges. We will be using playbooks to ensure we have the automation  for the kickstart process in mind.

The Playbook is tailored for using an external Vault to store the CA passphrase (my personal compliance border). If you don't have on at hand, just ensure to specify the same variable at the playbook run over and over.

There are two tags in the playbook:

* setup
* client

were **setup** is only needed once to ensure packages and directories as well as the CA is deployed. Any additional client you want to issue a Certificate for you just need the client tags to be executed.

The Client certificates are set to expire within **one hour** which should be fitting the average duration of your kickstart process of course. Furthermore, re-executing the playbook with the same client name will wipe the existing certificate and key but will not revoke the Certificate from the CA as the expiry is expected to happen quick.

Adjust you ansible inventory to have a group called tangserver which connects to the system of your choice 

```
# I am using a AAP deployed OCP-V VM called node1 as TANG server
tangserver:
  hosts:
    node1:
      ansible_host: vm/node1.default
```

this notation for OCP-V VM's expects you to have a ssh_config section doing a **proxycommand on virtctl** accordingly

Next we are going to execute the playbook in **setup** mode to deploy the TANG service, HTTPS service as well as the Root CA. (Remember to use the same **secret_ca_passphrase** for all run's)

Second, the **random_passphrase** is only used when setting a passphrase to the TANG server certificate.

```
ansible-playbook tang.yml \
  --tags setup \
  --extra-vars '{"secret_ca_passphrase":"806300f7-ddbb-4bfe-9418-c08f37ec4e93", "random_passphrase": "9418-c08f37ec4e93"}'
```

Unless you are using a Vault system the playbook run will have some errors shown which can be ignored.

We now should have a working TANG and HTTPS service which will refuse to accept a connection without showing a client certificate.

```
curl --cacert /etc/mtls/ca.crt \
    --resolve protected.tang.example.com:443:10.0.2.2 \
    https://protected.tang.example.com
curl: (56) OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
```

So far so good but we want to also create a Client and we will use that client to create our secret. The certificate will have a **subject** of **"cn=My First Client"**

```
ansible-playbook tang.yml \
   --tags client \
   --extra-vars '{"secret_ca_passphrase":"806300f7-ddbb-4bfe-9418-c08f37ec4e93","random_passphrase":"9418-c08f37ec4e93","client": "My First Client"}'
```

and the playbook has already started the stunnel for us. If we point our curl towards the stunnel  with schema **http** on port **443** we will see a **403** returned instead of a SSL connection error.

```
curl http://protected.tang.example.com:443 -w '%{http_code}\n' -o /dev/null -s
403
```

Now let's create our AAP Token (fake) that we want to be available in encrypted format for Clients.

```
uuidgen | clevis encrypt tang '{"url":"http://127.0.0.1:443"}' -y  > /var/www/html/protected/aap.token
```

Inspecting the content of the **aap.token** file to ensure everything worked out as expected (below is an example as the content will differ from your TANG key being randomly generated)

```
cat /var/www/html/protected/aap.token
eyJ...Paw
```

Let's see if we can decrypt the Token in the setup of **mTLS** through **stunnel** (again the example token will differ as we are using uuidgen to fake some data)

```
curl -s http://protected.tang.example.com:443/protected/aap.token | clevis decrypt
5d85766c-1f1c-43af-b468-63a33135b3f5
```

Cool now let's see what happens when we drop the stunnel connection

```
systemctl stop stunnel@tang
curl -s http://protected.tang.example.com:443/protected/aap.token | clevis decrypt
```

There's no output returned which is expected. If we break down the steps we will see that right now, there's no service Listening on port **443**.

```
curl -s http://protected.tang.example.com:443/protected/aap.token -v
*   Trying 127.0.0.2:443...
* connect to 127.0.0.2 port 443 failed: Connection refused
* Failed to connect to protected.tang.example.com port 443: Connection refused
```

Okay that might fail already script kiddies but we want to ensure it doesn't stop there. So let's use the **public IP** of the **HTTPS** service for TANG.

```
curl --resolve protected.tang.example.com:443:10.0.2.2 \
  -s https://protected.tang.example.com:443/protected/aap.token \
  --cacert /etc/mtls/ca.crt -v
* Added protected.tang.example.com:443:10.0.2.2 to DNS cache
* Hostname protected.tang.example.com was found in DNS cache
*   Trying 10.0.2.2:443...
* Connected to protected.tang.example.com (10.0.2.2) port 443 (#0)
...
* TLSv1.2 (IN), TLS header, Unknown (23):
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
* Closing connection 0
```

**mTLS** kicks in and therefor clevis would receive and **invalid** encrypted content and outputs nothing as well.

Now further we assume, that client Certificate has been lost to the wild and an external attacker want to **decrypt** our secret by accessing the **public IP** of  our HTTPS TANG service.

```
curl --resolve protected.tang.example.com:443:10.0.2.2 \
  -s https://protected.tang.example.com:443/protected/aap.token \
  --cacert /etc/mtls/ca.crt \
  --cert /etc/mtls/certificates/My\ First\ Client.crt \
  --key /etc/mtls/private/My\ First\ Client.key | \
clevis decrypt
Error communicating with server http://127.0.0.1:443
```

Even though we are able to retreive the encrypted secret

```
curl --resolve protected.tang.example.com:443:10.0.2.2 \
  -s https://protected.tang.example.com:443/protected/aap.token \
  --cacert /etc/mtls/ca.crt \
  --cert /etc/mtls/certificates/My\ First\ Client.crt \
  --key /etc/mtls/private/My\ First\ Client.key
eyJ...Paw
```

TANG refused to decrpyt it as we did encrypt it using our setup with stunnel.

I hope these use-cases are as fun for you to exercise through as they are for me, and fingers crossed that you can get at least to **use-case 1** improving the overall security if you are not already at **use-case 3**.

Enjoy

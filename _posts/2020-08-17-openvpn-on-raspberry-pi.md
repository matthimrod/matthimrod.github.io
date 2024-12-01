---
layout: post
title: OpenVPN On Raspberry Pi
subtitle: Setting up a VPN for personal use
tags: [vpn, howto]
author: Matt Himrod
---

#### Setting up a VPN for personal use

So you want to run an OpenVPN Server on a Raspberry Pi? I did too, but it wasn't easy to figure out the first time. If you want to skip all of this and do it the easy way, head over to https://pivpn.io/ for the one-line script. If you want to dive in, understand what's going on, and configure it your own way, keep reading.

## OpenVPN Server
First, I'm going to assume that you are already pretty comfortable at a Linux shell, you understand basic networking, and you have a Raspberry Pi with the OS image already on it. As a convention, commands that you need to type will be in fixed-width text, and the prompt that you should see will appear at the beginning of the line.

First, it's always a good idea to make sure your Raspberry Pi is up-to-date. Since I'm going to be doing a lot of things that require root, the first thing I'm going to do is elevate to root and do the updates.

```
pi@raspberrypi:~ $ sudo su -
root@raspberrypi:~ # apt-get update
root@raspberrypi:~ # apt-get upgrade
```

I'll be implementing certificate-based authentication with TLS encryption. The next step is to install OpenVPN and Easy-RSA: 

```
root@raspberrypi:~ # apt-get install openvpn easy-rsa
```

Next, we need to put some sample configuration files in place.

```
root@raspberrypi:~ # cp -R /usr/share/doc/openvpn /etc/openvpn
root@raspberrypi:~ # cp -R /usr/share/doc/openvpn/examples/sample-config-files /etc/openvpn
root@raspberrypi:~ # cp -R /usr/share/easy-rsa /etc/openvpn/
```

We need to use Easy-RSA to generate our certificates. Let's go into the Easy-RSA directory and get started by initializing Easy-RSA and generating our Certificate Authority. This is essentially the "root" of our VPN's trust. Note that I'm using "nopass" to build the certificates, but if you prefer to add a passphrase if you prefer by leaving that off.

```
root@raspberrypi:~ # cd /etc/openvpn/easy-rsa/
root@raspberrypi:/etc/openvpn/easy-rsa # ./easyrsa init-pki
root@raspberrypi:/etc/openvpn/easy-rsa # ./easyrsa build-ca nopass
```

Now we need to generate the key for the server itself. Note that you'll want to substitute the name of your server for server_name in the command.

```
root@raspberrypi:/etc/openvpn/easy-rsa # ./easyrsa build-server-full server_name nopass
```

Next, we need to generate the Diffie Hellman parameters. This one will take a while, so go make some coffee or get a snack.

```
root@raspberrypi:/etc/openvpn/easy-rsa # ./easyrsa gen-dh
```

For extra security, we should enable tls-auth, which requires that we generate another key that's shared among the server and all clients.

```
root@raspberrypi:/etc/openvpn/easy-rsa # cd /etc/openvpn
root@raspberrypi:/etc/openvpn # openvpn --genkey --secret ta.key
```

Ok. Now. The fun part. We get to set up the server! Let's start with a sample file.

```
root@raspberrypi:/etc/openvpn # cp sample-config-files/server.conf.gz .
root@raspberrypi:/etc/openvpn # gunzip server.conf.gz
```

Now to edit the configuration file:

```
root@raspberrypi:/etc/openvpn # nano server.conf
```

I leave a lot of the defaults in place such as the port, protocol, and a few others. There are a few options that are of specific importance. I'll go through the sections that I typically modify here. Note that there are two comment styles in the sample file. Documentation comments that begin with a hash sign (#) and comments denoting default or recommended values that begin with a seimicolon (;).

First, you need to add the CA Certificate, Server Certificate, Server Key, and Diffe Hellman parameters. I prefer to embed my certificates and keys in this file, especially on the clients. You do this by enclosing the text of the certificates in XML-like tags. You can also just reference the filenames as in the sample, but you'll have to add the full path. The embedded versions look something like this -- note that I've shortened the certificate text because this is just an illustration:

### Contents of /etc/openvpn/easy-rsa/pki/ca.crt
```
<ca>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUNP2LEVnQuakBRx1Z/N2wdoGu/IwwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLTWF0dCBIaW1yb2QwHhcNMjAwODE3MjMwMTM0WhcNMzAw
ODE1MjMwMTM0WjAWMRQwEgYDVQQDDAtNYXR0IEhpbXJvZDCCASIwDQYJKoZIhvcN
MMGGNlm1aUGrTuLv8XHAswt7bLR8CbTXVG6p1A70UfJLlKNc2Rq5+EFi8CgtMfmL
NT0nkLJETkJi2nbreqvxe4GNUzAq0YMcM73ohNOeQHuVB2NHScFLGOal/42foYbm
C+upRglCpJSxI2bgAJT5Kg5W5fZNhHWGii4uikAZjQ==
-----END CERTIFICATE-----
</ca>
```

### Contents of /etc/openvpn/easy-rsa/pki/issued/server_name.crt
<cert>
-----BEGIN CERTIFICATE-----
MIIDcjCCAlqgAwIBAgIRAM0/vSMPiMAtBpmmTj4FzC0wDQYJKoZIhvcNAQELBQAw
MDA1NzU3WjAWMRQwEgYDVQQDDAtzZXJ2ZXJfbmFtZTCCASIwDQYJKoZIhvcNAQEB
Hd4Lm1wKkNsVZBIIi8uozzeb+LIb5/YLV8tReYQM0u7JYe/Goo19tNz2Abvioei+
bNlvo6CNU+X5FWG7n0xTCymZ1/CqHKFagaB88hW+pm64DguxaevMyRReyUmUsCoq
3L4UUFScma1efXne2He8/e2I/ZTTKqtd1om7h4mhg5X7zwT5qLLuV2Vqm0ETFrRX
j61GdBwnT+8ES+eIgDsjsK7Bxk5TWQ==
-----END CERTIFICATE-----
</cert>

### Contents of /etc/openvpn/easy-rsa/pki/private/server_name.key
* Note that this file should be kept secret as it's the server's cryptographic private key.
```
<key>
-----BEGIN PRIVATE KEY-----
MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQCtF8UC2vSezBUt
/+MXiojPpwKkGSJSDbBqtGweAz4KHkvq/RSlVDTFG4n0ZxGrtsdcJwZ2W8eDN6xi
Vtr1j8DsTcERXJtx2trCcscPHCBVizXM/kxl/LTXHQw0fZt5yFK2wEtWJyhTPWFV
CjqqRbSadISPmdagyh4JXc4uFKjp2OJyKFSYJO7YgmMSM/6j6DLaXIzfmmRceki2
xLrF0JN2G7eYebTagTAPRPa0te4IPJNvpNbHdcie4piGVGcMyDA4obZUe+E/Ai49
zG63tWy0AMZEJdRWP7fomw==
-----END PRIVATE KEY-----
</key>
```

### Contents of /etc/openvpn/easy-rsa/pki/dh.pem
```
<dh>
-----BEGIN DH PARAMETERS-----
MIIBCAKCAQEA9hzVwmsqCWo9DQCZjvoyI8uOio8Rs5XgUOViIhRW8A20nKTTUYGz
+q75L9IOvhwwRVVFFy4c3eJ/W3kdXdHu9b+NBaVy6eFY+zMDP9LOU+L3pvf9U64K
H6NClVURxMR1SFVnA8WSh2UdUtNoH3NczYTD+0UibkPZJsrTR4IYmuKUs2JW9F1o
P6at3CtWN6jZNiYyTiJMrkT9yQ48jvQeewIBAg==
-----END DH PARAMETERS-----
</dh>
```
You'll next need to set the network topology. You should just remove the semicolon from the line that contains topology subnet.

Next, you'll define the VPN subnet. The way OpenVPN works is that it creates a network adapter to communicate with the clients. On the server side, you can route this network to the internet or to the server's local network (which we will cover a little later). You should be cautious about picking networks that are common default networks. For example. many routers use 192.168.0.1 or 192.168.1.1. The default here is 10.8.0.0, but I prefer to change this, so I'll use 172.30.0.1. Clients will be assigned addresses on this network.

```
server 172.30.0.0 255.255.255.0
```

Here's where the VPN starts becoming more useful. You will need to define the network on the server-side of the VPN so that the clients can route to it. My home router uses 10.0.0.0/24, so we'll push that to the clients. 

```
push "route 10.0.0.0 255.255.255.0"
```

Note that it's a good idea to avoid the common router networks with your local network. You don't want to connect from a friend's house and lose your ability to connect to the local network because of a routing conflict. You also can't have this be the same as the VPN subnet for the type of VPN we're configuring here.

The last thing we need to do is add the tls-auth key. As with the other crypto files, we can either embed this with XML-like tags or reference the file. Note the "key-direction" directive, which is 0 on the server and 1 on the clients.

```
key-direction 0
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
f0220e1a40956ca471ea792434341820
9296b0659c44e464d7e35aab7729c2c8
3bb54840ce4ddfc37330466cb0f8a992
09619759a45175f6a5088fad2cdb254d
2dbc992e4460199bf2a4b228bbbdc2e9
4273b0031e73abe8b92d97e547521d8c
-----END OpenVPN Static key V1-----
</tls-auth>
```

Now you need to make your Raspberry Pi into a router. Edit sysctl.conf with the following command:

```
root@raspberrypi:/etc/openvpn # nano /etc/sysctl.conf
```

You'll want to uncomment the following lines by removing the hash sign (#) from the start of each line:

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
```

Now reload sysctl.

```
root@raspberrypi:/etc/openvpn # sysctl -p /etc/sysctl.conf
```

We're almost done. We need to enable NAT in the built-in iptables firewall. This will allow our VPN to access the Internet through our local Internet connection should that be necessary. If you're using your VPN to create a secure connection from an untrusted network (such as public WiFi) then this is especially important. You'll want to both run this from your shell and add it to the /etc/rc.local script so that it's run every time your Pi boots.

```
root@raspberrypi:/etc/openvpn # iptables --table nat --append POSTROUTING --jump MASQUERADE
```

And of course, to edit rc.local:

```
root@raspberrypi:/etc/openvpn # nano /etc/rc.local
```

Paste this at the end of the file:

```
iptables --table nat --append POSTROUTING --jump MASQUERADE
```

Once you've set everything, you can start your OpenVPN server with systemctl.

```
root@raspberrypi:/etc/openvpn # systemctl start openvpn
```

Congratulations, you've set up the server! There's one last thing that you need to do to make this useful, and that's to set a port forwarding rule in your router. The way to do this depends on your specific router model. Unless you changed it in your server configuration file, you'll need to forward UDP port 1194 to the same port on your Raspberry Pi. 

## OpenVPN Client

What's next? We need to set up clients. I'll do another post later about how to set up a Raspberry Pi client for a site-to-site setup. It's not a lot different than setting up a regular client, but you have to do a few things with routing.

I'll assume you're using the latest OpenVPN application on the client. You can get this for most platforms including Android and iOS. We'll need to create a .ovpn file, which has the same structure as the sample client.conf that you'll find in /etc/openvpn/sample-config-files/, but you'll need to embed the crypto files.

First, we need to generate a certificate and key for the client similarly to how generated the server certificate and key. Note that you'll want to substitute the name of your client for client_name in the command. You'll want to do this for each client.

```
root@raspberrypi:~ # cd /etc/openvpn/easy-rsa/
root@raspberrypi:/etc/openvpn/easy-rsa # ./easyrsa build-client-full client_name nopass
```

You'll want to make a copy of the client config file for the client, and I recommend naming it with a .ovpn extension, especially for Windows and mobile clients. Most of the file stays as-is except for the things that are specific to the server. The first is the remote line, which tells the client where to connect to. You'll need your router's public IP address or a dynamic DNS name if you don't have a static IP address. In this example, the server's public IP address is 12.34.56.78 and the port number is the default port of 1194.

```
remote 12.34.56.78 1194
```

You'll use the same CA certificate and tls-auth key as the server, but you'll use the newly generated client certificate and key for their respective sections.

### Contents of /etc/openvpn/easy-rsa/pki/ca.crt
```
<ca>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUNP2LEVnQuakBRx1Z/N2wdoGu/IwwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLTWF0dCBIaW1yb2QwHhcNMjAwODE3MjMwMTM0WhcNMzAw
ODE1MjMwMTM0WjAWMRQwEgYDVQQDDAtNYXR0IEhpbXJvZDCCASIwDQYJKoZIhvcN
MMGGNlm1aUGrTuLv8XHAswt7bLR8CbTXVG6p1A70UfJLlKNc2Rq5+EFi8CgtMfmL
NT0nkLJETkJi2nbreqvxe4GNUzAq0YMcM73ohNOeQHuVB2NHScFLGOal/42foYbm
C+upRglCpJSxI2bgAJT5Kg5W5fZNhHWGii4uikAZjQ==
-----END CERTIFICATE-----
</ca>
```

### Contents of /etc/openvpn/easy-rsa/pki/issued/client_name.crt
```
<cert>
-----BEGIN CERTIFICATE-----
MIIDWjCCAkKgAwIBAgIRAITbzc/KlaioTvFDnNNT6b0wDQYJKoZIhvcNAQELBQAw
FjEUMBIGA1UEAwwLTWF0dCBIaW1yb2QwHhcNMjAwODE4MDE0NzQzWhcNMjMwODAz
MDE0NzQzWjAWMRQwEgYDVQQDDAtjbGllbnRfbmFtZTCCASIwDQYJKoZIhvcNAQEB
hj868jnqpWwk5N/xitJqZwnfX6zNQzCNJ6vKxShKjWPm2b9w40XfhLF9gzoTfgEA
OZePGTIAdVMNj2PhBua6SSxt1GmFwplb3msaGkAXjAgj0Ev3WpFh5yetiJZKjkv8
WPBXlR0PAQS1dpoSrF+ev3SO3YfarMX+CthRDeX8pPF+69FA4a4fBu6R8GQuyw==
-----END CERTIFICATE-----
</cert>
```

### Contents of /etc/openvpn/easy-rsa/pki/private/client_name.key
* Note that this file should be kept secret as it's the client's cryptographic private key.
```
<key>
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDWAEBJN2DPEMt4
3+xsExh+RY7upe0MMC8GmOErepiWnjt9adY+IhcvdJq2AYJWt0Fb0kYENtz/9BXT
z5+5NMYP++mhITuWdQSqIWqCjZOgvcbXcxWsW+o68EBznSvh6LW2d4drKKBDJ6p6
nuff+WxIHik1mXQPLijYOXEDetXpKHYzOJJJ5ln+jc5GHMyBlGGAsUzJVumEg4F9
nDocHy4l6Rvf9GkneAbt2KW6nNxIcHukd0hG6W5QEBUmLwoTJAIBl9SH0AyJpxA3
9EcBGJsYYkZAd9okEl9yxis=
-----END PRIVATE KEY-----
</key>
```

A few lines down, add the tls-auth key. This is the same key as was on the server, but the "key-direction" is 1 on the client.

```
key-direction 1
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
f0220e1a40956ca471ea792434341820
9296b0659c44e464d7e35aab7729c2c8
3bb54840ce4ddfc37330466cb0f8a992
09619759a45175f6a5088fad2cdb254d
2dbc992e4460199bf2a4b228bbbdc2e9
4273b0031e73abe8b92d97e547521d8c
-----END OpenVPN Static key V1-----
</tls-auth>
```

Lastly, there's one more thing that you may want to add to the config file. Personally, I make two versions -- one for when I just want to access my home network and a second for when I'm on an untrusted network like a coffee shop WiFi and I want to use the VPN as a layer of security. For the "full tunnel" version -- that is, the one where you want all traffic to go over the VPN -- add the following line to the end of the file.

```
redirect-gateway def1 bypass-dhcp
```

That's it! Save your config file and securely transfer it to your client. Import it into your OpenVPN client. You will have to test this from a different network than your Raspberry Pi is on, but you should be able to access resources on that network, and if you added the redirect-gateway line to your config file, you should see that you're accessing the Internet from your Raspberry Pi's public IP address. 
# How-To: IKEv2 VPN for Windows, macOS, Android and iOS

*Read this in other languages: [English](ikev2-howto.md), [简体中文](ikev2-howto-zh.md).*

---

**Important:** This guide is for **advanced users** only. Other users please use <a href="clients.md" target="_blank">IPsec/L2TP</a> or <a href="clients-xauth.md" target="_blank">IPsec/XAuth</a>.

---

Windows 7 and newer releases support the IKEv2 standard through Microsoft's Agile VPN functionality. Internet Key Exchange (IKE or IKEv2) is the protocol used to set up a Security Association (SA) in the IPsec protocol suite. Compared to IKE version 1, IKEv2 contains <a href="https://en.wikipedia.org/wiki/Internet_Key_Exchange#Improvements_with_IKEv2" target="_blank">improvements</a> such as Standard Mobility support through MOBIKE, and improved reliability.

Libreswan can authenticate IKEv2 clients on the basis of X.509 Machine Certificates using RSA signatures. This method does not require an IPsec PSK, username or password. It can be used with:

- Windows 7, 8.x and 10
- OS X (macOS)
- Android 4.x and newer (using the strongSwan VPN client)
- iOS (iPhone/iPad)

The following example shows how to configure IKEv2 with Libreswan. Commands below must be run as `root`.

Before continuing, make sure you have successfully <a href="https://github.com/hwdsl2/setup-ipsec-vpn" target="_blank">set up your VPN server</a>, and upgraded Libreswan <a href="https://github.com/hwdsl2/setup-ipsec-vpn#upgrade-libreswan" target="_blank">to the latest version</a>.

1. Find the VPN server's public IP, save it to a variable and check.

   ```bash
   $ PUBLIC_IP=$(wget -t 3 -T 15 -qO- http://ipv4.icanhazip.com)
   $ printf '%s\n' "$PUBLIC_IP"
   (Check the displayed public IP)
   ```

   **Note:** Alternatively, you may specify the server's DNS name here. e.g. `PUBLIC_IP=myvpn.example.com`.

1. Add a new IKEv2 connection to `/etc/ipsec.conf`:

   ```bash
   $ cat >> /etc/ipsec.conf <<EOF

   conn ikev2-cp
     left=%defaultroute
     leftcert=$PUBLIC_IP
     leftid=@$PUBLIC_IP
     leftsendcert=always
     leftsubnet=0.0.0.0/0
     leftrsasigkey=%cert
     right=%any
     rightid=%fromcert
     rightaddresspool=192.168.43.10-192.168.43.250
     rightca=%same
     rightrsasigkey=%cert
     narrowing=yes
     dpddelay=30
     dpdtimeout=120
     dpdaction=clear
     auto=add
     ikev2=insist
     rekey=no
     pfs=no
     ike-frag=yes
     ike=aes256-sha2,aes128-sha2,aes256-sha1,aes128-sha1,aes256-sha2;modp1024,aes128-sha1;modp1024
     phase2alg=aes_gcm-null,aes256-sha2,aes128-sha2,aes256-sha1,aes128-sha1
   EOF
   ```

   We need to add a few more lines to that file. First check your Libreswan version, then run one of the following commands:

   ```bash
   $ ipsec --version
   ```

   For Libreswan 3.23 and newer:

   ```bash
   $ cat >> /etc/ipsec.conf <<EOF
     modecfgdns="8.8.8.8, 8.8.4.4"
     encapsulation=yes
     mobike=no
   EOF
   ```

   **Note:** If your server runs Debian or CentOS and you wish to enable MOBIKE support, replace `mobike=no` with `mobike=yes` in the command above. DO NOT enable this option on Ubuntu systems.

   For Libreswan 3.19-3.22:

   ```bash
   $ cat >> /etc/ipsec.conf <<EOF
     modecfgdns1=8.8.8.8
     modecfgdns2=8.8.4.4
     encapsulation=yes
   EOF
   ```

   For Libreswan 3.18 and older:

   ```bash
   $ cat >> /etc/ipsec.conf <<EOF
     modecfgdns1=8.8.8.8
     modecfgdns2=8.8.4.4
     forceencaps=yes
   EOF
   ```

1. Generate Certificate Authority (CA) and VPN server certificates:

   **Note:** Specify the certificate validity period (in months) with "-v". e.g. "-v 36".

   ```bash
   $ certutil -z <(head -c 1024 /dev/urandom) \
     -S -x -n "IKEv2 VPN CA" \
     -s "O=IKEv2 VPN,CN=IKEv2 VPN CA" \
     -k rsa -g 4096 -v 36 \
     -d sql:/etc/ipsec.d -t "CT,," -2
   ```

   ```
   Generating key.  This may take a few moments...

   Is this a CA certificate [y/N]?
   y
   Enter the path length constraint, enter to skip [<0 for unlimited path]: >
   Is this a critical extension [y/N]?
   N
   ```

   **Note:** If you specified the server's DNS name (instead of its IP address) in step 1 above, you must replace `--extSAN "ip:$PUBLIC_IP,dns:$PUBLIC_IP"` in the command below with `--extSAN "dns:$PUBLIC_IP"`.

   ```bash
   $ certutil -z <(head -c 1024 /dev/urandom) \
     -S -c "IKEv2 VPN CA" -n "$PUBLIC_IP" \
     -s "O=IKEv2 VPN,CN=$PUBLIC_IP" \
     -k rsa -g 4096 -v 36 \
     -d sql:/etc/ipsec.d -t ",," \
     --keyUsage digitalSignature,keyEncipherment \
     --extKeyUsage serverAuth \
     --extSAN "ip:$PUBLIC_IP,dns:$PUBLIC_IP"
   ```

   ```
   Generating key.  This may take a few moments...
   ```

1. Generate client certificate(s), then export the `.p12` file that contains the client certificate, private key, and CA certificate:

   ```bash
   $ certutil -z <(head -c 1024 /dev/urandom) \
     -S -c "IKEv2 VPN CA" -n "vpnclient" \
     -s "O=IKEv2 VPN,CN=vpnclient" \
     -k rsa -g 4096 -v 36 \
     -d sql:/etc/ipsec.d -t ",," \
     --keyUsage digitalSignature,keyEncipherment \
     --extKeyUsage serverAuth,clientAuth -8 "vpnclient"
   ```

   ```
   Generating key.  This may take a few moments...
   ```

   ```bash
   $ pk12util -o vpnclient.p12 -n "vpnclient" -d sql:/etc/ipsec.d
   ```

   ```
   Enter password for PKCS12 file:
   Re-enter password:
   pk12util: PKCS12 EXPORT SUCCESSFUL
   ```

   Enter a secure password to protect the exported `.p12` file (when importing into an iOS device, this password cannot be empty). Repeat this step to generate certificates for additional VPN clients. Replace every `vpnclient` with `vpnclient2`, etc.

   **Note:** To connect multiple VPN clients simultaneously, you must generate a unique certificate for each.

1. (For macOS and iOS clients) Export the CA certificate as `vpnca.cer`:

   ```bash
   $ certutil -L -d sql:/etc/ipsec.d -n "IKEv2 VPN CA" -a -o vpnca.cer
   ```

1. The database should now contain:

   ```bash
   $ certutil -L -d sql:/etc/ipsec.d
   ```

   ```
   Certificate Nickname                               Trust Attributes
                                                      SSL,S/MIME,JAR/XPI

   IKEv2 VPN CA                                       CTu,u,u
   ($PUBLIC_IP)                                       u,u,u
   vpnclient                                          u,u,u
   ```

   **Note:** To display a certificate, use `certutil -L -d sql:/etc/ipsec.d -n "Nickname"`. To delete a certificate, replace `-L` with `-D`. For other `certutil` usage, read <a href="http://manpages.ubuntu.com/manpages/xenial/en/man1/certutil.1.html" target="_blank">this page</a>.

1. **(Important) Restart IPsec service**:

   ```bash
   $ service ipsec restart
   ```

1. Follow instructions below for your operating system. **Note:** If you specified the server's DNS name (instead of its IP address) in step 1 above, you must enter the DNS name in the **Server** and **Remote ID** fields.

   #### Windows 7, 8.x and 10

   1. Securely transfer `vpnclient.p12` to your computer, then import it into the "Computer account" certificate store. Make sure that the client cert is placed in "Personal -> Certificates", and the CA cert is placed in "Trusted Root Certification Authorities -> Certificates".

      Detailed instructions:   
      https://wiki.strongswan.org/projects/strongswan/wiki/Win7Certs

   1. On the Windows computer, add a new IKEv2 VPN connection:   
      https://wiki.strongswan.org/projects/strongswan/wiki/Win7Config

   1. Start the new VPN connection, and enjoy your IKEv2 VPN!   
      https://wiki.strongswan.org/projects/strongswan/wiki/Win7Connect

   1. (Optional) You may enable stronger ciphers by adding the registry key `NegotiateDH2048_AES256` and reboot. Read more <a href="https://wiki.strongswan.org/projects/strongswan/wiki/WindowsClients#AES-256-CBC-and-MODP2048" target="_blank">here</a>.

   #### OS X (macOS)

   First, securely transfer both `vpnca.cer` and `vpnclient.p12` to your Mac, then double-click to import them one by one into the **login** keychain in **Keychain Access**. Next, double-click on the imported `IKEv2 VPN CA` certificate, expand **Trust** and select **Always Trust** from the **IP Security (IPsec)** drop-down menu. When finished, check to make sure both `vpnclient` and `IKEv2 VPN CA` are listed under the **Certificates** category of **login** keychain.

   1. Open System Preferences and go to the Network section.
   1. Click the **+** button in the lower-left corner of the window.
   1. Select **VPN** from the **Interface** drop-down menu.
   1. Select **IKEv2** from the **VPN Type** drop-down menu.
   1. Enter anything you like for the **Service Name**.
   1. Click **Create**.
   1. Enter `Your VPN Server IP` (or DNS name) for the **Server Address**.
   1. Enter `Your VPN Server IP` (or DNS name) for the **Remote ID**.
   1. Leave the **Local ID** field blank.
   1. Click the **Authentication Settings...** button.
   1. Select **None** from the **Authentication Settings** drop-down menu.
   1. Select the **Certificate** radio button, then select the **vpnclient** certificate.
   1. Click **OK**.
   1. Check the **Show VPN status in menu bar** checkbox.
   1. Click **Apply** to save the VPN connection information.
   1. Click **Connect**.

   #### Android 4.x and newer

   1. Securely transfer `vpnclient.p12` to your Android device.
   1. Install <a href="https://play.google.com/store/apps/details?id=org.strongswan.android" target="_blank">strongSwan VPN Client</a> from **Google Play**.
   1. Launch the VPN client and tap **Add VPN Profile**.
   1. Enter `Your VPN Server IP` (or DNS name) in the **Server** field.
   1. Select **IKEv2 Certificate** from the **VPN Type** drop-down menu.
   1. Tap **Select user certificate**, then tap **Install certificate**.
   1. Choose the `.p12` file you copied from the VPN server, and follow the prompts.
   1. Save the new VPN connection, then tap to connect.

   #### iOS (iPhone/iPad)

   First, securely transfer both `vpnca.cer` and `vpnclient.p12` to your iOS device, then import them one by one as iOS profiles. To transfer the files, you may use AirDrop. Alternatively, host the files on a secure website of yours, then download and import in Mobile Safari. When finished, check to make sure both `vpnclient` and `IKEv2 VPN CA` are listed under Settings -> General -> Profiles.

   1. Go to Settings -> General -> VPN.
   1. Tap **Add VPN Configuration...**.
   1. Tap **Type**. Select **IKEv2** and go back.
   1. Tap **Description** and enter anything you like.
   1. Tap **Server** and enter `Your VPN Server IP` (or DNS name).
   1. Tap **Remote ID** and enter `Your VPN Server IP` (or DNS name).
   1. Leave the **Local ID** field blank.
   1. Tap **User Authentication**. Select **None** and go back.
   1. Make sure the **Use Certificate** switch is ON.
   1. Tap **Certificate**. Select **vpnclient** and go back.
   1. Tap **Done**.
   1. Slide the **VPN** switch ON.

1. Once successfully connected, you can verify that your traffic is being routed properly by <a href="https://www.google.com/search?q=my+ip" target="_blank">looking up your IP address on Google</a>. It should say "Your public IP address is `Your VPN Server IP`".

## Known Issues

1. The built-in VPN client in Windows may not support IKEv2 fragmentation. On some networks, this can cause the connection to fail or have other issues. You may instead try the <a href="clients.md" target="_blank">IPsec/L2TP</a> or <a href="clients-xauth.md" target="_blank">IPsec/XAuth</a> mode.
1. If using the strongSwan Android VPN client, you must <a href="https://github.com/hwdsl2/setup-ipsec-vpn#upgrade-libreswan" target="_blank">upgrade Libreswan</a> on your server to version 3.26 or above.
1. Connecting multiple IKEv2 clients simultaneously from behind the same NAT (e.g. home router) is not supported at this time. For this use case, please instead use <a href="clients-xauth.md" target="_blank">IPsec/XAuth</a> mode.

## References

* https://libreswan.org/wiki/VPN_server_for_remote_clients_using_IKEv2
* https://libreswan.org/wiki/HOWTO:_Using_NSS_with_libreswan
* https://libreswan.org/man/ipsec.conf.5.html
* https://wiki.strongswan.org/projects/strongswan/wiki/WindowsClients
* https://wiki.strongswan.org/projects/strongswan/wiki/AndroidVpnClient

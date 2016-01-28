# OPSI-Installation on Debian 8.x

This QuickInstall Guide describes how to a single OPSI depot server on Debian 8.x.

The minimal scenario for this is,

* You have a minimal Debian 8.x (aka jessie) system (Hostname: opsi)

For connecting your first Windows clients to OPSI, you need to have a very basic network setup up and running:

* An intranet network with a gateway to the internet
* Minimal DNS setup with a domain of the form ``<domain>.<tld>``, make sure that the IP address of the
  OPSI depot server reserve-resolve ok.
* Minimal DHCP setup for to-be-attached Windows clients

## Prepare DNS/Hostname setup

If you start with installing OPSI on a single server with no network
environment (you can add that later...), then you should make sure that
the OPSI machine's hostname is listed in ``/etc/hosts`` in the following way.

Modification of ``/etc/hosts`` on host ``opsi.<domain>.<tld>``, adapt the
host's IP address and the actual fully qualified domain name to your
needs:

```
root@opsi:~# diff -u /etc/hosts.dpkg-orig /etc/hosts
--- /etc/hosts.dpkg-orig	2016-01-22 15:35:56.148000000 +0100
+++ /etc/hosts	2016-01-22 15:34:28.036000000 +0100
@@ -1,5 +1,7 @@
 127.0.0.1	localhost
-127.0.1.1	opsi
+#127.0.1.1	opsi
+
+10.22.33.44    opsi.<domain>.<tld> opsi
 
 # The following lines are desirable for IPv6 capable hosts
 ::1     localhost ip6-localhost ip6-loopback
```


## OPSI-Server, Install the OPSI depot server

Run the following commands as super-user root.

1. Prepare APT for obtaining OPSI packages:

  ```
  # echo "deb http://download.opensuse.org/repositories/home:/uibmz:/opsi:/opsi40/Debian_8.0 ./" > /etc/apt/sources.list.d/opsi.list
  # wget -O - http://download.opensuse.org/repositories/home:/uibmz:/opsi:/opsi40/Debian_8.0/Release.key | apt-key add -
  # apt-get update
  ```

1. Make sure ``openbsd-inetd`` is installed, but Debian's ``tftpd`` package is not installed:

  ```
  # apt-get install openbsd-inetd 
  # update-inetd --remove tftpd || true
  # apt-get remove --purge tftpd
  ```

2. Start installing packages from the OPSI APT archive:

  ```
  # apt-get install opsi-atftpd
  # apt-get install opsi-depotserver
  ```

  The opsi-depotserver package post-installation script will guide you through the creation of an SSL certificate.
  Follow that process (handle by "debconf") of creating OPSI'S Server SSL Certificate.

3. Install OPSI's Configuration Editor (an application that can be
launched on any client for configuring OPSI-managed machines. The OPSI
configuration editor can be run as a Java Webstart application or as a
Java plugin container in a webbrowser.

  ```
   # apt-get install opsi-configed
   ```

  It is also possible later on to roll-out the OPSI Configuration Editor to an MS Windows machine.

4. Debian 8 (Jessie) specialities: The bootimage has issues when it comes
to mounting the ``opsi_depot`` share over mount.cifs. The actual tests
indicate, that these issue will not prompt if the winbindd daemon is not
running.


  ```
  # systemctl disable winbind
  ```

5. On Samba4 and above you have to set below option globally (edit ``/etc/samba/smb.conf`` for this):

  ```
  # Always allow execution of files, if no x-bit is set for them
     acl allow execute always = yes
     ; required for executing files without setting the files' x--bits on the underlying
     ; file systems!!!!
  ```

6. Create or choose an already existing admin user account. You can use the initial user you created during
installation of the Debian system (here: adminuser, already existing)

7. Add that user to the opsiadmin POSIX group

  ```
  # adduser adminuser opsiadmin
  ```

9. The OPSI Getting Started manual suggests making this user a Samba user. This is not required.

10. Furthermore, the official documentation states that the admin user
above should be a member of the pcpatch POSIX group. This is neither
required.

11. Correctly configure the password of user pcpatch for OPSI, POSIX, Samba. The ``pcpatch`` user account is used in OPSI's client agent for accessing resources (Samba, JSON-RPC API) on the OPSI server.

  ```
  # opsi-admin -d task setPcpatchPassword
  ```

  This command will ask you for a password. Enter that here and take a note of it.

12. Retrieve basic set of .opsi packages from UIB


  ```
   # opsi-product-updater -i -vv
  ```

## Accessing OPSI's config editor TTW

Visit ``https://opsi.<domain>.<tld>:4447/`` from any machine on the same
subnet. The client machine requires Java7 (or higher to be installed).
For authentication don't use the OPSI server's root account, but the admin user
(here: ``adminuser``) created during installation of the Debian system.


## On the first MS Windows Client

* Visit ``\\opsi\opsi_depot``
* Login as Samba user ``pcpatch``
* Browse to ``opsi-client-agent`` directory
* Execute ``service_setup.cmd``
* Wait for MS Visual C++ redist getting installed
* Wait for OPSI client agent to get installed
* When asked for username and password for the OPSI server, use account ``pcpatch`` and the earlier noted password for that user.

## Notes on OPSI products

*  swaudit / hwaudit: Run "always" vs. "setup" once! Check, how long it
takes during boot time and if it takes any considerable space on the
client's hard drive.

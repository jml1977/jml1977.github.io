---
layout: post
title:  "Getting Wifi working on a Lenovo P14S AMD Gen2"
date:   2021-07-10 16:48:29 +0200
categories: lenovo p14s amd gen2 wifi
---

New laptop day!  As expected, Wifi didn't work on my new Lenovo P14S AMD Gen2 due to the Realtek 8852AE Wireless chip.  Here's the steps I used to get it working.

The following got Wifi working for me, and is based on a couple of difference places:
- [Driver for Realtek 8852AE](https://github.com/lwfinger/rtw89)
- [Fedora Sysadmin Manual](https://docs.fedoraproject.org/en-US/fedora/f32/system-administrators-guide/kernel-module-driver-configuration/Working_with_Kernel_Modules/)

Firstly, install some dependencies.  These were done using Ethernet.

{% highlight shell %}
sudo yum install openssl kernel-devel mokutil keyutils
{% endhighlight %}

{% highlight shell %}
# from https://github.com/lwfinger/rtw89
git clone https://github.com/lwfinger/rtw89.git -b v5
cd rtw89
make
{% endhighlight %}

Doing a `make install` at this point gives an error about the modules not being signed.  This is due to Secure Boot being enabled (by default).  This can be confirmed with

{% highlight shell %}
$ mokutil --sb-state
{% endhighlight %}

Following the "Generating a Public and Private X.509 Key Pair" instructions from [here](https://docs.fedoraproject.org/en-US/fedora/f32/system-administrators-guide/kernel-module-driver-configuration/Working_with_Kernel_Modules/)

{% highlight shell %}
# cat << EOF > configuration_file.config
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts

[ req_distinguished_name ]
O = MyName
CN = MyName
emailAddress = MyEmailAddress

[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOF
{% endhighlight %}

Followed by generating a new public/private key, based on this:

{% highlight shell %}
 openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 \
 -batch -config configuration_file.config -outform DER \
 -out public_key.der \
 -keyout private_key.priv
{% endhighlight %}

Then, I signed both the modules

{% highlight shell %}
/usr/src/kernels/$(uname -r)/scripts/sign-file \
 sha256 \
 private_key.priv \
 public_key.der \
 rtw89pci.ko

/usr/src/kernels/$(uname -r)/scripts/sign-file \
 sha256 \
 private_key.priv \
 public_key.der \
 rtw89core.ko

# and install them
sudo make install
{% endhighlight %}

Then, following the "System Administrator Manually Adding Public Key to the MOK List" on the same page:

{% highlight shell %}
mokutil --import public_key.der
{% endhighlight %}

which asks for a password.  This password is needed again in a few minutes.

Then:

{% highlight shell %}
reboot
{% endhighlight %}

And the UEFI should notice the new request and prompt for the password during the reboot.  For me, it didn't pick it up on the first reboot, but it's possible to check that the request was waiting using `mokutil --list-new`, so rebooted again, and this time it did work.

Once logged back in:

{% highlight shell %}
sudo modprobe rtw89pci
{% endhighlight %}

And then the OS noticed that Wifi was available.  At this point, connect to your Wifi using the password from the Gnome desktop as normal and it worked.

As per the [instructions](https://github.com/lwfinger/rtw89), this will need to be done after each kernel update.
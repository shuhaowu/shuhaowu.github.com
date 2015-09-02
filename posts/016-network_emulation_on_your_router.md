title: Traffic Cop: Network (2G/3G) Emulation on an OpenWRT Router
published: no

A common issue in application development nowadays is that people do not always test their application against devices and networks that have very low performance. This is somewhat understandable, especially when not every one has the resources to setup a big testing rig testing high end, mid tier, and low end devices on a variety of different networks. This is not to mention that even when there are the means to test, people probably skip it as it is a giant hassle to do so.

During [Shopify's Hack Days][hackdays], the lovely [@amecila][amecila] and I worked on a project to simplify testing against slow performing networks. The basic idea is to be able to control the bandwidth, latency, and packet loss for individual devices connected to a special WiFi network. This system was named "Traffic Cop". 

Traffic Cop is an application running on a router with [OpenWRT][openwrt] that allows you to emulate different networks such as 3G, EDGE (2G) for individual devices connected to this router. 

All a client has to do is connect to this network, go to the router's IP address (at port 8080 by default), and select a network profile to simulate. You can then watch the client's ping latency, bandwidth, and packet loss rate match the ones indicated for that profile.

Spoiler! The finished product looks like the following:

<div class="center">
  <img src="/static/img/network-emulation-on-your-router/demo.gif" />
  <p><small>Note: the UI text has been slightly updated in the current version, with "None" changed to "No profile" and the caption set to "Best available connection".</small></p>
</div>

[trafficcop-github]: https://github.com/shuhaowu/trafficcop
[hackdays]: https://www.shopify.com/blog/7530964-inside-shopify-hack-days
[amecila]: https://github.com/amecila
[img2]: /static/img/aosp-intellij/project-structure-dep.png

------------------------------------------------------------------------------

### This already exists. Why? ###

This is probably the first thing that popped up in your mind after reading the above and you are right, [several][network-link-conditioner] [tools][android-network-delay] [already][network-emulator-toolkit] exists for this purpose.

One thing about these tools is that they are all locked down to a particular platform. As an example, [Network Link Conditioner][network-link-conditioner] is tied down to iOS and OS X only. Android does not have a network emulation tool on live devices as the feature is only available in the [emulator][android-network-delay]. So you are right when it comes to the existence of the idea, but I am not aware of any attempts to put this directly on a router.

Running this on a router has the advantage of being "cross platform", as long as the device you are testing your application on has some sort of networking capability (WiFi/Ethernet/Other) and has a browser to connect to the page to change your profile. This is much easier than downloading, installing, and then configuring an application to do this for your particular setup.

Furthermore, if your company has permanent test stations, you can just have a router nearby with this software running and all your devices connected. Since Traffic Cop supports per device network settings, all your devices can be using different settings and people coming over to test can set it themselves.

[network-link-conditioner]: http://nshipster.com/network-link-conditioner/
[android-network-delay]: https://developer.android.com/tools/devices/emulator.html#netdelay
[network-emulator-toolkit]: https://blog.mrpol.nl/2010/01/14/network-emulator-toolkit/
[netem]: http://www.linuxfoundation.org/collaborate/workgroups/networking/netem

### How does it work? ###

Traffic Cop runs on a router running [OpenWRT][openwrt], which you can think of as a Linux distribution built for routers. OpenWRT allow you to run a wide variety of software by installing them via [opkg][opkg], a lightweight package manager. The availability of software makes it a perfect target environment for what we are trying to do.

On a normal router, if you visited the router's IP address in your browser, you would likely get to a login page to the router's administration panel. On a Traffic Cop enabled router, you get select an internet profile for the device you used to connect to the page<sup>1</sup>.

This page is simply a static HTML + CSS + JS page that makes certain calls to a CGI backend written in shell (Ash, to be specific). In these scripts, a few common network profiles (3G, 2G) are defined. When you select a different profile, an Ajax call to the server executes these scripts.

These scripts then call out to a tool known as [`tc`][tc]. This tool is a front end to the bandwidth management features of the Linux kernel, which is very advanced and a [topic of its own][lartc-doc]. Using `tc`, we can then limit the bandwidth, increase the latency, and randomly drop packets for clients that chooses a degraded network profile.

*<sup>1</sup>: Some additional steps are required to enable this. By default, Traffic Cop runs on port 8080,*

[opkg]: http://wiki.openwrt.org/doc/techref/opkg
[openwrt]: https://openwrt.org/
[tc]: http://linux.die.net/man/8/tc
[lartc-doc]: http://lartc.org/howto/lartc.qdisc.html

### How does it really work? I mean, how do you use tc and stuff? ###

This is actually a somewhat complicated topic. It took us about a whole morning of reading to understand enough to implement the backend. The [LARCT documentation][lartc-doc] is very good at explaining all of this so I'll just give a brief explanation that may not be easily understood without reading the linked documentations. Feel free to skip the section if you have a lot of difficult reading it or otherwise don't care.

In OpenWRT, the local network is typically under the interface `br-lan`. Using `tc`, Traffic Cop creates a [`htb`][htb] qdisc<sup>2</sup> to limit the bandwidth, a [`netem`][netem] qdisc to delay packets and drop them randomly, and a filter to filter out particular IPs.

The `tc filter` requires a id to send traffic to for throttling, delaying, and/or dropping. Hence, the `htb` qdisc for a particular client gets assigned an id which comes from the last group of the IPv4 address. Since ids must be unique, the `netem` qdisc gets assigned an id of the last group of the IPv4 address multiplied by 100 plus 66 (to avoid collision of .2 and .200). The `netem` qdisc also is a child of `htb`, allowing the filter to simply send the traffic to the correct `htb` qdisc, which will pass it to `netem`.

Any traffic from clients without network profiles will be tossed into id 265, which has no limit and never collide with the control chain id of another client. This is created during `ifup br-lan`

This is a fairly dense section, you can see most of the code for individual clients [here][common-code] and the code for boot up [here][boot-code].

*<sup>2</sup>: qdisc is an algorithm that manages the queue of a device, either incoming (ingress) or outgoing (egress). [Credit: [LARCT][lartc-doc]]*
[htb]: http://luxik.cdi.cz/~devik/qos/htb/manual/userg.htm
[netem]: http://www.linuxfoundation.org/collaborate/workgroups/networking/netem
[common-code]: https://github.com/shuhaowu/trafficcop/blob/d3f58d873a411305207bd06535275b0d82fdb107/src/data/usr/lib/trafficcop/api/_common#L7-L41
[boot-code]: https://github.com/shuhaowu/trafficcop/blob/d3f58d873a411305207bd06535275b0d82fdb107/src/data/etc/hotplug.d/iface/50-trafficcop

### I want to run this myself, how? ###

During the two days of development, the code was built straight into an image that could be ran on our test device. Afterwards, I spent a little bit more time packaging it into an opkg package, which will work as soon as you install it. You can download the ipk file from https://github.com/shuhaowu/trafficcop/releases and install it with opkg on your OpenWRT router. You will need a couple hundred kilobytes of space for this.

Sample commands:

    root@OpenWRT:~# cd /tmp
    root@OpenWRT:/tmp# wget https://github.com/shuhaowu/trafficcop/releases/download/....
    root@OpenWRT:/tmp# opkg update && opkg install trafficcop*.ipk

After installing, you should be able to choose a network profile for your device by using that device and navigating to `http://{your-router-ip}:8080`.

As a note, with this setup, you can even have LuCI installed as well as Traffic Cop :)

### Closing Notes ###

This project taught me a great deal about Linux networking. Writing a web app in shell was also a great surprise (![:troll:](/static/img/trollface-emoticon.png)). Hopefully, this will be useful for people writing apps for different platforms.

For future work, the some aspect of running this and packaging this can be improve so I may be able to submit this into the official OpenWRT repository. Also, it would be nice to be able to specify your own network profile, as well as simulate jitter and other more network issues.

### Links ###

- Source code: [https://github.com/shuhaowu/trafficcop][trafficcop-github]
- Download: [https://github.com/shuhaowu/trafficcop/releases](https://github.com/shuhaowu/trafficcop/releases)

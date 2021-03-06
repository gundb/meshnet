Babel + WireGuard Mesh
======================

## Discussion on `matrix/#althea:matrix.org`

### Summary

- Should be able to set up Babel + WireGuard on ClearFog Pro running Armbian
- Installing Babel
  - Clone git://github.com/jech/babeld.git and cross-build for ARM
  - Assign an IPv6 address for each router in fd00::/8
    ```
    sudo ip addr add <some ipv6 address> dev <eth interface>
    ... repeat for more interfaces
    babeld <your arguments and tunings shouldn't need any> eth0 eth1... etc
    ```
  - Babel with IPv4 is full of gotchas
- Installing WireGuard
  - Need to be built as a kernel module which requires Armbian sources
  - Use for end-to-end encryption between nodes
  - Expect 400-800 Mbps on ClearFog Pro
- Local services (both solutions discusses are non-ideal, need further discussions)
  - Set up iptables to NAT IPv4 and client devices over the exit so Babel only deals in IPv6
  - Provide a default IPv6 route that allows accessing mesh devices
- Internet gateways
  - Exits are a list of keys that it would deploy a WireGuard tunnel for
  - Client nodes need to have keys to exit nodes manually added

### Chat logs

Mar, 2019

>ttk2 (@_discord_267031307945508864:t2bot.io)  
The GL is the best price/performance/availability ratio. You can run an exit in a pi locally but the issue then will be adding it to all the other routers (exits are added out of band as a key exchange mechanism) there's a curl command trick for that though.  

>benhylau  
Hi ttk2, can I run Babel + wireguard (node to node or e2e?) without payment daemon?

>ttk2 (@ttk2:matrix.org)  
sure it's not particularly difficult.  
do you want an openwrt firmware for that or?  
you can do it with a script too

>benhylau  
So I am thinking about Armbian base, it's the set up I linked above I am trying to prototype

>ttk2 (@ttk2:matrix.org)  
you just need to make sure every router gets a fd ipv6 address, as a note babel with ipv4 is full of gotchas while ipv6 'just works' if you assign any non link local address to the router.  
benhylau: so you need to run this two line script on startup  
```
sudo ip addr add <some ipv6 address> dev <eth interface>
... repeat for more interfaces
babeld <your arguments and tunings shouldn't need any> eth0 eth1... etc
```
>and your done  
darn messed up block code formatting

>benhylau  
So I want to peer the MicroTiks with stock firmware, then on the clearfog run Babel + wireguard, and connect an AP to each. If these work (and we are sure to go ahead with this routing protocol) I may mix some leaf nodes with the GL for lower cost

>ttk2 (@ttk2:matrix.org)  
shouldn't be an issue, babel has almost no dependencies so an arm binary should work on anything with the same architecture running linux  
should just be able to copy it over and then make an init script.  

>benhylau  
So what do I install in this case? Not the openwrt package  

>ttk2 (@ttk2:matrix.org)  
you wouldn't install anything, you would clone babel then cross build it  
here's a script that takes all the build artifects from openwrt and uses them to easily and quickly let you cross build rust  
https://github.com/althea-mesh/althea_rs/blob/master/scripts/openwrt_build_mips.sh  
just change the env varaibles to CC and LD instead of target_cc (which is a rust thing) then run gcc to build babel  
git clone git://github.com/jech/babeld.git  
then just copy the binary to /bin/ on the other side  
I mean as far as engineering a mesh firmware goes this is as simple as it gets :P

>benhylau  
Wow thanks, how is jech different from https://github.com/althea-mesh/babeld

>ttk2 (@ttk2:matrix.org)  
we've got our price extension, full path rtt extension and are otherwise a few commits behind, we need to rebase every so often.  
our price extension is currently too messy to merge upstream although we have a neater version of it designed it hans't been implemented yet. 

>benhylau  
AHH ok, and wireguard is in which?

>ttk2 (@ttk2:matrix.org)  
well wireguard might be a little harder, since you'll need to build it as a kernel module, you'd need the micotik kernel sources  
which they should release under the gpl... but you never know. 

>benhylau  
Clearfog pro, that's what runs the wireguard and routing, tiks will be stock

>ttk2 (@ttk2:matrix.org)  
essentially you would need to find their kernel sources, clone wireguard, setup wireguard to cross build the kernel module, copy it over and then load it. More involved than babel for sure since kernel modules are highly dependent on having the exact source tree of the kernel  
well for armbian it won't be an issue, their kernel sources are public  
https://forum.openwrt.org/t/mikrotik-gpl-source/6750

>benhylau  
Yea it should run Armbian...  
Do you guys run wireguard for node to node or e2e?

>ttk2 (@ttk2:matrix.org)  
depending on the kernel version of your device versus the last time they updated their gpl sources you may be SOL on the micotik  
both, we nest wireguard  
(yes it be like that)  
we could probable remove node to node and assume links are isolated and secured, but you know unmanaged switches exist.  
and so do sector antennas without client isolation enabled

>benhylau  
I don't expect to flash the tiks anyway, I think wireless equipment it's better to leave them as is, mess with the router board all we want :)

>ttk2 (@ttk2:matrix.org)  
so we chose the 15-20% perf hit for security by default
and so do sector antennas without client isolation enabled

>benhylau  
Hmm how does node to node help with this?  
Because same node, it won't encrypt at all?

>ttk2 (@ttk2:matrix.org)  
benhylau: so sector antennas act like unmanaged switches, meaning all nodes have the same broadcast domain, so nodes A, B, and C are on the same 'wire' and B could send packets with 'A' in the from field and get C billed for them.  
wireguard does authentication and encrypiton, we don't need the latter really but it's faster than any auth only altnerative at the moment so 🤷

>benhylau  
Heh wow, what speeds can you get from one of those little GL processors?

>ttk2 (@ttk2:matrix.org)  
not much of an issue with you guys, you can do the (much easier) end to end wireguard only  
100mbit without anything else getting in the way.  
it's actually mostly iptables overhead not the actual wireguard encryption, they have a hand tuned assembly arm encryption implementation, it uses the vector extension to great effect. 

>benhylau  
The clearfog has a 1.6G ARM, do you think it can saturate a 800 Mbps radio?

>ttk2 (@ttk2:matrix.org)  
I've tested similar processors, it's highly dependent on the memory, but I've seen 400-600ish let me know what you see  
so not quite, but pretty close.  
well 400-600 with two tunnels... yeha you can probably push to 800 

>benhylau  
Okay I am gonna give them a call today, see if I can order a 2 GB RAM version with poe  
And exit node is just a node with some extra configs and plugged into internet, then other nodes put its key?

>ttk2 (@ttk2:matrix.org)  
so our first gen exit nodes just has a list of keys that it would deploy a wireguard tunnel for (manual client config) but now our billing binary does dyanmic setup and registration  
but yes an exit node is really just a glorified wireguard configurating script  
you just script the steps to configure on each end, it's pretty easy

>benhylau  
The last thing is local devices, if I plug raspberry pis into the network, they need to be natted?

>ttk2 (@ttk2:matrix.org)  
well that depends on what you want, what we do is setup iptables to nat ipv4 and client devices over the exit.  
this has the upside of making sure babel only deals in ipv6

>benhylau  
This means the traffic will always go thru the exit and back?

>ttk2 (@ttk2:matrix.org)  
it's just a couple of iptables lines I can share  
yes, it makes the default ipv4 route (and thus the default route for client traffic) over the tunnel which will route all traffic that way.  
you could provide a default ipv6 rotue that allows accessing mesh devices but not the internet or some variant, but we haven't really explored that well  
so I can't share magic iptables lines to make it happen.  
yes, it makes the default ipv4 route (and thus the default route for client traffic) over the tunnel which will route all traffic that way. 

>benhylau  
Share this to me :) I want the other but perhaps this to start  
But then I am also mindful apps don't like IPv6 still, so... They kinda need the v4 nat

>ttk2 (@_discord_267031307945508864:t2bot.io)  
https://github.com/althea-mesh/althea_rs/blob/master/althea_kernel_interface/src/exit_client_tunnel.rs#L159  
https://github.com/althea-mesh/althea_rs/blob/master/althea_kernel_interface/src/exit_server_tunnel.rs#L117

>benhylau  
Thanks so much for explaining all this

>ttk2 (@ttk2:matrix.org)  
No problem!

## Resources

- [Althea documentation](https://forum.altheamesh.com/c/education-and-resources)
- [Althea exit server automation](https://github.com/althea-mesh/installer/commit/0e95832ff864e208eee45fbb085a88bcc3aba8ac)

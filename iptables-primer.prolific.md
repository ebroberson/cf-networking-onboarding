Iptables - a primer

## What
ASGs and c2c network policies are implemented using iptables rules. Think of iptables as a firewall implementation. (Iptables can do more than firewall things, but we have to start somewhere.)

All network traffic on linux machines is evaluated against the iptables rules written on the machine. To organize these rules, there are tables and chains. Tables have chains and chains have rules. Users (that's you!) can create custom chains and rules for their own needs. Traffic hits different tables and chains depending on if the traffic is ingress, egress, or East/West.

Let's look at the anatomy of an iptables rule.

![iptables rule anatomy](https://deliveryimages.acm.org/10.1145/2070000/2062737/10822f2.png)

The example above can be translated as:
- (**table/chain**) For any traffic that is evaluated against the filter table on the INPUT chain...
- (**match**)...if that traffic is using the tcp protocol...
- (**match**)...and if that traffic is sending traffic to port 80...
- (**jump rule/target**) ... then ACCEPT that traffic.

Once a packet hits ACCEPT, then it stops evaluating in that table and chain. It would also stop if it hit a DROP or REJECT target.

In this story you are going to skim/read two great resources on iptables rules.

## How
1. Give the [iptables man page](http://ipset.netfilter.org/iptables.man.html) a skim. At least the description, targets, and tables sections. It leaves out a lot of commonly used features that are covered in [the iptables-extensions man page](http://ipset.netfilter.org/iptables-extensions.man.html).

1. Flip through [Aidan's "iptables in Cloud Foundry" powerpoint](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

1. Check out [Julian Evan's Blog on iptables](https://jvns.ca/blog/2017/06/07/iptables-basics/).

## Resources
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)
[Julia Evans iptables basics](https://jvns.ca/blog/2017/06/07/iptables-basics/)

L: iptables-primer
---

Write your own iptables DROP rule in a docker container

## Assumptions
- You have the docker CLI installed

### What?
Experiment with writing your own iptables rules within the safety of your own docker container.

### How?

📝 **Get your docker container setup**
0. Run an ubuntu docker container and attach to it: `docker run --privileged -it ubuntu bin/bash`
0. Set up the docker container
    ```
    apt-get update
    apt-get install iptables
    apt-get install curl
    ```

📝 **What's the current state of the world?**
0. Look at the default iptables rules `iptables -S`.
    It should look like this:
    ```
    -P INPUT ACCEPT
    -P FORWARD ACCEPT
    -P OUTPUT ACCEPT
    ```
   Input, forward, and output are all names of chains. Currently there are no rules attached to these chains. What beautiful empty chains! This means that traffic is not being restricted by iptables.

0. See that traffic is not being restricted. `curl google.com`.  It works!

📝 **Make your own DROP rule**
Let's make a rule to DROP all traffic from the container so that the curl will fail.

0. Make your own custom chain.
 ```
 iptables -N drop-everything
 ```
0. Append a rule to your chain.
 ```
 iptables -A drop-everything -j DROP
 ```
0. View your handiwork. Oooooh. Ahhhhhhh.
 ```
 iptables -S
 ```
0. See if you can still `curl google.com`. What! The curl still works!
   That's because currently nothing is hitting your rule. You need to attach your custom chain to the INPUT, FORWARD, and/or OUTPUT chain in order for traffic to hit it.

    The INPUT, FORWARD, and OUTPUT chains are hit in different situations. (See diagram below)
    - The INPUT chain is hit by ingress traffic (remember, the traffic is coming *in*).
    - The FORWARD chain is hit by East/West traffic (remember... idk for this one).
    - The OUTPUT chain is hit by egress traffic (remember, the traffic is *e*xiting the container and going *out*).

   ![iptables chains and tables diagram](https://storage.googleapis.com/cf-networking-onboarding-images/iptables-tables-and-chains-diagram.png)

0. The request to google is egress traffic, so we want to attach out custom chain to the OUTPUT chain.
 ```
 iptables -A OUTPUT -j drop-everything
 ```

0. See if you can still `curl google.com`. You should see the error `Could not resolve host: google.com`.
0. Delete all of the rules and the chain that you created.
   Before you can delete the chain itself, you need to delete the rules attached to it using the `-D` flag.
   ```
iptables -D EITHER-INPUT-FORWARD-OR-OUTPUT -j drop-everything
iptables -D drop-everything -j DROP
   ```
   Then you can delete the chain itself using the `-X` flag
   ```
iptables -X drop-everything
   ```

Don't leave your docker container yet! You're going to need it in the next story!

### Expected Result

You should know how to...
- add/remove a iptables chain on a particular table
- add/remove a rule to an iptables chain


### Resources
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Julia Evans iptables basics](https://jvns.ca/blog/2017/06/07/iptables-basics/)
[Aidan's iptables ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)
[iptables primer](https://danielmiessler.com/study/iptables/)

L: iptables-primer

---

Write your own iptables firewall in a docker container

## What

Make a super basic firewall for your docker container. This (extremely practical) firewall will only let egress traffic exit if it is going to neopets.com.

## How

🤔 **Make your own rule**
0. Make your own chain.
0. Attach rule to that chain that accepts traffic if it is sent to ip 23.96.35.235 (neopets!) port 80 using tcp.
0. Attach a rule to that chain that drops all other traffic.
0. Add a jump rule to either the OUTPUT, FORWARD, or INPUT chains so that the traffic exiting the docker container will hit your custom chain.
0. Curl google.com. Does it fail?
0. Curl 23.96.35.235:80. Does it succeed?
0. Curl http://neopets.com. Does it fail or succeed? Why?
0. Practice deleting chains and rules: delete all of the rules and chains that you created.

### Expected Result
Hopefully you realize by now that iptables rules are very powerful and very fun :D

❓Why didn't curling http://neopets.com work?

### Extra Credit
1. Use iptables rules to make it so you can curl neopets.com, but not google.com

## Resources
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Julia Evans iptables basics](https://jvns.ca/blog/2017/06/07/iptables-basics/)
[Aidan's iptables ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)
[iptables primer](https://danielmiessler.com/study/iptables/)

L: iptables-primer
L: questions

---

[RELEASE] Iptables Primer ⇧
L: iptables-primer
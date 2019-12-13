OSI Networking Model Primer

## Assumptions
- None

## What

The stories have been throwing around terms like IP, HTTP, DNS, DNAT without talking about the broader networking context around these terms.

In this next track of work you will start looking at TCP as well. There are some crucial differences between HTTP Routes and TCP Routes, that will make more sense if you understand what layer of networking those different protocols are on.

In this story you are going to learn about the OSI model (Open Systems Interconnection model). In the OSI model, there are 7 networking layers and each one is built on top of the one below it.

![OSI model](https://storage.googleapis.com/cf-networking-onboarding-images/osi-layers.png)

## How

🎥 Watch Gabe's talk ["Networking 101 for Software Engineers"](https://www.youtube.com/watch?v=6_FHs_g1yw4)

Watch from start to 35:26.

The whole talk is great. If you have the time, watch the whole thing.

## Questions
❓What layer is TCP at?
❓What layer is IP at?
❓What layer is HTTP at?

L: tcp-routes
L: questions
---

User Workflow: TCP Routes

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have one TCP Router deployed. (Check by running `bosh vms` and looking for a VM called tcp-router). If you have more than one that's okay, but the steps written below assume that there is only one.

## What
GoRouter only handles incoming HTTP traffic.  All of the routes that we have been talking about so far are *HTTP* routes (though we often drop the HTTP in conversation and just call them routes). If you want to send og TCP traffic, then you are going to need to set up a TCP route. 

There is a parallel system for TCP routes similar to HTTP routes:
- An HTTP client connects to an HTTP route on an HTTP domain, though an HTTP load balancer, which sends traffic to HTTP Routers (GoRouters).
- TCP client connects to a TCP route on a TCP domain, through a TCP load balancer, which sends traffic to TCP Routers.

```
                                                   +-----------+
 +-----------+         +------------------+        |HTTP Router|        +------+
 |HTTP Client| ----->  |HTTP Load Balancer| -----> |(GoRouter) | -----> |      |
 +-----------+         +------------------+        +-----------+        |      |
                                                                        | App  |
 +-----------+         +------------------+        +-----------+        |      |
 |TCP Client | ----->  |TCP Load Balancer | -----> |TCP Router | -----> |      |
 +-----------+         +------------------+        +-----------+        +------+

```

Let's create a TCP Route and send traffic to it!

## How

📝 **Prep in Google Cloud Console**
1. Make sure that you have DNS set up properly for the TCP Router.
    In Google Cloud Console, go to the Zone Details for your env (Network Services --> Cloud DNS --> <your-env>-zone)
1. You should find a domain that starts with `tcp`, let's call this TCP_DOMAIN. This domain should have an IP next to it, let's call this TCP_LOAD_BALANCER_IP.
    ![example TCP domain DNS on GCP](https://storage.googleapis.com/cf-networking-onboarding-images/example-tcp-domain-dns.png)

1. In Google Cloud Console, find the Load Balancer with the ip TCP_LOAD_BALANCER_IP. (Network Services --> Load balancing)
    Here you will be able to see all of the VMs that the load balancer ...balances load between. In the example below, and most likely in your case, there is only one TCP Router deployed, so there will only be one VM listed.

    ![example TCP load balancer on GCP](https://storage.googleapis.com/cf-networking-onboarding-images/example-tcp-load-balancer.png)

1. Click the VM instance that the TCP load balancer sends traffic to. Find the VM's internal IP. Let's call this TCP_ROUTER_IP.
   ![example TCP router vm on GCP](https://storage.googleapis.com/cf-networking-onboarding-images/example-tcp-router-details.png)
1. In the terminal, check that the TCP_ROUTER_IP matches the IP that bosh reports for the TCP Router. It comes full circle!

📝 **Push a TCP server app**
1. Push [this tcp listener app](https://github.com/cloudfoundry/cf-acceptance-tests/tree/master/assets/tcp-listener) with no HTTP route.
 This app is listening for TCP traffic on port 8080 and it logs all of the messages sent to it.
 ```
 cf push tcp-app --no-route
 ```

📝 **Create a TCP Route**
1. See that you have a default-tcp router group. Router Groups are used to reserve ports for tcp routes.
 ```
 cf router-groups
 ```
1. Create a shared TCP domain
 ```
cf create-shared-domain TCP_DOMAIN --router-group default-tcp
 ```
1. See that `cf map-route --help` has different usage instructions for TCP routes and HTTP routes.
1. Create a route with the TCP domain and map it to tcp-app, let's call this TCP_ROUTE:TCP_PORT.
 ```
 cf map-route tcp-app TCP_DOMAIN --random-port
 ```

### Expected Result

📝 **test with curl**
Curl sends traffic via HTTP(S), but because HTTP is built on top of TCP, we can still use curl to test out TCP route.
1. In one terminal, run `cf logs tcp-app`
1. In another terminal `curl TCP_ROUTE:TCP_PORT`
1. See the HTTP headers show up in your logs

🤔 **test with netcat**
Netcat is a helpful utility for reading and writing traffic over TCP or UDP. You will use netcat to send tcp traffic.
1. In one terminal, run `cf logs tcp-app`
1. In another terminal, run a docker container and get some helpful tools
    ```
docker run --privileged -it ubuntu bin/bash
apt-get update -y
apt-get install netcat -y
    ```
1. Use `nc -h` to look at the help text and figure out how to connect to TCP_ROUTE:TCP_PORT.
1. If you successfully open a connection, it will hold it open so you can type anything. Mash some keys and press enter.
1. See your key mashes in the app logs. You sent that via TCP!

Don't delete your TCP app/route/domain yet! You'll need them in the next stories.

## Extra Credit
❓What happens if you try to send TCP traffic to an HTTP route? Why can you send HTTP traffic (kind of) over TCP, but not the other way around?

## Resources

[Basic Golang TCP Server](https://coderwall.com/p/wohavg/creating-a-simple-tcp-server-in-go)
[CF Docs - configure TCP domain](https://docs.cloudfoundry.org/adminguide/enabling-tcp-routing.html#-configure-cf-with-your-tcp-domain)
[CF Docs - HTTP vs TCP routes](https://docs.cloudfoundry.org/devguide/deploy-apps/routes-domains.html#-http-vs.-tcp-routes)
[CF Docs - create TCP routes](https://docs.cloudfoundry.org/devguide/deploy-apps/routes-domains.html#-create-a-tcp-route-with-a-port)
[Sending tcp traffic via netcat](https://askubuntu.com/questions/443227/sending-a-simple-tcp-message-using-netcat)
[netcat fun! by julia evans](https://jvns.ca/blog/2013/10/01/day-2-netcat-fun/)
[Did you brew install nc on your mac and it broke bosh? yup.](https://github.com/cloudfoundry/bosh-cli/pull/403)

L: tcp-routes
L: user-workflow
L: questions
---

Dig-ing TCP and HTTP Routes

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE
- You have a 2 [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed, that are named appA and appB
- You have a HTTP route mapped to appA called APP_A_ROUTE and one mapped to appB called APP_B_ROUTE

## What
`dig` is one of many network utilities that can be very helpful for debugging. Dig does a DNS lookup for a URL.

Let's play around with dig, TCP Routes, and HTTP Routes.

## How

📝 **Dig with a good URL**
1. Run `dig neopets.com`
    If you see `ANSWER 1` (which you should) that it was able to resolve the route. `ANSWER 0` means it was unable to resolve the route.
    In the `ANSWER SECTION` you should see an IP. (Likely, 23.96.35.235)
1. Put the IP that neopets.com resolved to in a browser. Is it neopets?

📝 **Dig with a bogus URL**
1. Resolve a bogus URL so you get ANSWER 0.

📝 **Dig with CF HTTP routes**
1. Use dig to resolve APP_A_ROUTE. Let's call this APP_A_IP
1. Curl that IP. What happens?

1. Use dig to resolve APP_B_ROUTE. Let's call this APP_B_IP
1. Curl that IP. What happens?
❓Why are APP_A_IP and APP_B_IP the same?
❓Why doesn't the IP resolve to either of the apps?

📝 **Dig with CF TCP routes**
1. Use dig to resolve the TCP Route. Let's call this TCP_APP_IP
1. Curl that IP. What happens?
❓Is TCP_APP_IP the same as APP_A_IP?
❓Why or why not?
❓How does traffic get to the apps if the IPs don't work?

🤔 **Sleuthing in your IAAS**
1. In your IAAS GUI, find what infrastructure that these IPs map to.

### Expected Results
All CF HTTP Routes resolve to the same IP. All CF TCP Routes resolve to the same IP. You should find the load balancers that map to these IPs.

## Extra Credit
❓Why does `dig neopets.com` work, but `dig http://neopets.com` does not work?

## Resources
[understanding the dig command](https://mediatemple.net/community/products/dv/204644130/understanding-the-dig-command)

L: tcp-routes
L: questions
---
Route Ports, Backend Ports, App Ports Oh My!

## What
Earlier in this track of TCP work you created a TCP route. This TCP route needed a port.  This is called a route port. There are 3 different types of ports in the Cloud Foundry ecosystem. They can be extremely confusing, so it is nice if there is clear vocabulary so you can easily describe which port you are talking about. 

The three types of ports are:

**Route Port** - this is the port that a client makes a connection to. For HTTP routes, this is always 80 (for http) or 443 (for https). For TCP routes, this port is configured and unique.

**Backend Port** - this is the high number port on the Diego Cell that the GoRouter or TCP Router proxies traffic to. This port is unique per app on each Diego Cell.

**App Port** - this is the port where the app is listening inside of the container. In CF this defaults to 8080.

Let's look at a diagram of these ports.

![ports for TCP traffic](https://storage.googleapis.com/cf-networking-onboarding-images/tcp-trafficflow-ports.png)

![ports for HTTP traffic](https://storage.googleapis.com/cf-networking-onboarding-images/http-traffic-flow-ports.png)

## Questions
❓Look at the help text for mapping a route (`cf map-route --help`). What are the different flags allowed for HTTP routes vs TCP routes?
❓How do these different flags align with what you learned in this story?

## Resources
[configuring app ports in CF](https://docs.cloudfoundry.org/devguide/custom-ports.html)

L: tcp-routes
L: questions
---
TCP vs HTTP Routes

## What

Earlier in this track, you learned that all CF HTTP Routes resolve to the IP of the HTTP load balancer and that all TCP Routes resolve to the IP of the TCP Load balancer.

But if _all_ routes resolve to the IP of a load balancer, then how does traffic actually get sent to an application????

Well, it depends if the route is HTTP or TCP.

One major difference between the HTTP protocol and the TCP protocol, is that HTTP packets contain more headers. These headers can carry all sorts of information, including what URL the request is being made to. (The client can even set arbitrary headers themselves!) GoRouter is able to look at this headers, see the requested URL, and route appropriately based on the information in the Routes Table. Because of this, all HTTP routes can have identical route ports (80 or 443).

![gorouter routing](https://storage.googleapis.com/cf-networking-onboarding-images/gorouter-traffic-routing.png)

TCP is barebones. TCP packets have very limited headers that only include the source and destination ports of the packets. Because of this, the TCP Router has _no_ knowledge of the URL. So the TCP Router needs something else to differentiate between apps. It can't be the destination IP because all TCP routes have the same destination IP. The only thing left, is the destination port. Because of this, all TCP routes must have unique route ports.

![tcp router routing](https://storage.googleapis.com/cf-networking-onboarding-images/tcp-traffic-routing.png)

## How
📝**Inspect HTTP headers**
1. Curl the networking api and look at the request headers
 ```
cf curl /networking/v1/external/policies -v
 ```
 You should get a response that looks like this:
 ```
REQUEST: [2019-04-24T11:01:49-07:00]
GET /networking/v1/external/policies <---- this is the header that contains the URL path
HTTP/1.1
Host: api.beanie.c2c.cf-app.com      <---- this is the header that contains the URL base
Accept: application/json
Authorization: [PRIVATE DATA HIDDEN]
Content-Type: application/json
User-Agent: go-cli 6.43.0+815ea2f3d.2019-02-20 / darwin
 ```

## Question
❓What would happen if two TCP routes had the same route port?

## Resources
[tcp header format](https://www.freesoft.org/CIE/Course/Section4/8.htm)
[http headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)

L: tcp-routes
L: questions
---

TCP Routes Table

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE

## What

The TCP traffic flow is nearly identical to the HTTP traffic flow. The big difference is that instead of an HTTP load balancer there is a TCP load balancer and instead of GoRouter there is a TCP Router.

Go back to the story _Route Propagation - Part 0.2 - HTTP Traffic Overview_ to review this flow.

In the story _Route Propagation - Part 4 - GoRouter_  you learned how to look at the route table for the GoRouter. In this story you are going to look at the analogous route table for the TCP Router.

## How

📝 **Try to list tcp routes**
1. List tcp routes via the routing api.
 ```
cf curl /routing/v1/tcp_routes
 ```
 Most likely you will get the error message:
 ```
{"name":"UnauthorizedError","message":"Token is expired"}
 ```

🤔 **Get correct permissions**
Based on the [routing api docs](https://github.com/cloudfoundry/routing-api/blob/master/docs/api_docs.md#list-tcp-routes), you need to have a client with routing.routes.read permissions.

There is probably already a client deployed with the correct permissions. Find out the name and password for this user from the bosh manifest.

1. Download your manifest
 ```
bosh manifest > /tmp/my-env.yml
 ```

1. Search for `routing.routes.read`

 You should find uaa client properties that look like this:

 ```
 routing_api_client:
            authorities: routing.routes.write,routing.routes.read,routing.router_groups.read
            authorized-grant-types: client_credentials
            secret: ((uaa_clients_routing_api_client_secret))
 ```
The name of the client is: routing_api_client. The password is in credhub under the key uaa_clients_routing_api_client_secret.

1. Use the credhub CLI to get the password.

📝 **Use uaac to get the oath token**

1. Run `uaac` to see if you have the uaa CLI installed.

1. If you don't have it installed, install it.
 ```
gem install cf-uaac
 ```

1. Target your uaa. (To determine this url you can run `cf api` and replace api with uaa.)
 ```
uaac target uaa.<YOUR-ENV>.com
 ```

1. Get the client information for the routing_api_client. It will prompt you for a password.
 ```
uaac token client get routing_api_client
 ```

1. Get the bearer token
 ```
uaac context
 ```
 You will see something like this (this one is truncated):
 ```
client_id: routing_api_client
access_token: eyJhbGciOiJ <------- This is the bearer token that you will need. Yours will be longer.
token_type: bearer
expires_in: 43199
scope: routing.router_groups.read routing.routes.write routing.routes.read
 ```

📝 **Get tcp routes**
1. This time when you curl, pass in the bearer token as a header.
 ```
cf curl /routing/v1/tcp_routes -H "Authorization: bearer BEARER_TOKEN" | jq .
 ```

### Expected Outcome

You should see one TCP route that looks like the one below (this one is edited for brevity):
```
{
    "router_group_guid": "e47c747a-d655-4ea8-5f1a-b59f21ad7852",
    "backend_port": 61004,         <--------- This is the backend port
    "backend_ip": "10.0.1.12",     <--------- This is the Diego Cell IP
    "port": 1025,                  <--------- This is the route port
    "isolation_segment": ""
}
```
## Questions
❓Go back to the story _Route Propagation - Part 4 - GoRouter_  and look at the example HTTP route table entry. What differences do you see between the TCP routes and the HTTP routes?
❓How does this difference match with what you understand about TCP and HTTP?

## Resources
[routing api docs](https://github.com/cloudfoundry/routing-api/blob/master/docs/api_docs.md#list-tcp-routes)

L: tcp-routes
L: questions
---

Why are my TCP routes broken? - Part 0 - Setup

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have completed the "iptables-primer" track
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE:TCP_PORT

## What
When writing these stories, I happened to have a particular deploy setup that caused TCP routes to break and took me awhile to debug. Hopefully this bug is fixed by the time you are reading this, so for this story you're going to have to deploy older code to get the broken state.

## How
🤔**Setup**
1. If your environment is running istio-release 1.2.0 you are good to go. Skip to the next section.
1. Save your current manifest so you can get back to it easily after this story.
 ```
bosh manifest > /tmp/bosh.yml
 ```
1. Deploy with istio-release 1.2.0 and use the [recommended opsfiles](https://github.com/cloudfoundry/istio-release/tree/v1.2.0/deploy/cf-deployment-operations).
   Make sure that the `experimental_enable_ingress_proxy_redirect` property on the `garden-cni` job is set to true

📝 **See it break**
1. See that `curl TCP_ROUTE:PORT` no longer works

🤔 **Now fix it!**
At this point it is a choose your own adventure.
There are 3 stories that you could follow, each with different levels of help text.

Option 1: Solo adventure. No map. No compass. - This option has no hand holding. You are left on your own to find and fix the bug.

Option 2: Some hints. You have a compass, but no map. - This option provides thoughtful questions and hints to probe you in the correct direction.

Option 3: Step by step directions. You have a compass and a map. - This option provides instructions on how to debug each component to figure out which component is broken.

I suggest starting with option 1 or 2 and then looking ahead to options 2 and 3 as you need help.

All of the previous stories that taught you about iptables, tcpdump, routing tables, etc should set you up how to debug CF. However, one thing that will become helpful, but that hasn't been covered previously is [how to get into a container's networking namespace as root](https://cloudfoundry.slack.com/archives/GFXGACQUB/p1554758849206400).

This your chance to put your debugging skills to the test. You can do it.
Ready. Set. Debug!

## Resources
[How to get into a container's networking namespace as root](https://cloudfoundry.slack.com/archives/GFXGACQUB/p1554758849206400)

L: tcp-routes
---
Why are my TCP routes broken? - Option 1 - hard

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE:TCP_PORT

## What
Find the bug that is breaking TCP routes and fix it.
This is option 1. This option has no hand holding. You are left on your own to find and fix the bug.

## How
📝 **See it break**
1. See that `curl TCP_ROUTE:PORT` no longer works

🤔 **Now fix it!**
1. Debug

### Expected Result
Find the bug. Remove the offender and get this TCP route working!
Don't forget to redeploy to your old state afterwards.

## Resources
[How to get into a container's networking namespace as root](https://cloudfoundry.slack.com/archives/GFXGACQUB/p1554758849206400)

L: tcp-routes
---

Why are my TCP routes broken? - Option 2 - medium

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE:TCP_PORT

## What
Find the bug that is breaking TCP routes and fix it.
This is option 2. This option provides thoughtful questions and hints to probe you in the correct direction.

## How
📝 **See it break**
1. See that `curl TCP_ROUTE:PORT` no longer works

🤔 **Now fix it! With hints!**
1. Draw out the data flow for TCP routes include: the client, the load balancer, the TCP router, the Diego Cell, and the app container.
 Any step in that process could be broken.

1. Start at the beginning of the flow and test if the traffic is successfully making it to the next hop.

1. Follow the helpful questions to systematically debug the system.

## Hints
🔎 **helpful questions**
❓Is the URL being resolved to the correct load balancer IP?
❓Is the traffic making it to the TCP router?
❓Does the TCP Router have information about the route in its route table?
❓Is the TCP Router information correct?
❓Is the traffic making it to the Diego Cell?
❓Are the DNAT rules on the Diego Cell correct?
❓Is the traffic making it to the app container?
❓Is the traffic making it to the app process?

🔎 **helpful tools**
- Use dig for DNS resolution. It turns a URL into an IP. See story _Dig-ing TCP and HTTP Routes_ for help.
- Use iptables to see if traffic is being directed somewhere (un)expected. See story _Route Propagation - Part 5 - DNAT Rules_ for help.
- Use tcpdump to see if traffic is making it to a container or vm. See story _Watch your c2c packets with tcpdump_ for help.
- Always look at logs
- See the resources section for how to get in the container's network namespace as root. So you can run all these helpful tool inside of the app container too.

### Expected Result
Find the bug. Remove the offender and get this TCP route working!
Don't forget to redeploy to your old state afterwards.

## Resources
[How to get into a container's networking namespace as root](https://cloudfoundry.slack.com/archives/GFXGACQUB/p1554758849206400)

L: tcp-routes
L: questions
---
Why are my TCP routes broken? - Option 3 - easy...er

## Assumptions
- You have a CF deployed
- The CF was deployed with the help of `bbl`
- You have a TCP server deployed named tcp-app
- You have a TCP route mapped to tcp-app called TCP_ROUTE:TCP_PORT

## What
Find the bug that is breaking TCP routes and fix it.
This is option 3: Step by step directions. You have a compass and a map. - This option provides instructions on how to debug each component to figure out which component is broken.

## How
📝 **See it break**
1. See that `curl TCP_ROUTE:TCP_PORT` no longer works

🤔 **Now fix it! With extra hints!**
1. Here is a diagram of the traffic flow for _working_ TCP routes.
 ![tcp route traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/tcp-trafficflow-ports.png)

 Any step in that process could be broken. Start at the beginning of the flow and test if the traffic is successfully making it to the next hop.

❓Is the URL being resolved to the correct load balancer IP?
1. Use dig with the tcp route to determine if DNS is resolving the route correctly. The IP should match the TCP Load Balancer. Look back at the story _Dig-ing TCP and HTTP Routes_ if you forget how.
 If the IP matches the load balancer, then you know the request is being sent to the correct place. Check the next hop.

❓Is the traffic making it to the TCP router?
1. Ssh onto the TCP router and become root.
1. Look at all network packets where the destination port is the TCP_PORT using tcpdump.
 ```
tcpdump 'port TCP_PORT'
 ```
1. Run  `curl TCP_ROUTE:TCP_PORT` while tcpdump is watching.
If tcpdump is seeing packets, that indicates that the load balancer is correctly proxying traffic to the TCP Router. It also means that the traffic is making it to the TCP Router.

❓Does the TCP Router have information about the route in the its route table?
1. Follow the steps in the story _TCP Routes Tables_ and see if there is an entry for TCP_PORT. If there is, then you know know that the router is being configured. Note down the Diego Cell IP and backend port (BACKEND_PORT).

❓Does the TCP Router's route table entry match app's backend information?
The TCP Router is proxying traffic for TCP_PORT to the Diego Cell IP and BACKEND_PORT, but are the variables correct?
1. CF ssh onto tcp_app and look at the environment variables. Do they match? If so, then you know that the TCP Router is proxying traffic to the correct location.

❓Is the traffic making it to the Diego Cell?
The TCP Router is trying to  proxy traffic to the Diego Cell, but is the traffic making it there?
1. Ssh onto the Diego Cell where tcp_app is running and become root.
1. Use tcpdump to look at traffic destined for BACKEND_PORT.
 ```
tcpdump 'port BACKEND_PORT'
 ```
1. Run  `curl TCP_ROUTE:TCP_PORT` while tcpdump is watching.
If tcpdump is seeing packets, that indicates that the traffic is making it to the Diego Cell.

❓Are the DNAT rules on the Diego Cell correct?
From the story _Route Propagation - Part 5 - DNAT Rules_ you learned that iptables rules redirect traffic that is sent to a Diego cell and backend port and send it to the overlay ip and app port. The same thing happens with TCP routes. Make sure that these DNAT rules are set up correctly.

1. Still on the Diego Cell, look at the DNAT rules.
 ```
iptables -S -t nat
 ```

1. Search through the rules for the one that includes the BACKEND_PORT for tcp_app. If it is set up correctly, then it should look something like this.

 ```
-A netin--b86c9487-2224-4264-47 -d DIEGO_CELL_IP/32 -p tcp -m tcp --dport BACKEND_PORT -j DNAT --to-destination OVERLAY_IP:8080
 ```

1. Compare the overlay IP in the DNAT rule with the overlay IP from the app's environment variables. They should match. If they match then this indicates that traffic should be correctly redirected to the correct app container.

❓Is the traffic making it to the app container?
1. Using the link in the resources, start a bash session inside of the networking namespace of the app.
1. Run `tcpdump`.
1. Run  `curl TCP_ROUTE:TCP_PORT` while tcpdump is watching.
1. Ctl + C out of tcpdump in the app container. (Idk why, but tcpdump doesn't tail correctly when you are in a network namespace like this. So you need to kill it before you will see the output.)
If you see traffic, that means that the packets are making to inside of the app container.

❓Is the traffic making it to the app process?
1. Still inside of the app's network namespace, look at the DNAT rules. Are there any rules here that are sending all tcp 8080 traffic to somewhere else?
1. Delete the bad DNAT rule and try curling the route again.
1. Success!

### Expected Result
Find the bug. Remove the offender and get this TCP route working!
Don't forget to redeploy to your old state afterwards.

## Resources
[How to get into a container's networking namespace as root](https://cloudfoundry.slack.com/archives/GFXGACQUB/p1554758849206400)

L: tcp-routes
L: questions
---
[RELEASE] TCP Routes ⇧
L: tcp-routes
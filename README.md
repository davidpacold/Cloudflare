# Cloudflare
Cloudflare Workers project

Deliverables:
1. A working application:
a. "Working" means that the site is reachable over an Argo Tunnel, and successfully
returns all HTTP request headers in the body of the response. 

b. A running Cloudflare Worker that achieves the specified logic. - Completed with the check for the cookie - the worker code is posted in the GitHub

c. Any instructions needed to access any of the above.
  The service can be accessed via several methods;
  directly -  nocfpoc.davidpacold.comanything?<Key>=<Value> 
  via Cloudflare network - poc.davidpacold.app/anything?<Key>=<Value>
  via a Cloudflare tunnel - argoapp.davidpacold.app/anything?<Key>=<Value>
Secure website accessible at poc.davidpacold.app/admin
    Username / Password: Admin/Admin


2. A report that addresses the following:
● A summary detailing how you implemented the technical requirements.  

```
                                                                                           
+-----------+                                                 +---------------------------+
| Google    |                                                 |macOS Host                 |
| Domain    |                      +--------------------------|        +---------+--------+
| Registrar | ------------------+  |                          |        |         | Argo   |
| & DNS     |                   |  |                          |        |HA Proxy |Tunnel  |
+-----------+                   |  |                          |        |         |        |
   |                      +-----------------+                 |        +---------+--------+
   |                      |                 |                 |        | VMware Container |
   |                      |   Cloudflare    |                 |        | Service          |
   |   +------------------|   Platform      |                 |        |                  |
   |   |                  |                 |                 |        |     +------------+
   |   |                  +----/---|----\---+                 |        |     |            |
   |   |                      /    |     \                    |        |     | webservice |
+------------+               /     |      \                   |        |     | httpbin    |
|            |              /      |       \                  |        |     |            |
|            |    +----------+----------+----------+          |        |     |            |
| User       |    |          |          |          |          +--------+-----+------------+
|            |    |CF Workers| CF DNS   |CF Tunnel |                 |                     
|            |    |          |          |Service   |                 |                     
|            |    +----------+----------+----------+                 |                     
|            |                                                       |                     
|            |                                                       |                     
|            |-------------------------------------------------------+                     
+------------+                                                                             
```

The diagram above is meant to illustrate the solution I have implemented in a logical and easy to consume manner - not necessary reflecting the ports / protocols / connection orders etc, but to show the technical stack and the relationships between the different components. 

The httpbin web service is accessible via 3 world accessible endpoints 

nocfpoc.davidpacold.com
poc.davidpacold.app
argoapp.davidpacold.app

First the endpoint nocfpoc.davidpacold.com has a DNS entry hosted as part of the domain registration hosted by Google. 
This DNS entry points to the IP address of my home router, where I have configured port 80 and 443 to forward onto my MacBook Pro. This connection is accepted by the HA Proxy service where the connection is forwarded from port 80 to port 8080 that the web service is actually listening on. 

This connection flow will bypass the Cloudflare network and allow us to experiment with and understand the different headers that are included in the request made over the Cloudflare network based connections which we will explore later on. 

A trace route of this domain confirms that my traffic stays within my ISP network.

The endpoint poc.davidpacold.app has the domain registered and hosted by Google as well, but domain configuration in Google has delegated the Name Servers to Cloudflare. In the Cloudflare network I have configured an A record that points to the IP address of my home router.

A trace route of this domain confirms that my traffic now flows into the Cloudflare Network. 

Finally the endpoint argoapp.davidpacold.app has a similar configuration where the domain is registered and hosted by Google, but the domain configuration for davidpacold.app has delegated the Name Servers to Cloudflare. For the argoapp host, rather than an A record being configured, I have configured a CNAME record that points to a host in the cfargotunnel.com domain. 

The host in the cfargotunnel.com domain is the service endpoint that the argo tunnel service running on my Macbook is made world accessible from. So by navigating to argoapp.davidpacold.app the request is translated and passed to the Cloudflare Tunnel Service, where is then sent over the established tunnel to my macbook.
  


As the user makes a connection to the site via the URLs that are routed though the Cloudflare network either via the traditional path, or via the Argo Tunnel, there is an additional action performed along the way. When the path of the url includes either /anything, /admin or /logout the Cloudflare platform invokes a Cloudflare Worker and runs a bit of Javascript to evaluate the request. In the case of the /anything being present in the url path, a Worker is invoked that checks for the user agent that is present in the request. If the user agent includes, bot, PostmanRuntime/7.28.4 (the version of Postman I happen to have) the Worker will block the resource request and respond with that notice. If the user agent is Mozilla/5.0, the agent that Chrome and Safari happen to present, it will be allowed and routed on to the web service I host. Finally if the user agent happens to be cURL, they will be redirected to the Cloudflare developers website. There is one additional check that applies to the cURL and Postman checks, and that is looking for the presence of a cookie in the request. Should the cookie key value pair be equal to: “cf-noredir”=“true” then instead of being blocked or redirected, the request will be allowed to pass though to the web service. 

For the case where the URL path includes /admin or /logout, there is actually no backend web service supporting those endpoint, rather those requests are fully satisfied within the Cloudflare worker environment. There are some excellent examples of how to use Cloudflare Workers on their documentation site located here:https://developers.cloudflare.com/workers/
To make this example even easier to understand how it was implemented solely in the Workers environment there happens to be an pre-made example that we can take advantage of:
https://developers.cloudflare.com/workers/examples/basic-auth

By using this sample script, we can create a new worker that will handle routing to the /auth and /logout paths, and the Worker will be invoked and our user will be challenged with a Authentication request when loading the /auth endpoint. 

I found the documentation and presence of example code to be very helpful and allowed someone who has limited use of server-less computing an easy on ramp to doing some pretty neat things.

In the diagram there are several other technologies listed that I used to complete my solution. I briefly mentioned the Argo Tunnel or Cloudflare Tunnel services. This is a service that you can leverage to make your applications accessible via the Cloudflare network with out having to acquire or deploy and configure traditional Internet facing equipment, like Public IP addresses, firewalls, web application firewalls, load balancers, etc.. The Argo tunnel is a simple to configure and run service that maps your internal apps to an address in the Cloudflare platform. We can then access that Cloudflare address, and have a secure connection back to the resource. One of the big differences in using the tunnel vs a traditional web app deployment, is the direction of the connections from the public space into the private network space. Historically, those connections have gone from a less secure (public internet) to a privileged (your private) network space. When leveraging a tunnel the connection direction is flipped and the connection is made originating from your private network and going outbound to the Cloudflare platform. This is a much more secure method of making something world accessible.

In my testing and verification of the different configurations, I can see this working, by removing the port-forwarding on my home network, so the url requests to nocfpoc.davidpacold.com and poc.davidpacold.app will fail. However, the url requests to argoapp.davidpacold.com will succeed. 



As noted in the diagram and briefly in the above description, there are several other technologies used in my deployment. 

The Connection between the User and the Cloudflare network is secured with a SSL certificate provided by LetsEncrypt, and can be seen in the users browser when navigating to the poc.davidpacold.app service. The connection between the Cloudflare network and my server is secured with a SSL certificate provided by the Cloudflare Origin CA certificate service. This presented an issue that I needed to overcome. Due to an earlier deployment choice I had made, I had to deploy a very basic instance of HA Proxy. 

I had seen the availability of the httpbin service as a Docker container, and decided to use this as an opportunity to learn and try out a feature update to the VMware Fusion application. In a prior release of the VMware Fusion application the ability to run and deploy not only traditional VM's but Container based applications was added. Knowing this, and the availability of the httpbin app as a docker image, seemed like a quick and easy way to get the service up and running. 

This was indeed a very quick way to get the needed application up and going, but presented an issue when trying to add the mentioned SSL Certificate. Due to the docker image based app, i was unable to configure it directly with the required certificate. Thats where HA Proxy came to the rescue. I used HA Proxy as a front end to the Docker Based web service, that presented the correct SSL certificate and terminated the HTTPS session and then did basic HTTP from the HA Proxy to the httpbin service. With that configuration, all services turned green and worked. 




● Did you learn anything new during this assignment?

This exercise definitely cleared a few cobwebs from my mind and I really enjoyed the challenge of solving it. There was a totally new tool to me that I used, the VMware Fusion Container Service, while I was aware of it, this was the first time I had spent some time deploying an app with it. So there was a bit of a learning curve there. 

On the Certificate management front, I was familiar with Let's Encrypt, and had even generated certificates with them in the past. However I had always used the default cypher methods, so with the requirement of the homework to use EC based key-pairs, I had to do some research on that process. 

Finally the part that took the most trial and error, and research was writing the Javascript for the Workers configuration. I found the editors, the code sandbox and reference examples very useful, to quickly try / fail / update / retest 



● How did you fill the gaps (if any) in your knowledge during the process?

As I noted above, I found the Cloudflare docs to be very handy, the sample code snippets were fantastic starting points to get me off and running on a new platform. 

Aside from the docs pages, I used Google for help with the Lets Encrypt and Certbot tools to generate the EC based key pairs for SSL certs, and some just plain old trial and error of different configurations. 

Additionally it always helps me to draw things out on a notepad to diagram the flows and connections to better understand if something isn’t working, where the issue is. 


● What was your experience with Argo Tunnels and Cloudflare Workers?

Argo tunnel was really easy to configure and deploy. Where I got a little tripped up was due to the solution design I had created, I wanted multiple URLs with different routing paths to the same endpoint. I had to ensure my SSL certs had SAN attributes that would support all the different hostnames. 

The Cloudflare Workers were a very easy way to consume and initiate server-less workloads. It was really nice that they supported multiple languages, since I had a basic understanding of Javascript, and the examples were writing in it, I chose to use that Language to write my scripts in. 

In the case of my /admin and /logout pages, I thought the idea of the Workers handling the inbound requests completely on their own with no other backend server was a great way to deploy functions on the internet with out having to allocate or deploy on-prem or cross cloud infrastructure. Abstracting that idea from just my simple /admin and /logout pages, consider the ability to deploy lots of other functionality of modern web applications with no other additional infrastructure, while a big change in the concepts of app development and deployment, the iteration and functionality it provides like A/B testing is really exciting. 


○ What issues did you run into (if any) and how did you overcome them?

The biggest issue I ran into was some SSL configuration items when configuring the Cloudflare network to Origin server. This was due to the design design of deploying the web service (httpbin) as a docker container. When the Docker image was deployed, I was unable to get into the container to set up the SSL certificates to ensure the connection worked as expected. 

To resolve the issue of the web-service being HTTP only, I did a very basic HAProxy deployment that could serve the SSL certificate as expected, and then do SSL termination and forward on the traffic as regular HTTP traffic to my internal service. This allowed me to use the required SSL configurations, with the same backend app I had deployed. 



○ How would you describe each product in simple language?

Cloudflare Workers allow you to run applications, or snippets of applications, such as individual functions, on top of the global Cloudflare network. By taking advantage of Cloudflare Workers, your code runs close to the user, scales to meet demand automatically, deploy and iterate quickly with no infrastructure to maintain. 

Cloudflare Tunnels allow you to deploy applications securely no matter where they are, Internal network, Lab systems or other Cloud platforms. By deploying your applications with Cloudflare Tunnels you get the ability to manage your application traffic with Cloudflare WAF, and your Tunnel deployed applications also benefit from the broad security tools Cloudflare is known for, such as DDoS protection. 



○ What use cases you can see each product being useful for?

I think in both cases I can see the programmability of the Workers and Tunnel, the access to manage them via APIs as huge wins for organizations that are either moving too or have adopted the ideas of a Continuous integration and delivery pipeline. As they iterate builds, build up / tear down micro services, or move functions out of the application and into the server-less model of the Workers, the speed of deployment is a huge win. 

Over the last 2 years as we have seen a mass distribution of users out of corporate networks and onto their home networks, the ability to deploy the Tunnel based apps, and Worker based apps to get closer to them while retaining a secure posture is very important. So things like workers where the functions are distributed / executed as close the user as possible improves the experience of the person consuming the app.
 

● Would HTTP response headers be different if you were not using Cloudflare?
Yes - In fact due to the same endpoint being accessible in 2 ways, over the Cloudflare network, and directly, as discussed above we can very easily run the same test that echos back the headers and we can easily identify the different headers that are present in the Cloudflare based connection. 


Here is an example of the Cloudflare fronted response:
```
{
  "args": {
    "name": "david"
  }, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", 
    "Accept-Encoding": "gzip", 
    "Accept-Language": "en-US,en;q=0.9", 
    "Cdn-Loop": "cloudflare; subreqs=1", 
    "Cf-Connecting-Ip": "71.199.128.136", 
    "Cf-Ew-Via": "15", 
    "Cf-Ipcountry": "US", 
    "Cf-Ray": "6aab8d2e03f6f1aa-ATL", 
    "Cf-Visitor": "{\"scheme\":\"https\"}", 
    "Cf-Worker": "davidpacold.app", 
    "Host": "poc.davidpacold.app", 
    "Sec-Ch-Ua": "\"Google Chrome\";v=\"95\", \"Chromium\";v=\"95\", \";Not A Brand\";v=\"99\"", 
    "Sec-Ch-Ua-Mobile": "?0", 
    "Sec-Ch-Ua-Platform": "\"macOS\"", 
    "Sec-Fetch-Dest": "document", 
    "Sec-Fetch-Mode": "navigate", 
    "Sec-Fetch-Site": "none", 
    "Sec-Fetch-User": "?1", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.69 Safari/537.36"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "71.199.128.136", 
  "url": "https://poc.davidpacold.app/anything?name=david"
}
```
And here is an example of the direct access response:
```
{
  "args": {
    "Name": "David"
  }, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "en-US,en;q=0.9", 
    "Host": "nocfpoc.davidpacold.com", 
    "Sec-Ch-Ua": "\"Google Chrome\";v=\"95\", \"Chromium\";v=\"95\", \";Not A Brand\";v=\"99\"", 
    "Sec-Ch-Ua-Mobile": "?0", 
    "Sec-Ch-Ua-Platform": "\"macOS\"", 
    "Sec-Fetch-Dest": "document", 
    "Sec-Fetch-Mode": "navigate", 
    "Sec-Fetch-Site": "none", 
    "Sec-Fetch-User": "?1", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.69 Safari/537.36"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.16.179.1", 
  "url": "http://nocfpoc.davidpacold.com/anything?Name=David"
}
```

By compairing the two responses we can identify the different headers are:
```
    "Cdn-Loop": "cloudflare; subreqs=1", 
    "Cf-Connecting-Ip": "71.199.128.136", 
    "Cf-Ew-Via": "15", 
    "Cf-Ipcountry": "US", 
    "Cf-Ray": "6aab8d2e03f6f1aa-ATL", 
    "Cf-Visitor": "{\"scheme\":\"https\"}", 
    "Cf-Worker": "davidpacold.app", 
    "origin": "172.16.179.1" (when going direct)
    "origin": "71.199.128.136" (when going over the CF network route)
```

● How do you imagine that a target customer will find this experience?

I would imaging someone on the customer side who works in this space to find doing something similar a good experience. I thought the challenge unsupported by a Cloudflare team member was quite satisfying to reflect on, definitely something that by reading the docs they could work though as I did. I imagine the experience of doing the same thing with a guided hand from a Cloudflare team member would no doubt show how easy it is to build some really high performing apps. 


Please let us know if you have any questions with any of the above!
# LAB-Project---Resolve-Connectivity-Issues

This lab project was created as a simple showcase of how I can troubleshoot networking issues. 

Scenario: Ubuntu Server is unable to connect to the internet. Cannot pull security patches.

Tools Used:
- Ping | Traceroute | SecureCRT

#TROUBLESHOOTING EXPLANATION and SOLUTION

When reading the issue described by the ticket, I decided to approach this issue from the end device, which means starting at the Ubuntu Server and working my way to other networks. To start, I tested connectivity to the default gateway, as this is the primary means by which this device can reach other networks. I used the ping command and was successful. At this point, I had a choice of using ping to continue testing connectivity, one device and command at a time. However, I chose to use the traceroute command as this command allows me to test connectivity with multiple devices in a single command. The results of this test depicted a back-and-forth of replies. These replies were from the 10.10.80.1 and 10.10.80.254 interfaces. One was on Router 4, and another on Router 5.
 
Since the replies came from these devices and no other device responded to the traceroute, then it meant that the Ubuntu Server had connectivity with those two devices and no other. I needed to look further into the configurations and routes on the Routers. I discovered that Router 4 only had one static and no dynamic routes. Not only that, but the static route was a default route that was forwarding all traffic to Router 5. This configuration explained the back-and-forth behavior of the traceroute responses received. I re-configured the default route, ensuring all incoming traffic is forwarded to Router 3 instead of Router 5. However, with only this static route, I knew there would need to be more to resolve the issue. I tested connectivity to Router 4's neighbor routers, Router 3 and Router 5. This test was successful.
 
At this point, I had the option of configuring static routes on Router 4 and Router 5. However, I wanted to see the routes configured on Router 3 and Router 5. I looked at their configurations and discovered that Router 3 is configured with the dynamic routing protocol OSPF. This meant that instead of creating multiple routes statically, I could configure OSPF on Router 4 and Router 5 so that they may learn these routes dynamically. I configured OSPF on the respective routers and tested connectivity to the 10.10.60.0/24 network. Each test was successful. With the tests succeeding, I finally tested connectivity from the Ubuntu Server to exterior networks. I pinged the 10.10.60.0/24 network. It was successful. Lastly, I performed a traceroute to the 10.10.20.0/24 network, ending with a final firewall response at the 10.10.60.254 IP address. With these results, I determined the issue to be resolved.


#TROUBLESHOOTING STEPS:
#“Ubuntu_Server”

Step 1.	Login to “Ubuntu_Server” and ping Default Gateway of 10.10.90.254. Success!

Step 2.	Use the >traceroute command to the 10.10.20.0/24 network and determine where the route has an issue. Specific command used: traceroute 10.10.20.1

      i. Results: 
	      1.	Response from 10.10.90.254
	      2.	Response from 10.10.80.254
	      3.	Response from 10.10.80.1
	      4.	2 more no response followed by a response from 10.10.80.254.
	      5.	At this point the command goes into what appears to be a loop.
       
Step 3.	The results indicate an issue with the routes between Routers 4 and 5. Specifically, 
              Router 4 may be missing a route to the 10.10.70.0/24 network or with its default route.


![Picture1](https://github.com/user-attachments/assets/92dccb72-6292-4522-9c92-2a386268f251)

![Picture2](https://github.com/user-attachments/assets/5060f9c3-a365-4515-a152-927d0d008586)


“Routers”

Step 4.	Login to Router 4. We need to inspect the current routing configuration and determine 
              what may be missing or misconfigured. To see all the current running configuration on 
              the Vyos Router I type -> configure -> show. Here I look for a few things:
      i.	IP addresses of the interfaces
      ii.	Static and dynamic routes that were defined.
      
Step 5.	I can see that “eth7” interface is assigned the 10.10.80.254/24 Ip address and “eth0” interface is assigned the 10.10.70.1/24 IP address. This means that Router 4 should be able to reach at least Router’s 3 and 5. I run the ping commands to confirm.

      i.	Results:	
      1.	Ping to Router 5 (10.10.80.1): Success!
      2.	Ping to Router 3 (10.10.70.254): Success!
      
Step 6.	The results inform me that I can communicate with both neighbor routers. Next, I inspect the routing table on Router 4. Using the -> show protocols command I can see the current routing configuration. Only 1 static route exits in router 4 and it is a default route.

      i.	Route:
          1.	0.0.0.0/0 -> next hop 10.10.80.1 
	  
Step 7.	This default route will forward all packets received to Router 5. So, any traffic received, even traffic originating from Router 5, after reaching Router 4 will be forwarded to Router 5. This is the cause for the loop discovered by the traceroute before.

Step 8.	Update default route on Router 4 so the next hop is instead Router 3. 

      i.	Updated default route: 0.0.0.0/0 next-hop 10.10.70.254 
      
Step 9.	Delete the previously discovered default route via 10.10.80.1.

Step 10. At this point, I suspect that traffic will still not leave the network as there is only one route configured on the router; however other routes are required to proceed.

Step 11. Login and Inspect routers 3 and 5 to determine which routes must be configured. Use the -> configure -> show command on each router and compare protocols.

      i.	Router 5 contains only one route. It is a default static route to 10.10.80.254. More routes are required here.
      ii.	Router 3 contains two routes. One is a default, static route to 10.10.60.254. The other is an OSPF configuration on area 0, configured to allow all configured ports to forward and received traffic.

![Picture3](https://github.com/user-attachments/assets/cec80d75-f20a-40b9-9059-7caee8827d11)


Step 12. The discovery of the dynamic routing protocol OSPF on Router 3 allows me to determine the next necessary step. To resolve the issue, Router 4 and Router 5 require an OSPF configuration so that the routers learn the correct routes dynamically throughout the network.

“Router 4”
Step 13. Configure OSPF on router 4 allowing all configured interfaces to participate in route sharing.

      i.	Command: set protocols ospf area 0.0.0.0 network 0.0.0.0/0
      ii.	Command: commit
      
Step 14. Show the current router on Router 4 again. -> configure ->show protocols.

      i.	Result:
          1.	OSPF: area 0.0.0.0 network 0.0.0.0/0
          2.	Static: 0.0.0.0/0 next-hop 10.10.70.254
	  
Step 15. Try to ping the 10.10.60.0/24 network.

      i.	Command: > ping 10.10.60.1
      ii.	Result: Success!

![Picture4](https://github.com/user-attachments/assets/1df99803-7418-447d-b4e4-99be38e76661)

      
Step 16. Now Router 4 can access other networks. Router 5 must be configured for OSPF next.

“Router 5”
Step 17. Login to Router 5. Configure this router for OSPF using the same command previously used on Router 4.

      i.	Command: set protocols ospf area 0.0.0.0 network 0.0.0.0/0
      ii.	Command: commit
      
Step 18. Show the currently configured routes.

      i.	 >show protocols
      
Step 19. Attempt to ping the 10.10.60.0/24 network.

      i.	Command: ping 10.10.60.1
      ii.	Result: Success!

![Picture5](https://github.com/user-attachments/assets/a7ec54e6-1adb-441e-9ad5-c4945dabd112)

Step 20. With successful results, all that is left is to test connectivity to other devices from the “Ubuntu Server”.
“Ubuntu Server”
Step 21. Login to “Ubuntu Server”. Test connectivity to other devices using the ping and 
               traceroute commands.
      i.	Command: ping 10.10.60.1. Result: Success!
      ii.	Command: traceroute 10.10.20.1. 
      1.	Result: Success. Response from:
        a.	10.10.90.254
        b.	10.10.80.254
        c.	10.10.70.254
        d.	10.10.60.254
Step 22.	With successful connection tests to devices on other networks, the issue is resolved.

![Picture6](https://github.com/user-attachments/assets/92517e8f-a443-40de-845c-6646af76f822)

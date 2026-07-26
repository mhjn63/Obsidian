
[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

Which component of the Elastic Stack represents the interface analysts use to visualize log data?
```
Kibana
```
Which Elastic component would you use to manage the agents deployed across your client's infrastructure?
```
Fleet Server
```
Which port does the Kibana web interface run on by default?
```
5601
```
Access the Kibana menu in the top left of the Kibana home page. What is the name of the first section that is listed?
```
Analytics
```
What is the name of the default integration used to collect system metrics and log data from your host?
```
System
```
In your VM terminal, create a new user with useradd testuser.
In Discover, enter the query process.name: `useradd` to highlight the `useradd` events.
What is the `event.dataset` field value of the associated logs?
```
system.auth
```
Add testuser to the sudoers group with `gpasswd -a testuser sudo`.
Enter the query process.name: `gpasswd` to highlight the log.
What is the full message field value for the event created?
```
user testuser added by root to group sudo
```
What is the event.dataset field value for the Apache access logs?
```
apache.access

```
Head to `/secret.html` on the TryHatMe site.
Using Discover, investigate the user_agent.original field from the recent requests.
What is the hidden flag value?
```
THM{access_log_secrets!}
```
Investigate the newly ingested VPN log data. Who is the most active user on the network?
```
s.summer
```
What is the source.ip of the user you identified in the previous question?
```
72.14.24.1
```
Create a new Pie chart visualization using the search `event.action: "auth_fail"`.  
In Slice by, add the `user.name` field and set the Metric to `Count`.  
What percent of `auth_fail` events does the user `p.mallow` account for? (e.g., 82.7%)
```
62.5%
```
Using your dashboard, experiment with creating a visualization for the `vpn.server.region` field.  
Which field value represents the least accessed `vpn.server.region`?
```
us-west-1
```
Investigate the `vpn.client.ip` field.  
Which `vpn.client.ip` was only logged 26 times within the available data?
```
10.10.10.116
```





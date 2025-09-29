---
title: Migrating k3s single stack cluster to dual stack (and new IP range)
author: Wilf Silver
pubDatetime: 2025-09-29T16:00:00Z
slug: migrating_cluster
featured: true
draft: false
tags:
  - k8s
  - ipv6
  - k3s
description: I managed to successfully migrate my 4 node cluster to a dual stack and new IP range, here is how I managed to do it
---

## Premise

I initially set up my cluster as a test, this test expanded quickly and had "production" applications put on it (specifically used by other people), which basically means that if I lose data, it affects more people than I would like. However, when initially setting it up I made a few mistakes:

1. It was set up in a single stack configuration (as you have to specifically opt in to dual stack). This means that even though I updated my router to hand out IPv6es to my nodes, it did not accept IPv6 communications (though I later found a setting in traefik which might have fixed it anyway).
2. The main two original nodes had (imo) bad names, and so I wanted to rename them.
3. The IP range for my router was `192.168.1.1/20` which may look not too bad (except the `/20`) but the main issue was that my VPN caused issues for some devices on some networks (because of IP collisions) and so I wanted to move to a new IP range (Specifically `10.0.0.1/20`). This causes more issues as in K3S an IP address is considered static and cannot be changed once a node is added.

And so I planned to fix these all in one go. In hindsight, probably the most problematic fix was the node renaming, due to longhorn requiring you to set them up as a new node (and causing so many issues).

As a summary I do love this quote for this whole process, please do also take this as a warning:

> this is not something we test or support
>
> -- <cite>[@brandond on GitHub](https://github.com/k3s-io/k3s/discussions/11280#discussioncomment-11246118)</cite>

### Basic cluster overview

My setup is very similar to the base k3s install, changing nothing (except disabling `traefik` so I can configure my own). Then for storage I use [longhorn](https://longhorn.io/) and then [velero](https://velero.io/) to back up to an onsite and offsite s3 storage.

In terms of node structure:

* I have one control node (sadly, but I plan to be changing this in the future) on bare metal
* 3 agent nodes, one on bare metal and 2 in a VM

All of which run Fedora Server, meaning they have a GUI accessible through `9090`. Speaking of which, if you are attempting this yourself, make sure you have access to each node over SSH all times and maybe even physical access. I performed this remotely, but did need physical access as one node decided to randomly turn off (but once turned back on it was fine).

> [!NOTE]
> The most important thing I found to have set up was tailscale to all devices with VPN access and advertising routes for both old and new IP range.

To make my life easier, I also have two DNSes, one public (pihole) one in the cluster (and of course the CoreDNS for internal use) and one backup one on the router. You should note that during this time, the pihole DNS on the cluster will NOT be accessible (basically) and so you want to make sure that all key hostnames are also stored on the DNS on your router (or on a computer that is not part of the cluster). Before this migration, I moved all computers on the network to not use the DNS server in the cluster.

## Getting IPv6 addresses on your network

As many people will have done this in the past, I will keep this brief. But for a dual stack cluster to work as I wanted it to, all nodes required an IPv6 address.

Firstly, you will need to check that your internet service provider supports IPv6, then in the router settings there should be an option to use DHCPv6 and a prefix delegation size. Note that you will probably want a static IPv6, as K3S does not like when a node's IP address changes. You should refer to your provider to determine what prefix delegation you should use, for me, it was 64. With ubiquity, it was under the "Internet" tab and choosing "WAN1", though this seems to change a lot.

Next, you will then have to pass these IPv6 addresses to the nodes in your cluster. For this, you will have to configure IPv6 for your network, which you can use DHCPv6 and specify a range.

Once this is done, when you restart the network on your devices, they should be given IPv6 addresses, and you can search for [ipv6 testing website](https://test-ipv6.com/) to check that it is working.

## BACKUP

> [!WARNING]
> Backing up is probably the most important step to this whole thing, due to the removal of each node, you may lose all your data.

Cack the data up, and I really would not recommend using k3s base storage, **please migrate all your storage into a cluster friendly storage option like longhorn**. This personally saved me, but also it did then cause more issues due to the data redundancy on the different nodes.

For backups, I used both longhorns inbuilt backup tool and velero (making sure all important data was successfully backed up, as it doesn't particularly like pvs). You should also test that your backups do work before continuing with this migration,  as there is a large chance you will have to use it.

I did have to rely on longhorns backup for some persistent volumes due to going too fast and nodes going down randomly during this transition.

## Plan (with hindsight)

So this is a basic plan with a few tweaks from my experience, but mostly worked pretty well, but there were problems as discussed in the later section.

1. Setup IPv6 addresses on all nodes (as discussed above).
2. Setup tailscale on all nodes, all advertising routes on both `192.168.0.0/20` and `10.0.0.0/20`, this allows when VPNing in to access both networks in the transition
   ```sh
   sudo tailscale up --advertise-exit-node --advertise-routes="192.168.0.0/20,10.0.0.0/20"
   ```
3. Scale up your CoreDNS deployment to at least 2 nodes (to make sure that it is up as much of the time as possible)
4. Prepare the router for the transition, my one had some requirements.
5. Cut off access to the cluster
6. Make a backup of all volumes
7. Added: scale down all pods that use a volume to 0. As discussed later, longhorn nearly crashed my entire cluster trying to figure out what was wrong -- though please note this is not tested and potentially could stop replication to other nodes in the cluster (which would be unfortunate)
8. Optional: remove the control node from the cluster `kubectl delete node ...`, protecting from if it loses the original IPv4
9. Change the IP range.

> [!NOTE]
> Once the IP range has changed you cannot restart any of the nodes (especially the control node), as otherwise it will refuse to boot


Once the IP range, each node individually has to be removed and re-added, starting with the control node (due to if it loses its IPv4 it's doomed):

1. Assign a new static IP address for the node in the router (I mostly did this so I could do the conversion without looking it up), making sure to update any local DNS with this new IP.
2. Restart NetworkManager (or other) to gain **additional** IP address. Note that you should still have an IP address in the old range.
3. Optional: change hostname
4. Install K3S:

   ```sh
   # For the control node, include dual stack args (making sure --flannel-ipv6-masq is specified)
   curl -sfL https://get.k3s.io | sh -s - server --cluster-cidr=10.42.0.0/16,2001:db8:42::/56 --service-cidr=10.43.0.0/16,2001:db8:43::/112 --flannel-ipv6-masq --node-ip=<new-ipv4-here>,<ipv6-here> ...

   # For agent nodes
   curl -sfL https://get.k3s.io | K3S_URL=https://k8s.mydomain.com:6443 K3S_TOKEN=mytoken::server::something sh -s - agent --node-ip=<new-ipv4-here>,<ipv6-here> ...
   ```

   > [!NOTE]
   > The use of `--node-ip` to make sure that it doesn't get assigned the wrong IP address (as the node should currently have two IPv4 addresses)

5. Update `k3s-shutdown` (which gracefully shutdown my nodes) with new node name and maybe new control node name/ip
6. If renamed node, setup storage on longhorn and remove delete the previous node in the admin UI.
7. **Wait**. Make sure that any replication of the data has completed before continuing.

I found that some nodes randomly went down (in longhorn's view, but were still up), so I just prioritised those next.

Once all the nodes were re-added, all the problems started to appear, but as long as K3S was up and running, everything else can be fixed. It is just a matter of time.

You will also want to restart the nodes at some point as well to shed the trailing IPv4 address and maybe even reinstall `k3s` without `--node-ip` option, to keep the manual config as small as possible. From doing this myself, I found that IPv6 addresses on the cluster are, in fact, less static than IPv4 (due to some of my nodes having two IPv6 addresses and me picking the wrong one initially).

## Problems

### Nothing is attaching and longhorn is basically unresponsive

I found that longhorn managers got stuck trying and failing to recover volumes that had faulted. Additionally, to this, the `driver-deployer` pod was failing to get responses from `longhorn-backend` and the pods with storage were all saying they can't attach storage due to time-outs. This personally took me around 3 hours to fix due to the multitude of different errors which were happening at the same time. But I did manage to qualm the storm by doing the following:

* As mentioned before, scaling down all nodes which used storage to zero. This would mean that all "attach" and "detach" requests stopped overwhelming longhorn, but you will also probably want to make sure "Offline Replica Rebuilding" was turned off in the settings, so the longhorn managers don't continue to try and recover the nodes.
* Additionally, it seemed that `longhorn-backend` was failing to be queried from the DNS, and so I scaled up and down the CoreDNS as appropriate (mostly making sure that it could be consistently queried as I think some of my nodes were not liking hosting the DNS, but I fixed this with updating some of the K3S parameters)
* After doing this, I was able to wait about 30 minutes before the longhorn UI become much more responsive, and I could perform actions again such as recovering for backups where the volumes had faulted or that there were no nodes with the volume.

Once the UI was responsive, I was more confident to scale back up deployments/statefulsets which required volumes (making sure to only do two at a time to make sure that longhorn had time to handle the requests).

### Nodes randomly shutdown

I have not actually found the reason for this, but two nodes shutdown soon after this migration and were inaccessible, though they are configured to automatically turn on if there is power available.

However manually turning them back on made them come back online and have yet to randomly shutdown again (and it has been nearly a week as of writing this).

## Post migration

Once you've redeployed, you can add back any port forwarding you disabled/correct the IP address, and we can start testing this IP, but first we must update traffic.

Using the helm chart you must change `service.spec.ipFamilyPolicy` from `SingleStack` to `PreferDualStack` or `RequireDualStack` and `service.spec.ipFamilies` to include both `IPv4` and `IPv6` so it looks like:

```yaml
# ...

service:
  spec:
    ipFamilyPolicy: RequireDualStack
    ipFamilies:
      - IPv4
      - IPv6

# ...
```

This makes it that traefik listens to both IPv4 and IPv6 requests. This potentially could have solved the whole wanting to get IPv6 working on my cluster without having to make it Dual Stack, but I haven't verified nor looked into it. You should then be able to request the IPv6 address of a node and get a 404 response.

To then allow the whole internet to access your cluster nodes via IPv6, you will have to edit your router's firewall to accept whatever ports (e.g. `80` and `443`) to your nodes IP addresses. Personally, I only made two nodes in my cluster accessible via the whole internet just in case of a DDoS, I wanted to make sure that some of my nodes would still be accessible for any debugging or solutions.

## Conclusion

I found this surprisingly easy once I had figured out the limitations and how to work around them. The only issue being longhorn, which definitely took a while to fix due to trying to figure out what was going wrong. But overall this went quite well.

If you are doing something similar, I wish you luck though.

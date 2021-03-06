# Router for Mininet

- Routing table:

We start by reading the data from rtable.txt. We then sort it with
QuickSort that has complexity on average case: O (n * log (n)). Sorting is
will be done based on the prefix and mask as follows:

* ascending: prefix & mask
* in case of a tie, ascending according to the mask

Part of a routing table would look like this:


`    192.168.0.0 192.168.1.2 255.255.255.0 1 43200`\
`    192.168.0.0 192.168.1.2 255.255.255.128 1 43200`\
`    192.168.0.0 192.168.1.2 255.255.255.192 1 43200`\
`    192.168.0.0 192.168.1.2 255.255.255.224 1 43200`\
`    192.168.0.0 192.168.1.2 255.255.255.240 1 43200`\
`    192.168.0.0 192.168.0.2 255.255.255.248 0 43200`\
`    192.168.2.0 192.168.3.2 255.255.255.128 3 174272`\
`    192.168.2.0 192.168.3.2 255.255.255.192 3 174272`\
`    192.168.2.0 192.168.3.2 255.255.255.224 3 174272`\
`    192.168.2.0 192.168.3.2 255.255.255.240 3 174272`\
`    192.168.2.0 192.168.2.2 255.255.255.248 2 174272`


We are looking for the best_route entry in the table.
Binary search will return the first entry it finds a match
(prefix & mask = dest_ip & mask) -> might not be the best match though.
So we have to go through the following entries in the table
(because the entry matching the largest mask
will be the last), until we find an entry that does not fit.
The one before that will be the best match.

### The actual implementation

in a while (1), in which we receive packets:

* check the type of package received:

    - it's an IP packet (has ether_type 0x800):

        - if it has the wrong checksum: drop
        - if the lifetime does not allow forwarding (ttl <= 1): drop
        - we check if the package is for us (for one of the router interfaces)
        - we are looking for the best route, applying the procedure explained above
        * if there is no best_route, it means that the destination is
        unreachable by the router: send an ICMP packet
        back to the source, with the required fields
        * if there is a best_route:
        - update ttl and checksum
        - we look in the ARP table for the MAC address to which we must forward the package.
        * If the package is for us the MAC will be one of the router's interfaces, 
        not in the ARP table
        * If we do not find it and we are not the destination of the package,
        we put the packet in the queue and send an ARP request on the interface (output)
        We move on, we receive the new package. If it will be an ARP
        reply with the desired MAC, we go to the branch "it is an ARP package"
        * If we find it, we are not the destination => forward
        (we modify the source MACs (with the MAC of the interface on which the package comes out) and
        destination (with MAC next_hop)
        * Otherwise: we are the destination:
        - check if it is an ICMP package, if not: drop
        - initialize the ICMP header, check the checksum
        - wrong checksum: drop
        - is an ICMP ECHO request package? No => drop, Yes =>
        we build an ICMP ECHO reply and send it on the same interface


    - it's an ARP package (has ether_type 0x806):

        - initialize the ARP header

        * If operation code == 1 => it's an ARP request for the router => we will
        we need to send the MAC of one of our interfaces (we check who has the
        requested IP (target_ip), extract the MAC, build
        an ARP reply packet and send it back, on the interface we have
        received the request

        * If operation code == 0 => it's an ARP reply => it means we receive the
        MAC for which we made an ARP request earlier => it's time to send
        the packages in the queue. We initialize the headers, find
        target_mac in the ARP table (we know for sure we will find it now), update ttl,
        modify ether_dhost and ether_shost, find the best route. If we do not find a route,
        make an ICMP package with unreachable destination and send it
        back. If we find it, we will send the package.



 chef sets the hostname to clustername-uuid, where the uuid is the one seen in the orchestrator ssh menu.  This makes it
 possible to correlate logs back to their hosts after extracting them to another system for analysis.  The same chef recipe
 adds the local hostname to /etc/hosts on each host to allow local name resolution.  There is no interaction with dns, none
 should be required.  Nothing outside the host is aware of the hostname. All cluster services are configured by IP.

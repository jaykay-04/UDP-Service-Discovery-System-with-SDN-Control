# UDP-Service-Discovery-System-with-SDN-Control
UDP-based service discovery system with SDN integration. Servers advertise over UDP with TTL heartbeats, a registry tracks live services, and clients discover and invoke them. When a server crashes, the registry alerts a Ryu controller, which installs an OpenFlow drop rule on the switch; recovery removes it automatically. Built with Mininet + Ryu.

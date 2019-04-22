# docker-canary
A lightweight container designed to watch for whether it's public IP represents a known `CLEARNET_IP` and, as a result, restart a VPN container.

Bit of a specific usecase for people leveraging the [OpenVPN Client](https://github.com/dperson/openvpn-client) trick (where you connect containers to a `service` network to force all their traffic to go out via that VPN).

Relies upon the `$CLEARNET_IP` environment variable (which should represent the known non-VPN IP you want to ensure is never exposed).

## Usage
In a `docker-compose.yml` file you'd set this up as follows:

```
version: "3.6"
services:
  # Use this for the ability to force other containers
  # to route all their traffic through a VPN connection
  # https://github.com/dperson/openvpn-client
  # Any other containers you want routed through the VPN
  # should have this added in their section.
  # network_mode: "service:openvpn"
  # and then add their ports to the OpenVPN container to
  # expose (if wanted).
  openvpn:
    container_name: 'openvpn'
    image: dperson/openvpn-client
    # cap_add, security_opt, and volume required for the image to function
    cap_add:
      - net_admin
    environment:
      - DNS  # forces DNS resolvers from provider
      - FIREWALL  # if VPN drops then kill all traffic
    networks:
      - default
    read_only: true
    tmpfs:
      - /run
      - /tmp
    restart: unless-stopped
    security_opt:
      - label:disable
    stdin_open: true
    tty: true
    volumes:
      - /dev/net:/dev/net:z
      - ${USERDIR}/docker/vpn:/vpn
      # Put .ovpn configuration file in the /vpn directory (in "volumes:" above or
      # launch using the command line arguments, IE pick one:
      #  - ./vpn:/vpn
      # command: 'server;user;password[;port]'
  # Use the canary image to run a query and ensure we can only access internet via
  # a VPN. Check the IP address from the command when containers are up.
  canary:
    container_name: 'canary'
    image: saracen9/docker-canary
    environment:
      - CLEARNET_IP=${CLEARNET_IP}
    network_mode: "service:openvpn"
    volumes:
      # We're going to allow this container to control others
      # (i.e. we can start/stop them based on VPN status)
      - /var/run/docker.sock:/var/run/docker.sock
```

Example log output (see `openvpn` container restarting)

```
canary        | Sun Apr 21 21:42:18 UTC 2019: Container IP is 82.24.111.21
canary        | Sun Apr 21 21:42:29 UTC 2019: ## Entering Canary mode
canary        | Sun Apr 21 21:42:35 UTC 2019: Container IP is 82.102.27.171
canary        | Sun Apr 21 21:42:35 UTC 2019: VPN IP address not detected
openvpn       | Sun Apr 21 21:42:35 2019 event_wait : Interrupted system call (code=4)
openvpn       | Sun Apr 21 21:42:35 2019 SIGTERM received, sending exit notification to peer
...
openvpn       | Sun Apr 21 21:42:40 2019 SIGTERM[soft,exit-with-notification] received, process exiting
openvpn exited with code 0
openvpn       | Sun Apr 21 21:42:43 2019 TLS: Initial packet from [AF_INET]82.102.27.170:443, sid=74875991 eab00940
...
openvpn       | Sun Apr 21 21:42:51 2019 Initialization Sequence Completed
canary        | Sun Apr 21 21:42:35 UTC 2019: Attempting restart of OpenVPN Client
canary        | Sun Apr 21 21:42:53 UTC 2019: Restarting canary
canary exited with code 137
canary        | Sun Apr 21 21:43:03 UTC 2019: Container IP is 82.102.27.171
canary        | Sun Apr 21 21:43:19 UTC 2019: ## Entering Canary mode
```

## TODO
At the moment the containers to restart are hardcoded into the `start-canary` script. Going forwards it would be best to separate out this into a separate, user-mountable, file as all containers will lose connectivity once the vpn is restarted.
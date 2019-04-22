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

## TODO
At the moment the containers to restart are hardcoded into the `start-canary` script. Going forwards it would be best to separate out this into a separate, user-mountable, file as all containers will lose connectivity once the vpn is restarted.
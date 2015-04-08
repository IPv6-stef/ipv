1. Make sure there are no running radvd processes.
   # ps -ef | grep radvd

2. Create admin tenant network objects:
   Via admin tenant:
   a. Create external network with subnet.
      # neutron net-create ipv6_ext_net --router:external=True

   b. Create subnet and attach it the external network.
      * Please note that the subnet and gateway address may change according to your setup configurations.
      # neutron subnet-create <external_net_id>2001:db8:0::2/64 --name ipv6_ext_net_subnet --gateway 2001:db8:0::1 --ipv6-ra-mode slaac --ipv6-address-mode slaac --ip-version 6

   c. Create a neutron router.
      # neutron router-create router1

   d. Set the IPv6 external network as the router default gw.
      # neutron router-gateway-set <router-id> <external-network-id>
 
The Logic behind this step is that currently there is a known limitation with Nova Metadata (cloud-init) service, which does not supported IPv6, meaning: All instances that need cloud-init will have to be connected both to IPv6 and IPv4 networks.Create Tenants and IPv4 Networks

3. a. Create  tenants (tenant_a) with user.
      # keystone bootstrap --user-name tenant_a --pass 123456 --role-name tenant_a --tenant-name tenant_a
      

   b. Grant admin privileges for each user to it's tenant. 
      # keystone user-role-add --user tenant_a --role admin --tenant tenant_a
      


4. Create tenant_a tenant network objects:
   Via tenant_a:
   a. Create IPv4 internal network (name it internal_ipv4_a).
      # neutron net-create internal_ipv4_a --shared

   b. Create IPv4 subnet and attached to internal_ipv4_a
      # neutron subnet-create <internal_ipv4_a_id>192.168.1.0/24 --name internal_ipv4_a_subnet --ip-version 4



5. Add router interfaces to all Networks.
   Via admin tenant:
    Add the IPv4 networks
    # neutron router-interface-add <router1_id> <internal_ipv4_a_id>
    
[RADVD][SLAAC] Basic functionality and In-Tenant Connectivity
6. Create tenant_a RADVD SLAAC DHCPv6 network:
   Via tenant_a:
   a. Create IPv6 internal network (name it tenant_a_radvd_slaac).
   # neutron net-create tenant_a_radvd_slaac --shared

   b. Create IPv6 RADVD SLAAC DHCPv6 subnet and attached to tenant_a_radvd_slaac
      * For detailed usecases explanation, see IPv6 address assignment usecases To be tested at:https://mojo.redhat.com/docs/DOC-985240 
      # neutron subnet-create<tenant_a_radvd_slaac_id> 2001:db3:0::2/64 --name tenant_a_radvd_slaac_subnet --ipv6-ra-mode slaac --ipv6-address-mode slaac --dns-nameserver 2001:4860:4860::8888 --ip-version 6


7. Add router interfaces to all Networks.
   Via admin tenant:
    Add the IPv6 networks
    # neutron router-interface-add <router1_id><tenant_a_radvd_slaac_subnet_id>

Make sure there are no running radvd processes.
   # ps -ef | grep radvd

8. Boot Two instances and link them both to IPv4 and IPv6 networks.
   Via tenant_a:
   a. Generate a key pair.
      # nova keypair-add tenant_a_keypair > tenant_a_keypair.pem
      # chmod 600 tenant_a_keypair.pem

   b. Boot two instances (Upload in image to glance if needed).
      # nova boot tenant_a_instance_radvd_slaac --flavor m1.small --image <image_id> --min-count 2 --key-name tenant_a_keypair --security-groups default --nic net-id=<internal_ipv4_a_id> --nic net-id=<tenant_a_radvd_slaac_id>



9. Check for the following network flows:
   
   a. Ping the other instance IPv6 address, In our example: 2001:db3::f816:3eff:fe93:429f (use ping6). 

   b. Ping the IPv6 default GW, In our example: 2001:db3::1 (use ping6).

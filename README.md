# GCP base lab

## Introduction

This repo helps you build a simple Lab environment in GCP with a single VPC, an Ubuntu VM, Cloud Router for Interconnect, and VPN.

You can use it for diverse scenarios like interconnecting with other cloud providers such as Azure or AWS, or by emulating an on-premises environment and testing interconnectivity with other remote networks. You pretty much can use and expand based on your creativity and needs. Enjoy it.

## Prerequisite

You are required to use GCP CLI (gcloud) by using either of these two options:

1) Run commands using [GCP cloud shell](https://shell.cloud.google.com)
2) Install GCP CLI (gcloud) on your Windows or Linux machine by following instructions: [Installing the gcloud CLI](https://cloud.google.com/sdk/docs/install)

:point_right: Tip: When running on Linux elevate your shell to root (-s).

## Lab steps

1 - Define your variables. Change the values below based on your needs:

```bash
# Define your variables
project=angular-expanse-327722 #Set your project Name. Get your PROJECT_ID use command: gcloud projects list 
region=us-central1 #Set your region. Get Regions/Zones Use command: gcloud compute zones list
zone=us-central1-c # Set availability zone: a, b or c.
vpcrange=192.168.0.0/24 # Set VPN CIDR
envname=onprem #Enviroment Name you want to create
vmname=vm1 #VM Name
mypip=$(curl -4 ifconfig.io -s) #Gets your Home Public IP or replace with that information. It will add it to the Firewall Rule for remote access to your VM.
```

2 - Create VPC, Firewall Rules and creates Ubuntu VM

```bash
#Create VPC
gcloud compute networks create $envname-vpc --project=$project --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create $envname-subnet --project=$project --range=$vpcrange --network=$envname-vpc --region=$region

#Create Firewall Rule
gcloud compute firewall-rules create $envname-allow-traffic-from-azure --network $envname-vpc --allow tcp,udp,icmp --source-ranges 192.168.0.0/16,10.0.0.0/8,172.16.0.0/16,35.235.240.0/20,$mypip/32 --project=$project

#Create Unbutu VM:
gcloud compute instances create $envname-vm1 --project=$project --zone=$zone --machine-type=f1-micro --network-interface=subnet=$envname-subnet,network-tier=PREMIUM --image=ubuntu-1804-bionic-v20220126 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-balanced --boot-disk-device-name=$envname-vm1 
```

## Hybrid connectivity (Interconnect or VPN)

For hybrid connectivity you have two options: Interconnect or VPN, or both depending on your needs.

### Interconnect with Partner

1) Create Cloud Router and Interconnect with Partner (Megaport, Equinix and others)

```bash
#Cloud Router: #***********Validate************
gcloud compute routers create $envname-router --project=$project --region=$region --network=$envname-vpc --asn=16550

#DirectConnect with Connectivity Partner:
gcloud compute interconnects attachments partner create $envname-vlan --region $region --edge-availability-domain availability-domain-1 --router $envname-router --admin-enabled --project=$project

```

2 - Get the pair key on the output above to setup connection with GCP interconnectivity partner. Example:

```bash
Please use the pairing key to provision the attachment with your partner:
      890c4350-XXXX-YYYY-ZZZZ-db2b6de5768b/us-central1/1
```

### VPN

Note: the scope for this VPN gateway is classic (single instance) and uses static routing.

1 - Create VPN Gateway and forwarding rules for IPSec (ESP,IKE and NAT-T)

```bash
gcloud compute target-vpn-gateways create $envname-vpn-gw --project=$project --region=$region --network=$envname-vpc 
gcloud compute addresses create $envname-vpn-pip --project=$project --region=$region
gcloud compute forwarding-rules create $envname-vpn-rule-esp --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=ESP --target-vpn-gateway=$envname-vpn-gw-gw 
gcloud compute forwarding-rules create $envname-vpn-rule-udp500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=500 --target-vpn-gateway=$envname-vpn-gw-gw 
gcloud compute forwarding-rules create $envname-vpn-rule-udp4500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=4500 --target-vpn-gateway=$envname-vpn-gw
```

2 - Get VPN Public IP information to be configured in the other VPN device side:

```bash
gcloud compute addresses describe $envname-vpn-pip --region=$region --project=$project --format='value(address)'
```

3) Create GCP VPN tunnel to the other side VPN device

```bash
# Define Variables = Please add your custom values on the lines below
sharedkey=@password@  #specify your own share key
peervpnpip=1.1.1.1 #specify Peer VPN Public IP address
destcidr=10.0.0.0/8 #specify remote VPN network to be reached.
vpntunnelname=vpn-to-remote-site-a #specify the tunnel name, usually the remote site name.

gcloud compute vpn-tunnels create $vpntunnelname --project=$project --region=$region --peer-address=$peervpnpip --shared-secret=$sharedkey --ike-version=2 --local-traffic-selector=0.0.0.0/0 --remote-traffic-selector=0.0.0.0/0 --target-vpn-gateway=$envname-vpn-gw 
gcloud compute routes create $vpntunnelname-route-1 --project=$project --network=$envname-vpc --priority=1000 --destination-range=$destcidr --next-hop-vpn-tunnel=$vpntunnelname --next-hop-vpn-tunnel-region=$region
```

4) Check VPN connection status

```bash
gcloud compute vpn-tunnels describe $vpntunnelname \
   --region=$region \
   --project=$project \
   --format='flattened(status,detailedStatus)'
```

## Clean up

```bash
gcloud compute vpn-tunnels delete $vpntunnelname --region $region --quiet
gcloud compute routes delete $vpntunnelname-route-1 --quiet
gcloud compute forwarding-rules delete $envname-vpn-rule-esp --region $region --quiet
gcloud compute forwarding-rules delete $envname-vpn-rule-udp500 --region $region --quiet
gcloud compute forwarding-rules delete $envname-vpn-rule-udp4500 --region $region --quiet
gcloud compute target-vpn-gateways delete $envname-vpn --region $region --quiet
gcloud compute addresses delete $envname-vpn-pip --region $region --quiet
gcloud compute instances delete $envname-vm1 --project=$project --zone=$zone --quiet
gcloud compute firewall-rules delete $envname-allow-traffic-from-azure --quiet
gcloud compute networks subnets delete $envname-subnet --project=$project --region=$region --quiet
gcloud compute networks delete $envname-vpc --project=$project --quiet
```

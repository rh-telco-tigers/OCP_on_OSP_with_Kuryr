# OpenShift with Kuryr on top of OpenStack

# Introduction to Kuryr CNI
Kuryr is a CNI plugin using Neutron and Octavia to provide networking for pods and services. It is primarily designed for OpenShift clusters running on OpenStack virtual machines. Kuryr improves the network performance.

Kuryr is recommended for OpenShift deployments on encapsulated OpenStack tenant networks in order to avoid double encapsulation: running an encapsulated OpenShift SDN over an encapsulated OpenStack network i.e whenever VXLAN/GRE/GENEVE are required. Conversely, implementing Kuryr does not make sense when provider networks, tenant VLANs, or a 3rd party commercial SDN such as Cisco ACI or Juniper Contrail are implemented.

## Kuryr Architecture
Kuryr components are deployed as pods in the kuryr namespace. The kuryr-controller is a single container service pod installed on a node as a deployment OpenShift resource type. The kuryr-cni container installs and configures the Kuryr CNI driver on each of the OpenShift masters and compute node as a daemonset. The Kuryr controller watches the OpenShift API server for pod, service and namespace create, update and delete events. It maps the OpenShift API calls to corresponding objects in Neutron and Octavia. This means that every network solution that implements the Neutron trunk port functionality can be used to back OpenShift via Kuryr. This includes open source solutions such as OVS and OVN as well as Neutron-compatible commercial SDNs. That is also the reason the OpenvSwitch firewall driver is a prerequisite instead of the ovs-hybrid firewall driver when using OVS.

For more deatils please refer to [kuryr-kubernetes](https://github.com/openstack/kuryr-kubernetes/blob/master/doc/source/devref/kuryr_kubernetes_design.rst) github page

# Installing Red Hat Openstack Platform 16.2 with Octavia
For testing purpose we will be deploying All-in-One OSP deployment using tripleo. For steps please refer to [this project](https://github.com/rh-telco-tigers/OSP16.2-AIO)

# Deploying Red Hat Openshift Container Platform 4.8

We will use IPI method to deploy OCP on top of OSP. Prerequiste for installing OCP on OSP are
- Red Hat OSP 16.2 installed with octavia
- A project in which OCP instances will be created
- Project should meet the minimum quota
- A user who has access to project and also have swiftoperator role in the project

# Creating environment in OSP

1. Creating project in Openstack
   ```bash
   [stack@aio ~]$ openstack project create --description 'Openshift 4.8' ocp48 --domain default
   [stack@aio ~]$ openstack project list
   [stack@aio ~]$ openstack project list
   +----------------------------------+---------+
   | ID                               | Name    |
   +----------------------------------+---------+
   | 2b1201670e264cb98a68f8f52a07c581 | service |
   | 618498a4b6e54bc7b4fbc45ffefe3852 | admin   |
   | 636beab46fdf4e7da8b150b9c3d9c01f | ocp48   |
   +----------------------------------+---------+
   ```
2. Create a user and make it a member of the project and assign swiftoperator role to it.
   ```bash
   [stack@aio ~]$ openstack user create --project ocp48 --password PASSWORD ocpadmin
   +------------+----------------------------------+
   | Field      | Value                            |
   +------------+----------------------------------+
   | email      | None                             |
   | enabled    | True                             |
   | id         | 6322872d9c7e445dbbb49c1f9ca28adc |
   | name       | new-user                         |
   | project_id | 636beab46fdf4e7da8b150b9c3d9c01f |
   | username   | new-user                         |
   +------------+----------------------------------+
   [stack@aio ~]$ openstack role add --user ocpadmin --project ocp48 swiftoperator
   ```
3. Modify the quota for project ocp48
   ```bash
   [stack@aio ~]$ sudo openstack quota set --secgroups 250 --secgroup-rules 1000 --ports 1500 --subnets 250 --networks 250 ocp48
   ```
4. Create two floating ip and add records in DNS
   ```bash
   [stack@aio ~]$ openstack floating ip create --description "API" ext-net
   [stack@aio ~]$ openstack floating ip create --description "Ingress" ext-net
   ```
# Deploying Openshift Cluster
1. Create the install-config for the cluster
   ```bash
    $ ./openshift-install create cluster --dir=pluto --log-level=debug
   ```
2. Update the install-config.yaml
   ```yaml
   apiVersion: v1
   baseDomain: home.lab
   compute:
   - architecture: amd64
     hyperthreading: Enabled
     name: worker
     platform: {}
     replicas: 3
   controlPlane:
     architecture: amd64
     hyperthreading: Enabled
     name: master
     platform: {}
     replicas: 3
   metadata:
     creationTimestamp: null
     name: pluto
   networking:
     clusterNetwork:
     - cidr: 10.128.0.0/14
       hostPrefix: 23
     machineNetwork:
     - cidr: 10.0.0.0/16
     networkType: Kuryr
     serviceNetwork:
     - 172.30.0.0/16
   platform:
     openstack:
       lbFloatingIP: 192.168.50.237
       ingressFloatingIP: 192.168.50.212
       cloud: standalone
       defaultMachinePlatform:
         type: ocp.small
       externalDNS: [192.168.50.1,192.168.1.1]
       externalNetwork: ext-net
       trunkSupport: true
       octaviaSupport: true
   publish: External
   pullSecret: '<pull secret from cloud.redhat.com>'
   sshKey: |
     <your ssh public key>
   ```
3. Deploy cluster
   ```bash
   $ ./openshift-install create cluster --dir pluto --log-level debug
   ......
   time="2021-09-21T15:54:16-04:00" level=debug msg="Cluster is initialized"
   time="2021-09-21T15:54:16-04:00" level=info msg="Waiting up to 10m0s for the openshift-console route to be created..."
   time="2021-09-21T15:54:16-04:00" level=debug msg="Route found in openshift-console namespace: console"
   time="2021-09-21T15:54:16-04:00" level=debug msg="OpenShift console route is admitted"
   time="2021-09-21T15:54:16-04:00" level=info msg="Install complete!"
   time="2021-09-21T15:54:16-04:00" level=info msg="To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/Users/aaggarwa/Work/Lab/ocp-clusters/osp/ocp4.8/pluto/auth/kubeconfig'"
   time="2021-09-21T15:54:16-04:00" level=info msg="Access the OpenShift web-console here: https://console-openshift-console.apps.pluto.home.lab"
   time="2021-09-21T15:54:16-04:00" level=info msg="Login to the console with user: \"kubeadmin\", and password: \"QSKqk-6qNTa-RQz4Y-B3juj\""
   time="2021-09-21T15:54:16-04:00" level=info msg="Time elapsed: 0s"
   ```
4. Once the cluster is deployed successfully. Login to OSP dashboard and notice that each pod has its own network
   ![OSP Network](https://github.com/rh-telco-tigers/OCP_on_OSP_with_Kuryr/blob/main/images/OSP-Network.png)
5. Load balancers for each namespace
   ![OSP LB](https://github.com/rh-telco-tigers/OCP_on_OSP_with_Kuryr/blob/main/images/OSP-LB.png)
6. Now let us deploy a sample app
   ```bash
   $oc new-project test-project
   $oc new-app --git https://github.com/openshift/nodejs-ex.git   
   ```
7. After successfull deployment we will see a network and LB created in Openstack
      

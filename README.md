# IBM Kubernetes Service private server endpoints proxy

## Objective

This is a automated way to configure a bastion host within your VPC so that it is able to forward traffic to the private service endpoints of Red Hat OpenShift on IBM Cloud (ROKS) or even IBM Kubernetes Service (IKS) clusters. This is particularly useful when using a VPN to connect to the VPC from an enterprise network as today, the IP addresses for the private service endpoints for the cluster control plane can not be tunneled across the VPN.

## Prerequisites

You will need to have a Red Hat OpenShift on IBM Cloud cluster created in a gen2 VPC with the cluster configured for private service endpoints only. In the same VPC, you will need a bastion host vm with CentOS 8 or Red Hat Enterprise Linux 8 where you will use this playbook to install and configure the proxy.

## Usage

1. To configure the variables in the [vars/main.yaml](./vars/main.yaml) for your environment you will need to identify the hostname and ports used for the cluster. From a workstation logged in with the IBM Cloud CLI run:

    ```console
    $ ibmcloud oc cluster get --cluster my-private-cluster --output json | jq -r '.masterURL'
    https://c104-e.private.us-east.containers.cloud.ibm.com:32359
    ```

    Here the host name is `c104-e.private.us-east.containers.cloud.ibm.com` and the Cluster API port is `32359`. This is one of the two ports that need to be proxied for the cluster. To get the other port, query this API endpoint for the Oauth endpoint:

    ```console
    $ curl -sk -XGET  -H "X-Csrf-Token: 1" 'https://c104-e.private.us-east.containers.cloud.ibm.com:32359/.well-known/oauth-authorization-server' | grep token_endpoint
    "token_endpoint": "https://c104-e.private.us-east.containers.cloud.ibm.com:30652/oauth/token"
    ```

    The second port to forward is `30652` in this example. Edit the [vars/main.yaml](./vars/main.yaml) file and update the `backend` and the `ports` array with your values.

1. Edit the `hosts` to add your bastion machine IP.

1. Run the playbook with :

    ```bash
    ansible-playbook -i hosts -u root playbook/main.yaml
    ```

1. After running the playbook, you'll need to update the host resolution for systems on the network that uses VPN to connect to the VPC. This can be done through an internal DNS that overrides the public name resolution for your cluster endpoint hostname. Alternately, this can be done by editing the workstations `hosts` file which is located in `/etc/hosts` on MacOS and Linux and in `C:\Windows\System32\drivers\etc\hosts`. For either add a line with the IP address of the bastion host internal to the VPC, for example:

    ```text
    10.241.0.8  c104-e.private.us-east.containers.cloud.ibm.com
    ```

    Once added, the workstation should be able to use both `oc login` with an IBM Cloud API key for either a user in the account or a service ID with permissions to access the cluster. For more information see [Managing API keys](https://cloud.ibm.com/docs/account?topic=account-manapikey_)

    ```console
    oc login -u apikey -p <IBM Cloud API key> --server=https://c104-e.private.us-east.containers.cloud.ibm.com:32359
    ```

    It will also be possible to open the OpenShift Console for the cluster using the IBM Cloud web user interface.

## Use with multiple clusters

This approach is applicable for multiple clusters in a region as long as the hostname for the Cluster API and Oauth endpoint is the same. To add support for additional clusters, just add their port values to the `ports` array in [vars/main.yaml](./vars/main.yaml).

## Tested on

This has been tested and works on:

- CentOS 8

## License & Authors

If you would like to see the detailed LICENCE click [here](./LICENCE).

- Author: Tim Robinson <timro@us.ibm.com>

```text
Copyright:: 2020- IBM, Inc

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

[haproxy 1.8 Intro]: https://cbonte.github.io/haproxy-dconv/1.8/intro.html

## Acknowledgements

A big tip of the hat to JJ Ashgar who recently published a git repo with striking similarity required to what was needed for this project. Check it out at [tigervnc-crc-playbook](https://github.com/jjasghar/tigervnc-crc-playbook)

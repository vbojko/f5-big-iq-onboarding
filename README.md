/!\ **Under development** /!\

Performs series of on-boarding steps to bootstrap a BIG-IQ system
to the point that it can accept configuration.

This can be used for *lab*, *proof of concept* or *production* BIG-IQ deployments.

It provides examples of 3 type of configurations, small, medium or large which can be adapted depending on customer's network/size/requirements.

Consult the [Planning and Implementing a BIG-IQ Centralized Management Deployment](https://techdocs.f5.com/kb/en-us/products/big-iq-centralized-mgmt/manuals/product/big-iq-centralized-management-plan-implement-deploy-6-1-0.html) for for details.

![Deployment Diagram](./images/diagram_onboarding.png)

Once the inventory files are set with the necessary information (IP, license, dns, ntp, ...), the Ansible playbooks can be launched from your local machine or a remote linux machine, as long as you have network connectivity to the management IP addresses of the targeted BIG-IQ instances to onboard/configure.

BIG-IQ Onboarding with Docker and Ansible
-----------------------------------------

1. Choose your configuration:

    - **Small**: 1 BIG-IQ CM standalone, 1 BIG-IQ DCD
    - **Medium**: 1 BIG-IQ CM standalone, 2 BIG-IQ DCD
    - **Large**: 2 BIG-IQ CM HA, 3 BIG-IQ DCD

2. Deploy BIG-IQ instances in your environment.

    - [AWS](https://aws.amazon.com/marketplace/pp/B00KIZG6KA?qid=1495059228012&sr=0-1&ref_=srh_res_product_title)
    - [Azure](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/f5-networks.f5-big-iq?tab=Overview)
    - [VMware](https://downloads.f5.com/esd/eula.sv?sw=BIG-IQ&pro=big-iq_CM&ver=6.1.0&container=v6.1.0&_ga=2.95373976.584487124.1557161462-1415455721.1549652512)
    - [Openstack](https://downloads.f5.com/esd/eula.sv?sw=BIG-IQ&pro=big-iq_CM&ver=6.1.0&container=v6.1.0&_ga=2.200814506.584487124.1557161462-1415455721.1549652512)
    - [HyperV](https://downloads.f5.com/esd/eula.sv?sw=BIG-IQ&pro=big-iq_CM&ver=6.1.0&container=v6.1.0&_ga=2.133130250.584487124.1557161462-1415455721.1549652512)

    Go to the [BIG-IQ Knowledge Center](https://support.f5.com/csp/knowledge-center/software/BIG-IQ?module=BIG-IQ%20Centralized%20Management&version=6.1.0) and follow the setup guide.

    Number of instances to bring up:

    - **Small**: 2 BIG-IQ instances (1 for CM, 1 for DCD)
    - **Medium**: 3 BIG-IQ instances (1 for CM, 2 for DCD)
    - **Large**: 5 BIG-IQ instances (2 for CM, 3 for DCD)

    Public Cloud deployments ([AWS](https://techdocs.f5.com/kb/en-us/products/big-iq-centralized-mgmt/manuals/product/big-iq-centralized-management-and-amazon-web-services-setup-6-0-0.html)/[Azure](https://techdocs.f5.com/kb/en-us/products/big-iq-centralized-mgmt/manuals/product/big-iq-centralized-management-and-msft-azure-setup-6-0-0.html)):

    - Deploy the instances with min 2 NICs
    - Create an EIP and assign it to the primary interfaces for each CM instances
    - Make sure you have the private key of the Key Pairs selected
    - Copy your private key in the under the f5-bigiq-onboarding directory and name it ``privatekey.pem``and apply correct permission ``chmod 600 privatekey.pem``
    - Configure the network security group for the ingress rules on each instances

      *Example for AWS: (10.1.1.0/24 = VPC subnet, sg-06b096098f4 = Security Group Name, 34.132.183.134/32 = [your public IP](https://www.whatismyip.com))*

      Ports | Protocol | Source 
      ----- | -------- | ------
      | 80  | tcp | 10.1.1.0/24 |
      | 443 | tcp | 10.1.1.0/24 |
      | 22 | tcp | 10.1.1.0/24 |
      | 0-65535 | tcp | sg-06b096098f4 |
      | All traffic | all | 34.132.183.134/32 |      
  
3. From a linux machine with access to the BIG-IQ instances.

    - Install [Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/) ([AWS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html) or [Azure](https://docs.docker.com/docker-for-azure/))
    - Install [Git](https://git-scm.com/download/linux)

    Example for Amazon Linux EC2 instance:
    ```
    sudo yum update -y
    sudo amazon-linux-extras install docker -y
    sudo service docker start
    sudo yum install git -y
    ```

    Then, clone the repository:

    ```
    git clone https://github.com/f5devcentral/f5-big-iq-pm-team.git
    ```

4. Update the ansible inventory files with the correct information (management IP, self IP, BIG-IQ license, master key, ...).

    Notes:
    
    - It is not recommended to set ``bigiq_onboard_discovery_address`` for deployment in AWS or Azure (the management IP address will be used automatically if not set). In this case ``register_dcd_dcd_listener`` should be set to the DCD management IP (``bigiq_onboard_server``)
    - ``bigiq_onboard_server`` in AWS and Azure should be the private IP address assigned to eth0

  ```
  cd f5-big-iq-pm-team/f5-bigiq-onboarding
  ```

  - if **Small** selected, edit:

  ```
  vi inventory/group_vars/bigiq-global.yml
  vi inventory/group_vars/bigiq-cm-01.yml
  vi inventory/group_vars/bigiq-dcd-01.yml
  vi inventory/hosts
  ```

  - if **Medium** selected, edit:

  ```
  vi inventory/group_vars/bigiq-global.yml
  vi inventory/group_vars/bigiq-cm-01.yml
  vi inventory/group_vars/bigiq-dcd-01.yml
  vi inventory/group_vars/bigiq-dcd-02.yml
  vi inventory/hosts
  ```

  - if **Large** selected, edit:

  ```
  vi inventory/group_vars/bigiq-global.yml
  vi inventory/group_vars/bigiq-cm-01.yml
  vi inventory/group_vars/bigiq-cm-02.yml
  vi inventory/group_vars/bigiq-dcd-01.yml
  vi inventory/group_vars/bigiq-dcd-02.yml
  vi inventory/group_vars/bigiq-dcd-03.yml
  vi inventory/hosts
  ```

5. Build the Ansible docker images containing the F5 Ansible Galaxy roles.

  ```
  sudo docker build . -t f5-bigiq-onboarding
  ```

  Validate Docker and Ansible are working correctly: (Ansible version should be displayed)

  ```
  sudo docker run -t f5-bigiq-onboarding ansible-playbook --version
  ```

5. Change default shell on all instances to bash, and set the admin's password.

  ```
  ./ansible_helper ansible-playbook /ansible/playbooks/bigiq_onboard_pretasks.yml -i /ansible/inventory/hosts
  ```

6. Execute the BIG-IQ onboarding playbooks.

  - if **Small** selected, run:

  ```
  ./ansible_helper ansible-playbook /ansible/playbooks/bigiq_onboard_small_standalone_1dcd.yml -i /ansible/inventory/hosts
  ```

  - if **Medium** selected, run:

  ```
  ./ansible_helper ansible-playbook /ansible/playbooks/bigiq_onboard_medium_standalone_2dcd.yml -i /ansible/inventory/hosts
  ```

  - if **Large** selected, run:

  ```
  ./ansible_helper ansible-playbook /ansible/playbooks/bigiq_onboard_large_ha_3dcd.yml -i /ansible/inventory/hosts
  ```

7. Open BIG-IQ CM in a web browser by using the management private or public IP address with https, for example: ``https://<bigiq_mgt_ip>``.

8. If you choose the **Large** configuration, go to the [BIG-IQ Knowledge Center](https://techdocs.f5.com/kb/en-us/products/big-iq-centralized-mgmt/manuals/product/big-iq-centralized-management-plan-implement-deploy-6-1-0/04.html) to configure HA.

9. Start managing BIG-IP devices from BIG-IQ, go to the [BIG-IQ Knowledge Center](https://techdocs.f5.com/kb/en-us/products/big-iq-centralized-mgmt/manuals/product/big-iq-centralized-management-device-6-1-0/02.html#concept-3571).

For more information, go to the [BIG-IQ Knowledge Center](https://support.f5.com/csp/knowledge-center/software/BIG-IQ?module=BIG-IQ%20Centralized%20Management&version=6.1.0).


Miscellaneous
-------------

- In case you need to restore the BIG-IQ system to factory default settings, follow [K15886](https://support.f5.com/csp/article/K15886) article.

- Disable SSL authentication for SSG (**LAB/POC only**):

```
echo >> /var/config/orchestrator/orchestrator.conf
echo 'VALIDATE_CERTS = "no"' >> /var/config/orchestrator/orchestrator.conf
bigstart restart gunicorn``
```

Troubleshooting
---------------

n/a

### Copyright

Copyright 2014-2019 F5 Networks Inc.

### License

#### Apache V2.0

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and limitations
under the License.

#### Contributor License Agreement

Individuals or business entities who contribute to this project must have
completed and submitted the [F5 Contributor License Agreement](http://f5-openstack-docs.readthedocs.io/en/latest/cla_landing.html).
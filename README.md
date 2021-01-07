[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

# Ansible playbook for Searchguard enabled ELK

Special thanks to fedelemantuano for the [kibana role](https://github.com/fedelemantuano/ansible-kibana) and the [ansible elasticsearch role](https://github.com/elastic/ansible-elasticsearch) which has been used extensively. Enables out of the box installation of a fully functional cluster (with monitoring enabled) with a basic elastic license courtesy of [elastic](https://elastic.co) and secured through TLS and Basic auth courtesy [searchguard](https://search-guard.com/) and its open source license.

## Please modify the scripts to meet your needs especially the `group_vars/all.yml` and `disk.yml` file!

# Preparation:
1. Install ansible first: 
    ``` sh
    $ sudo python -m pip install ansible
    ```
    [Must be installed on machine having connectivity to all other machines where one wants to install ES, Kibana, etc through SSH]
2. You may need to keep the private key for ssh handy in case it is used to access Machines (eg: In the case of AWS).  Let us name the key `your_key.pem`
3. Have IPs of all the systems handy and note down which systems will be your Master node, Data node and Coordinating Node.
4. This script uses pre-built ES roles. Run the following only after ansible has been installed:
    ``` sh
    $ ansible-galaxy install elastic.elasticsearch,7.10.1
    ```
    ``` sh
    $ ansible-galaxy install fedelemantuano.kibana
    ```

# Changing Script values:
The following need to be checked and changed:
Remember the IP list you made earlier? It will help you now.

1. Input all IPs in the [inventory.ini](./inventory.ini) file in required positions (decide which machines would be your master node, data node, etc and put them under corresponding blocks). For example if 1.1.1.1 is master, 1.2.2.1 is coordinating, 1.3.3.1 is data and 1.2.2.1 is kibana, my inventory.ini file will look like this:
    ``` yaml
     [hosts]
      1.1.1.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
      1.2.2.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
      1.3.3.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
     [data]
      1.3.3.1
     [masters]
      1.1.1.1
     [coordinating]
      1.2.2.1
     [kibana]
      1.2.2.1
     ```
     Though not recommended, by default host key check during ssh is disabled by default for usability.
2.  Download the [Search Guard TLS Tool](https://maven.search-guard.com/search-guard-tlstool/1.8/search-guard-tlstool-1.8.zip)     (Please note it needs a Java runtime to run, install java and make sure JAVA_HOME exists)
3.  Edit the [sg-certs.yml](./sg-certs.yml) file and paste necessary info wherever needed as follows:
    ```yaml
    ca:
        root:
            dn: "CN=root.ca.example.in,OU=CA,O=example\\, tech pvt. ltd.,DC=example,DC=in"
            keysize: 2048
            pkPassword: none
            validityDays: 36500
            file: root-ca.pem
    defaults:
        validityDays: 36500
        pkPassword: changeit
        nodesDn:
            - CN=instance.example.in,OU=Ops,O=example\\, tech pvt. ltd.,DC=example,DC=in"
        nodeOid: "1.2.3.4.5.5"
        httpsEnabled: true
        reuseTransportCertificatesForHttp: true
    nodes:
    - name: instance
        dn: "CN=instance.example.in,OU=Ops,O=example\\, tech pvt. ltd.,DC=example,DC=in"
    clients:
    - name: others
        dn: "CN=others.example.in,OU=Ops,O=example\\, tech pvt. ltd.,DC=example,DC=in"
    - name: security
        dn: "CN=security.example.in,OU=Ops,O=example\\, tech pvt. ltd.,DC=example,DC=in"
        admin: true
    ```

4.  Make sure there is a folder called `out` in the root directory of this project. If not, create one. Then run the following command:

    ```sh
    $ search-guard-tlstool-1/tools/sgtlstool.sh -c /[root-git-directory]/sg-certs.yml -ca -crt -v -o true -t /[root-git-directory]/out
    ```

5. Continuing this example, the [all.yml](./group_vars/all.yml) file in the group_vars folder must be modified as follows. 
    ``` yaml
    kibana_encryption_key: [32 number alphanumeric key]
    kibana_es_user: "admin"
    kibana_password: "admin"
    kibana_version: "7.10.1"

    elastic_bootstrappass: "admin"
    elastic_superuser: "admin"
    elastic_superuserpass: "admin"

    es_data_dir: "/es-data"

    es_master_heap_size: "15g"
    es_data_heap_size: "15g"
    es_coord_heap_size: "12g"
    es_cluster_name: [cluster-name]

    es_common_pem_path: "/[path-to-git-root-folder]/out/instance.pem"
    es_common_key_path: "/[path-to-git-root-folder]/out/instance.key"
    kibana_ca_cert: "/[path-to-git-root-folder]/out/root-ca.pem"
    es_admin_pem_path: "/[path-to-git-root-folder]/out/security.pem"
    es_admin_key_path: "/[path-to-git-root-folder]/out/security.key"
    es_ca_path: "/[path-to-git-root-folder]/out/root-ca.pem"

    searchguard_es_plugin: "/[path-to-folder-on-ansible-master]/search-guard-suite-plugin-7.zip"
    searchguard_kibana_plugin: "/[path-to-folder-on-ansible-master]/searchguard-kibana-7.10.1-48.0.0.zip"

    sg_transport_keyfile: certs/instance.key
    sg_transport_pemfile: certs/instance.pem
    sg_transport_keypass: changeit

    sg_http_enabled: true
    sg_http_keyfile: certs/instance.key
    sg_http_pemfile: certs/instance.pem
    sg_http_keypass: changeit

    sg_ssl_ver: false

    admin_keypass: changeit
    admin_trustpass: changeit

    sg_trusted_cas_transport: /etc/elasticsearch/certs/root-ca.pem
    sg_trusted_cas_http: /etc/elasticsearch/certs/root-ca.pem
    sg_admin_dn: CN=security.example.in,OU=Ops,O=example\, tech pvt. ltd.,DC=example,DC=in
     ```
4. Next you need to make sure [disk.yaml](./disk.yml) is changed to match the disk names that is used by your machines. `nvme1n1`, `xvdb` and `xdf` are most commonly used for AWS EC2 instances and have been included in the script. If yours is different, change the disk type.

# Running the script:
Just run the following from the machine connected to all other machines through ssh

``` sh
$ ansible-playbook main.yml -i inventory.ini --user ec2-user --key-file your_key.pem
```
**(Use ctrl + c to exit ansible if it gets stuck at checking if kibana restarted)**

**Make sure the cluster is up**

Now kibana should work!

***Note 1***: Search guard TLS tool is packaged with this repo for convenience and is in no way created or authored by the maintainer of this repo. All credits goes to [searchguard](https://searchguard.com)

***Note 2***: This repo is not as secure out of the box and the passwords need to be changed if the system needs to be used in production


# FTP Server and DNS Configuration using Vagrant + Ansible

## Overview

This project involves the deployment and configuration of DNS (bind9) and FTP (vsftpd) servers using Vagrant and Ansible. The main objective is to provide hands-on experience in configuring FTP servers, addressing two scenarios: anonymous access and local user authentication, and implementing security measures through encryption. Additionally, a DNS server is configured to manage the domain `sri.ies`.

## Features

- **Secure Configuration**: SSL/TLS encryption for all local user connections.
- **Chroot Jail**: Restricts users to their designated directories.
- **Automated Testing**: Verifies functionality and security for both anonymous and local user servers.

## Requirements

### Prerequisite Knowledge

1. Familiarity with Linux Debian 12.
2. Basic knowledge of network protocols and services.
3. Basic knowledge of DNS protocol and configuration.

### Necessary Resources

- **Vagrant 2.4.1** or latest.
- **Ansible 2.5** or latest.
- **Python3**.


## Infraestructure

The infrestructure was built with [Vagrant](https://www.vagrantup.com/), which uses [VirtualBox](https://www.virtualbox.org/) as a provider to create the virtual machines. This machines were provisioned with [Ansible](https://docs.ansible.com/) with its properly configuration files.

- We have this machine to deploy the **FTP Server**. How we can see, it have two networks cards, one for each server service.

  ```ruby
  config.vm.define "ftp" do |s|
    s.vm.network "private_network", ip: "192.168.57.20"
    s.vm.network "private_network", ip: "192.168.57.30"
    s.vm.hostname="ftp"
    s.vm.provision "ansible" do |ansible|
      ansible.config_file = "./ansible.cfg"  
      ansible.playbook = "ansible/ftp.yml"
      ansible.inventory_path = "ansible/inventory/inventory.yml"
    end
  end
  ```
> *FTP Server machine. Extract from [Vagrantfile](../Vagrantfile)*


- This will be the machine for the **DNS Server**.

  ```ruby
  config.vm.define "dns" do |d|
    d.vm.network "private_network", ip: "192.168.57.10"
    d.vm.hostname="dns"
    d.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/ns.yml"
      ansible.config_file = "./ansible.cfg"
      ansible.inventory_path = "ansible/inventory/inventory.yml"
    end
  end
  ```
> *DNS Server machine. Extract from [Vagrantfile](../Vagrantfile)*

## FTP Server Configuration

### 1. Configuration of the Anonymous FTP Server

1. Install FTP server software on the assigned machine:

    ```yml
    - name: Install FTP package
      ansible.builtin.package:
        update_cache: yes
        name: vsftpd
    ```
    > *This was done in the [ftp.yml](/ansible/ftp.yml)*

2. Configure the FTP server to allow anonymous connections:
   - Copy the file `/etc/vsftpd.conf` for the mirror server.
   - Edit `/etc/mirror.conf` and set:

     ```bash
      # config/vsftpd/mirror.conf

      # Anonymous FTP server configuration
        banner_file=/srv/ftp/.message
        listen=YES
        listen_address=192.168.57.20
        listen_ipv6=NO
        anonymous_enable=YES
        no_anon_password=YES
        local_enable=NO
        write_enable=NO
        data_connection_timeout=30
        anon_max_rate=5120
        dirmessage_enable=YES
        use_localtime=YES
        connect_from_port_20=YES
        secure_chroot_dir=/var/run/vsftpd/empty
        pam_service_name=vsftpd
     ```

  > *With this configuration we secure that the anonymous users have no write permissions and can enter  without using a password.*

### 2. Configuration of the FTP Server for Local Users

1. On the same machine with a second network adapter, we create the local server.

2. Configure the FTP server to authenticate users using OS accounts:

   - We copy `/etc/vsftpd` file but this time for the local users of the FTP-server
   - Edit `/etc/ftp.conf` and set:

     ```bash
      # config/vsftpd/ftp.conf

      # Anonymous FTP server configuration
        banner_file=/srv/ftp/.message
        listen=YES
        listen_address=192.168.57.30
        listen_ipv6=NO
        anonymous_enable=NO
        local_enable=YES
        write_enable=YES
        dirmessage_enable=YES
        use_localtime=YES
        connect_from_port_20=YES
        secure_chroot_dir=/var/run/vsftpd/empty
        pam_service_name=vsftpd

      # Jailed users
        chroot_local_user=YES
        chroot_list_enable=YES
        chroot_list_file=/etc/vsftpd/vsftpd.chroot_list
        allow_writeable_chroot=YES
     ```

3. User charles is chrooted, laura is not.
    - We created this users with ansible too.

      ```yml
      - name: Create local user charles
        user:
          name: charles
          password: "$6$zrhHqgEzqBWfELPs$mtw4BECKUKCxNHvyiPZYhat/k6ussxcwyLmbRAIEBavxe9Hg1ONoiEBvSvGVEeO6NQJbkXnVofpUBxn5EWjLO/"

      - name: Create local user laura
        user:
          name: laura
          password: "$6$jjpYQgP5MUfGI3D5$T2a3aeKGc6aJIkl9CwiOYAEO6qRWGPoCdUDQnAVq2Cu0fJnE9W1riE1JOAGk7nsxFEg7FCG10URBmHL5BJyNa/"
      ```

> *All of this configured in the ftp.conf.*

### 3. Configuration of the FTP Server for multi-user target

  1. Create the systemd files for FTP Server

   - Create the `vsftpd-mirror.service`, set:

     ```bash
        [Unit]
        Description=Service for Anonymous FTP server
        After=network.target

        [Service]
        ExecStart=/usr/sbin/vsftpd /etc/vsftpd/mirror.conf
        ExecReload=/bin/kill -HUP $MAINPID
        ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty
        KillSignal=SIGTERM
        Restart=on-failure
        Type=simple

        [Install]
        WantedBy=multi-user.target
     ```

   - Create the `vsftpd-ftp.service`, set:

      ```bash
      [Unit]
      Description=Service for Local Users FTP server
      After=network.target

      [Service]
      ExecStart=/usr/sbin/vsftpd /etc/vsftpd/ftp.conf
      ExecReload=/bin/kill -HUP $MAINPID
      ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty
      KillSignal=SIGTERM
      Restart=on-failure
      Type=simple

      [Install]
      WantedBy=multi-user.target
      ```

  2. Copy the files.

  - copy the files using ansible using this command:

    ```yml
    - name: Copy configuration files 
    ansible.builtin.copy:
      src: "{{ item.src }}"
      dest: "{{ item.dest }}"
      owner: root
      group: root
    loop:
      - { src: "{{ ftp_path }}ftp.conf", dest: /etc/vsftpd/ }
      - { src: "{{ ftp_path }}mirror.conf", dest: /etc/vsftpd/ }
      - { src: "{{ ftp_path }}.message", dest: /srv/ftp }
      - { src: "{{ ftp_path }}vsftpd.chroot_list", dest: /etc/vsftpd/ }
      - { src: "{{ ftp_path }}systemd/vsftpd-mirror.service", dest: /etc/systemd/system/ }
      - { src: "{{ ftp_path }}systemd/vsftpd-ftp.service", dest: /etc/systemd/system/ }
    ```

  > *All of this is set inside [ftp.yml](/ansible/ftp.yml)*

### 3. The Importance of Encryption in Secure File Transfer

Encryption is critical in secure file transfer to protect sensitive data from being intercepted or tampered with during transmission. Here’s why it’s important:

- **Data Confidentiality**: Ensures that only authorized users can read the transmitted data by encrypting the content.
- **Data Integrity**: Prevents unauthorized modification of files during transfer.
- **Authentication**: Validates the identity of the server and the client to avoid impersonation attacks.

In this project, SSL/TLS is enabled for the local user FTP server to secure file transfers:

- `ssl_enable=YES` ensures all communication between the client and server is encrypted.
- Certificates (`rsa_cert_file` and `rsa_private_key_file`) are used to establish a secure connection.

#### Implementation of Encryption (SSL/TLS)

1. **Configure the SSL/TLS security layer on the FTP server**:

   - Generate SSL certificate and key:

    > *We created it using ansible.*

     ```yml
      - name: Install dependencies to obtain SSL certificate
        ansible.builtin.package:
          update_cache: yes
          name: python3-cryptography 
      - name: Generate SSL Private Key
        openssl_privatekey:
          path: /etc/ssl/private/ssl-cert-priv.key
          size: 2048

      - name: Generate sign request (CSR)
        openssl_csr:
          path: /etc/ssl/private/ssl-sign.csr
          privatekey_path: /etc/ssl/private/ssl-cert-priv.key
          common_name: "ssl-ftp-cert"
          country_name: "ES"
          organization_name: "IZV"
          email_address: "jrodcas899@ieszaidinvergeles.org"

      - name: Obtain SSL public key
        openssl_certificate:
          path: /etc/ssl/certs/ssl-cert-pub.pem
          privatekey_path: /etc/ssl/private/ssl-cert-priv.key
          csr_path: /etc/ssl/private/ssl-sign.csr
          provider: selfsigned
     ```

  - Edit `ftp.conf` and `mirror.conf`, set:

     ```bash
        # ssl configuration
        rsa_cert_file=/etc/ssl/certs/ssl-cert-pub.pem
        rsa_private_key_file=/etc/ssl/private/ssl-cert-priv.key
        ssl_enable=YES
        allow_anon_ssl=YES
     ```

  ### 4. Testing the FTP Server

  #### Testing FTP server via Filezilla

  **1. Testing mirror server**

  - Connecting mirror server

  ![image_putting_mirror_server](/docs/imgs/putting_ftp_server.png)

  - SSL certificate appears.

  ![watching_ssl_certificate](/docs/imgs/testing_ssl.png)

  - Connected in the mirror server.

  ![connected](/docs/imgs/connecting_mirror_server.png)

  **2. Testing local server**

  - Connecting local server.

  ![local_server_connection](/docs/imgs/connecting_local_server.png)

  - SSL certificate appears.

  ![ssl_local_server](/docs/imgs/ssl_local_server.png)

  - Connected as laura in the local server.

  ![connected_laura](/docs/imgs/connected_laura.png)

  - Connected as charles in the local server.

  ![connected_charles](/docs/imgs/conected_charles.png)


  #### Testing FTP sever via python test

  We can automate this test with a Python script in `tests/` directory running the following command:

   ```bash
   $ python3 tests/test-ftp.py
   ```

  ![ftp_test](/docs/imgs/ftp-test.png)

## DNS Server Configuration

### 1. Install Bind9 package

  Install a second virtual machine with a DNS server authoritative for the domain `sri.ies`:

   - Install BIND9:

     ```yml
      - name: Install Bind9
        ansible.builtin.package:
          update_cache: yes
          name: bind9
     ```
  > *This is in [ns.yml](/ansible/ns.yml)*

### 2. Configure the named.conf files

   - Configure `named.conf.local` to define zones:

     ```bash
        zone "sri.ies" {
            type master;
            file "/var/lib/bind/db.sri.ies";
            allow-transfer { 192.168.57.10; };
        };

        zone "57.168.192.in-addr.arpa" {
            type master;
            file "/var/lib/bind/rev.sri.ies";
            allow-transfer { 192.168.57.10; };
        };
     ```

   - Configure the `named.conf.options`.

      ```bash
        options {
                directory "/var/cache/bind";

                forwarders {
                        1.1.1.1;
                };

                dnssec-validation yes;
                listen-on { any; };
                listen-on-v6 { any; };
        };
      ```

  ### 3. Configure the DNS zones

   - Create the zone file `/var/lib/bind/db.sri.ies`:

     ```bash
      ;
      ; DNS configuration for the FTP-Server
      ; 
      ;
      $TTL    604800
      $ORIGIN sri.ies.

      @       IN      SOA     ns.sri.ies. root.sri.ies. (
                        3         ; Serial
                  604800         ; Refresh
                    86400         ; Retry
                  2419200         ; Expire
                  604800 )       ; Negative Cache TTL

      @       IN      NS      ns.sri.ies.
      ns      IN      A       192.168.57.10
      mirror  IN      A       192.168.57.20
      ftp     IN      A       192.168.57.30
     ```

   - Create the zone file `/var/lib/bind/rev.sri.ies`:

      ```bash
      ; Reverse configuration for the FTP-server
      ; 
      ;
      $TTL    604800
      $ORIGIN 57.168.192.in-addr.arpa.

      @       IN      SOA     ns.sri.ies. root.sri.ies. (
            3         ; Serial
          604800         ; Refresh
          86400         ; Retry
          2419200         ; Expire
          604800 )       ; Negative Cache TTL

      @       IN      NS      ns.sri.ies.
      10      IN      PTR     ns.sri.ies.
      20      IN      PTR     mirror.sri.ies.
      30      IN      PTR     ftp.sri.ies.
      ```
  
  - Providing the server with its configuration files.

    ```yml
    - name: Copy DNS configuration files
      ansible.builtin.copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        owner: bind
        group: bind 
      loop:
        - { src: "{{ dns_path }}named", dest: /etc/default/ }
        - { src: "{{ dns_path }}named.conf.options", dest: /etc/bind/ }
        - { src: "{{ dns_path }}named.conf.local", dest: /etc/bind/ }
        - { src: "{{ dns_path }}db.sri.ies", dest: /var/lib/bind/ }
        - { src: "{{ dns_path }}rev.sri.ies", dest: /var/lib/bind/ }
     ```  

With this configuration we secure the DNS server is working.

### 4. Testing DNS server

You can try it by using the [test-dns.sh](/tests/test-dns.sh)

![dns_result](/docs/imgs/test-dns.png)

> *This is the result you should expect.*


## Testing Ansible playbooks

This project contains a **Continuous Integration** workflow made with **Github Actions** features to ensure the quality of the code and the correct syntax in the Ansible files.

  - **Deploy yaml file was created to trigger the action when a commit is done**. How you can see the test are running in Ubuntu 24.04 and there is a job to install the necessary dependencies for run [Ansible Lint](https://ansible.readthedocs.io/projects/lint/) and another one to run it.

    ```yml
    name: ansible-lint
    on:
      push:
        branches: ["main"]
      workflow_dispatch:

    jobs:
      build:
        name: Ansible Lint
        runs-on: ubuntu-24.04
        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Install community.crypto from Ansible galaxy
            run: |
              python -m pip install --upgrade pip
              pip install ansible ansible-lint
              ansible-galaxy collection install community.crypto
      
          - name: Run ansible-lint
            run: |
              ansible-lint ansible/
    ```

  - When we do a `git push` command the test are triggered and passed.

    <div align="center">
      <img src=".imgs/ansible-lint-passed.png" alt="ansible-lint">
    </div>

## Conclusion

This project provides hands-on experience in configuring and securing FTP servers, preparing students for real-world challenges in system and network administration. By following the steps outlined, students will gain valuable skills in managing FTP and DNS servers, as well as implementing encryption for secure file transfers.

## Contributors

- [Jorge Rodriguez](https://github.com/sporestudio) 
- [Juan Diego](https://github.com/JuanDiego1406)
- [Miguel Angel](https://github.com/leogamer644)

## License

This project is licensed under the GNU General Public License - see the [LICENSE](../LICENSE) file for details.

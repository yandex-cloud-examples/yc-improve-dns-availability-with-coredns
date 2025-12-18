# Improving fault tolerance using CoreDNS cache

This solution employs CoreDNS in caching mode to mitigate potential DNS issues during infrastructure failures.

## Prerequisites:

* You have an existing Yandex Cloud folder.
* You have installed [**yc**](https://yandex.cloud/docs/cli/quickstart).

## About the solution

If your external DNS servers fail, this solution will provide a DNS response for each request whose response was previously cached.
* Deploying this will create an Ubuntu 20.04 VM in your Yandex Cloud, and conventionally connect the VM to an appropriate subnet.
* That VM will run a separate CoreDNS server in caching mode.
* Then, the regular DNS service will be disabled, with all requests routed to the new CoreDNS.
* If your external DNS servers fail, CoreDNS will keep successfully responding to requests whose responses were previously cached.

## Usage

### Input parameters for [variables.tf](./variables.tf)
* `vm_name`: Name of your VM in Yandex Cloud.
* `vm_zone`: Name of the availability zone in Yandex Cloud.

### Deployment
1. In [env-yc-prod.sh](./env-yc-prod.sh), check that the YC runtime environment is properly configured. Replace the `prod` profile name with that of the one you need.
1. Activate the environment:

    ```bash
    source env-yc-prod.sh
    ```

1. Initialize Terraform:

    ```bash
    terraform init
    ```

1. Deploy the VM with CoreDNS:

    ```bash
    terraform apply
    ```

### Testing

1. Save the external address of the created VM to an environment variable:

    ```bash
    export vm_ext_ip=`terraform output vm_ext_ip_address | sed 's/"//g'`
    ```

1. Connect to the VM:

    ```bash
    ssh -oStrictHostKeyChecking=no admin@$vm_ext_ip
    ```

1. Check that your DNS service works properly, also warming up the CoreDNS cache:

    ```bash
    ping www.yandex.ru
    ping github.com
    ping kubernetes.io
    ```

1. With `iptables`, disable the regular DNS server:

    ```bash
    export dns_ip=`grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" /etc/coredns/Corefile`
    sudo iptables -A OUTPUT -d $dns_ip -j DROP
    sudo iptables -L -n -v
    ```

1. Check your DNS service once again using the addresses from Step 3. Make sure the responses are correct:

    ```bash
    ping www.yandex.ru
    ping github.com
    ping kubernetes.io    
    ```

1.  Try to get an IP address for a name never used before (not in the DNS cache). This should return `Temporary failure in name resolution`:

    ```bash
    ping www.terraform.io 
    ```

### Shutting down the solution

```bash
unset vm_ext_ip
terraform destroy
```

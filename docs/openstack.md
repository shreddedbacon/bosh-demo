Most of what is here is only used for this demo, some key places to look at are the following if you want to find out main differences.

* Deployment Manifest
* Operations Files
* Cloud Configuration

# Reference Information
This section is for the demo only

## LoadBalancer
* 192.168.101.69

## SSH Tunnels
A lot of the networking in this section is specific to the demo performed at the Infracoders meetup, be aware it probably won't work elsewhere.
```
ssh -nNt -L 2222:192.168.101.91:22 10.8.0.10 #jumpbox
ssh -nNt -L 7000:192.168.101.30:80 10.8.0.10 #openstack dashboard
ssh -nNt -L 7001:192.168.101.69:80 10.8.0.10 #openstack loadbalancer
ssh -nNt -p 2222 -L 7002:192.168.209.7:8080 ubuntu@localhost #jumpbox->director->turbulence
```
## Access
### Web
* [Openstack Dashboard](http://localhost:7000/dashboard/project/instances/)
* [Openstack Load Balancer Pool](http://localhost:7000/dashboard/project/ngloadbalancersv2/0a839cc2-abb1-4801-9eb3-caa616374628/listeners/6eea0e96-419a-431f-adf2-8061d366ca54/pools/246dce06-1504-41d2-8805-f1a2ae1e8337)
* [Load Balancer](http://localhost:7001)
* [Turbulence Dashboard](https://localhost:7002/)

### Jumpbox
```
scp -p 2222 /tmp/static-site.tgz ubuntu@localhost:/tmp/static-site.tgz
ssh -p ubuntu@localhost
```

# Resources
## Deployment Manifest
Create the deployment manifest on the jumpbox, this is exactly the same manifest as used in virtualbox
```
cd
mkdir demo
cat << "EOF" > demo/deployment.yml
---
name: static-web

releases:
- name: staticsite-boshrelease
  version: latest
- name: nginx
  version: latest

stemcells:
- alias: default
  os: ubuntu-xenial
  version: latest

instance_groups:
- name: webserver
  instances: 1
  stemcell: default
  vm_type: default
  azs: [z1]
  persistent_disk_type: default
  networks:
  - name: default
  jobs:
  - name: staticsite
    release: staticsite-boshrelease
    properties:
      docroot: ((staticsite_docroot))
  - name: nginx
    release: nginx
    properties:
      nginx_worker_processes: auto
      nginx_worker_connections: 1024
      nginx_servers:
      - server_name: ((staticsite_domain))
        docroot: ((staticsite_docroot))
        port: ((staticsite_http_port))
        index: "index.html"
        access_log: /var/vcap/sys/log/nginx/access.log
        error_log: /var/vcap/sys/log/nginx/error.log
        custom_data: ((nginx_config))

update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
EOF
```
We are also going to use some variables to set up the docroot and listening domain and port for nginx too these are defined in the deployment manifest using `((variable_name))`
```
cat << "EOF" > demo/variables.yml
staticsite_domain: staticsite.demo
staticsite_docroot: /var/vcap/store/nginx/www/document_root
staticsite_http_port: 80
nginx_config: |
  location / {
    try_files $uri $uri/ =404;
  }
EOF
```
## Operations Files
We want to creat some operations that override the behaviour of our original default deployment.
```
cat << "EOF" > demo/ops-instances.yml
---
- type: replace
  path: /instance_groups/name=webserver/instances
  value: 6
- type: replace
  path: /instance_groups/name=webserver/vm_type
  value: lbmicro
- type: replace
  path: /instance_groups/name=webserver/networks/name=default/name
  value: demonet01
EOF
```

!!! info
    Our operations are only changing the vm_type and the number of instances we want to deploy, and which network they will live in. The vm_type `lbmicro` is defined in the Cloud Configuration.

## Cloud Configuration
```
cat << "EOF" > demo/cloud-config.yml
---
azs:
- name: z1
  cloud_properties:
    availability_zone: nova

vm_types:
- name: default
  cloud_properties:
    instance_type: m1.micro
- name: tiny
  cloud_properties:
    instance_type: m1.tiny
- name: micro
  cloud_properties:
    instance_type: m1.micro
- name: small
  cloud_properties:
    instance_type: m1.small
- name: medium
  cloud_properties:
    instance_type: m1.medium
- name: large
  cloud_properties:
    instance_type: m1.large
- name: xlarge
  cloud_properties:
    instance_type: m1.xlarge
- name: lbsmall
  cloud_properties:
    instance_type: m1.small
    loadbalancer_pools:
      - name: demo-pool
        port: 80
- name: lbmicro
  cloud_properties:
    instance_type: m1.micro
    loadbalancer_pools:
      - name: demo-pool
        port: 80

disk_types:
- name: default
  disk_size: 1024
  cloud_properties:
    type: nfs
- name: micro
  disk_size: 5_120
  cloud_properties:
    type: nfs
- name: small
  disk_size: 10_240
  cloud_properties:
    type: nfs
- name: medium
  disk_size: 20_480
  cloud_properties:
    type: nfs
- name: large
  disk_size: 30_720
  cloud_properties:
    type: nfs

networks:
- name: default
  type: manual
  subnets:
  - azs: [z1]
    dns: [192.168.101.1]
    range: 192.168.209.0/24
    gateway: 192.168.209.1
    static: [192.168.209.10-192.168.209.99]
    reserved: [192.168.209.2-192.168.209.9,192.168.209.100]
    cloud_properties:
      net_id: 16533825-c891-4e2b-8124-490a0b5bae4e
      security_groups: [demo-bosh, demo-web, demo-all]
- name: demonet01
  type: manual
  subnets:
  - azs: [z1]
    dns: [192.168.101.1]
    range: 192.168.140.0/24
    gateway: 192.168.140.1
    static: [192.168.140.10-192.168.140.99]
    reserved: [192.168.140.2-192.168.140.9,192.168.140.100]
    cloud_properties:
      net_id: ea119a90-86a0-44fc-b86c-3aa6c50dd3e0
      security_groups: [demo-bosh, demo-web]
- name: vip
  type: vip

compilation:
  workers: 4
  az: z1
  reuse_compilation_vms: true
  vm_type: medium
  network: default
EOF
```

!!! info
    You can see that we have cloud_properties defined for some things in this manifest, where our virtualbox cloud configuration doesn't.
    Cloud properties are used to tell the CPI what to use when it builds, or what to attach something to.
    In the case of `lbmicro` we are telling it that the instance_type to use in openstack is an m1.micro, and that once built they need to be assigned to the loadbalancer pool called `demo-pool`.

    Similar things are done for the network definition where it tells the CPI which networks to build in, and which security groups to assign to vms built in those networks.

# Deploying
Now we are ready to deploy it into openstack.
```
cd bucc
```
Upload our cloud configuration
```
bosh ucc ../demo/cloud-config.yml
```
Upload our releases
```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-openstack-kvm-ubuntu-xenial-go_agent?v=97.17
```
```
bosh upload-release https://github.com/shreddedbacon/nginx-boshrelease/releases/download/v1.2.7/nginx-1.2.7.tgz
```
```
bosh upload-release /tmp/static-site.tgz
```
And finally deploy it
```
# no LB
bosh -d static-web d ../demo/deployment.yml -l ../demo/variables.yml
```
```
# with LB
bosh -d static-web d ../demo/deployment.yml -o ../demo/ops-instances.yml -l ../demo/variables.yml
```

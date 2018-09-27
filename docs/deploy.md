# Manifest
Great, we are nearly done. We need to create a deployment manifest that tells BOSH how to deploy our release.

Create a manifest directory in our release, so we can keep everything there.
```bash
mkdir manifests
```
We are also going to use an existing nginx BOSH release here too, so we need to make sure the manifest knows how to set it up too.
```bash
cat << "EOF" > manifests/deployment.yml
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

!!! note
    In our manifest, you will notice that a lot of things are defined as `default`. This allows our manifest to be as simple as possible, and we need to make sure our cloud manifest knows how to handle defaults.

We are also going to use some variables to set up the docroot, listening domain, and port for nginx. These are defined in the deployment manifest using `((variable_name))`.
```bash
cat << "EOF" > manifests/variables.yml
staticsite_domain: staticsite.demo
staticsite_docroot: /var/vcap/store/nginx/www/document_root
staticsite_http_port: 80
nginx_config: |
  location / {
    try_files $uri $uri/ =404;
  }
EOF
```

Great, we are nearly there.

# Stemcell
Our deployment calls for a stemcell called ubuntu-xenial. We should probably upload the stemcell into our director.

```bash
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-xenial-go_agent?v=97.17
```

```bash
#already downloaded
bosh upload-stemcell ~/Downloads/bosh-stemcell-97.17-warden-boshlite-ubuntu-xenial-go_agent.tgz
```
This can take some time, its about 400MB

# Nginx Release
We also told our deployment to use nginx. We should probably upload the nginx release too.
```bash
bosh upload-release https://github.com/shreddedbacon/nginx-boshrelease/releases/download/v1.2.7/nginx-1.2.7.tgz
```

# Check Director
Check that our director has a stemcell associated to it
```bash
bosh stemcells
```
And also check that our 2 releases are there.
```
bosh releases
```

# Cloud Configuration
We need to tell BOSH about our virtualbox "cloud" IaaS.

Let's create the cloud configuration now.
```bash
cat << "EOF" > virtualbox-cloud-config.yml
azs:
- name: z1

vm_types:
- name: default

disk_types:
- name: default
  disk_size: 1024

networks:
- name: default
  type: manual
  subnets:
  - azs: [z1]
    dns: [8.8.8.8]
    range: 10.244.0.0/24
    gateway: 10.244.0.1
    static: [10.244.0.34]
    reserved: []

compilation:
  workers: 5
  az: z1
  reuse_compilation_vms: true
  vm_type: default
  network: default
EOF
```
!!! info
    The cloud configuration describes our infrastructure. You will notice we have the following
    * `default` vm_type, in our manifest, our vm to build is a default type.
    * `default` disk_type, in our manifest this is what our `persistent_disk_type` refers to
    * `default` network, in our manifest, our instance group is associated to this default network
    Since our director is using virtualbox and warden cpi, there isn't much else we can describe.

Now update the director so it know how to use our cloud.
```bash
bosh update-cloud-config virtualbox-cloud-config.yml
```

# Deploying
Now it's time to actually deploy our release.

```bash
bosh --deployment=static-web deploy manifests/deployment.yml --vars-file=manifests/variables.yml
```
or
```bash
bosh -d static-web d manifests/deployment.yml -l manifests/variables.yml
```

# Check Status
Once the deployment is done, you can check the status of our VM and the services we are running on them.
## List Deployments
```bash
bosh deployments
```
## List VMs
```bash
bosh -d static-web vms
```
## List processes running on instances
```bash
bosh -d static-web instances --ps
```

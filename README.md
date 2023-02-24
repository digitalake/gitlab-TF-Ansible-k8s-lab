<h1 align="center">FINAL</a> 
<img src="https://github.com/blackcater/blackcater/raw/main/images/Hi.gif" height="32"/></h1>

> __CHECK THE SOURCE CODE FROM LINKS IN "LINKS" SECTION__

### <a name="road">ROADMAP</a>
  - #### [LINKS](#links)
  - #### [INTRODUCTION](#intro)
  - #### [GITLAB](#git)
  - #### [GITLAB TERRAFORM](#terraform)
  - #### [GITLAB KUBESPRAY](#kubespray)
  - #### [GITLAB HELM DEPLOY](#deploy)
  
---

### <a name="links">LINKS</a> | [ROADMAP](#road)

 __ALL THE PROJECT SOURCE CODE:__
 - [GITLAB TERRAFORM REPO LINK](https://gitlab.com/gl-basecamp-final/terraform-infra)
 - [GITLAB KUBESPRAY REPO LINK](https://gitlab.com/gl-basecamp-final/ansible-kubespray)
 - [GITLAB HELM DEPLOY REPO LINK](https://gitlab.com/gl-basecamp-final/ansble-wordpress-deploy)
 - [WP INSTALL RESULT LINK](https://vantus.dns.army/)
  
---


### <a name="intro">INTRODUCTION</a> | [ROADMAP](#road)

__Task definition__

_Terraform:_
- create VPC in GCP/Azure		
- create instance with External IP		
- prepare managed DB (MySQL)		
			
_Ansible:_
- perform basic hardening (keyless-only ssh, unattended upgrade, firewall)			
- deploy K8s (single-node cluster via Kubespray)		
			
_Kubernetes:_
- prepare ansible-playbook for deploying Wordpress		
- deploy WordPress with connection to DataBase

As we can see, there are 3 primary steps in this task. For performing all of those steps the techs needed are:
- Terraform
- Ansible
- Kubernetes cluster
- GCP account:
  - with access to the CC resources
  - service account with right permissions
  - API-services enabled
  
For task completion i needed the variant of automizing all of those step's communication. I decided to use GitLab CI and create some repos with pipelines.

So, i'll try to shortly describe all the steps i needed.
  
---


### <a name="git">GITLAB SOLUTION</a> | [ROADMAP](#road)

GitLab is a web-based Git repository manager that provides Git repository management, code reviews, issue tracking, continuous integration and deployment, and wiki functionality. It offers features such as: Git repository management, continuous integration and delivery (CI/CD).

I decided to create the group with 3 repos to separate the steps of building and configuring my infra to deploying the soft on it:
- Infrastructure build--> terraform
- Infrastructure configure (deploy k8s)--> ansible (kubespray)
- Deploy (deploy wordpress)--> ansible, helm

![image](https://user-images.githubusercontent.com/109740456/221228108-b5fe3c48-3699-4930-a8d2-c4efa34b0365.png)

To enable running CI/CD pipelines i needed to create the pipeline files named(by default) .gitlab-ci.yml and the gitlab runner. Gitlab runner is the pipeline job executor. I could use shared runners but i decided to create my own runner with docker executor for running the pipeline steps inside the docker containers.
So after i registered the runner i was able to see him being avaliable in my projects. I ennabled the runner for using in several projects. 

Here is my runners config from /etc/gitlab-runner/config.toml:
```
concurrent = 10
check_interval = 5
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "wks4-notebook"
  url = "https://gitlab.com/"
  id = 21340033
  token = "***************"
  token_obtained_at = 2023-02-22T17:39:19Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
  [runners.docker]
    tls_verify = false
    image = "ruby:2.7"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    allowed_pull_policies = ["never", "always", "if-not-present"]
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
 ```
 
 The runner has got the "core" tag assigned so, to make gitlab schedule some job using this runner i needed to use "core" tag defined in my jobs.
 
 For using docker runner i needed docker images as an env for running the jobs.
 
 I will go through and describe each of my repos and their .gitlab-ci.yml files.
   
---


 ### <a name="terraform">GITLAB TERRAFORM</a> | [ROADMAP](#road)
 
 Terraform code file-structure:
 ```
 
├── ansible-artifacts
│   └── k8s-cluster.yml ---> ready-for-use kubespray config
│
├── ansible-templates
│   ├── addons.tpl      ---> kubespray additional soft config template
│   ├── helm-values.tpl ---> helm values file template with db values
│   └── inventory.tpl   ---> the inventory will be used with ansible
│
├── db.tf            ---> db configuration
├── firewall.tf      ---> firewall rules
├── generation.tf    ---> local resources for generation ansible-artifacts
├── networking.tf    ---> network and subnetwork configuration
├── output.tf        ---> useful outputs with ips
├── providers.tf     ---> providers section
├── terraform.tfvars ---> variables values
├── variables.tf     ---> the variable defined
└── vms.tf           ---> the vm instances configuration
```

The main thing about TF in CI is a problem with local backend. To provide normal-working TF, i needed to migrate my state to the remote backend. Because of the GCP cloud provider i'm using, i decided to use the Google Cloud Storage bucket and specified te remote backend.

```
 backend "gcs" {
    credentials = "./your/credentials/file/path.json"
    bucket      = "bucket-name"
    prefix      = "foo/bar"
```

![image](https://user-images.githubusercontent.com/109740456/221238409-ddb77101-10c4-496f-a012-688af25855d3.png)

The pipeline has got 5 stages:
  - validate (tf fmt and validate check) 
  - plan     (tf plan)
  - apply    (tf apply)
  - gcp-push (push the generated artifacts into the bucket)
  - trigger-kube (trigger the kubespray pipeline via API)
  
 The main idea is to make TF generate the configs for executing ansible-playbook for pulling them later in another pipeline.
 
 Pipeline execution:
 
![image](https://user-images.githubusercontent.com/109740456/221241604-1f23570c-dfd7-4e9b-af4d-de2741087d3a.png)

![image](https://user-images.githubusercontent.com/109740456/221241830-33a833f9-09e0-4451-b837-598ba6728126.png)

Apply job results:

![image](https://user-images.githubusercontent.com/109740456/221242196-5271fd34-06e8-4f88-8b2a-ba3f8ca999c4.png)

Push  job results:

![image](https://user-images.githubusercontent.com/109740456/221242555-893c870b-84a0-4610-9f40-c175383e060a.png)
  
---


### <a name="kubespray">GITLAB KUBESPRAY</a> | [ROADMAP](#road)

File-structure:
```
.
└── ansible
    ├── {inventory.ini} generated by terraform and pulled from bucket
    ├── post-conf.yml
    └── roles
        └── kubeconf-migrate
            └── tasks
                └── main.yml ---> task for copying kubeconfig
```

The most impotant part in this pipeline is kubespray execution. The configs rendered by terraform from templates are used by kubespray to perform the k8s cluster installation. 

The parts of templates:

addons.tpl
```
# MetalLB deployment
metallb_enabled: true
metallb_speaker_enabled: true
metallb_ip_range:
%{ for content_key, content_value in content ~}
   - "${content_value}/32"
%{ endfor ~}
```

helm-values.tpl
```
db:
  host: ${db_ip}
  user: ${db_user}
  name: ${db_name}
  password: ${db_password}
```

inventory.tpl
```
[all]
%{ for content_key, content_value in content ~}
${content_key} ansible_host=${content_value}
%{ endfor ~}

[kube_control_plane]
%{ for content_key, content_value in content }
%{~ if length(regexall("master", content_key)) > 0 ~}
${content_key}
%{ endif ~}
%{~ endfor ~}

[etcd]
%{ for content_key, content_value in content }
%{~ if length(regexall("master", content_key)) > 0 ~}
${content_key}
%{ endif ~}
%{~ endfor ~}

[kube_node]
%{ for content_key, content_value in content ~}
${content_key}
%{ endfor ~}

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

Just because im using the for_each construction, its also an  interesting task - to create the proper template for dynamic ansible inventory.

The first step of this pipeline is pulling the artifacts pushed by terraform from the GCS.

The result:

![image](https://user-images.githubusercontent.com/109740456/221245789-57857521-b627-49d6-bc6a-4b0cf88e23e5.png)

While using kubespray, the kubespray repo is being cloned and the necessary files are copied to prepare the kubespray configuration.
Then the Kubespray deploys K8s on the node/nodes.

Job result:

![image](https://user-images.githubusercontent.com/109740456/221246391-8de47566-eea1-404e-ba08-1c95936e7343.png)

And then the small playbook is executed as an afterscript to copy the kubeconfig into the user's home repo to grant access to kubectl.

Pipeline result:

![image](https://user-images.githubusercontent.com/109740456/221246942-c16dfcf1-1f29-45cf-8ba6-fe9010c20b22.png)
  
---

<a name="deploy">GITLAB HELM DEPLOY</a> | [ROADMAP](#road)

The last pipeline file-structure:
```
.
└── ansible
    ├── deploy.yml ---> deploy playbook
    ├── hardening.yml ---> playbook to perform basic hardening
    └── roles
        ├── deploy
        │   ├── files ---> here is the chart-files for wordpress and helper-services
        │   │   └── charts
        │   │       ├── Chart.yaml ---> helm-values.yaml is not here cause it is being pulled 
        |   |       |                   as an artifact generated by terraform from the bucket 
        │   │       └── templates
        │   │           ├── ingress.yaml
        │   │           ├── nginx-ctl.yaml
        │   │           ├── path_provisioner.yaml
        │   │           ├── production-issuer.yaml
        │   │           └── wordpress.yaml
        │   ├── tasks
        │   │   └── main.yaml
        │   └── templates
        │       └── values.j2 ---> the primary values file jinja2 template for the chart 
        └── hardening
            ├── files
            │   ├── 50unattended-upgrades ---> config for enabling unattended-upgrades
            │   ├── passwd ---> password rules
            │   └── ssh_config ---> ssh rules 
            └── tasks
                └── main.yaml
```

The values.j2 template is a standart jinja2 template and the values are being passed into the template from the GitLab env vars.

While the pipeline is running, the necessary artifacts(additional values file for the chart and the inventory) are being pulled from the bucket and the values file is being generated from jinja2 template.

Ansible runs twice:
  - for deploying wordpress
  - for performing hardening
  
Pipeline scheme:

![image](https://user-images.githubusercontent.com/109740456/221251021-407eb046-2543-44d7-a745-3a8f240ca78e.png)

Hardening job result:

![image](https://user-images.githubusercontent.com/109740456/221251296-9c267a15-cefb-4c4c-ad6f-6f7b4f07e5f0.png)

Deploy job result:

![image](https://user-images.githubusercontent.com/109740456/221251453-a5900516-3780-4599-bb56-ef4ca4e632ad.png)

After the deploy its able to check the result:

![image](https://user-images.githubusercontent.com/109740456/221251739-2b893f27-5d0c-45b1-b46e-20206b2abdf7.png)







































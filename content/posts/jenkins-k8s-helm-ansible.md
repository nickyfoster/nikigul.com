---
author:
  name: "Nikita Guliaev"
date: 2021-09-28
title: 'Deploying Jenkins on AWS EKS with Helm and Ansible: A Step-by-Step Guide'
series:
- Tutorials
---

## Deploying Jenkins Helm chart to AWS EKS with Ansible

## Introduction
In this tutorial we're going to set up Jenkins CI/CD tool on AWS EKS cluster using [Jenkins official Helm chart](https://charts.jenkins.io/) and Ansible.

Using the Jenkins Helm chart alone is already a great way to automate your deployments, but adding ansible to the mix brings even more benefits. With Ansible, you can use infrastructure as code (IaaC) to easily manage things like Jenkins plugins, Docker image tags, and ports. 

Using the Jenkins Helm chart alone is already a great way to automate your deployments. Adding Ansible to this process will bring the following **benefits**:
- IaaC (Jenkis plugins, Docker images tags and other parameters).
- Jinja templating for Helm chart values.
- Automate the initialization of jobs (and potentially other tasks) during Jenkins boot.

## Prerequisites
In this tutorial, we will be using AWS Elastic Container Service for Kubernetes (EKS) as our Kubernetes cluster. If you don't have an EKS cluster set up yet, you can also use minikube to run a Kubernetes cluster locally on your machine.

**(Optional)** If you already have an EKS cluster deployed, you can use the following command to update your kubeconfig and enable access to the cluster:
```bash
$ aws eks update-kubeconfig --region aws_region_code --name eks_cluster_name --alias short_name
```

## Jenkins Configuration as a Code Plugin

Next, we'll take a look at the values.yml file from the Jenkins Helm chart and override any variables that we need to customize for our deployment.

Jenkins Configuration as Code (JCasC) is now a standard component in the Jenkins project. JCasc configuration is passed through Helm values under the key `controller.JCasC`.
The section ["Jenkins Configuration as Code (JCasC)" of the page "VALUES_SUMMARY.md"](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/VALUES_SUMMARY.md#jenkins-configuration-as-code-jcasc) lists all the possible directives.

You can find more details on how to specify particular plugins setup in JCasC at https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/README.md#configuration-as-code


## Ansible Role

To automate the deployment of Jenkins with Ansible, we'll create a simple Ansible role that performs the following steps:
- Render the values.yml file from the Jenkins Helm chart
- Add and update Helm repository
- Deploy the Helm chart

```yaml
# roles/jenkins/tasks/main.yml
---
- name: Create temporary file
  tempfile:
    suffix: .helm_values.yml
  register: helm_values_file

- name: Templating Helm Values file
  template:
    src: values.yml.j2
    dest: "{{ helm_values_file.path }}"

- name: Add Helm repository
  command: helm repo add jenkinsci https://charts.jenkins.io
  changed_when: False

- name: Update Helm repository
  command: helm repo update
  changed_when: False

- name: Deploy using Helm
  command: >-
    helm upgrade jenkins jenkinsci/jenkins
    --install
    --create-namespace
    --namespace {{ jenkins_namespace }}
    --version {{ jenkins_chart_version }}
    --values {{ helm_values_file.path }}
  register: shellout
  failed_when: "(shellout.rc != 0) and ('cannot re-use a name that is still in use' not in shellout.stderr)"
  changed_when: "'Installing it now' or 'has been upgraded' in shellout.stdout"
```
Here's an example of the `values.yml.j2` template file from our Ansible role. We're only specifying the variables that we want to override from the default values.yml file in the Jenkins Helm chart. All other variables will use the default values specified in the chart:

```jinja
{# roles/jenkins/templates/values.yml.j2 #}

controller:
  tag: {{ jenkins_controller_image_tag }}
  imagePullPolicy: {{ jenkins_controller_image_pull_policy }}
  resources:
    requests:
      cpu: {{ jenkins_req_cpu }}
      memory: {{ jenkins_req_mem }}
    limits:
      cpu: {{ jenkins_lim_cpu }}
      memory: {{ jenkins_lim_mem }}

  jenkinsUrl: {{ jenkins_base_url }}
  jenkinsAdminEmail: {{ jenkins_admin_email }}

  ingress:
    enabled: {{ ingress.enabled}}
    ingressClassName: {{ ingress.class_name }}            
    annotations: {{ ingress.annotations }}

  additionalPlugins: {{ jenkins_additional_plugins }}
  scriptApproval: {{ jenkins_approved_signatures }}
  
  initScripts:
    - |
      {% filter indent(width=6) %}{% include 'init0-sample-pipeline-job.groovy' %}{% endfilter %}

agent:
  podTemplates: {{ jenkins_agent_pod_templates }}

```
In the template file above, you'll notice that we've specified an additional Jinja template called `init0-sample-pipeline-job.groovy.j2`. This is a Jenkins hook script that will initialize a simple pipeline job during Jenkins startup. Hook scripts allow you to run custom scripts at different points in the Jenkins boot process. You can read more about hook scripts in the https://www.jenkins.io/doc/book/managing/groovy-hook-scripts/.

```jinja
{# roles/jenkins/templates/init0-sample-pipeline-job.groovy #}

import hudson.plugins.git.*
import jenkins.model.*
import org.jenkinsci.plugins.workflow.cps.*
import org.jenkinsci.plugins.workflow.job.*

def instance = Jenkins.getInstance()

def pipelineGitUrl = "{{ pipeline_git_url }}"
def pipelineGitBranch = "{{ pipeline_git_branch }}"
def pipelineGitCredsId = "{{ pipeline_git_creds }}"
def pipelineName = "{{ pipeline_name }}"

def pipelineScm = new GitSCM(
    GitSCM.createRepoList(pipelineGitUrl, pipelineGitCredsId),
    [new BranchSpec(pipelineGitBranch)],
    false,
    [],
    null,
    null,
    []
)

def pipelineJenkinsfilePath = "{{ pipeline_jenkinsfile_path }}"
def pipelineDefinition = new CpsScmFlowDefinition(pipelineScm, pipelineJenkinsfilePath)
pipelineDefinition.setLightweight(true)

def pipelineJob = new WorkflowJob(instance, pipelineName)
pipelineJob.setDefinition(pipelineDefinition)
def pipeline = instance.getItemByFullName(pipelineName)

if (pipeline) {
    cliPipeline.setDefinition(pipelineDefinition)
} else {
    instance.add(pipelineJob, pipelineJob.name)
}

```

Below, we've defined the default variables for our Ansible role in the `defaults/main.yml` file:

```yaml
# roles/jenkins/defaults/main.yml

---
jenkins_chart_version: "4.2.17"
jenkins_namespace: "jenkins"

jenkins_controller_image_tag: "2.377"
jenkins_controller_image_pull_policy: "IfNotPresent"

jenkins_agent_image_tag: "latest"

jenkins_admin_email: "example@mail.com"
jenkins_base_url: "jenkins-test.example.com"

jenkins_req_cpu: "1"
jenkins_req_mem: "2Gi"
jenkins_lim_cpu: "2"
jenkins_lim_mem: "3Gi"

jenkins_additional_plugins: {}
jenkins_agent_pod_templates: {}
jenkins_approved_signatures: []

pipeline_name: "demo-pipeline"
pipeline_git_url:  "https://github.com/nickyfoster/jenkins-helm-ansible-tutorial.git"
pipeline_git_branch: "main"
pipeline_git_creds: ""
pipeline_jenkinsfile_path: "Jenkinsfile"

ingress:
  enabled: yes
  class_name: nginx
  annotations: 
    kubernetes.io/ingress.class: "nginx"
```

## Ansible Playbook

Next, we'll use this role in our playbook to deploy the Jenkins Helm chart to our Kubernetes cluster.

```yaml
# ./jenkins.yml
---
- name: Deploy Jenkins
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Set context for kubectl
      command: "kubectl config use-context {{ kubectl_context_name }}"
      changed_when: false

    - name: "Deploy Jenkins"
      include_role:
        name: jenkins
```

To follow Ansible best practices, we'll redefine our role's variables in the `./group_vars/all.yml` file.

In my AWS EKS cluster, I'm using an Application Load Balancer, so I needed to make some minor adjustments to the ingress configuration. If you're also using an ALB, you can find more information about configuring the Kubernetes ingress to work with it in the https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/#ssl.

```yaml
# part of ./group_vars/all.yml

ingress:
  enabled: yes
  class_name: alb
  annotations: 
    external-dns.alpha.kubernetes.io/hostname: "{{ jenkins_base_url }}"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: "CHANGEME"
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
```

Another advantage of running Jenkins on a Kubernetes cluster is the ease of configuring Jenkins agents in the cloud. With Kubernetes, you can quickly and easily set up and manage Jenkins agents as containers, making it easier to scale your build infrastructure as needed.

```yaml
# part of ./group_vars/all.yml

jenkins_agent_pod_templates:
   python: |
      - name: python
        label: python
        serviceAccount: jenkins
        containers:
          - name: python
            image: python:3
            command: '/bin/sh -c'
            args: 'cat'
            ttyEnabled: true
            privileged: true
            resourceRequestCpu: '400m'
            resourceRequestMemory: '512Mi'
            resourceLimitCpu: '1'
            resourceLimitMemory: '1024Mi'
```

Now we can run our playbook and wait for the Jenkins pod to become available. Once the pod is up and running, Jenkins will be ready for use.

```bash
$ ansible-playbook jenkins.yml -i path_to_your_inventory
```


## Accessing Jenkins after deployment

If you don't have a load balancer or don't want to set up ingress on your cluster, you can use port-forwarding to access your Jenkins installation. To do this, run the following command:

```bash
$ kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
```

This will forward traffic from your local machine's port 8080 to the Jenkins service running in your cluster. You can then access Jenkins by visiting http://localhost:8080 in your web browser.

To log in as an administrator, you'll need the Jenkins admin password. You can retrieve it by running the following command:

```bash
$ kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password
```

Now that you've successfully deployed Jenkins, you can use the [Jenkins Configuration as Code (JCasC)](https://github.com/jenkinsci/configuration-as-code-plugin) plugin to manage your Jenkins configuration as code. The JCasC plugin allows you to define your Jenkins configuration in a file, which you can then check into version control and use to automate the setup and configuration of your Jenkins instance. For more information, you can check out the `http[s]://<your_jenkins_url>/configuration-as-code`.

## Summary

In this tutorial, we covered how to deploy Jenkins, a popular continuous integration and delivery tool, on an Amazon Web Services (AWS) Elastic Container Service for Kubernetes (EKS) cluster using the Jenkins Helm chart and Ansible. We created an ansible role to automate the deployment process and customize the Jenkins installation with the help of Jinja templating. We also discussed how to use the Jenkins Configuration as Code (JCasC) plugin to manage your Jenkins configuration as code. The full source code for the ansible role and demo playbook can be found in a [GitHub repository](https://github.com/nickyfoster/jenkins-helm-ansible-tutorial).


## Notes

Information on how to setup EKS PV:

https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/

My EKS cluster is configured with the following add-ons.
- [AWS Load Balancer](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 
- [ExternalDNS controller](https://github.com/kubernetes-sigs/external-dns)

References:
- https://helm.sh/
- https://plugins.jenkins.io/configuration-as-code/
- https://github.com/jenkinsci/helm-charts



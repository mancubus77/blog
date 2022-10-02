DevSecOps with Tekton and OpenShift
===================================

Incoming call from… CISO
========================

If you are a project lead and you see an incoming call from CISO or his deputy it’s not a good sign. And also I hope you never paid BTC to bad boys who stolen private information of your customers because one of your devs forgot to disable _admin/admin_ on public-facing application or used an old and vulnerable JS library. This is my one of my [favorite stories](https://medium.com/hackernoon/im-harvesting-credit-card-numbers-and-passwords-from-your-site-here-s-how-9a8cb347c5b5) about how easy people can get access to sensitive data, just because devs use libraries without source code review. Devs are brave and creative, they want to write good and efficient software and try new things. If you forbid them to do that, you are slowing an innovation or can lose good people. But new tech might be good and dangerous at the same time. If you are a team lead and work with Kubernetes I assuming your teams use only clean [UBI](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image) images and well know libraries. But how to breed creativity/innovation and keep your job and reputation at the same time? There are many ways to do that, but today we will speak about DevSecOps.

DevOps + Sec = Dev Sec Ops
==========================

DevOps is not a buzzword anymore, you or your company do have them and they bring enormous benefits for the business. These warriors help to automate product delivery with CI/CD pipelines and let devs just push a code and get reports in Stash/Rocketchat/Telegram about build/deployment status. But how is about to add **Sec**urity bit to ensure that automation delivers secure products? That’s how **DevSecOps** appeared. I’m not very good in describing methodologies, so here is a [good explanation](https://www.redhat.com/en/topics/devops/what-is-devsecops#:~:text=DevSecOps%20means%20thinking%20about%20application,DevOps%20workflow%20from%20slowing%20down.) from Red Hat what is DevSecOps.

**Use Case**
============

Let’s assume our team **λ** started to work on a simple application that has front end and backend deployed on OpenShift cluster. Front-end is internet-facing with an exposed HTTPS Route and a simple backend. Also let’s imagine that our company is adopting GitOps, so developers have limited access to the cluster (at least on Staging and Prod) namespaces. Therefore they can not deploy anything via `kubectl`and `oc` commands. And to unify processes DevOps defined a structure of repositories that each team has to follow:

```
.
├── app
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
└── kubernetes
    ├── base
    │   └── be
    │       ├── deployment-config.yaml
    │       ├── image-builder.yaml
    │       └── service.yaml
    └── kustomization.yaml
```

Where **app** folder  used  for application source code and Dockerfile, and **kubernetes** for project manifest files compatible with Kustomize. We have 2 sub-teams for FrontEnd and Backend and they can make commits whenever they want without asking permission. In this case, minimum delivery pipeline (which is Tekton) might look like this

![](https://miro.medium.com/max/1400/1*8FYumqjhpzEyknF4gxpapQ.png)FrondEnd and BackEnd pipeline

_NB: This is a simple case, so we omit all integration tests in CI/CD, for sure in real life you will have more steps like unit tests and integration testing. Also in real life, we won’t have 2 parallel tasks in one pipeline_

If you want to have a look at the pipeline to follow our further discussion, please feel free to apply it to your cluster

```
\# Deploy simple Tekton pipeline
oc apply -k "[https://github.com/mancubus77/devsecops-demo-tekton.git/?ref=simple](https://github.com/mancubus77/devsecops-demo-tekton.git/?ref=simple)"\# Run task
tkn pipeline start build-and-deploy \\
\-w name=sharedworkspace,volumeClaimTemplateFile=[https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/01\_pipeline/03\_persistent\_volume\_claim.yaml](https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/01_pipeline/03_persistent_volume_claim.yaml)
```

_I hope you had a look at repository content before :-)_

So, we have delivery pipeline and each time devs make a commit git triggers CI/CD pipeline, and project will be built and deployed on the cluster. So far we solved a few challenges:
\* We defined unified repository structure
\* We are compatible with GitOps
\* We can separate teams and let them work individually

OK, what is about Security?
===========================

Good question. Nothing stops our Devs from testing new technologies and libraries, but how **do we know** that they are secure and harmless? The answer is simple **we do not**, as we do not check repo nor code quality. So any mistake or vulnerability can jeopardize or compromise a project. To avoid this, let add a few steps for our delivery pipeline:

*   Code scanning of backend code via Python [Bandit](https://github.com/PyCQA/bandit)
*   Code scanning of Node code with [njsscan](https://github.com/ajinabraham/njsscan)
*   Image scanning with [Trivy](https://github.com/aquasecurity/trivy)
*   Manifest scanning with [kube-linter](https://github.com/stackrox/kube-linter)

Therefore, we not only automating CI/CD we also ensure that security vulnerabilities or dodgy code not being deployed in a staging/production environment. Let’s have a look at how the pipeline looks like with DevSecOps’ish approach:

![](https://miro.medium.com/max/1400/1*ChuDK0ddlDs3so-l6UwbWQ.png)DevSecOps pipeline execution

If you wish to have a look at new pipeline deploy updated pipeline on your cluster:

```
oc apply -k [https://github.com/mancubus77/devsecops-demo-tekton.git](https://github.com/mancubus77/devsecops-demo-tekton.git)
```

Let’s stop on each step and explain what we can mitigate while we are running it

```
tkn pipeline start build-and-deploy \\
\-w name=shared-workspace,\\
volumeClaimTemplateFile=[https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/01\_pipeline/03\_persistent\_volume\_claim.yaml](https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/01_pipeline/03_persistent_volume_claim.yaml)
```

**Python-Scan —** scan Python for hardcoded passwords or dodgy libraries. For example, my source code had a typical Flask pattern

![](https://miro.medium.com/max/1400/1*OEfrvY1U4fc31NhEL-fhIw.png)The issue with port binding

And Bandit didn’t allow me to proceed, because binding a port to all interfaces is not a good idea.

![](https://miro.medium.com/max/1400/1*8R4UVpzNerucpBB02Va50Q.png)Error message from Bandit about port binding

**Kube-Linter** — a great product from StackRox, it checks manifest files and reveals configuration issues. Unfortunately, it doesn’t support OpenShift APIs, and as I use “DeploymentConfig” which is not recognized by Kube-Linter, so I bypassing the error. But please consider this tool if you do use other Kubernetes distro.

![](https://miro.medium.com/max/1400/1*SEUYDovm71Q-pdJR5S5WJg.png)Kube-lint error message

Kube-Linter could catch orphan services or absence of CPU/Memory limits.

**build-and-scan**— This task has a few steps to build and scan containers on vulnerabilities. To make scanning in a true Cloud-Native way, instead of mounting the image to Trivy we use Docker side-car container with Trivy container to scan freshly built image.

The steps are:

*   Build an image with buildah and push the image to Docker side-car container
*   Scan the new image with Trivy
*   Push image to the local OpenShift registry

And our base python image from Docker Hub has a lot of surprises:

![](https://miro.medium.com/max/1400/1*A78WzdkO5N3XzCJEJEp3Rw.png)Part of Dockerfile for Backend![](https://miro.medium.com/max/1400/1*KGu9x1hxnKtcz8b1MdVXgg.png)Trivy scan results for backend

Pretty interesting, right? This image has 19 critical vulnerabilities that will be working hand in hand with your code. Please be noted that the severity can be tuned and probably some warnings can be skipped, but it depends on your faith in the code or how convincing you might be in front of InfoSec

How is our Frontend container?

![](https://miro.medium.com/max/1256/1*1oI0OGJS4DjRSEhuNgd08Q.png)Dockerfile frontend![](https://miro.medium.com/max/1400/1*bBNumtd8rXmnW1l5cPMzGw.png)Trivy scan result for frontend

Not better actually, 32 critical vulnerabilities.

**deploy-k8 —** Is a service task that takes our manifest file and deploys them in our namespace. You do not need to create a new serviceaccount as it will be using one from Tekton.

Remediation
===========

The best way to avoid all issues above is to use certified and clean images, for example [from Red Hat](https://catalog.redhat.com/software/containers/search). Let’s change Dockerfile to use Python from Red Hat UBI
`registry.redhat.io/rhel8/python-38`
and trigger pipeline again to see how it change the situation.

![](https://miro.medium.com/max/1400/1*ZRmHs-uQkKy4IwE9MZ-wPA.png)Image scanning with Red Hat image

As you can see we still have some warnings but there are no critical issues, so we can review them and probably relax severity.

Conclusion
==========

If you do not want to be at google news feed with the name of your company and the words `[“Breach”, “Leak”, “Security”]` in the same title or be blackmailed by hacker groups, you might want to consider DevSecOps practices in your organization. DevSecOps does not slow down or stop innovations but allows you to raise red flags to developers if something needs attention.

_PS: There are many security scanners, I used Trivy, Bandit, and njsscan because we used them in our projects before. Feel free to make your own research._

Resources
=========

\[1\] Tekton pipelne (master and simple branches) [https://github.com/mancubus77/devsecops-demo-tekton](https://github.com/mancubus77/devsecops-demo-tekton)
\[2\] App Frontend (NodeJS), [https://github.com/mancubus77/devsecops-demo-fe](https://github.com/mancubus77/devsecops-demo-fe)
\[3\] App Backend (Python), [https://github.com/mancubus77/devsecops-demo-be](https://github.com/mancubus77/devsecops-demo-be)

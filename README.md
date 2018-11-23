#### I'll take this from an AWS standpoint. Tool of choice would still be terraform, yes.
- Set up terraform backend on S3
- Set up terraform workspaces for each developer
- Have a production and staging infra, which are managed by Jenkins (or other tool) jobs running your terraform files
- Provision basics: VPC, as many private subnets as useful to your region, and a public subnet as a SSH bastion and a NAT gateway.
- Provision security items and backing infra such as RDS, S3, IAM roles etc
- Provision your actual application environment - here I'll assume kubernetes for convenience. Ideally this is a mix of spot, reserved and on-demand EC2 instances to keep costs low.
- Setup monitoring and alerting with Cloudwatch and Prometheus (operator is great)
- Set up log collection (either TICK or ELK).
- Deploy the application (which I don't believe is part of the question, but I need to include this for step 10)
- Include autoscaling (HPA and cluster autoscaler if you're on k8s), adjust requests and limits based on usage of your application - you should have all the data you need to make these decisions from the earlier steps
- Not quite "a" server anymore but this I believe will put you in a good position for your application!

-----------------------------------------------------

#### "We're launching a new product and we need the infrastructure deployed. Explain how you would do this and detail the various components."
- I'm going to assume you are on the cloud, since you can't have a good idea of exactly how many servers you need to calculate a 3-year ROI.
> From an operations point of view, I'd be looking at server size (CPU, RAM, Disk),
  - Yawn, those are just the "instance-type" pulldown. You shouldn't care what the values are, because they will change as the developer figure things out.
> load balancing, redundancy, some kind of failover, DNS, security etc.
  - Yup, don't forget monitoring and alerting.
> I've also got a good idea of what CI/CD is.
  - Do you? You supplied a lot of links, but didn't summarize what you would do. And is CI/CD only for the application, or does it apply to the infrastructure too?
> I guess what I'm looking for is a generic architecture checklist
  - There is no perfect 'generic' architecture. But the question asked for "infrastructure", and you only painted a picture for a server. What about databases? What about async jobs? What about caching? CDN?

> "We're launching a new product and we need the infrastructure deployed. Explain how you would do this and detail the various components."
- My answer:

  - First, let's make sure you really need this. Most products should probably start with a PaaS like Heroku or GCE/GKE, or Lambda. Don't pay for people to muck with servers if you don't need to. Most apps are not so unique that they need unique infrastructure.
  - Ok, if you think you need servers. Great. Let's architect this for best practices: 12-factor apps, infrastructure as code, immutable infrastructure, and deploy pipelines.
  - There are lots of ways to get there (Netflix stuff, CloudFoundry), but my favorite is Kubernetes. This gives a nice dotted line between Developers and Operators. Developers think of things "inside the container", plus the application linkages between the services. Ops thinks about boxes and alerting.
  - There should be automation that builds + deploys all the infrastructure + code to the staging AWS account, then once it passes tests, runs it all in the production account. Developers can spin up temporary environments in their own account, but it gets burned to the ground every night.
  - You should be able to run the chaos monkey (and/or gorilla if the biz needs it), and let it kill off any server in production. You should not need to SSH into your infrastructure, except for occasional debug sessions. All changes should be done in a branch in Git, and preferably peer reviewed before merging.
  - Your infrastructure + application should be entirely defined in a few Git repos. Each master branch should 'deploy itself' to production after a commit (with a manual approval step at first.) For anything that isn't 100% magic like that, you should have a Runbook that ensures anyone on the team can perform that work (perform a database fail-over, bounce the applications, block an IP, etc.)
  - For a small team, if you'd like to keep infrastructure management down, I might recommend using ECS with Fargate to run all your containers, and then using RDS to run your DB. For automated deployments, its kind of up to you. Maybe CodePipeline, maybe Jenkins, maybe Drone or something else.

-----------------------------------------------------

We use Fargate, and its great for avoiding having to maintain EC2 infrastructure. However you should know:
- Fargate is pricey
- You no longer get access to the underlying infrastructure
- No volume mounts

Everyone's use cases are different but this has worked well for my company.
- When deploying a new service it will be hosted on the following (in order of preference)
- Lambda While in the free tier, dirt cheap to free. If performance is not an issue and workloads are light to medium we prefer lambda.
- When we hit performance constraints we move to 2. Fargate Fargate allows us to execute long running taske similar to lambda but for us these are usually tasks > lambda 5 min window. These usually run in the background If performance and uptime is of utmost importance 3. ECS Its not kubernetes but it gets the job done.
- We use a mixture of terraform and serverless framework depending on the use case.
 
#### 5 independent cloud services
  - backblaze for storage
  - cloudflare/other cdn
  - databricks/floydhub for ml
  - packet.net for everything else
  - Sendgrid
  - Gitlab
  - PagerDuty

I would suggest that you pick something (maybe a statically generated blog) and start using these technologies to host it (and learning as you go). I'd suggest this order:

- Buy a VPS (shared kernel) -- just to see what people in environments where shared kernel is OK (this is normally the usecase of wordpress/other framework centric plans), note what you can and can't install, how you can get past requirements by locally building software from source, etc.
- Buy a VPS (KVM) -- know the difference between KVM (along the way you'll probably learn about qemu and virtualbox, know how they differ)
- Buy a full dedicated server -- learn a tiny bit about what happens when you need beefier hardware (I recommend hetzner's robot marketplace), and the benefits of being able to run it yourself.
- Buy a domain name (you can use AWS or Cloudflare or my personal favorite Gandi.net)
- Learn about A/AAAA/CNAME/TXT/MX records, enough to point your domain name at the IP for the server you bought, for the appropriate subdomain(s)
- Deploy the blog (or whatever project you choose) to the server (whichever one you chose) by copying the code and running a process that binds to port 80
- Deploy the blog to the server by running a process that copies the code, runs it but binds to another port (let's say 8080 or 8000), and is reverse-proxied to by NGINX (which is listening @ port 80)
- Deploy the blog to the server with the NGINX-proxied setup, but this time use systemd to make the process an enabled service that starts up when the server starts -- you should be able to restart your server now and have the blog come up.
- Deploy the blog to the server by running the process with docker (remember, docker is just sandboxed/isolated processes). For extra points here, figure out how to make docker and systemd process management to play nice, CoreOS has some good blog posts on this IIRC.
- To flex your Docker chops, have NGINX actually run from a container (not as an installed system-level program), pointing at the other container.
- Completely wipe/reset your server, and make your way to the NGINX-powered setup with Ansible. The only part Ansible might not be able to do right away is getting you a machine, but ansible has provisioning so that's kind of a lie (but one that's kind of OK to tell yourself at first).
- Attempt to start learning Kubernetes by reading through the concepts, setting up a cluster the hard way
- Get your blog up and running on k8s
- Teardown your cluster completely, then use kubeadm to rebuild it -- you should be writing config in a manner that once you have a cluster up you can just point at it, and run scripted kubectl commands .
- Teardown your cluster completely, build it back up again kops to try on AWS.
- Try and deploy an application with storage requirements/more ambitious requirements.


My list would be:

- Ansible
- Prometheus
- Kubernetes
- CI/CD (not just CI/CD, but specifically Jenkins declarative Pipelines)
  - despite downvotes, jenkins is what you’ll be working with in 60+% of companies. at least learn your way around.
- Ansible - easy learning curve. Python - as above, and allows extension of Ansible. CentOS and Ubuntu - two major players. Docker Terraform AWS + GCP (in equal measures)
- Terraform is darn fun, I would learn it first


I'm on a 4 person team. We use a combination of GitLab CI/CD and Jenkins. Jenkins has become mostly a way to manage automated jobs that don't necessarily relate to a deployment process. We pretty much use GitLab exclusively for deploying changes to staging and then production environments using Puppet. We also run a pretty even split right now between Windows and Linux systems, with Linux starting to take over a bit more of the mix. This is such a critical piece of our process that everyone on the team is expected to know how to develop for, run, and troubleshoot the pipelines. Obviously each of us has a higher skill level in certain areas. I am the PowerShell/C#/JS, networking, and unit testing Guru. We have another guy who is the Ruby and Git guru, etc. But we all have skills in the same areas and are competent enough to get by. But everyone is responsible for the build process as it's a core competency for how we work.

We were inspired by the Phoenix Project and then read Jez Humble's CI book and we started with simple things that made our lives easier. First we just got to know and understand Jenkins and Git and automated some of the daily chores that were of zero value to us (like account renewals for JV staff) and eventually got to the point where we were running automated unit tests and builds on our in-house tools. From these we started with Puppet. We took our most fragile and least understood system (build by a third party in a language none of us knows well) and completely puppetized it. Then we connected it to a GitLab CD pipeline. From these we added in our Sensu monitoring and then added all of our production systems into Puppet for Sensu. Once we had the confidence that we knew what we were doing and were on the right track we started with other services that problematic for us in some way. Eventually the process improved to the point that we actually had time to start working on engineering integrations and performance improvements. Our systems are monitored, documented (the pipeline code is the documentation), and we don't get crap alerts at night any more. For us I believe the key was starting on something that made us all see the benefit. The first time we got the pipeline to spin up a completely new system and just build itself from a GitLab code push one of my workmates shook his head and his voice cracked when he said, "This changes everything. Everything." And we never looked back.

It's definitely a different way of thinking. What sort of environment are you working in? Mostly Windows? What was revolutionary for me was learning that I could leverage PowerShell unit testing (Pester) to not just unit test the build code but to also perform the integration testing. So when a pipeline goes from code validation (unit tests, style checking, etc) I can leverage the same processes to actually verify that the system did what we expected it to and export it to a dashboard. So I can be sure that my code is manipulating the inputs correctly (unit tests with mocks) and then I can see that the VMs were built to the specs we desired, code was deployed, etc (integration testing).

Start small. Then add more to it. The most important thing at each step is that you are sure it is a completely deterministic system. You should know that it can only be poked in so many ways and what will happen when it gets poked. If that makes sense...

For our first project we first made sure that we could ensure that the tools we required to run the app were all installed on the system. So here was how we went.

1. Can we have Puppet domain join the server, install nano, Git, Ruby, MySQL etc on the server?

2. Can puppet make sure that the code is present on the server?

3. Can Puppet actually configure the web app to run?

4. Can Puppet configure the reverse proxy with an SSL redirect?

5. Can Puppet restore the DB backup if the DB is not already present on the system?

6. Can Puppet install Sensu checks/metrics?

OK, now we have the system fully up and running on a clean, up-to-date system (previously it was on server 2003 and we had no idea how it worked or was built), can we connect it to GitLab so that GitLab can push changes to Puppet?

7. Create a CI/CD pipeline from Git to Puppet to the existing VM.

8. Create a staging environment and a Git branch called staging so code is first pushed there.

9. Dynamically build the VM for the staging environment and run automated tests before the code can be pushed to production.

Each step on the path brought us value. Initially we knew nothing about the system. So going through the process of Puppetizing it got us valuable documentation. Your situation is not likely the same but you probably have a fragile system in your environment that you can take and do something similar to. Even if it is just getting the configuration documented and put into version control. Take small steps that you are pretty sure will succeed. Chain them together into something bigger as yo gain experience and confidence. You can't do everything at once. Small changes compounded over time make huge results.

Make sure you check out Pluralsight. I have had a subscription to them for several years now and they have courses on Jenkins, Chef, Puppet, unit testing, etc. Udemy is nice but the a la carte pricing can be a bit rough if you want to watch a lot of courses... On the other hand, if you aren't watching anything a subscription plan sucks, too...

I would use Ansible:

- Versioning - All edits to your playbooks are version controlled
- Deployments are defined as code - Makes changes trivial and quick
- Desired configruation - Rerun a playbook to bring a machine back into compliance.
- Every deployment is up-to-date

You could use a combo:

- Ansible to build a "master" image then deploy that image, this offsets the deployment time problem of builds with just Ansible BUT brings added overheads such as:
- storage requirements for images
- required "refresh" intervals to update the images.

Ansible (or some other tools of automation that can condense Infrastructure as Code) brings some benefits to the table:

You "pay" for the number of different bakes you need, not for the total number of "machines" dealt with (as with baking different images and cloning them after). But
- forking and extending gets significantly easier, and
- code (YAML files or playbooks) can be stored in a code repository, so you can actually blame changes and roll back if anything breaks.
- And after you tune your configuration playbooks, you can continue with more configuration playbooks: the same tool allows you to handle additional administration tasks.

But if you dockerize, I see how that could turn irrelevant. Still it makes a lot of sense in physical machines and VM's.

If you don't have the time, or your team doesn't/isn't likely to take care in this regards, you may be better with VM's. Packer exists for a reason. You could certainly still use Docker for development as it makes the architecture much more repeatable and documents it as code, even if it's not the final product in production.

Small shop or not, one of the side effects of using containers is how they force you to follow certain design patterns. When those patterns aren't followed and things break, the answer isn't to stop using the technology -- it's to encourage sharing of knowledge and better understanding of that technology.

A lot of negativity around Docker is based on misunderstandings, misuse, and a dislike of the implementation. While there are pros and cons with any technology, I have yet to encounter a scenario where VMs provided a better all around solution to containers. Once you embrace Docker (or containerization in general) for what it is, you find yourself applying its usage patterns without even thinking about it.

Docker certainly isn't a silver bullet, but:

- it can be the sole dependency of a project

- it encourages environment-agnostic applications

- it can speed up onboarding of new developers

- its supported by a number of orchestration frameworks: compose, swarm, ECS, Kubernetes, etc.

Sometimes those alone provide enough benefit to use Docker over plain VMs.

Why not both? Both solve different problems. (Although there's some overlap.)

As for development prospective - both approach are good and Jetbrains products have good support for both cases. If you choose VM over Docker you should also look at Vagrant. In another case I suggest you to familiarize with docker-compose.

With Docker it is easy to spin up different environments, tear down, rebuild, share, etc.

Make sure you setup persistent storage and you won't run into storage issues.

For deployments on large scale containers are the future. Find the right framework, ie swarm or kubernetes, that works for you and enjoy the benefits of high availability, auto scalingand self healing. The fact you made a mistake just tells you you need to change the way you setup and operate the containers nothing else. And don't beat yourself up for what you did, remember only people that work make mistakes.

What was the data stored in? You should really aim to have your containers be stateless so this is a non-issue. If they can’t be stateless, mount the data directories with a -v so the important data is persisted on the underlying host machine. As others have pointed out, this same thing can happen with VMs. You just need to take the correct precautions.

For starters, using containers without some kind of orchestrator managing them (e.g. Kubernetes or Docker Swarm) will lead to more issues over time wrt containers going down/data going missing.

As for the containers vs VMs debate, it depends. VMs can do what containers do and simplify a lot of the operational overhead that comes with running many containers at once (esp. around networking, storage and security), but they are bigger, less portable (think: VMware vs Xen vs KVM,) and take longer to deploy.

Containers are smaller and make it much easier to deploy an app consistently on many different kinds of hardware (if your app allows for it, i.e. it doesn't have any dependencies on the host it runs on). This means that developers don't have to worry as much about the underlying hardware than they do in the case of VMs. However, running lots of them requires a tool like Kubernetes or Docker Swarm to keep track of it all, and getting primitives like networking, storage and inter/intra-process communication right are "more" difficult than they are in the VM world. As well, with the exception of OpenShift (to an extent), administrating containers requires a much higher comfort level with the command line than VMs do. This tends to be challenging for many of the teams I work with that use the command line "as needed."

I normally advocate for containers because most apps don't need an entire VM to run on, which allows development teams to spend less time worrying about hardware and infrastructure and more time on making apps better. However, making apps that work well in an ecosystem like this involves cultural changes around the way teams typically write and deploy software. Because of this and the fact that most big companies are already deeply invested in virtualization, I usually recommend keeping VMs in the short term (using something like Vagrant to manage their deployment) and focussing on making "the process" right so that the move to containers "just makes sense."

Hey buddy. Don't stress, you are already off to a good start. Terraform / AWS go hand in hand. 

I thought the AWS Solutions Architecture Associate course was a good intro.

Terraform is essentially just a way to not use the AWS console. It has almost no syntax to learn and my entire usage is along the lines of:

Need to launch an Ec2 instance, Google terraform ec2 definition, copy paste definition, change a few things.

To me, the hardest part of all of this is the networking.

Kubernetes is a bitch but actually not that crazy. Standalone docker experience is a must before starting to use kubernetes.

It’s easy to become overwhelmed. And you cannot learn everything. Start with the basics. Learn how to provision basic infrastructure with Terraform. This will also teach you some aspects of Azure or AWS. Then focus on deploying a simple WebApp. Then start building on that. Deploy Infrastructure > WebApp> Datasource and configure with Ansible. Real world scenarios are easier to get focus on.

Also, I see myself using Ansible less and less because immutable infrastructure. I say this so you understand that you will learn a tool that you will end up never using at all after a few years.

Don't forget monitoring and log management ELK etc, CI/CD servers like Jenkins etc... very important too.

First continue to learn ansible and get to the point where you can right a custom role for some app. Learn to make it generic so anyone can add their vars and its custom for their setup

From there take on the other things like cicd/terraform/docker and kubernetes.

Remember some of us have been doing this for 10+ years. We didn't learn everything all at once. The key is to always keep learning. Sometimes for me I break things up. I do it in baby steps and build to trying to master something.


The basic idea is to build an artifact in one stage and then deploy it to staging environment and then to production. Now whether this artifact is a docker image or anything else is up to you.

Preferably, there should be just one build stage and then multiple deploy stages using the same artifact, to make sure, you're not deploying different versions for staging and production.

So for your use case, this would probably mean something like:

build -> test -> build-image -> deploy-staging -> deploy-prod

This is the very basic idea and could be extended to suit your needs. What's your use case, are you deploying docker images already?

Terraform / AWS go hand in hand. I thought the AWS Solutions Architecture Associate course was a good intro.

Terraform is essentially just a way to not use the AWS console. It has almost no syntax to learn and my entire usage is along the lines of:

Need to launch an Ec2 instance, Google terraform ec2 definition, copy paste definition, change a few things.

To me, the hardest part of all of this is the networking.

Kubernetes is a bitch but actually not that crazy. Standalone docker experience is a must before starting to use kubernetes.

It’s easy to become overwhelmed. And you cannot learn everything. Start with the basics. Learn how to provision basic infrastructure with Terraform. This will also teach you some aspects of Azure or AWS. Then focus on deploying a simple WebApp. Then start building on that. Deploy Infrastructure > WebApp> Datasource and configure with Ansible. Real world scenarios are easier to get focus on.

google SRE free and DevOps handbook.

I would suggest spinning up your own dev environment in AWS. Think gitlab, Jenkins, maybe the $10 license of jira/confluence, plus some way to connect in. Maybe do logging and alerting too. Take notes, then tear it down. Then automate the same setup with ansible and packer, or similar tech. Then break it down. Then build it with terraform. Now tearing it down and building it is easy.

Now maybe set up a kubernetes cluster using kubernetes the hard way...

Note that getting to the end of this may take a long time. Don’t get discouraged, just setting up servers and securing them is really good practice.

And don’t let the long time gruff veterans get to you ;) This stuff isn’t magic. It’s just hard.

FYI, dont leave things up for a long time because you can rack up quite an AWS bill :)

Last— this leaves out programming. I would start picking up a programming language, be it python or ruby or java. That’s sort of out of the scope of this, but I learn things the hands on way, so if you do too just build, build, build.

First continue to learn ansible and get to the point where you can right a custom role for some app. Learn to make it generic so anyone can add their vars and its custom for their setup

From there take on the other things like cicd/terraform/docker and kubernetes.

Remember some of us have been doing this for 10+ years. We didn't learn everything all at once. The key is to always keep learning. Sometimes for me I break things up. I do it in baby steps and build to trying to master something.

Personally, I would spend the budget on a few months of cloud spend on AWS if that’s your platform of choice - do check out GKE in Google Cloud if possible though if you’re interested in k8s. Everything that you need to get started on these things are freely available, but the experience using them is priceless.

Create a simple API backend in Go backed by a database and/or memory store of choice; Deploy it to AWS in a managed EKS cluster or create a cluster yourself if you really want a challenge; Use and connect to managed AWS databases (e.g. RDS) or run it in the cluster; Manage the infrastructure via Terraform; Handle its secrets via a Vault service in the cluster; Monitor it with Prometheus; Load test it to breaking point, monitor it and learn to prevent and recover it; Performing rolling updates to the app and test canary deployments; Update and scale the infrastructure via Terraform or set it up to be automatically scalable, etc.

Spending time doing the above will typically teach you much more than any course or workshop in my opinion.

To expand on his excellent suggestion.. Incorporate CI/CD into your learning process while you do it. Build, test, tag, push, and deploy all your services using Jenkins, Drone, Travis, etc.

And there are a ton of really good free resources on the net (e.g., youtube, katacoda.com, kubernetes-the-hard-way). But to be honest, like he said, if you are somewhat familiar with the stuff, just digging into the docs and getting hands on building stuff is the best way to learn.

And if you want to stay cloud agnostic, use the money to build a small lab. Enough to build a 3 to 4 node cluster (use spare/used pc's, raspberry pi's, ebay rack-mount servers, etc), a 5-port switch and some network cables.

For example, I bought a 4-blade 2u supermicro box for $250 off of ebay, a cheap 8 port 10/100 switch, and a ubiquiti EdgeRouter x. The server is very loud of course, so maybe not great for the house, but you get the idea. I frequently tear down and rebuild hosts to the point that I am starting to look into PXE booting to make things easier.

I'm AWS certified. What I did to pass the exam: Use the AWS Free trial if you didn't. Almost everything can be learned using that tier.

If you want courses: https://www.udemy.com/aws-certified-solutions-architect-associate best starting course ever.

The problem with AWS is the amount of tech involve. I use AWS everyday at my job and the half of the services are a mistery for me :/

For Go (my main language) and Vault (little knowledge, we use Ansible Vault which is not the same but covers our needs), don't waste a dime. The docs are awesome.

Kubernetes. Spend money on cloud providers. Docker and Kubernetes have amazing docs.

People keep talking about dependency issues but that’s not why every modern company is shifting their workload to orchestrated containers. Yes, Docker lowers the friction between developer and production environments by ensuring the environment devs code in is exactly the same as the environment code runs on in production. That’s true.

But the main reason to use containers is that, if you want to effectively run microservices and use your cloud compute cost-effectively, you really have to use container orchestration (kubernetes, nomad, mesos). Here’s an analogy. Container orchestration is kind of like an OS for your cloud. Just as a CPU efficiently schedules processes by interacting with various resources in your computer, Kubernetes efficiently schedules containers (which are essentially single process if you’re doing microservices correctly) across your machines in the cloud. You need containers because there’s no other way to kill a process or let it expire on one machine and then revive it on another machine at a different time.

Part of DevOps is taking that XML config updating per environment and deploying code the same way every time through your CICD pipeline. A GUI is pointless. Everything should be organized, and repeatable with very little user interaction, besides maintaining the deployment code base and hitting the deploy button in your CICD tool of choice.

No, at my work we use powershell and teamcity. All of the code is source controlled and the scripts we write deploys all the code. We just tell it what environment and press deploy. Obviously there's a learning curve but when you focus on automating each piece it becomes repeatable and plug and play, the key is organization and making sure you're doing things the cleanest way possible in code and in process. It does take a DevOps skill set to get that going properly though.

This is not a dumb question, at ALL. This shit is really complicated, and don't let blowhards gaslight you into believing that they all understand it, just because they can use it. So the word "container" doesn't mean anything super specific - which doesn't help. Broadly speaking, you can think of containers like virtual machines that share the same running kernel. That last part is the important one - since it's pretty mind-bending and novel. For the sake of your sanity I'm gonna limit this to Docker. I'll write all this out about halfway for my own benefit, since I need to train my own team up on this stuff in the next few months.

The Linux kernel added a few very interesting new features over the last several years, and containers are a nice "shiny" way to use them all at once. Those features are Namespaces, CGroups, and seccomp-bpf. Together with chroot, which you might already know, containers happen. CGroups let you fence off CPU resources for a process.(if you're old enough to remember ye olde Renice, this is like that on steroids), Namespaces let you do cool stuff like have 2 PID 1s(!), 2 applications running on the same port(!) and something to do with the filesystem which made my brain hurt when I tried to make sense of it. seccomp-bpf just fences off what system calls your process can make(your container, for instance, cannot connect itself to the network or reboot its host). Chroot limits a user's filesystem access by pretty much making a "jail" where `/` is a new location of your choice. As you might imagine, all that magic comes at the expense of abstractions and translations for networking, filesystems, and DNS. I'd study Docker networking separately once you get the nuts and bolts. (then envision seven more levels of abstraction hell, and that's Kubernetes networking)

​

So all that was Linux...which is the primary runtime for containers. But people don't write code on Linux most of the time - they use MacOSX or Windows. The MacOSX runtime is a little closer to native since it's using the Darwin/BSD kernel...but Windows is an entirely different thing. Remember how I said they share a running kernel? So Windows had to implement it's own Linux subsystem to have a running kernel for Linux containers in Docker to access. This stuff is so important to the business that Microsoft added Linux to Windows.

Generally speaking, the primary allure of Docker is the same one that we've all heard a ZILLION times in this business - "write once and run anywhere!" Except this time it's pretty much true!

The usual lifecycle looks like this:

Write your application in an IDE and make it work.
Use a "Dockerfile" to provide instructions for how to make that application work in a container(pull these dependencies, open this port, start this process, etc).
Build an image with this and push it to a container repository, like Dockerhub, that AWS one, or Google Container Registry. Use a tag to indicate a version or some other tiny bit of metadata.
You or your ops team(probably via Jenkins) then pull that image(often based on tag)and deploy it in Kubernetes or one of the Docker things like Compose or Swarm. At that particular level, even more abstraction and fun happens, but do not look into this abyss yet or you will turn into a pillar of salt.


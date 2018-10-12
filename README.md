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
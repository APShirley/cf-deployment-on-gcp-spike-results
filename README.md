## __Notes for the spike__ (deploy director)

#### First attempt
We followed [these docs](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/bosh/README.md) and got a director running quickly.
- Install gcloud cli (`brew cask install google-cloud-sdk`)

- Log in (`glcoud init`) and point at our environment

- Next `export GOOGLE_CREDENTIALS=$(cat ./CF-Release-Integration-3df44d304ba5.json)`

- Terraform `terraform plan .`

- Terraform `terraform apply .`

- ssh to bastion vm with `gcloud compute ssh bosh-bastion`

The process is fairly straightforward and could be adapted to deploy the director with bosh-init locally instead of from a "bastion" jump box deployed in the same network on gce by allowing network access to the director using firewall rules.

After deploying, targeting the director is as simple as running `bosh target 10.0.0.6 my-bosh` `bosh login admin admin` from the bastion vm.

#### Second Attempt

Deploying without the jump box using terraform has some caveats:

- Setting up ssh may not be possible using only terraform and sane firewall rules, firewall rules allow traffic to all vms in the network, if other vms are placed in the director's private network they will all have ssh traffic forwarded to them.
- Setting up ssh using terraform and [forwarding rules](https://cloud.google.com/compute/docs/load-balancing/network/forwarding-rules) is also problematic, as you can not create forwarding rules for an instance that does not yet exist, and the instance is brought up by bosh-init after having terraformed the other resources. This is a chicken-egg problem with no immediately obvious solution.
- There is currently a bug in bosh-init which means ssh will not work for project names longer than 13 characters (they are aware, fix on the way).

Setting up the second deployment requires getting the static ip terraform created from the terraform.tfstate file, and inserting it into the manifest in the appropriate places (marked in comments). You also need to add the cf-ssh tag to the private network to match the firewall rules we created with terraform.

~~To target the director from this spike: bosh -t 104.198.98.95, username: admin, password: TgszSiT2vMnf071ZojkA3xM3s1k=~~ (we deleted this)

### Third Attempt

Found out that in order to get GCE backend service health checks to work, we require [routing-release](https://github.com/cloudfoundry-incubator/routing-release/commit/d5c94f315a087ea1fdffcacf2c0025c271377390) with the healthcheck_user_agent set to the user-agent used by gce health check (GoogleHC/1.0)


### Fourth and Final Attempt
We decided to delete everything that could be deleted within the account and repeat the whole process, precisely documenting exactly what was necessary. We did this to confirm our own understanding of the new network complexity, most of which was put in place by flailing on the GUI while we were learning in the previous attempt.

These steps don't represent the iterative experimentation of this attempt, just the final, repeatable sequence. In order to make these steps work, we've given all instances ephemeral public IP addresses, added the cf-public tag to a "public" network that needs to be added to instances that must be exposed to public traffic (router and the diego ssh proxy). We also codified basic cert generation in a script.

As a note, before we start we **DESTROY EVERYTHING**, leaving only the empty GCE project. We did this with the IaaS, not with bosh, terraform, et al. Basically, we used variations of `gcloud compute instances list | grep vm | grep us-west1-a | cut -d\  -f1 | parallel gcloud compute instances delete {} -q --zone=us-west1-a`, then scrounged through the UI to find things that `terraform destroy` complained about the existence of and deleted them by hand. Bosh & Friends are able to tear down only what they've created; cleaning up manual tinkering is painful.

#### Get yo'self a BOSH DIRECTOR
0. From the top of the project, `export GOOGLE_CREDENTIALS=$(cat CF-Release-Integration-3df44d304ba5.json)`. Of course, other accounts will need other creds in this file.
0. From the terraform directory, `terraform apply`
0. From the bosh-init directory, modify the `manifest.yml` on the lines with the comment `# Replace with the static IP from the terraform state file`. We got the correct IP address not from the state file, which is big and ugly, but from `gcloud compute addresses describe director-address`, which is small and sane.
0. Update IP address in the cert generation script to match the one for the bosh director.
0. Run the cert generation script.
0. Update the cert in the bosh manifest with the generated cert.
0. Export appropriate values for `BOSH_CA_CERT`, `BOSH_USER`, and `BOSH_PASSWORD`.
0. Generate an ssh key for ssh'ing to the bosh director: `ssh-keygen keys/key -C vcap`. (This corresponds to the `ssh_tunnel` section in `bosh-init/manifest.yml`.
0. Use the Google Cloud UI to add the ssh key to the project's "Metadata" section.
0. From the bosh-init directory, `bosh create-env manifest.yml`.
0. Target that env so you don't have to specify it all the gosh-darn time: `bosh env [DIRECTOR IP ADDRESS]`

#### AND A CF TOO
0. Add DNS entries pointing to the appropriate values from the output of `gcloud compute addresses list`:
  - A wildcard entry for the `cf-static-ip` value
  - A ssh. entry for the `diego-ssh-static-ip` value
0. Update/check the values in env-vars.yml to ensure they're right for your environment - make sure they match your new DNS records, for instance.
0. `bosh update-cloud-config cloud-config.yml`
0. From `deployments/cf/`, run `bosh deploy --var-files=env-vars.yml -d cf cf-deployment.yml`
0. You are done. Be happy.

Cloud Config Notes:
- We've combined the IP ranges previously used in z3 into z2 in the private subnet, because the GCE environment doesn't have a third AZ.

#### Useful links:
https://cloud.google.com/sdk/docs/quickstart-mac-os-x

https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/bosh/README.md

Cunnie's bosh-init manifest example (not terraformed)
https://github.com/cunnie/deployments/blob/master/bosh-gce.yml

## __Deploying CF__
We followed [these docs](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/cloudfoundry/README.md#deploy-cloudfoundry) to deploy CF.

We are deploying the [new bosh-2-0 manifest](https://github.com/cloudfoundry/cf-deployment/), using the new [golang bosh cli](https://main.bosh-ci.cf-app.com/pipelines/bosh:cli/resources/release-bucket-darwin).


#### Useful links:
https://github.com/cloudfoundry-incubator/bosh-google-cpi-release/blob/master/docs/cloudfoundry/README.md#deploy-cloudfoundry



## __Deploying Internetless CF__

Deploying without internet access for any vms at all (except for the director which we will treat as a jumpbox of sorts).

First steps:

\*Note: we are doing these experiments manually via ui first, we can automate as a next step

 0. Figure out how to load balance the cf routers internally.
  - Internal load balancers require a health check, we are using the /v2/info endpoint as a workaround.
  - Internal load balancers can't point directly at specific instances, so we have to define instance groups of 1 instance in each AZ.
 0. Figure out how to load balance the diego ssh_proxy internally.
  - GCP load balancers may not work because internal load balancers require a health check (feedback for the Diego team).
  - We are using a TCP connection to 2222 on the diego ssh proxy as a _proxy_ for a health check.  This will not be useful in production, since there is no way to take down the ssh proxies without also losing traffic (since the health check and normal traffic both pass through the same port on the same load balancer)
 0. Figure out DNS resolution (pointing at internal load balancers instead).
  - We modified DNS record sets in Route 53 to point them at the new local (10.10.x.x) IPs of the LBs.

After we get this working, we will remove external network access and fix the things that break. We will additionally need to add the CATS errand to this deployment which means consuming cf-release.

 0. After removing network access (redeploying without external ips) the `cloud_controller_ng` job on the api vms failed to start.
  - We determined the bug was occurring in the following way:
   0. Monit would attempt to start all the jobs in the instance group and wait for them to finish (30 seconds from start of job).
   0. The `cloud_controller_ng` job would wait for the blobstore to become available
   0. The `consul_agent` job hadn't come up yet (this is internetless so DNS is resolved by monit)
   0. Monit would decide that the cc job hadn't come up, and mark it as `execution failed`


GCP Internetless Summary:

0. Decide on two private IP addresses for your router load balancer and your diego ssh-proxy load balancer.
0. Create a CF deployment manifest with no external IP addresses (you can use vm extensions to accomplish this task).  Make sure that the deployment has running-security-groups that allow connections to your router and ssh-proxy load balancer addresses. Also, make sure that the dns for the cloud-config points to google's internal DNS server at the 10.x.0.1 (the first .1 address in the network.  For example, 10.200.0.1 in a 10.200.0.0/16 network).
0. In Google Compute, create separate instance groups for each combination of az and router/diego ssh proxy.  If you have a single AZ, there will be two instance groups.  If you have two AZs, there will be four -- one for each [az1,az2]\*[ssh-proxy,router] combination.
0. In Google Compute, Create two private Google Compute load balancers using the IP addresses you decided up front.  One for the router instance groups, and one for the ssh-proxy instance groups.  The router instance groups should forward ports 80,443,4443 to the router instance groups. Its health check should check the /v2/info endpoint on port 80 of the router.  The ssh-proxy instance groups should forward port 2222 to the diego ssh proxy instance groups.  Its health check should just do a tcp connection to 2222. These are temporary workarounds that are not great. The healthcheck endpoint should stop responding before the routers stop processing requests to give them time to drain during a rolling deploy for example. There are stories to add a healthcheck endpoints to both the diego ssh proxy and the router job.
0. Setup a DNS server to point to the load balancer for the routers.  The ssh.domain-name.cf-app.com should point to the ssh-proxy.
0. Run CATs.  Most of them should pass. These should and will fail: `internet_dependent`, `diego_docker` (uses dockerhub), `security_groups` (explicitly tests the opposite of this configuration).

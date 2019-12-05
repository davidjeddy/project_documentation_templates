
David E 8:22 AM
This is not a high prio. task but I wanted to re-visit our conversation concerning GL/TF using access tokens to AAA to the GCP bucket containing the TF state file. I have what I think are the main steps of action but want to review them and my reference architecture implementation. Again, not important but would like have the conversation..
Charles 8:56 AM
okey dokey
in the ci job, container should auth against vault using the ephemeral creds on the node.  those creds are permitted to request the short lived SA tokens.  one would be the equiv of the terraform.tenerum.io service account in the main org IAM now
so auth via ephemeral, retrieve creds for run, do work, expire creds (this should happen automatically)
for things that are NOT in the pipeline that need to auth against vault (like application space) we use the “approle_auth” backend
David E 8:59 AM
Where do the creds on the node come from?
Charles 8:59 AM
resource "vault_approle_auth_backend_role_secret_id" "certificate-issuer" {
  backend   = "approle"
  role_name = "certificate-issuer"
}
ephemeral creds on the node
google thing
David E 9:00 AM
Tl;DR process:
- Use Vault to create a service token to access GCP bucket that contains TF state
  - GitLab Runner (GL) node uses Google creds to auths against Vault
  - Vault confirms authentication of GL runner
  - GL Runner requests access token to GCP Bucket containing the TF state of project
  - Vault confirms Authorization of request and returns token
  - GL triggers TF in pipelinme providing bucket access token as well as auth to change resources inside GCP
  - TF executes validate/plan/apply steps as is typical
  - GL indicates to Vault to expire the token OR timeout is reached and token is expired
  - GCP bucket is no longer accessible using originally provided token.
(edited)
y/n?
Charles 9:00 AM
really probably need to learn the google iam side very well first
David E 9:00 AM
Also, its 6a over there, go to bed. :stuck_out_tongue:
Charles 9:01 AM
yeah, except the scope for the SA that vault requests is goign to be more than reading that bucket
b/c it will use the same SA to do all the actual terraform work (creating projects, deleting k8s masters, etc)
David E 9:02 AM
Ok, I was thinking the auth to read/write the state file would be different than exec. changes in GCP. But it makes sense to use the same SA. Let me update the steps.
Charles 9:03 AM
first step will be to define the sa scopes you want to request
then configure the scopes for which SA’s are allowed to request it
David E 9:04 AM
:thumbsup:
I am think I will setup a reference project  that does just this; once I get the process working we can apply it to other projects. A sort of reference architecture project.
Charles 9:04 AM
# Enable gcp generation by privileged accounts
# https://www.vaultproject.io/docs/secrets/gcp/index.html
resource "vault_gcp_secret_backend" "gcp" {
  default_lease_ttl_seconds = 3600
  max_lease_ttl_seconds   = 86400
}
data "template_file" "gcp-dns-admin-json" {
  template  = "${file("${path.module}/gcp-secret-bindings/dns-admin-binding.json")}"
  vars {
    GOOGLE_CLOUD_PROJECT    = "${data.google_project.master.name}"
  }
}
i had all this working before i nuked it the first time
David E 9:07 AM
If you had it working, any point in my trying to re-implement it other than practice/experience in doing it? (edited) 
Charles 9:07 AM
vault-gcp-auth.tf 
# Enable gcp service account authentication.  This is used by service accounts.
# https://www.vaultproject.io/docs/auth/gcp.html
resource "vault_auth_backend" "gcp" {
  path = "gcp"
  type = "gcp"
Click to expand inline (36 lines)


this file enables the gcp auth backend, configures a role for gcp-terraform (those policies don’t exist anymore, have to make new policies)
so when an ephemeral account auths against here, he can request one of those roles and gets a SA back.  i didn’t completely finish this, so it’s just returning the “terraform@tenerum-io.iam.gserviceaccount.com” account we use everywhere (but with a short lived token)
vault_gcp_auth <-- authorizes an existing gcp service account
vault_gcp_secret <-- returns (short-lvied) service account tokens to authorized clients
David E 9:11 AM
:thumbsup:
Charles 9:11 AM
probably still good to do it to learn how it works and to get familiar with gcp iam
David E 9:11 AM
sounds good.
Charles 9:12 AM
i didn’t get it working completely, just enough to auth those pipeline executions with master blaster permissions.  with my strategy, any node would be able to escalate permissions to be able to delete the whole domain :smile:
David E 9:12 AM
opps
Charles 9:21 AM
so that’s the next thing --
let’s run with the idea that we will push the gitlab runners off to a secure node
a node that only runs terraform jobs maybe
and that’s the only SA that’s allowed to request the escalation

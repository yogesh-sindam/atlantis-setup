########################################

## 1. Setup your server.yaml file
Create your config file:

$sudo nano /etc/atlantis/server.yaml

repos:
  - id: "localhost/root/your-repo"
  - id: "localhost/root/*" # Allow all repos under "root"
  
# Optional: Define workflows if needed
workflows:
  default:
    plan:
      steps: ["init", "plan"]
    apply:
      steps: ["apply"]
	  
	  
## 2. Set up the Atlantis server command
Now, start Atlantis with the auth and URL configs as CLI arguments:
atlantis server \
  --atlantis-url="http://localhost:4141" \
  --gitlab-user="your-gitlab-user" \
  --gitlab-token="your-gitlab-token" \
  --gitlab-webhook-secret="your-webhook-secret" \
  --repo-allowlist="localhost/root/*" \
  --config="/etc/atlantis/server.yaml"
##############################################
QUESTION
can we integrate a simple policy check  also in this yaml
##################################################
Yes — you can integrate a policy check directly in server.yaml using custom workflows. 
Let’s add a basic one that ensures Terraform resources have a tags block for better accountabili

1. Setup your server.yaml file
Create your config file:

$sudo nano /etc/atlantis/server.yaml


1. Extend server.yaml with a policy check
 
repos:
  - id: "localhost/root/your-repo"
  - id: "localhost/root/*" # Allow all repos under "root"
  
workflows:
  default:
    plan:
      steps:
        - run: terraform init
        - run: terraform plan -out=tfplan
        - run: conftest test *.tf --policy=policies/terraform.rego
    apply:
      steps:
        - run: terraform apply tfplan
		
Explanation: run: conftest test *.tf --policy=policies/terraform.rego: This integrates OPA (Open Policy Agent) using conftest to enforce Terraform policy checks during the plan stage.
terraform apply will only run if the policy passes.

2. Write a simple terraform.rego policy
Create a folder called policies/ and add a file:

mkdir policies
nano policies/terraform.rego

package main

# Check all resources have "tags"
deny[msg] {
  resource := input.resource
  not resource.config.tags
  msg := sprintf("Resource %s missing tags", [resource.name])
}

## 3. Install Conftest (if not installed)
If you don’t have conftest yet:

bash
$sudo apt-get install conftest

## 4. Test Atlantis with the policy check
When you run atlantis plan, if your Terraform file is missing tags, you’ll get a failure like:

 2. Set up the Atlantis server command
Now, start Atlantis with the auth and URL configs as CLI arguments:
atlantis server \
  --atlantis-url="http://localhost:4141" \
  --gitlab-user="your-gitlab-user" \
  --gitlab-token="your-gitlab-token" \
  --gitlab-webhook-secret="your-webhook-secret" \
  --repo-allowlist="localhost/root/*" \
  --config="/etc/atlantis/server.yaml"

###############################
another simple rego file 
##

package terraform

# Check if the "policy" configuration block exists in the Terraform plan.
deny[msg] {
  # Look for resource blocks with the "policy" configuration.
  count([r | r := input.resource_changes[_]; r.change.after.policy]) == 0

  msg := "Policy configuration is missing. Please ensure the 'policy' block is defined."
}
#########################

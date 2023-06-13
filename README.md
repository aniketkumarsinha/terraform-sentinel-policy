<div align="center"><h1>Implementing Policy-as-Code to Terraform workflow using Hashicorp Sentinel</h1></div>

![Sentinel Cover Image](https://res.cloudinary.com/practicaldev/image/fetch/s--gXKaJXCo--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nc9cdrlq301bl54d0gqw.png)

In this project, we are implementing Policy-as-Code (PaC) to our Terraform workflow using Hashicorp Sentinel. PaC is a way of defining and enforcing policies for your infrastructure as code, which can help you ensure compliance, security and best practices across your organization. Sentinel is a language and framework for writing and applying policies to Terraform and other Hashicorp products.

## Why use Policy-as-Code?

Policy-as-Code has many benefits for managing your infrastructure as code. Some of them are:

- It allows you to codify your policies and store them in version control, which makes them easier to track, review and audit.
- It enables you to apply your policies consistently and automatically across your environments, which reduces human errors and increases efficiency.
- It empowers you to enforce your policies at different stages of your workflow, such as plan, apply or destroy, which gives you more control and visibility over your infrastructure changes.
- It supports you to write policies that are flexible and expressive, which can handle complex scenarios and logic.

## How to use Sentinel with Terraform?

Sentinel integrates seamlessly with Terraform Cloud and Terraform Enterprise, which are platforms for collaborating and automating your Terraform workflows. To use Sentinel with Terraform, you need to:

- Write your policies in Sentinel language and save them as .sentinel files in your repository.
- Configure your Terraform organization and workspace to enable Sentinel and specify which policies to apply.
- Run your Terraform commands as usual and see how Sentinel evaluates your policies against your configuration and state.

Sentinel policies can be applied at different levels of granularity, such as organization, workspace or run. You can also use different enforcement modes, such as advisory, soft-mandatory or hard-mandatory, depending on how strict you want your policies to be.
1. Advisory: Failed policies never interrupt the run, just post a warning.
2. Soft-mandatory: lets an organization owner or a user with override privileges proceed with the run in the event of failure. Terraform Cloud logs all overrides.
3. Hard-mandatory: requires that the policy passes. If a policy fails, the run stops. You must resolve the failure to proceed.

Learn more about Sentinel: [Introduction to Sentinel, HashiCorp Policy as Code Framework By Armon Dadgar, CTO Hashicorp](https://youtu.be/Vy8s7AAvU6g).

## Terraform without Sentinel:
![Terraform without Sentinel](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tlyth8stff6gxsorhqhk.png)

## Terraform with Sentinel:
![Terraform with Sentinel](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fkx2ms86mjmaaudh5zmd.png)

---

We are using a policy to restrict VM size, which basically means that if the VM size mentioned in our infrastructure matches the list VM sizes mentioned in our policy, then the Policy checks will pass and proceed to Apply phase, otherwise it will stop with the error.

And we are adding this policy directly to Terraform Cloud via UI.
- So go to your workspace page in Terraform Cloud - https://app.terraform.io/app/{username}/workspaces
- Move in to Settings from left pane.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qn76di9ommlkh87cly65.png)
- Move to `Policies` tab from left pane and tap on 'Create a new policy' button. This will lead to the page where you can add your sentinel policy code.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cox8vmy1nmg8n5vh94ls.png)
- Finally Create a new policy
 - Choose **Sentinel** in `Policy framework`.
 - Add the policy name.
 - Add the description.
 - Choose the enforcement level (advisory, soft-mandatory or hard-mandatory)
 - Add the Policy Code. We are using this code:
 ```
 ##### Imports #####

import "tfplan"
import "strings"

##### Functions #####

# Find all resources of a specific type from all modules using the tfplan import
find_resources_from_plan = func(type) {

  resources = {}

  # Iterate over all modules in the tfplan import
  for tfplan.module_paths as path {
    # Iterate over the named resources of desired type in the module
    for tfplan.module(path).resources[type] else {} as name, instances {
      # Iterate over resource instances
      for instances as index, r {

        # Get the address of the instance
        if length(path) == 0 {
          # root module
          address = type + "." + name + "[" + string(index) + "]"
        } else {
          # non-root module
          address = "module." + strings.join(path, ".module.") + "." +
                    type + "." + name + "[" + string(index) + "]"
        }

        # Add the instance to resources map, setting the key to the address
        resources[address] = r
      }
    }
  }

  return resources
}

# Validate that all instances of a specified resource type being modified have
# a specified top-level attribute in a given list
validate_attribute_in_list = func(type, attribute, allowed_values) {

  validated = true

  # Get all resource instances of the specified type
  resource_instances = find_resources_from_plan(type)

  # Loop through the resource instances
  for resource_instances as address, r {

    # Skip resource instances that are being destroyed
    # to avoid unnecessary policy violations.
    # Used to be: if length(r.diff) == 0
    if r.destroy and not r.requires_new {
      print("Skipping resource", address, "that is being destroyed.")
      continue
    }

    # Determine if the attribute is computed
    if r.diff[attribute].computed else false is true {
      print("Resource", address, "has attribute", attribute,
            "that is computed.")
      # If you want computed values to cause the policy to fail,
      # uncomment the next line.
      # validated = false
    } else {
      # Validate that each instance has allowed value
      if r.applied[attribute] else "" not in allowed_values {
        print("Resource", address, "has attribute", attribute, "with value",
              r.applied[attribute] else "",
              "that is not in the allowed list:", allowed_values)
        validated = false
      }
    }

  }
  return validated
}

##### Lists #####

# Allowed Azure VM Sizes
allowed_sizes = [
  "Standard_A1",
  "Standard_A2",
  "Standard_D1_v2",
  "Standard_D2_v2",
]

##### Rules #####

# Calls the validation function
vms_validated = validate_attribute_in_list("azurerm_virtual_machine", "vm_size", allowed_sizes)

# Main rule
main = rule {
  vms_validated
}
 ```

 - For now keep the Policy set as blank because we haven't created any policy set yet.
 - And then tap on Create policy button.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ckofyermqa0zptlgj0r1.png)
- It's time to create a Policy Set. Policy sets are groups of policies which may be enforced on workspaces. In settings itself, move to `Policy Set` from left pane, and click on 'Create a new policy set'.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/97htuz0ig0zld6hacba0.png)
 - In `Connect to VCS`, choose 'No VCS Connection', as we not using any GitHub repo for our policy. We can also use any repository containing policies.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ucyz979s193ojnk62rya.png)
 - It will directly lead you to `Configure settings` page.
  - Choose **Sentinel** in `Policy framework`.
  - Add a name.
  - Add description to policy set.
  - In `Scope of policy`, choose the scope according to your need. We are restricting the scope to selected workspace.
  - Choose the workspace. We are using an already created workspace. To learn how to create a workspace and deploy to Azure using the workspace, check out this blog: [Deploy Azure Infrastructure using Terraform Cloud](https://dev.to/this-is-learning/deploy-azure-infrastructure-using-terraform-cloud-3j9d)
  - And tap on to `Connect policy set` button to create the policy set.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/78yeftf102a9kajyhcee.png)
 - We are done with creation of the policy set.
- Move back to policies to add this created policy set to your policy. And tap on Update policy button.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/arj1yaeu8b5hxokhusd9.png)

So, finally we are done with adding our sentinel policy to Terraform Cloud.

It's time to check the workflow.
So, in our infra, we have used VM with size, "Standard_D1_v2", which is a part of allowed VM sizes, so let's see the workflow outcome.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o21oa2xvuyjg84ick766.png)
We can that we have new phase named as 'Sentinel policies passed', and since the infra matches the policy condition it passes and proceeded to Apply phase.

Let's try with some different VM size.
We are changing the VM size to "Standard_D2_v5" which is not a part of allowed VM sizes, and running the workflow.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cl41qcpe9snsznqhax4u.png)
We can see that the sentinel policy phase failed and stop the workflow there itself because we applied enforcement mode as hard-mandatory.

Error message: Resource azurerm_virtual_machine.vm[0] has attribute vm_size with value Standard_D2_v5 that is not in the allowed list: ["Standard_A1" "Standard_A2" "Standard_D1_v2" "Standard_D2_v2"]
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/34pei9g22o8dopy9hvrz.png)

Let's me just give you a small brief that how the policies are working. So, the Plan phase generates the mock files (containing the output of Plan phase) which is used as input to Sentinel policy phase, and then the policy phase checks with the policy and accordingly pass or fail.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/epj9vmhh8xki94c5qz82.png)
We can even download those mock files to see the results.

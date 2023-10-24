# Part 5: Continuous Deployment (CD)

## Pre-requisites
- Complete [Part 0], [Part 1], [Part 2], [Part 3] and [Part 4]

## Summary 

After a successful run of the CI pipeline, your team is looking to complete the process with a CD pipeline that will handle the deployment of the model without introducing any downtime in production (otherwise termed a "hot swap").

The goal of this section is to get a fully functional CD pipeline that will:
    
1. Trigger based on the creation of a Pull Request (PR) to main.
2. log in to Azure using a Service Principal to be able to leverage the Azure ML CLI commands in your workflow.
3. Create a model API endpoint (web service) using an Azure ML Managed Endpoint and deploy the model to the endpoint into one of the two deployment slots (blue/green slots, which will switch staging/production roles).
    - Test the deployment to the endpoint of the new model.
    - On the success of the test, swap the deployment to accept 100% of the service endpoint traffic (and therefore become 'production').
4. Add a Branch Protection rule in GitHub.

## Task 1:

1. You define triggers as part of a GitHub Actions workflow. The triggers for this workshop have already been defined in `.github/workflows/workshop_cd.yml`. Please review this file to understand how we've established a trigger mechanism to enable a deployment of code that has successfully passed CI and is ready to be deployed to production.

    Review the key elements of the trigger section:

    - 'workflow_dispatch': this enables running the CD pipeline on demand from the GitHub UI as this will greatly facilitate testing. In practice, you would eventually remove this trigger type and fully depend on the rest of the automation.

    - 'pull_request': defines the trigger to kick in when an integration is open to main, and a specific set of files have been modified in the pull request (any code change, or CD definition change that would justify pushing a new deployment live). See [Events that Trigger Workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) for details about the trigger syntax and the type of controls available to build your custom trigger rules.

2. The CD workflow will rely heavily on the Azure CLI to control the infrastructure and implement the automation of the model deployments. Therefore, we need to set up this workflow to login into Azure via a Service Principal to be able to leverage the Azure CLI.

    > **Action Items:**
    > 1. Open up the `workshop_cd.yml` file in your repo under _yourname-dev branch_ (.github/workflows location)
    > 2. Update the 'creds: ${{ secrets...' section in this file to set up your secret name. Follow the instructions in this file annotated with #setup.

    >**Note:** If the name is already updated and matches with the name of the Azure Credentials secret, you can leave as it is.

    > **Note:** Please refer to [Use the Azure login action with a service principal secret](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-a-service-principal-secret) to create the proper Azure Credentials if you haven't done so already (you should have already defined such secret to complete the CI part of the workshop, i.e. [Part 4]).

3. We will now configure our Azure ML deployments and the GitHub workflow which will automate these deployments.

    - Two files control your Azure ML deployments:
      - `/core/scoring/endpoint.yml`: this is your endpoint. Think of this as a virtual load balancer to the actual 'deployments' (actual web services hosting your models,i.e. deployments)
      - `/core/scoring/deployment.yml`: this defines an actual deployment to an endpoint. 

    You can have as many deployments as you want behind an endpoint. The endpoint traffic routing enables you to control which parts of the traffic to the endpoint get routed to which deployment. In this workshop, we take the blue/green approach where we'll have 2 deployments (named green and blue respectively), which will take turns playing the role of production/staging. We only have one deployment file defined though, as we automatically override the name of the deployment as part of a custom GitHub action which we'll review later in this section.

**Action Items:**
 
1. Open up the `endpoint.yml` file in your repo under _yourname-dev branch_ `src/workshop/core/scoring/endpoint.yml` 

2. Edit `endpoint.yml` file to set up the name of your endpoint. This name needs to be unique within the region you are deploying into as the endpoint name is part of the endpoint URI. Look for #setup in that file. (Add your **Deployment ID** as a suffix to your endpoint name to make it unique.), select **Commit changes...**. On the **Commit changes** page, select **Commit changes**.

3. Now navigate to `src/workshop/core/scoring/deployment.yml` and edit `deployment.yml` to set up the name of the endpoint this deployment belongs to the same name you defined just above. Look for #setup in that file. Select **Commit changes...**. On the **Commit changes** page, select **Commit changes**.

Now let's configure the GitHub Actions workflow file that controls the CD process located at `.github/workflows/workshop_cd.yml`

**Action Item:**
- Edit `workshop_cd.yml` to set up your Azure resource group name and Azure ML workspace name which are being passed as parameters to a set of custom GitHub Actions. Look for #setup and follow the instructions in the file. Select **Commit changes...**. On the **Commit changes** page, select **Commit changes**.
    

    As you've now noticed, 3 actions control the overall CD flow at the end of the workflow definition. Let's have a look into them in more detail, feel free to open their code and review how this logic has been implemented. The key considerations for each file are as follows:
    
    - `.github/actions/aml-endpoint-deploy/action.yaml`: this action does quite a few things:
      - Creates an endpoint if it doesn't exist yet using your endpoint.yml definition.
      - Checks the traffic on the endpoint, which returns a list of deployments and their respective traffic. Based on that information, the action determines which deployment name to use (green or blue). The action will deploy to whichever deployment has 0% traffic (or create one if none exists yet)
      - Deploys the latest version of the model (note that the code retrieves the latest version of the model automatically and ignores the version set in the deployment.yml file) to ensure we always release the latest registered model version.
      - The deployment has a traffic of '0%' by design as we do not want to enable it to support traffic until it's been tested.

    - `.github/actions/aml-endpoint-test/action.yaml`: This action is quite simple and does the following:
       - Finds the deployment to test by reading the endpoint, and looking for the deployment with 0% traffic
       - Tests the 0% traffic endpoint. Note that we do not have a real test it, we just check that the endpoint is 'live' but you'd be most likely checking the output against an expected response or analyzing the response with a simple Python test code. Consider looking at [Test new Deployment](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-safely-rollout-managed-endpoints#test-the-new-deployment) to understand your options when it comes to testing endpoints, either via an az ml command or by calling the endpoint URI directly but specifying a header hint to route your test request to the 0% traffic deployment.
    
    - `.github/actions/aml-endpoint-swap/action.yaml`: This action is also quite simple and consists of two main steps:
      - Read the endpoint to see which deployment is at 0% vs 100%
      - Operates a traffic update operation to swap around the traffic routing and effectively enables the latest model version to support 100% of the traffic, i.e. becoming 'production'.

**Action Items:**

1. Commit your configuration changes and push them up to Git Hub in your own development branch.

2. Go to the GitHub under 'Actions', select 'workshop_cd', select **Run workflow** drop-down, select your own development branch, and then select **Run workflow**.

3. Once triggered, click on it to open up the details and monitor its execution.

   >**Note:** Run this one more time at least to observe the entire flow and the 'swap' of deployments happening automatically with each green/blue swap alternating between supporting 0% of the traffic and 100% of the traffic as they get 'pushed to production'.

4. The last step to control CD is to set up a GitHub branch protection rule to require a successful CD run to be able to merge any code into 'main'. This important point will guarantee that the 'main' branch only accepts stable code (and therefore model as an artifact of this code) that has been successfully rolled to production and goes to 'main'. This 'main' branch is therefore always reflecting what's actually in production.

    GitHub offers an easy way to define such a policy.

    >**Note:** Wait for the pipeline to successfully run.
    
    > **Action Items:**
    > - Go to your Github repo, and click on 'Settings'
    > - Click on 'Branches' under 'Code and automation'
    > - Click on 'Add branch protection rule' under the 'Branch protection rules' to create a new rule, keep all defaults and set the following:
    >     - Branch name pattern: `main`
    >     - Require a pull request before merging: **CHECK**
    >     - Require status checks to pass before merging: **CHECK**
    >     - Status checks that are required: type in **'Workshop-Deployment'** in the search box and select it (it should auto-complete). This name is the job name defined in the workshop_cd.yml file.
    >     - Click **Create** to enable this rule on your repo.

5. You can easily test this rule by **creating a pull request to main from integration**.

   > **Action Items:**   
   > - If you have no changes in integration, first commit a simple change in your own dev branch by adding a comment to the score.py script for instance.
   > - Bring this over to integration via a Pull Request from your dev branch to integration.
   > - Go to the GitHub UI under 'Actions', select 'workshop-ci', and trigger it to run now on your own development branch.
   > - Once the CI workflow has been completed, a Pull Request from integration to main will be automatically created.
   > - Observe the status of the Pull Request to main: it should have triggered the CD run (based on the workshop_cd.yml triggers definition), and there should be a rule that prevents merging the Pull Request until the CD workflow completes succesfully.

## Success criteria

- The CD pipeline runs successfully each time a PR request to 'main' is opened. Please test this by triggering a new CI run (which on success should generate a PR to main), or creating your own PR to main.
- Each CD run updates the deployment which is currently at 0%, and then swaps it to 100% once tested properly via the CD test custom action.

## Reference materials

- [GitHub Actions](https://github.com/features/actions)
- [GitHub Actions: Workflow Triggers](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [Github Actions: Use the Azure login action with a service principal secret](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-a-service-principal-secret)
- [Azure ML CLI v2](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-train-cli)
- [Azure ML CLI v2 Examples](https://github.com/Azure/azureml-examples/tree/main/cli)
- [Azure ML Managed Endpoints](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-deploy-managed-online-endpoints)
- [Azure ML Safe Rollout of Managed Endpoints](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-safely-rollout-managed-endpoints)

## Congratulations!

This completes this workshop. We hope you've learned a lot of key concepts and are ready to take this as a template to customize for your own needs and accelerate your ML use cases.

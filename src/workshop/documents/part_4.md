# Part 4: Continuous Integration (CI)

## Pre-requisites
- Complete [Part 0], [Part 1], [Part 2], [Part 3]

## Summary
After learning about how GitHub can be leveraged for MLOps, your team decides to start by automating the model training and evaluation process with a CI pipeline. Continuous Integration (CI) is the process of developing, testing, integrating, and evaluating new features in a staging environment where they are ready for deployment and release. 

## Task 1:

1. If you are in the workshop folder, Run the following command to switch back files until you reach your repo i.e. MLOpsTemplate)

    ```bash
    cd ..
    ```
2. Locate the CI pipeline template under ```.github/workflows/workshop_ci.yml``` by running the following command:

    ```bash
    cd .github/workflows/
    nano workshop_ci.yml
    ```
    
3. Add all the needed information for the resource group name, workspace name, location and secrets for Azure and Github. They are all tagged with ```#setup```. 

    > **Action Item:** Update resource group name, workspace name, location, Azure Secret and Github Secret inside workshop_ci.yml file.
    
    > **Note:** You can use the arrow keys to move around in the file. Press the "CTRL + X" keys to close the file. You will be prompted to save your changes. Press the "y" key to save your changes and then press enter to exit.

4. Now Let's consider a common scenario in an ML development team. One of the team members is going to work on a new feature (examples can be changes to feature engineering, hyper-parameter selection, type of the model, etc.). For this work, a common pattern is to first fork and clone the repository on your local machine (which you already have done in Step 0).  Then you need to switch to the ```yourname-dev``` local branch which you created in step 3.

    > **Action Item:** Run the following command to switch to ```yourname-dev``` branch
    
    ```bash
    git checkout yourname-dev
    ```
    This takes you to yourname-dev branch, so your current working branch is set to yourname-dev. 

    > **Action Item:** Run the following command to ensure you are in the correct branch.  
    
    ```bash
    git branch
    ```
    > **Note:** Hopefully "yourname-dev" branch is colored green with a * next to it.

5. In this step we want to make some changes to our ML code, locate and open the following file: ```/src/workshop/core/training/ml_training.py```
    > - Run the following command to switch back files until you reach your repo i.e. MLOpsTemplate)
    ```bash
    cd ..
    ```
    > - Locate the CI pipeline template under ```/src/workshop/core/training/ml_training.py``` by running the following command:
    ```bash
    cd src/workshop/core/training/
    nano ml_training.py
    ```
    >**Action Item:** Update `ml_training.py`, you can search for #setup and modify `alpha` to: `model = Ridge(alpha=100)`
    
    >**Note:** You can use the arrow keys to move around in the file. Press the "CTRL + X" keys to close the file. You will be prompted to save your changes. Press the "y" key to save your changes and then press enter to exit.

    The default for the model is set to 100,000. By updating alpha we think it will improve the model performance, let's find out! Make sure to save the changes to the file. Now we want to commit these changes to the local branch and push them to our GitHub repository. This will update the remote GitHub branch on the repository.

6. Run the following commands in sequence (one by one) to stage changes, commit them and then push them to your repo. Git status shows the files that have been modified. It's a useful command to know what's the latest status of the files.

    > **Action Items:** Run the following commands sequentially:
    ```bash
    git status
    ```
    ```bash
    git add .
    ```
    ```bash
    git commit -am "a short summary of changes made- put your own comments here"
    ```
    ```bash
    git push origin yourname-dev
    ```
7. At this point you have made some changes to your code and have pushed the changes to your branch on the repository. In order for us to make these changes permanent and take them eventually to deployment and production, we need to place these changes in the "integration" branch.

    >**Action Items:**
    >- Go the your browser and go to your repository. 
    >- Click on the "pull requests" tab and Click on "New pull request". 
    >- Set the `base` branch to `integration` and the `compare` branch to `yourname-dev`.
    >- IMPORTANT NOTE: Make sure the integration branch you choose as the base is pointing to your `forked` repository and NOT the Microsoft MLOpsTemplate repository.
    >- Click on "Create pull request".
    >- Click on "Merge pull request".
    >- Click on "Confirm Merge".

    This creates a pull request to the integration branch and merges it. As a reminder, the integration branch is a branch which is as up-to-date as the main branch but we use it to evaluate the new feature. Here we made some changes to the model, and we want to make sure the new model passes the evaluation. If not, it will stop us from going to the CD process and making changes to the main branch where our production code lives.

8. The merge to the integration branch triggers the workshop_ci workflow. Click on the Actions tab on your repository and you will see the CI workflow running after a few minutes. Click and examine all the steps, note that the CI Workflow is running following the steps in the ```workshop_ci.yml``` file which you located earlier. Note that in the first few lines of this file, we have defined the workflow to be triggered when a pull request is merged in the "integration" branch.

    The CI workflow has multiple steps, including setting up the Python version, installing the libraries needed, logging in to Azure running the training model pipeline and evaluating the model. As a part of this workflow, the updated model from our current changes is compared to our best previous model and if it performs better it passes the evaluation step (more details below).

    >**Note:** If it shows you **Workflows aren't being run on this forked repository**, select **I understand my workflows, go ahead and enable them**. Select **Run workflow**, and select the **integration** branch, and select **Run workflow**.
    
    You can check out different steps of the training pipeline under: ```/src/workshop/pipelines/training_pipeline.yml```. 
    
    >**Note:** At this point, it takes about 10 minutes for the pipeline to run.
    
    If all steps pass (you can check the status under the actions in the repository), a new pull request is made to the main branch. If the workflow fails, there could be a few different reasons, you can open the workflow steps on the actions tab of the repository and examine it. Most likely if it fails in this case is due to the evaluation part, where our new model performs worse than our best previous model and doesn't pass the evaluation step and the whole workflow fails. To resolve that please read the optional reading section at the bottom of this page.

    >**Note:** By design the CI workflow will fail if the new updated model does not perform better than our best previous model and that is expected. The CI workflow prevents promoting a new model that does not pass the evaluation step. 

> **IMPORTANT NOTE:** On success on the CI workflow, a Pull Request (PR) to main is created from the integration branch. This is by design as per the definition of the CI workflow (see the last step in the workflow yml file).
>
> What you will notice will happen, is that another workflow, the CD workflow, is triggered (you could go to 'Actions' in Git Hub and see that 'workshop-cd' appeared and is running). We will cover this workflow in the next section. Its trigger is based on having a Pull Request open to the main, which is how we automate the CI -> CD chain.
>
> <u>At this point the CD workflow will fail, and this is expected</u>, because we haven't configured it yet (the yaml at this point is incorrect and pointing to incorrect Azure resources for instance).
>
> **Another important observation:** If you go to the Pull Request, you can see that you'd be allowed to merge the Pull Request to the main, even though 'workshop-cd' fails. <u>DO NOT MERGE IT</u>, instead <u>CLOSE IT</u>, but observe that it is inappropriate to have the option to close the PR.

>We definitely do not want to allow moving some code to 'main' if something in the integration branch is broken (at this point, the workflow itself is broken, but it could be anything, like the scoring script). Take note of that, as we will set up in the next section a <u>branch protection system</u> that will prevent such a merge from being possible unless the CD workflow is successful.

> OPTIONAL READING: For the evaluation and comparison of the current model with our best previous model, we have included some code in the following script: ```/src/workshop/core/evaluating/ml_evaluating.py```. Note that on line 85 of the script we are comparing the R-square of the current model with our best previous model in order to decide if we want to allow any changes to the model and main branch. You might want to edit this and relax it a little bit in order for the evaluation step to pass if you already have a really good model registered. Note that you can change the evaluation metrics based on your actual use case in the future.

## Success criteria
- Trigger CI workflow when a pull request is merged to the integration branch
- Successfully run the CI workflow which also includes the AML pipeline
- Create a Pull Request to the main branch if new code results in higher performing model

## Reference materials

- [GitHub Actions](https://github.com/features/actions)
- [GitHub Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-token)
- [GitHub Actions Workflow Triggers](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [Azure ML CLI v2](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-train-cli)
- [Azure ML CLI v2 Examples](https://github.com/Azure/azureml-examples/tree/main/cli)




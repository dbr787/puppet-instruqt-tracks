---
slug: set-up-code-manager-and-deploy-code-to-production
id: zcognslglvff
type: challenge
title: Set up Code Manager and deploy code to production
teaser: Deploy code from your feature branch to the primary server, enabling you to
  test changes on nodes in a separate Puppet environment.
notes:
- type: text
  contents: |-
    Configuring Code Manager to move code from source control to the primary server
     makes it easier for developers to safely and securely deliver their code.

    After developers' code changes are reviewed, approved, and merged, Code Manager authenticates the dedicated Code Deployer user role to automatically push the code to the primary server. Assigning a dedicated user to be a Code Deployer limits access to the primary server, which is considered a best practice.

    The code is then available to test in a separate Puppet environment, such as a branch for a specific feature, a staging environment, or a simulated production environment.

    In this lab you will:
    - Create a dedicated code deployment user that you will use to authenticate code deployment.
    - Configure Code Manager to authenticate and download your control repo from the Git server.
    - In your control repo, create a feature branch from the main branch, enabling you to develop safely without affecting production.
    - Add a module to the Puppetfile in your feature branch to test a code deployment.
    - Deploy code from your feature branch to the primary server, enabling you to test changes on nodes in a separate Puppet environment.

    Click **Start** when you're ready to begin.
tabs:
- title: Windows Agent
  type: service
  hostname: guac
  path: /#/client/c/winagent?username=instruqt&password=Passw0rd!
  port: 8080
- title: PE Console
  type: service
  hostname: puppet
  path: /
  port: 443
- title: Primary Server
  type: terminal
  hostname: puppet
- title: Git Server
  type: service
  hostname: gitea
  path: /
  port: 3000
- title: Linux Agent 1
  type: terminal
  hostname: nixagent1
- title: Practice Lab Help
  type: website
  hostname: guac
  url: https://puppet-kmo.gitbook.io/practice-lab-help/
- title: "Bug Zapper \U0001F99F⚡"
  type: website
  hostname: guac
  url: https://docs.google.com/forms/d/e/1FAIpQLScGEdG86t-YZ6nVXeC6pZbiCQ3htJZlTu4e1_V3Vr3J2u-vIw/viewform?embedded=true
difficulty: basic
timelimit: 3600
---
# Clone the control repo on your Windows development workstation
1. On the **Windows Agent** tab, from the **Start** menu, open **Visual Studio Code**.
2. Enable autosave so that you don't have to remember to save your changes. Click **File** > **Auto Save**.
3. Open the `C:\CODE` directory. Click **File** > **Open Folder**, navigate to the `C:\CODE` directory and click **Select Folder**.
    ✏️ **Note:**  If prompted, click **Accept** to trust code in this directory.

4. In VS Code, open a terminal. Click **Terminal** > **New Terminal**.
5. In the VS Code terminal window, run the following command:

        git clone git@gitea:puppet/control-repo.git
---
# Add the apache module to the Puppetfile
1. Check out the `webapp` feature branch:
    ```
    cd control-repo
    git checkout webapp
    ```
2. Navigate to the **Puppetfile** at the root of the **control-repo** directory and open it.  Add the apache module by copying the code below into the end of the Puppetfile:
    ```
    mod 'puppetlabs-apache',
      :git => 'http://gitea:3000/puppet/apache.git',
      :ref => 'main'
    ```
3. Commit and push your changes to the `webapp` feature branch:
    ```
    git add .
    git commit -m "Add Apache module dependency"
    git push
    ```

---
# Create a dedicated code deployment user
🔀 Switch to the **PE Console** tab.

1. Log in with username `admin` and password `puppetlabs`.
2. In the left navigation panel, under the **Admin** heading, click **Access Control**.
3. On the **Users** tab, enter the following information, and then click **Add local user**:
     - In the **Full name** field, enter `Code Deployer`.
     - In the **Login** field, enter `deployer`.
4. Click the **Code Deployer** user, and then click the **Generate password reset** link in the upper-right corner.
5. Copy and paste the generated link into a new browser tab. In the **NEW PASSWORD** field, enter the password `puppetlabs`, and then click **Reset password**.
6. Close the browser tab. On the **PE Console** tab, close the **Password Reset Link** window.
7. Navigate to **Access Control** > **User roles** tab, and then click on the **Code Deployers** role.
8. From the **User name** list, select `Code Deployer`, click **Add User**, and commit the changes (click **Commit** in the bottom-right corner).

---
# Configure Code Manager
1. Navigate to the **Node Groups** page, expand the **PE Infrastructure** group, and then click **PE Master**.
2. On the **Classes** tab, scroll to **Class: puppet_enterprise::profile::master**. From the **Parameter name** list, select the parameter shown, enter the relevant value, and then click **Add to node group**. Repeat for each parameter shown:

      |  Parameter                     |  Value                                                 |
      |--------------------------------|--------------------------------------------------------|
      |  code_manager_auto_configure   |  true                                                  |
      |  r10k_remote                   | "git@gitea:puppet/control-repo.git"                    |
      |  r10k_private_key              | "/etc/puppetlabs/puppetserver/ssh/id-control_repo.rsa" |

3. Commit the changes.

    🔀 Switch to the **Primary Server** tab.



4. Run `puppet agent -t` to apply the changes and configure Code Manager. Run the agent until it no longer applies corrective or intentional changes.

---
# Deploy with Code Manager
1. Test the connection to the control repository:
    ```
    puppet code deploy --dry-run
    ```

2. Observe the following output:
    ```
    ...
    Found 2 environments.
    [
      {
        "environment": "production"
      },
      {
        "environment": "webapp"
      }
    ]
    ```
    The output indicates that Code Manager was able to connect to and read all the Git branches in the control repo.

3. Check the contents of the Puppet code directory.  You can find the Puppet codedir with the command:
    ```
    puppet config print codedir
    ```
    Use the following command to view the contents of the **environment** directory:
    ```
    ls -lah /etc/puppetlabs/code/environments/
    ```

4. Note that the directory environment `webapp` branch has not been deployed.

5. Deploy the `webapp` branch manually:
    ```
    puppet code deploy webapp --wait
    ```
6. Check the contents of the Puppet code `environments/webapp` directory:
    ```
    ls -lah /etc/puppetlabs/code/environments/webapp
    ```

    🔀 Switch to the **Windows Agent** tab.

7. Compare these contents with the `control-repo > Puppetfile` contents in the control repository on the Windows server. Note that the directory environment `webapp` branch has been deployed and the `modules` subdirectory now contains the Apache module because the new version of the Puppetfile has instructions to deploy it from your Git server.

---
# Configure a webhook to deploy code automatically
  🔀 Switch to the **Primary Server** tab
1. Use the command below to generate and show a login token for the code deployment user:
    ```
    puppet access login deployer --lifetime 180d
    ```
2. Use the command below to show the access token, and save it to your notes:
    ```
    more /root/.puppetlabs/token
    ```
   🔀 Switch to the **Git Server** tab
3. Login with username `puppet` and password `puppetlabs`.  Once you are logged in, look for the **Repositories** sidebar on the right.  Click the link to go to the **puppetlabs/control-repo** repository.
4. Click the **Settings** link on the upper right
5. Click to open the **Webhooks** tab
6. Click on the **Add Webhook** button on the upper right, and select **Gitea** in the dropdown menu
7. Paste the following link into the **Target URL** field:
    ```
    https://puppet:8170/code-manager/v1/webhook?type=github&token=
    ```
8. Paste the Puppet admin token that you copied earlier at the end of the link and click the **Add Webhook** button at the bottom of the screen
9. You will be taken to a list of webhooks; click on the URL for the webhook that you just added
10. Click on **Test Delivery** at the bottom of the page.  After a few seconds, you should see an entry in the list with a green checkmark next to it.

🎈 **Congratulations!** In this lab you created a dedicated code deployment user that you used to authenticate to deploy code. You then configured Code Manager to authenticate and download your control repo from the Git server. Next, you created a feature branch in your control repo, which allowed you to develop safely without affecting production. Then, to test a code deployment, you added a module to the Puppetfile on your feature branch Finally, you deployed code from your feature branch to the primary server, enabling you to test changes on nodes in a separate Puppet environment.

To continue, click **Next**.

---
**Find any bugs or have feedback? Click the **Bug Zapper** tab near the top of the page and let us know!**



## Module 3 - Part 1: Configuring EDA

### 3.0 Exercise: Creating GitHub Personal Access Token

Before configuring credentials in AAP, you need to create a Personal Access Token (PAT) for GitHub authentication.

1.  **Generate GitHub Personal Access Token:**
    * Log in to your GitHub account
    * Navigate to **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
    * Click **Generate new token** → **Generate new token (classic)**
    * Configure the token:
        * **Note**: `AAP EDA Workshop Token`
        * **Expiration**: `7 days` (or as per your security policy)
        * **Select scopes**: Check the following permissions:
            - `repo` (Full control of private repositories)
            - `workflow` (Update GitHub Action workflows)
    * Click **Generate token**
    * **IMPORTANT**: Copy the token immediately and store it securely - you won't be able to see it again!

2.  **Record Your Credentials:**
    * **GitHub Username**: Your GitHub username (e.g., `your-username`)
    * **GitHub Token**: The personal access token you just generated (starts with `ghp_...`)

**Security Note**: Personal access tokens provide the same level of access as passwords but are more secure because they can be limited in scope and easily revoked.

### 3.1 Exercise: Configuring Credentials in EDA

1.  **In the EDA UI (within AAP), go to Automation Decisions -> Infrastructure -> Credentials, and create the following:**
    * **Image Registry Credential:**
        * Name: `Red Hat Registry Credential`
        * Type: `Container Registry`
        * URL: `registry.redhat.io` #from which we will pull the DE image
        * Usename: No need, public registry
        * Token: No need, public registry
    * **AAP Controller Credential:**
        * Name: `AAP Controller Credential`
        * Type: `Red Hat Ansible Automation Platform`
        * Host: `https://<URL of the AAP controller route>/api/controller/`
        * User: `admin`.
        * Get Password: `oc get secret <instance-name>-controller-admin-password -o jsonpath='{.data.password}' | base64 --decode`
    * **Git Repo Credential:**
        * Name: `GitHub Credential`
        * Type: `Source Control`
        * Username: Your GitHub username (from section 3.0)
        * Password/Token: Your GitHub Personal Access Token (from section 3.0)
    
### 3.2 Exercise: Creating the Project in EDA

1.  Go to `Automation Decisions` -> `Projects` -> `Create project`.
2.  Name: `OpenShift Alerting Project`.
3.  Point to your forked Git repository URL and select the `GitHub Credential` you created in section 3.1.
4.  Source Control branch: `main`.
5.  Unmark `Validae SSL`.
6.  Click `Save` and wait for the project to sync.

### 3.3 Exercise: Referencing the Decision Environment

Before creating the Rulebook Activation, you need to configure the Decision Environment that will execute your Event-Driven Ansible rulebooks.

1. Go to `Automation Decisions` -> `Decision Environment` -> `Create new`.
2. **Name**: `EDA Decision Environment`
3. **Image**: `registry.redhat.io/ansible-automation-platform-25/de-supported-rhel9:latest`
4. **Description**: `Red Hat supported Decision Environment for Event-Driven Ansible`
5. **Credential**: Leave Empty # (registry.redhat.io is publicly available. In your prod network, you'll reference your registry creds)
6. Click `Create`.

**Important**: This decision environment will be used in the next section when creating your Rulebook Activation.

### 3.4 Exercise: Creating the Rulebook Activation

1.  Go to `Automation Decisions` -> `Rulebook Activations` -> `Create rulebook activation`.
2.  Name: `alertmanager-listener`
3.  Project: `OpenShift Alerting Project`.
4.  Rulebook: `extensions/eda/rulebooks/rulebook.yml`.
5.  Decision Environment: Select `EDA Decision Environment` (the Red Hat pre-built image configured in the previous section).
6.  Log Level: `Debug`.
7.  Service Name: `alertmanager-listener`. #Same as the one we configured in the Route.
8.  Credential: `AAP Controller Credential`.
9.  Rulebook activation enabled? `Marked V`.
10.  Click `Create rulebook activation`.
11.  **Apply the Route:**
    * Now that the activation has created the `alertmanager-listener` service, apply the route manifest you created earlier.
        ```bash
        # Make sure you are in your local workshop repo directory
        oc apply -f extensions/eda/k8s-objects/eda-route.yml
        ```
12.  **Get the Webhook URL:**
    * Find the exposed URL for the webhook. This is the endpoint you'll give to Alertmanager.
        ```bash
        ROUTE=`oc get route alertmanager-listener -n <aap-namespace> -o jsonpath='{.spec.host}'`
        echo https://$ROUTE
        ```
    
      * Your full webhook URL will be `https://<route-hostname-from-above>/alerts`. Note that we use **https** now because of the edge termination.
13.  **Testing**
    * Run the following `CURL` command to check for the validity of the rulebook.
      ```bash
      curl -X POST https://$ROUTE/alerts -H 'Content-Type: application/json' -d '{"alerts":[{"annotations":{"description":"Workshop!"},
         "labels":{"alertname":"test",
                   "env":"dev",
                   "severity":"custom"},
         "status":"firing"}]}'
      ```
     * See in the rulebook activation history the data of your CURL command.

## Module 3 - Part 2: Configuring AAP

### 3.5 Exercise: Configuring Credentials in AAP
**Git Repo Credential:**
  * Name: `GitLab/GitHub Credential`
  * Type: `Source Control`
  * Username: Your GitHub username (from section 3.0)
  * Password/Token: Your GitHub Personal Access Token (from section 3.0)

**OpenShift Cluster Credential:**
1. Create a new Credential Type for OpenShift (`openshift_api_url`, `openshift_token`):
   * Navigate to `Automation Execution` -> `Infrastructure` -> `Credential Types` -> `Create New`
   * Name: `OCP Login`

   * Input configuration:
      ```yaml
      fields:
      - id: dev_ocp_api
        type: string
        label: DEV OCP API Endpoint
      - id: dev_ocp_token
        type: string
        label: DEV OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace
      - id: prd_ocp_api
        type: string
        label: PRD OCP API Endpoint
      - id: prd_ocp_token
        type: string
        label: PRD OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace
      ```

    * Injector configuration:
      ```yaml
      env:
        DEV_OCP_API: '{{ dev_ocp_api }}'
        DEV_OCP_AUTH_TOKEN: '{{ dev_ocp_token }}'
        PRD_OCP_API: '{{ prd_ocp_api }}'
        PRD_OCP_AUTH_TOKEN: '{{ prd_ocp_token }}'
      ```

    2. Create a new credential of this type for your cluster(s):
    * Navigate to `Authentication Execution` -> `Infrastructure` -> `Credential` -> `Create New`
    * Name: `OCP Login Creds`
    * Credential: `OCP Login`
    * Type Details:
      * DEV OCP API Endpoint: `<https://dev_URL:6443>`
      * DEV OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace: `<PRINT SAVED TOKEN FROM Module 2 HERE>`
      > `echo $TOKEN`
      * PRD OCP API Endpoint: `<https://prd_URL:6443>`
      * PRD OCP Token of ServiceAccount aap-eda-sa in openshift-monitoring namespace: `<PRINT SAVED TOKEN FROM Module 2 HERE>`


### 3.6 Exercise: Creating the Project in AAP

1.  Go to `Authentication Execution` -> `Projects` -> `Create project`.
2.  Name: `OpenShift Automations`.
3.  Point to your forked Git repository URL and select your Git credentials.
4.  Branch: `main`
5.  Unmark `Validae SSL`.
6.  Click `Save` and wait for the project to sync.

### 3.7 Exercise: Creating a project template in AAP

1.  Go to `Authentication Execution` -> `Templates` -> `Create New`
2.  Name: `reboot-problematic-node` # Same as referenced in the `run_job_template` in the rulebook
3.  Inventory: `Demo Inventory` # Irrelevant because we are using `oc login` + `ocp api` as access method to our remote cluster based on the `env` from the alert
4.  Credentials: `OCP Login Creds` # from 3.5. step 2
5.  Project: `Openshift Automations`
6.  Job Type: `Run`
7.  Playbook: `playbooks/drain_and_reboot_node.yaml`
8.  `Extra variables` -> `Prompt on launch` -> `Mark V` # A MUST !!! so the `vars_prompt` section in the playbook will work
9.  Click `Save`

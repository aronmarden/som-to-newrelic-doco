# Walkthrough: Sending Salesforce Order Management Data to New Relic

This guide will walk you through the entire process of sending data from your Salesforce Order Management (SOM) instance into New Relic. We will use the official [newrelic-salesforce-exporter](https://github.com/newrelic/newrelic-salesforce-exporter) tool, which runs inside a Docker container.

The process is straightforward and we are going to accomplish it in three main steps:

1.  **Create a Salesforce 'Connected App'**: This will give our exporter the secure credentials it needs to access your Salesforce data.
2.  **Create a Configuration File**: This file will tell our exporter exactly what data to get from Salesforce and where to send it in New Relic.
3.  **Run the Exporter**: We will use a single Docker command to start the exporter and begin sending data.

By the end of this guide, your Salesforce Order Management data will be flowing into New Relic, ready for you to query, dashboard, and alert on.

---

## Step 1: Create a Salesforce 'Connected App' for API Access

**Goal:** To get a secure **Consumer Key** and **Consumer Secret** from Salesforce.

**Why:** The exporter application needs to log in to Salesforce to retrieve data, just like a user would. Instead of using a person's username and password, applications use a "Connected App" which provides a unique `client_id` (Consumer Key) and `client_secret` (Consumer Secret) for secure, trackable API access.

### Instructions:

1.  **Navigate to App Manager:**
    * In your Salesforce instance, go to **Setup**.
    * Use the Quick Find box to search for and select **App Manager**.
    * Click the **New Connected App** button in the top right.

2.  **Fill in Basic Information:**
    * **Connected App Name:** Give it a clear name, like `New Relic SOM Exporter`.
    * **API Name:** This will auto-fill. You can leave it as is.
    * **Contact Email:** Enter your email address.

3.  **Enable API (OAuth) Settings:**
    * Check the box for **Enable OAuth Settings**. This is the section that generates our API credentials.
    * **Callback URL:** Enter a placeholder. `https://localhost/oauth/callback` is a safe and common choice.
    * **Selected OAuth Scopes:** This defines what the app is allowed to do. Select the following two scopes and add them to the **Selected OAuth Scopes** list:
        * `Access and manage your data (api)`
        * `Perform requests on your behalf at any time (refresh_token, offline_access)`
    * Click **Save**. It may take a few minutes for Salesforce to apply the changes.

4.  **Retrieve Your Credentials:**
    * After saving, you will be on the management page for your new app. Click **Manage Consumer Details**.
    * You may need to verify your identity with a code sent to your email.
    * You will now see your **Consumer Key** and **Consumer Secret**.
        * **Consumer Key** = `client_id`
        * **Consumer Secret** = `client_secret`
    * **Action:** Copy both of these values and save them in a secure place. We will need them in the next step.

---

## Step 2: Create the `config.yml` File to Instruct the Exporter

**Goal:** To create a configuration file named `config.yml`.

**Why:** This file acts as the instruction manual for our exporter. It tells the exporter:
* **How to connect to Salesforce**: Using the `client_id` and `client_secret` you just got from Step 1.
* **What data to get**: We will define specific queries to pull data from your Salesforce Order Management objects.
* **Where to send it**: We will provide your New Relic Account ID and an Ingest License Key.

### Instructions:

1.  Create a file named `config.yml` in a new, empty folder on your computer.
2.  Copy the entire block of code below and paste it into your `config.yml` file.
3.  Carefully replace all the placeholder values (e.g., `YOUR_...`) with your actual information.

```yaml
integration_name: com.newrelic.labs.sfdc.som.exporter
run_as_service: false
instances:
  - name: sfdc-som-instance # A descriptive name for this Salesforce connection
    arguments:
      api_ver: "60.0" # Use a recent, supported Salesforce API version
      token_url: "https://YOUR_[DOMAIN.my.salesforce.com/services/oauth2/token](https://DOMAIN.my.salesforce.com/services/oauth2/token)" # Replace YOUR_DOMAIN with your Salesforce domain
      auth:
        grant_type: password
        client_id: "YOUR_CONSUMER_KEY_FROM_CONNECTED_APP"      # Paste the Consumer Key from Step 1 here
        client_secret: "YOUR_CONSUMER_SECRET_FROM_CONNECTED_APP"  # Paste the Consumer Secret from Step 1 here
        username: "YOUR_SALESFORCE_API_USERNAME"
        password: "YOUR_SALESFORCE_API_USER_PASSWORD_AND_TOKEN" # Password followed by security token
      date_field: CreatedDate
      generation_interval: Hourly # Or Daily, depending on your needs
      time_lag_minutes: 0
      logs_enabled: no # Set to 'no' to only use the specific queries below
    labels:
      environment: sfdc-nonprod # Or 'sfdc-prod'

# Define the specific SOQL queries to run against SOM objects
queries:
# - query: "SELECT Id, EventType, CreatedDate, LogDate, LogFile, Interval FROM EventLogFile" # General purpose event log files
- query: "SELECT Id, OrderNumber, Status, TotalAmount FROM OrderSummary" # Example for Order Summary
- query: "SELECT Id, Status, FulfillmentType FROM FulfillmentOrder" # Example for Fulfillment Orders
- query: "SELECT Category, Message, OccurredDate FROM ProcessException WHERE AttachedToId != null" # Example for Process Exceptions on orders


# --- New Relic Ingest Configuration ---
newrelic:
  data_format: events # Use 'events' for structured data, or 'logs'
  api_endpoint: US # Or EU
  account_id: "YOUR_NEW_RELIC_ACCOUNT_ID"
  license_key: "YOUR_NEW_RELIC_INGEST_LICENSE_KEY" # An Ingest License Key (starts with 'NRII...')
```

**What you are configuring:**
* `client_id` & `client_secret`: These are the credentials you saved from Step 1.
* `username` & `password`: The credentials of a Salesforce user who has permission to view the SOM objects you are querying. The password must have the user's Salesforce Security Token appended to the end.
* `queries`: These are the exact queries the exporter will run. The examples provided target common SOM objects, but you can customize them for your needs.
* `account_id` & `license_key`: This tells the exporter which New Relic account to send the data to. Use a New Relic **Ingest - License** key.

---

## Step 3: Run the Exporter with Docker

**Goal:** To start the exporter.

**Why:** This is the final step that brings everything together. We will run a command that starts the Docker container. The container will automatically find and read your `config.yml` file from Step 2, connect to Salesforce using the credentials from Step 1, and start sending data to New Relic.

### Instructions:

1.  Open a terminal or command prompt.
2.  Navigate into the directory where you saved your `config.yml` file.
3.  Run the single command below, replacing the one placeholder value for the license key.

```bash
docker run -t --rm --name salesforce-exporter \
   -v "$PWD/config.yml":/usr/src/app/config.yml \
   -e NEW_RELIC_APP_NAME="New Relic Salesforce SOM Exporter" \
   -e NEW_RELIC_LICENSE_KEY="YOUR_NEW_RELIC_USER_LICENSE_KEY" \
   newrelic/newrelic-salesforce-exporter
```

**What this command does:**
* `docker run ...`: Starts the exporter application inside a Docker container.
* `-v "$PWD/config.yml":...`: This is a critical part. It mounts your local `config.yml` file into the container, so the application can read it.
* `-e NEW_RELIC...`: These environment variables are for monitoring the exporter application itself in New Relic APM. Use a **User** key here, which is different from the Ingest key you used in the config file.

---

## Verifying Success

After running the command, you will see log output in your terminal. Look for messages indicating a successful connection to Salesforce and data being sent to New Relic. If there are any errors (e.g., bad password, incorrect permissions), they will be displayed here.

After a successful run, your Salesforce Order Management data will appear in your New Relic account. You can find it by going to **Query your data** and looking for event types with the names of the objects you queried, such as `OrderSummary`, `FulfillmentOrder`, and `ProcessException`.

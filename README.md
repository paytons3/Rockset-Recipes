# Rockset Recipe: Send an Automated Email Report

This recipe provides a script to create an automated email report for product listings stored in Rockset. 

We will walk through how to create a Steam Product Listings collection from a public S3 bucket. We will then create an automated email report using a Scheduled Query Lambda with the number of products listed for more than $1.99 in the past 7 years relative to the current date.

You can copy the full Pythons script without step-by-step descriptions via the attached `python-script` file.

Link to Recipe on Rockset Website: https://docs.rockset.com/documentation/recipes/send-automated-email-reports

## **Requirements**
Python >= 3.6
pip install rockset time json os

*_The things you have to change to successfully run this script are **rocksetApiKey**, **apiServerHost**, **webhookAuthorization**, **senderEmail**, **senderName**, **receivingEmail**, and **receivingName**_.

## Step 1: Initialize the Client

Before we can start, we'll need to initialize the Rockset client. Create an API key in the API Keys tab of the Rockset Console. The region can be found in the dropdown menu at the top of the page.

```
# Set the API Key as an environmental variable in the shell via:
# export ROCKET_API_KEY='<insert key here>'
rocksetApiKey = os.getenv("ROCKSET_API_KEY")
apiServerHost = Regions.usw2a1 #ex: Regions.usw2a1

rs = RocksetClient(host=apiServerHost, api_key=rocksetApiKey)
```

## Step 2: Define Parameters for Scheduled Query Lambda

Here we will define the parameters that will be included in our [Scheduled Query Lambda](https://docs.rockset.com/documentation/docs/query-lambdas#scheduled-query-lambdas). We will be using SendGrid to create this email automation, so be sure to [set up a SendGrid account and API key](https://docs.sendgrid.com/for-developers/sending-email/api-getting-started).

```
queryLambdaName = "steamProductListingsReport"
webhookAuthorization = "Bearer YOUR_SENDGRID_API_KEY"
webhookURL = "https://api.sendgrid.com/v3/mail/send"
senderEmail = "SENDER_EMAIL"
senderName = "SENDER_NAME"
receivingEmail = "RECEIVING_EMAIL"
receivingName = "RECEIVING_NAME"
pastDays = "2555" # Number of past days (7 years) since current date to get product listings from.
gamePrice = "1.99"
subjectLine = "Automated e-mail - Total Products Listed"
emailContent = "Total products from last 7 years to date listed with price greater than "+gamePrice+": {{QUERY_RESULTS}}" # Content of automated email. Uses special character sequence {{QUERY RESULTS}} which returns the result of the query.
cronSchedule = "0 16 * * *" # UNIX cron format for every day at 16 UTC.
numberOfExecutions = 1 # Can change to `None` for unlimited number of executions.
```

## Step 3: Create Workspace

Next, we'll create a `steam_data` [Workspace](https://docs.rockset.com/documentation/docs/workspaces). It is good practice to separate different use cases into different workspaces.

```
def create_workspace(rs, workspaceName):
    try:
        print(f"Creating workspace `{workspaceName}`...")
        api_response = rs.Workspaces.create(
            name=workspaceName,
        )
        print(f"Workspace `{workspaceName}` created!")
    except ApiException as e:
        print("Exception when creating workspace: %s\n" % json.loads(e.body))
```

```
create_workspace(rs, workspaceName)
```

## Step 4: Define an Ingest Transformation

[Ingest Transformations](https://docs.rockset.com/documentation/docs/ingest-transformation) are SQL queries applied to data _before_ it is stored in Rockset. 

```
ingestTransformation = """
    SELECT
        CAST(release_date as DATE) AS release_date, 
        CAST(price as FLOAT) AS price, 
        _input.*
    EXCEPT
        (_meta, 
        release_date, 
        price)
    FROM
        _input
    """
```

## Step 5: Create a Collection

Next, we'll create a `steam_product_listings` [Collection](https://docs.rockset.com/documentation/docs/collections) from a public S3 bucket, and we'll pass our ingest transformation from the previous step into the `field_mapping_query` parameter.

```
def create_collection(rs, workspaceName, collectionName, ingestTransformation):
    
    try:
        print(f"Creating collection `{collectionName}`...")
        api_response = rs.Collections.create_s3_collection(
            field_mapping_query=FieldMappingQuery(
                sql=ingestTransformation,
            ),
            name=collectionName,
            workspace=workspaceName,
            sources=[
                S3SourceWrapper(
                    bucket="s3://rockset-community-datasets",
                    prefix="public/steam-games-reviews/steam_games.json",
                    region="us-west-2"
                ),
            ]
        )
        print(f"Collection `{collectionName}` created!")
    except ApiException as e:
        print("Exception when creating collection: %s\n" % json.loads(e.body))
```

```
create_collection(rs, workspaceName, collectionName, ingestTransformation)
```

## Step 6: Wait for Collection Ready

We will now wait for the collection to be in `READY` state. It will only take ~3 minutes to bulk ingest all the steam product listing data from the public S3 bucket.

```
def wait_for_collection_ready(rs, workspaceName, collectionName, max_attempts=30):
    print(f"Waiting for the `{collectionName}` collection to be `Ready` (~5 minutes)...")
    for attempt in range(max_attempts):
        api_response = rs.Collections.get(collection=collectionName, workspace=workspaceName)

        if api_response.data.status == "READY":
            print(f"Collection `{collectionName}` ready!")
            break
        else:
            time.sleep(60)
            
    print("Collection still not ready. Check collection status in console.")
```

```
wait_for_collection_ready(rs, workspaceName, collectionName)
```

## Step 7: Define Report Query

Here we will define a query that returns the results we want to include our report. We will aggregate the total number of product listings that were listed in the past 7 years from the current date with a listing price greater than $1.99.

```
report_query = f"""
    SELECT
        COUNT(*) AS number_products_listed
    FROM
        {workspaceName}.{collectionName}
    WHERE
        release_date >= CURRENT_DATE() - DAYS(:past_days)
        AND price > :game_price;
    """
```

## Step 8: Create Query Lambda

[Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas) (QLs) are parameterized SQL queries that can be executed from a dedicated REST endpoint. Here, we'll save our report data query from the previous step into a QL named `steamProductListingsReport`. 

```
def create_query_lambda(rs, workspaceName, collectionName, queryLambdaName, sqlQuery):
    description = "get number of product listings greater than a certain price created at or after the specified date"

    try:
        print(f"Creating query lambda `{queryLambdaName}`...")
        api_response = rs.QueryLambdas.create_query_lambda(
            name=queryLambdaName,
            workspace=workspaceName,
            sql=QueryLambdaSql(
                default_parameters=[
                    QueryParameter(
                        name="past_days",
                        type="int",
                        value=pastDays
                    ), 
                    QueryParameter(
                        name="game_price",
                        type="float",
                        value=gamePrice
                    )
                ],
                query=sqlQuery,
            ),
        )
        print(f"Query lambda `{queryLambdaName}` created!")
    except ApiException as e:
        print(f"Exception when creating query lambda: %s\n" % json.loads(e.body))
```

```
create_query_lambda(rs, workspaceName, collectionName, queryLambdaName)
```

## Step 9: Define Webhook Payload
We will now define a webhook payload for our SendGrid webhook configuration. This payload will include the necessary information for our automated email. 

```
payload_content = """
        {\"personalizations\": 
        [ {      
        \"to\": [{          
            \"email\": \""""+receivingEmail+"""\",         
            \"name\": \""""+receivingName+"""\"       
        }],     
        \"subject\": \""""+subjectLine+"""\"    
        }],
        \"content\": [{     
            \"type\": \"text/plain\",     
             \"value\": \""""+emailContent+"""\"
        }], 
        \"from\": {   
            \"email\": \""""+senderEmail+"""\",   
            \"name\": \""""+senderName+"""\" 
        }, 
        \"reply_to\": {    
            \"email\": \""""+senderEmail+"""\",    
            \"name\": \""""+senderName+"""\"
        }}
        """
```

## Step 10: Create Scheduled Query Lambda

You can schedule your QLs for automatic execution and configure certain actions to be taken based on the results of the executed query lambda using [Scheduled Query Lambdas](https://docs.rockset.com/documentation/docs/query-lambdas#scheduled-query-lambdas). We will now create a Scheduled QL to send our automated email report.

```
def created_scheduled_ql(rs, workspaceName, queryLambdaName, cronSchedule, webhookURL, webhookAuthorization, payload, numberOfExecutions):

    # Webhook payload for the SendGrid email.

    try:
        print(f"Creating scheduled query lambda for `{queryLambdaName}`...")
        api_response=rs.ScheduledLambdas.create(
            apikey=rocksetApiKey,
            cron_string=cronSchedule,
            ql_name=queryLambdaName,
            tag="latest",
            total_times_to_execute=numberOfExecutions,
            webhook_auth_header=webhookAuthorization,
            webhook_payload=payload,
            webhook_url=webhookURL,
            workspace=workspaceName
        )
        print(f"Scheduled query lambda for `{queryLambdaName}` created!")
    except ApiException as e:
        print(f"Exception when creating scheduled query lambda: %s\n" % json.loads(e.body))
      
```

```
created_scheduled_ql(rs, workspaceName, queryLambdaName, cronSchedule, webhookURL, webhookAuthorization, numberOfExecutions)
```

## Step 12: Clean Up

That's all! As a courtesy to you, I've included a function to delete the workspace, collection, and query lambda that was created below at your leisure.

```
def clean_up_demo(rs, workspaceName, collectionName, queryLambdaName, scheduledQLrrn, max_attempts=10):

  	# Deleting Schedule
    try:
      print(f"Deleting schedule `{scheduledQLrrn}`...")
            api_response = rs.ScheduledLambdas.delete(
              scheduled_lambda_id=scheduledQLrrn,
            )
        # Checking if Schedule is deleted
        for attempt in range(max_attempts):
          try:
            api_response = rs.ScheduledLambdas.get(
              scheduled_lambda_id=scheduledQLrrn
            )
         	except ApiException as e:
             if json.loads(e.body).get("message_key") == "SCHEDULED_LAMBDA_NOT_FOUND": 
               	print(f"Schedule `{scheduledQLrrn}` deleted.")
                break
          time.sleep(30)
      except ApiException as e:
            print(f"Exception when deleting schedule: %s\n" % json.loads(e.body))
  
    # Deleting Query Lambda
    try:
        print(f"Deleting Query Lambda `{queryLambdaName}`...")
        api_response = rs.QueryLambdas.delete_query_lambda(
            query_lambda=queryLambdaName, 
            workspace=workspaceName
        )
        # Checking if Query Lambda is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.QueryLambdas.get_query_lambda_tag_version(
                    query_lambda=queryLambdaName, 
                    workspace=workspaceName, 
                    tag="latest"
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "QUERY_NOT_FOUND": 
                    print(f"Query Lambda `{queryLambdaName}` deleted.")
                    break
            time.sleep(30)   
    except ApiException as e:
        print(f"Exception when deleting Query Lambda: %s\n" % json.loads(e.body))

    # Deleting Collection
    try:
        print(f"Deleting Collection `{collectionName}`...")
        api_response = rs.Collections.delete(
            collection=collectionName, 
            workspace=workspaceName
        )
        # Checking if Collection is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.Collections.get(
                    collection=collectionName, 
                    workspace=workspaceName
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "COLLECTION_DOES_NOT_EXIST": 
                    print(f"Collection `{collectionName}` deleted.")
                    break
            time.sleep(30)
    except ApiException as e:
        print(f"Exception when deleting Collection: %s\n" % json.loads(e.body))

    # Deleting Workspace
    try:
        print(f"Deleting workspace `{workspaceName}`...")
        api_response = rs.Workspaces.delete(
            workspace=workspaceName
        )
        # Checking if Workspace is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.Workspaces.get(
                    workspace=workspaceName
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "WORKSPACE_NOT_FOUND": 
                    print(f"Workspaces `{workspaceName}` deleted.")
                    break
            time.sleep(30)
    except ApiException as e:
        print(f"Exception when deleting workspace: %s\n" % json.loads(e.body))
```

```
scheduledQLrrn = get_scheduled_ql_rrn(rs, queryLambdaName)
clean_up_demo(rs, workspaceName, collectionName, queryLambdaName, scheduledQLrrn)
```

## What's Next?

Want to learn more about what you can automate using Scheduled Query Lambdas? Check out the following:
- Blog: [5 Tasks You Can Automate in Rockset Using Scheduled Query Lambdas](https://rockset.com/blog/5-tasks-you-can-automate-in-rockset-using-scheduled-query-lambdas/)
- Workshop: [Automate Your Workflow with Rockset’s Scheduled Query Lambdas](https://www.youtube.com/watch?v=CuQYCE0Bbsc)
- Recipe: [Auto-Resume/Suspend VI on Custom Schedule](https://docs.rockset.com/documentation/recipes/auto-resumesuspend-vi-on-custom-schedule)


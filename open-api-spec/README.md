# Create MongoDB api setup

## MongoDB Setup
If you are new to MongoDb Atlas, you can subscribe to it from [Google Cloud Marketplace](https://www.mongodb.com/products/platform/atlas-cloud-providers/google-cloud?utm_source=google&utm_campaign=search_gs_pl_evergreen_atlas_general_prosp-brand_gic-null_apac-in_ps-all_desktop_eng_lead&utm_term=mongodb%20on%20google&utm_medium=cpc_paid_search&utm_ad=p&utm_ad_campaign_id=6501677905&adgroup=84316982521&cq_cmp=6501677905&gad_source=1&gclid=Cj0KCQjwmt24BhDPARIsAJFYKk26Mjhe4PffhYvYm9yTDgiAoGNp9MiKzEQG9wgp0LLzTC0qb0ilblMaAvDwEALw_wcB). 
1. Configure a MongoDB cluster on GCP. For instructions, refer to [How to Set Up a MongoDB Cluster](https://www.mongodb.com/resources/products/fundamentals/mongodb-cluster-setup).
2. Follow the [instructions](https://www.mongodb.com/docs/guides/atlas/sample-data/) to Load sample dataset.
   * Log in to MongoDB Atlas console, On cluster navigation page and naviage to Databases. click on 3 dots (...) and view all clusters. Click on 3 dots in front of your cluster name and Click on Load sample documents.

3. To obtain MongoDB cluster connection string from the Connect UI on the MongoDB Atlas console navigate to your Atlas Home screen and click on Connect for the AWS cluster you want to connect, Select the Private Endpoint, and Connection Method.
Copy the SRV connection string. We use this SRV connection string in the subsequent steps.


## Google Cloud Function setup
1. Open [Google Cloud function](https://console.cloud.google.com/functions) and click on **create function**.
2. Provide a **function Name** and **select a region**. Allow Unauthenticated access leave the other options to default and click on next.
3. Select Python as a **Runtime** (Select the latest version available).
4. Copy the code from file '_app.py_' and paste it to '_main.py_' file on cloud function console.
5. Update the </MongoDB Connection String/> with the connection string MongoDB MongoDB Atlas.
6. Rename the **Entry Point** as "mongodb_crud"
7. copy the requirements.txt from this folder to requirements.txt on Google Cloud function.
8. Deploy the Function. 
9. Copy and store locally the Https Endpoint(URL) for triggering the Cloud Function.
10. Navigate to the details page of cloud function and copy and store the service account name used by the function.

## Load the OpenAPI specs for Veretx AI Extensions.
1. Update the file 'mdb_extension_openapi_specs.yaml' with the Google Cloud Functions trigger.
2. Upload the file to GCS bucket.


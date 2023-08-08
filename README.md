# TechConf Registration Website

## Project Overview
The TechConf website allows attendees to register for an upcoming conference. Administrators can also view the list of attendees and notify all attendees via a personalized email message.

The application is currently working but the following pain points have triggered the need for migration to Azure:
 - The web application is not scalable to handle user load at peak
 - When the admin sends out notifications, it's currently taking a long time because it's looping through all attendees, resulting in some HTTP timeout exceptions
 - The current architecture is not cost-effective 

In this project, you are tasked to do the following:
- Migrate and deploy the pre-existing web app to an Azure App Service
- Migrate a PostgreSQL database backup to an Azure Postgres database instance
- Refactor the notification logic to an Azure Function via a service bus queue message

## Dependencies

You will need to install the following locally:
- [Postgres](https://www.postgresql.org/download/)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Azure Function tools V3](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack)

## Project Instructions

### Part 1: Create Azure Resources and Deploy Web App
1. Create a Resource group
2. Create an Azure Postgres Database single server
   - Add a new database `techconfdb`
   - Allow all IPs to connect to database server
   - Restore the database with the backup located in the data folder
3. Create a Service Bus resource with a `notificationqueue` that will be used to communicate between the web and the function
   - Open the web folder and update the following in the `config.py` file
      - `POSTGRES_URL`
      - `POSTGRES_USER`
      - `POSTGRES_PW`
      - `POSTGRES_DB`
      - `SERVICE_BUS_CONNECTION_STRING`
4. Create App Service plan
5. Create a storage account
6. Deploy the web app

### Part 2: Create and Publish Azure Function
1. Create an Azure Function in the `function` folder that is triggered by the service bus queue created in Part 1.

      **Note**: Skeleton code has been provided in the **README** file located in the `function` folder. You will need to copy/paste this code into the `__init.py__` file in the `function` folder.
      - The Azure Function should do the following:
         - Process the message which is the `notification_id`
         - Query the database using `psycopg2` library for the given notification to retrieve the subject and message
         - Query the database to retrieve a list of attendees (**email** and **first name**)
         - Loop through each attendee and send a personalized subject message
         - After the notification, update the notification status with the total number of attendees notified
2. Publish the Azure Function

### Part 3: Refactor `routes.py`
1. Refactor the post logic in `web/app/routes.py -> notification()` using servicebus `queue_client`:
   - The notification method on POST should save the notification object and queue the notification id for the function to pick it up
2. Re-deploy the web app to publish changes

## Monthly Cost Analysis
Complete a month cost analysis of each Azure resource to give an estimate total cost using the table below:

| Azure Resource           | Service Tier                                                 | Monthly Cost |
| ------------             | ------------                                                 | ------------ |
| *Azure Postgres Database*| Single Server - Basic - 1 vCore - 5 GB                       | 25.32        |
| *Azure Service Bus*      | Basic - Max Msg Size 256kb - 1M Ops                          | 0.05         |
| *Azure Function App*     | Basic - 1vCpu - 1.75Gb Memory - 10Gb Storage - 3 Instances   | 12.41        |
| *Azure Web App*          | Free                                                         | 0            |
| *Total*                  |                                                              | 37.78        |

## Architecture Explanation
The main purpose of this app is to create and manage notifications and trigger sending email process to attendee. Due to these reasons, the app architecture should be split into two as follow:
- Web client as App Service: Since the purpose of the client is mostly create new notification and query notifications and attendees from database, the web traffic reqired is not too high. Therefore, Free Tier App Service is enough for these processes. CRUD action to attendees and notifications table are the most used functions in the client, so the budget should focus more on database and background process.
- Postgres DB: Because of the logic is much focus on query data, single server database should be enough. I choose the minimun specs in Basic plans because it is more budget friendly, and the data scaling of this project in the future is not so big that HyperScale or Multi-server database server is needed
- Azure Function App: Based on the logic of the app, the background process requires more resources than the web client. The function app and Service Bus plan should base on how much emails and attendees are estimated. Due to the current size of the database, Basic plans for both of these services should be more than enough. In addition, these services are also very easy to scale up if more resources are required without having to scale the app service. But for now, Basic plans is most optimized and budget-friendly.

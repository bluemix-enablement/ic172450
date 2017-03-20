<img src="images/header.png" width=100% height=auto>


<div class="title">

<H1>Hands-on Lab<br
<H1>Session 2450<br
<H1>Working with OpenWhisk in IBM Bluemix</H1>

</div>

<div class="author">
<H2>Budi Darmawan, Bluemix Enablement</H2>
<H2>Pam Geiger, Bluemix Enablement</H2>
</div>

<div class="page-break"></div>

<div class="copyright">

© Copyright IBM Corporation 2017  

IBM, the IBM logo and ibm.com are trademarks of International Business Machines Corp., registered in many jurisdictions worldwide. Other product and service names might be trademarks of IBM or other companies. A current list of IBM trademarks is available on the Web at *Copyright and trademark information* at www.ibm.com/legal/copytrade.shtml.  

This document is current as of the initial date of publication and may be changed by IBM at any time.  

The information contained in these materials is provided for informational purposes only, and is provided AS IS without warranty of any kind, express or implied. IBM shall not be responsible for any damages arising out of the use of, or otherwise related to, these materials. Nothing contained in these materials is intended to, nor shall have the effect of, creating any warranties or representations from IBM or its suppliers or licensors, or altering the terms and conditions of the applicable license agreement governing the use of IBM software. References in these materials to IBM products, programs, or services do not imply that they will be available in all countries in which IBM operates. This information is based on current IBM product plans and strategy, which are subject to change by IBM without notice. Product release dates and/or capabilities referenced in these materials may change at any time at IBM's sole discretion based on market opportunities or other factors, and are not intended to be a commitment to future product or feature availability in any way.
</div>

<div class="page-break"></div>


# Social Review Microservice implemented with OpenWhisk actions


## Introduction

This project is built to demonstrate how to build a Microservices application implemented as OpenWhisk actions to access an IBM Cloudant NoSQL database. It provides basic operations of saving and querying reviews from a database as part of a Social Review function. Additionally, the review text is analyzed using the Watson Tone Analyzer service to determine whether to flag negative reviews for further inspection.  The project covers the following technical areas:  

 - Leverage OpenWhisk actions and REST API gateway to build a Serverless Microservices application.  
 - Use IBM Cloudant NodeJS library to access a Cloudant database.  
 - Use OpenWhisk triggers to fire an OpenWhisk action on Cloudant database changes.  
 - Use Watson Tone Analyzer REST API to analyze text.  

## About Watson Tone Analyzer

Tone Analyzer uses linguistic analysis to detect three types of tones in written text: emotions, social tendencies, and writing style. The Tone Analyzer service can be to understand the emotional context of conversations and communications. You can then use this insight to respond in an appropriate manner. Tone Analyzer is used when you need to understand text more deeply than simple positive and negative sentiment. You can use the service to deeply understand written text. Common uses for the Tone Analyzer service include:  
  - Analyze communications and improve message effectiveness  
  - Optimize the tones in your communication to increase the impact on your audience  
  - Understand and route call center customers based on their tone  
  - For bloggers and journalists, fine-tuning of writing to reflect a specific personality or style  

It takes any text as input and outputs a hierarchical representation of the analysis of the terms in the input message in JSON format.

In this lab exercise, you will provision a cloudant database to store the reviews for a social review application. The OpenWhisk cloudant package is used to configure Cloudant to fire a trigger when a change is made to the social review database. You create OpenWhisk actions to invoke the Watson Tone Analyzer to analyze the text when a new review is stored in the database, or when a change is made to an existing review. The analysis results are stored into the socialreview database.  

### Provision Cloudant Database in Bluemix

1. Login to the virtual machine as `bmxuser` and password of `passw0rd`.  
2. Open a Firefox browser by clicking **Activities** and then clicking the Firefox icon.

 ![OpenBrowser](images/OpenBrowser.png)

3. Login to your Bluemix console at **console.ng.bluemix.net** for the US region. You may typically use a different region, but for these exercises, please use the US region.
4. Click **Catalog**, then under Services, **Data and Analytics**.
5. Click on **Cloudant NoSQL DB**.

 ![CloudantService](images/AddCloudant.png)  

6. Give your Cloudant service a unique name like `cloudantdb-socialreview-[yourinitials]`, replacing *[yourinitials]* with your own initials, such as pfg in this example.  

  ![CreateCloudant](images/cloudantname.png)

7. For testing, use the default **Lite** plan and click **Create**  
8. Once the service has been created, click **Service Credentials** to view the service credentials.  

 ![Credentials](images/Credentials.png)  

9. Click **View Credentials** to display the credentials for the Cloudant NoSQl DB Service. The Social Review microservice requires the `url` property.  

 ![ViewCredentials](images/ViewCredentials.png)

### Provision Watson Tone Analyzer in Bluemix

Next you provision an instance of the Tone Analyzer service to analyze the reviews that are posted.  

1. In your Bluemix console, Click **Catalog**, then under **Services** click **Watson**.
2. Click **Tone Analyzer**. <br>

 ![Tone Analyzer](images/ToneAnalyzer.png)  

3. Name your Watson Tone Analyzer service `tone-analyzer-[yourinitials]`, replacing *[yourinitials]* with your own initials.
4. For testing, use the default **standard** plan and click **Create**.  

 ![CreateToneAnalyzer](images/CreateToneAnalyzer.png)  

5. Once the service has been created, note the service credentials under **Service Credentials**.  In particular, the Social Review microservice requires the `username`, `password`, and `url` properties.

## Download and configure the OpenWhisk CLI
1. Open a terminal Window by selecting **Activities** and **Show Applications** ![Icons](images/0008-iconview.png) from the panel and typing **XTerm** into the search field. Click the Xterm icon to open the terminal.

    ![](images/0009-xterm.png)  

2. Get the code for this lab by entering the following command:  

   ```
   git clone -b openwhisk https://github.com/bluemix-enablement/BMX2450OpenWhisk.git
   ```  

 This creates a directory named BMX2450OpenWhisk.

3. Open a new browser window and download the OpenWhisk CLI for your platform from this link [https://console.ng.bluemix.net/openwhisk/cli](https://console.ng.bluemix.net/openwhisk/cli). When the Bluemix Console opens, verify that you are in the org and space that correspond to where your Cloudant service was created.  

 ![BluemixOrgandSpace](images/BluemixOrgandSpace.png)  

4. Click **Download the Linux CLI**. If the button shows a different platform, click where it says `Click Here`.

 ![DownloadCLI](images/DownloadCLI.png)  

5. The file name is **OpenWhisk_CLI-linux.tgz**. Click **Save File** and then **OK**.
6. In the terminal Xterm window, change to the download directory and enter the following command to extract the command line interface:
<pre><code>tar -xvf OpenWhisk_CLI-linux.tgz</pre></code>
7. You must add the *wsk* path to the PATH variable. First, you must find the full path to where you just downloaded the file. In the terminal, type  
`pwd wsk`

8. Note the response, which is something like *home/usr/Downloads*.  
   **NOTE**: normally, you would move this file to a more appropriate directory. For this exercise, you can just leave it in the *Downloads* directory.  
9. In the terminal, type the following:  
`export PATH=$PATH:/[path to your file]`

10. Verify that the cli works by entering the command:  
<pre><code> wsk</pre></code>  
Usage information for the wsk command is dislayed.  
 ![wskusage](images/wskUsage.png)
11. Return to the browser tab with the OpenWhisk CLI information and copy the command to configure the OpenWhisk CLI. Note that this is customized for your Bluemix id and the org and space that you are logged into, which is why you verified this information in step one.<br>
 ![Copy](images/copycreds.png)
12. Paste the link into your terminal window and run it.

 ![WSKAuth](images/Wskauth.png)  

 This sets your OpenWhisk namespace and authorization key.
13. Run the following to automatically create OpenWhisk packages with the Cloudant credentials in your space:
   ```
   wsk package refresh
   ```

   This should result in a package containing your Cloudant database credentials.  

 ![Packagerefresh](images/Packagerefresh.png)

 <p>Enter the following to list your OpenWhisk packages.

  ```
   wsk package list
  ```  

 ![PackageList](images/PackageList.png)  


 <div class="page-break"></div>
 -----

## Deploy the OpenWhisk package and actions
![OpenWhiskConcepts](images/OpenWhiskConcepts.png)<br>
Some OpenWhisk terminology, as a reminder:<br>
**Event Sources**  
Event sources, such as devices, queues, databases, and webhooks, emit classes of events in the form of triggers.<br>
**Triggers**  
Triggers are the class of events (including device readings, published messages, and data changes) that are emitted by event sources.<br>
**Actions**  
Actions are functions that encapsulate code – written in any supported language by implementing a single method signature – to be executed in response to a trigger.<br>
**Rules**  
Rules represent the declarative association between a trigger and an action, defining which action(s) should be executed in response to an event.<br>
**Packages**  
Packages encapsulate external services in a reusable manner and assemble them into triggers and actions.  


For these lab exercises, a set of actions is being provided. In this case, the actions are written in JavaScript, but can also be Swift, or run in a Docker container. You will first create a socialreview package and then add the actions to that package. The properties that are added to the package are available to the actions in the package, so the access information for the Cloudant database and the Watson Tone Analyzer only need to be defined at the package level.  

1. Change directories to the BMX2450OpenWhisk directory that contains the code that you downloaded earlier. If you are currently in the Downloads directory, the command will be:
   ```
   cd ../BMX2450OpenWhisk/
   ```   
2. Use the OpenWhisk CLI to create a `socialreview` package.  Pass the `url` property from the Cloudant service instance created, and the `username`, `password`, and `url` properties from the Watson Tone Analyzer service instance.

   ```
    wsk package create socialreview --param cloudant_url <cloudant url> --param watson_url <watson tone analyzer url> --param watson_username <watson tone analyzer username> --param watson_password <watson tone analyzer password --param cloudant_reviews_db socialreviewdb

   ```
   It looks similar to the following  
  ![createPackage](images/createPackage.png)

3.. You are creating four OpenWhisk actions:  

  - **initCloudant**: connects the the Cloundant instance that is running in Bluemix using the url that you defined when you created the package. The database credentials are retrieved from Bluemix and the socialreviewdb is created if it hasn't been created yet. The name fo the database when also defined when you created the package, in the cloudant_reviews_db parameter.
  - **saveReview**: This action saves new reviews to the database.
  - **getReviews**: Retrieves reviews from the database.
  - **analyzeTone**: Invokes Tone Analyzer to analyze the tone of the review, and stores the analysis in the cloudant database with the review entry. It uses the Tone analyzer url, username, and password that you defined when you created the package in the previous step.  


3. Upload all of the actions under the created package. The code for the actions is in the openwhisk/actions directory. All of the actions in the package inherit the properties we created in the package (`cloudant_url`, `watson_url`, `watson_username`, `watson_password`, `cloudant_reviews_db`) :  

   ```
   # wsk action create socialreview/initCloudant openwhisk/actions/initCloudant.js
   # wsk action create socialreview/saveReview openwhisk/actions/saveReview.js
   # wsk action create socialreview/getReviews openwhisk/actions/getReviews.js
   # wsk action create socialreview/analyzeTone openwhisk/actions/analyzeTone.js
   ```  

The four actions are created in the socialreview package:  

![createActions](images/createActions.png)  

3. View the actions and packages. From the command line, enter

    ```
    #wsk package list
    ```

    You now have the socialreview package in addition to the cloudant package.
    ![PackageList](images/ListPackages.png)
    To see the actions, enter:
    ```
    #wsk action list
    ```

    You see the four actions that you just added to the socialreview package,
   ![ActionList](images/actionList.png)
4. View the information in the OpenWhisk Dashboard. The OpenWhisk Dashboard in Bluemix can be used to Develop, Monitor, and Manage your OpenWhisk packages, actions, and triggers as well. In your Bluemix interface, open the navigation menu and click **Apps** then **OpenWhisk** to open the Dashboard.<br>
 ![OpenWhiskDashboard](images/OpenWhiskDashboard.png)<br>
 The Dashboard opens to the Getting Started Window. Click **Manage** to see your packages and actions. You see the socialreview package with the four actions that you added, as well as the package for your Cloudant service. Scroll through the list to see the actions that are available as part of that package.
 ![DashboardManage](images/DashboardManage.png)

5. Actions can be executed directly from the Dashboard or from the command line, as well as invoked automatically by a trigger. We'll take a look at each of these options.  
First, execute the initCloudant OpenWhisk action to create the Cloudant databases and indexes required by the Social Review microservice. Since you are already looking at the Dashboard, let's invoke this action here.
  - In the OpenWhisk Dashboard, click **Develop**. The actions that you created are listed under My Actions. Any Sequences, Rules, and Triggers would be displayed here as well.
  - Click **initCloudant** to select that action. The Node.js source for the action is displayed.
 ![initCloudant](images/initCloudant.png)<br>
   Notice that you have a number of options. You can Create a new action, run this action, link the action into a sequence, or automate the action. You can also make changes to the action source code from here as needed.
   - Click Run this Action. The Invoking an action window is displayed. You can specify JSON input from this window that will be passed as input to the action.
   ![InvokeAction](images/InvokeAction.png)
   - Scroll down to see the parameters that are bound to the action. These are the parameters that you defined to the socialreview package. You can use the JSON input field to override any of these parameters, as needed.
   ![BoundParameters](images/BoundParameters.png)
   - The only input that this action requires is the Cloudant URL, so click **Run with this value**. If you configured your Cloudant URL properly, you should see a successful result.<br>
   ![SuccessfulAction](images/SuccessfulAction.png)
6. Verify that the database was initialized successfully.
   - From the Bluemix navigation menu, click **Services** and **Dashboard**.   
   - Locate your Cloudant DB instance and click the name to open the Cloudant Manage window.
    - Click **Launch** to launch the Cloudant Dashboard.
    - Click Databases. You see that a socialreviewdb and a socialreview-staging database have been created.
    ![SocialreviewDB](images/socialreviewDB.png)

7. Create the OpenWhisk REST API gateway for the OpenWhisk actions. The OpenWhisk API gateway is a new, experimental feature, which enables you to easily expose your OpenWhisk actions as RESTful endpoints. You can assign actions to specific endpoints, and even have verbs (get, put, post delete) from the same endpoint assigned to different actions. Here you expose the getReviews and saveReview actions as REST APIs with the following commands:

   ```
   # wsk api-experimental create /api /reviews/list get socialreview/getReviews
   # wsk api-experimental create /api /reviews/comment post socialreview/saveReview
   ```
![CreateAPIs](images/APICreation.png)

8. Create an OpenWhisk trigger called `reviewTrigger` on the staging database `socialreviewdb-staging`.  This uses the Whisk built-in trigger from the generated Cloudant package. Replace `<org>` and `<space>` with the Bluemix org and space that you are using for this lab and replace `<your Cloudant db name>` with the name of your Cloudant database. You can retrieve this information by running the **wsk package list** command.  Reviews are initially added to this staging database, and then moved to the socialreview database after being analyzed.
  ![OrgInfo](images/OrgInfo.png)

   ```
   # wsk trigger create reviewTrigger --feed /<org>_<space>/<your Cloudant DB name>/changes --param dbname socialreviewdb-staging
   ```
 The trigger is successfully created.
 ![CreateTrigger](images/CreateTrigger.png)

6. Create a rule that fires the `analyzeTone` action when `reviewTrigger` is triggered.  This analyzes the text of posted reviews and uses the output to decide whether to unflag the review so it is returned by the API. Once the text is analyzed, it will be inserted into the socialreviewdb database.

   ```
   # wsk rule create handleReviewPosted reviewTrigger socialreview/analyzeTone
   ```
  - Return to the OpenWhisk Dashboard and click **Develop**. Your new rule and new trigger are now listed.<br>
  ![RuleDashboard](images/RuleDashboard.png)<br>
  The flow you just created looks like this:<br>
  ![OpenWhiskFlow](images/OpenWhiskFlow.png)

## Verify the Social Review Microservice

1. Check the created OpenWhisk endpoints, for example:
   ```
   # wsk api-experimental list
      ```
  You see your GET and POST APIs.
  ![APIList](images/APIList.png)

2. Create a positive review using the API, replacing the API endpoint with your API endpoint from the listing in the previous step. You can change the reviewer name, comment, review date, and email as desired.
<div class="page-break"></div>

   ```
   # curl -X POST -H "Content-Type: application/json"    -d '{ "comment": "I love this product!", "rating": 5, "reviewer_name": "Pam Geiger", "review_date": "01/19/2016",
   "reviewer_email": "pgeiger@us.ibm.com"}' <your api endpoint>/reviews/comment?itemId=13402
   ```
The command completes successfully.
 ![PostivePost](/images/postsuccess.png)

3. Verify the Results.
  - Open the OpenWhisk Dashboard in Bluemix.
  - Click **Monitor**
  You see activity information for all of the OpenWhisk activities. In the activity log, you see that the `saveReview` action is called, which saves the review to the `socialreviewdb` database, initially flagging it.
   - the `reviewTrigger` is fired,
   - which triggers the `handleReviewPosted` rule,
   - which executes the `analyzeTone` action.  the review text, "I love this product!", is analyzed and determined to be positive, and the comment is unflagged and updated into the `socialreviewdb` database with the JSON document returned by the Watson Tone Analyzer attached.

   ![AnalyzeTone](images/AnalyzeTone.png)

4. Call the GET API to get the reviews for the item:
   ```
   # curl -X GET -H "Accept: application/json" <your url endpoint>/api/reviews/list?itemId=13402
   ```
 It has been stored in the socialreview database and is not flagged.
 ![GetReviews](images/GetReviews.png)
5. Now submit a negative review:
   ```
   # curl -X POST -H "Content-Type: application/json" -d '{ "comment": "I hate this product!", "rating": 1, "reviewer_name": "Jack Sprat", "review_date": "01/19/2016",  "reviewer_email": "jsprat@gmail.com"}' <your api endpoint>/api/reviews/comment?itemId=13402

   ```
 ![NegativeReview](images/NegativeReview.png)
6. Observe in the OpenWhisk monitor that the same sequence is fired.
7. Open the Cloudant Dashboard and click **Databases**
  - Click socialreviewdb to open the database.
  - Click **All Documents**
  - Click **Edit Document** for the first document in the database.
  ![EditDocument](images/EditDocument.png)
  - You see the review with the tone analysis information. Note that for the negative review the scores are high for anger, disgust and feat.
  ![NegativeTone](images/NegativeTone.png).
  - Click **Cancel** and edit the postive review. Notice that this is reviews is not flagged since it is positive, and the ratings for anger, disgust, and fear are low,while the rating for joy is high.<br>
  ![PostiveReview](images/PostiveReview.png)

This completes the lab exercises. 

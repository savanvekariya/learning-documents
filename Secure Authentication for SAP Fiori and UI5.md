Implementing secure authentication for an SAP Fiori/UI5 application involves a specific sequence: preparing your identity provider, defining security metadata, instantiating the service, and finally configuring your application's entry point.
## Step 1: Establish Trust with SAP Identity Services (IAS) [1] 
Before the application can recognize your users, your [SAP BTP Subaccount](https://help.sap.com/docs/SAP%20SECURE%20LOGIN%20SERVICE/c35917ca71e941c5a97a11d2c55dcacd/bd38e9deab2743aa8a3fb8aaa5b12210.html) must trust your [SAP Identity Authentication Service (IAS)](https://discovery-center.cloud.sap/card/a870731d-57e7-4410-8291-16e935484e0e) tenant. [2] 

   1. Get Metadata: In your IAS Administration Console, go to Applications & Resources > Tenant Settings > SAML 2.0 Configuration and download the Metadata XML.
   2. Add Trust: In the SAP BTP Cockpit, navigate to Security > Trust Configuration. Click Establish Trust (or New Trust Configuration) and upload the IAS XML file.
   3. Link IAS to App: In your IAS console, create a new "Application" representing your BTP subaccount to complete the bidirectional trust. [2, 3, 4, 5, 6] 

## Step 2: Define the Security Descriptor (xs-security.json) [7] 
This file tells the XSUAA service what roles and permissions your app requires. [7, 8] 

   1. Create the file in your project root with this basic structure:
   
   {
     "xsappname": "my-fiori-app",
     "tenant-mode": "dedicated",
     "scopes": [
       { "name": "$XSAPPNAME.Display", "description": "Display access" }
     ],
     "role-templates": [
       {
         "name": "Viewer",
         "description": "View-only role",
         "scope-references": ["$XSAPPNAME.Display"]
       }
     ]
   }
   
   [7, 8, 9] 

## Step 3: Create the XSUAA Service Instance [10] 
Now, turn that JSON definition into a live security service in the cloud. [10, 11] 

   1. CLI Command: Open your terminal and run:
   cf create-service xsuaa application my-auth-service -c xs-security.json.
   2. Verify: Ensure the service is created in your BTP Cockpit under Instances and Subscriptions. [6, 12, 13, 14] 

## Step 4: Configure the App Router (xs-app.json) [15] 
The [App Router](https://github.com/SAP/cloud-bulletinboard-ads/blob/Documentation/Security/Exercise_22_DeployApplicationRouter.md) is the "bouncer" that redirects unauthenticated users to the IAS login page. [16, 17] 

   1. Set Authentication: In your xs-app.json, change the authenticationMethod to route.
   2. Define Protected Routes:
   
   {
     "authenticationMethod": "route",
     "routes": [
       {
         "source": "^/index.html",
         "target": "index.html",
         "authenticationType": "xsuaa"
       }
     ]
   }
   
   This ensures that whenever a user tries to load index.html, the App Router checks for a valid OAuth JWT token. [15, 18, 19] 

## Step 5: Assign Roles to Users
Even with everything set up, users can't enter unless you give them the role. [20] 

   1. Create Role Collection: In the BTP Cockpit under Security > Role Collections, create a new collection (e.g., "App_Viewers") and add the "Viewer" template from Step 2.
   2. Assign User: Go to Security > Users, select your user, and [assign the Role Collection](https://developers.sap.com/tutorials/hana-cloud-cap-add-authentication..html) to them. [1, 7, 21] 

Would you like a sample mta.yaml file to see how these components are bound together for deployment?

[1] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-sap/automation-of-trust-setting-role-collections-and-ias-user-groups-in-sap/ba-p/14213468)
[2] [https://help.sap.com](https://help.sap.com/docs/SAP%20SECURE%20LOGIN%20SERVICE/c35917ca71e941c5a97a11d2c55dcacd/bd38e9deab2743aa8a3fb8aaa5b12210.html)
[3] [https://developers.sap.com](https://developers.sap.com/tutorials/abap-custom-ui-trust-cf..html)
[4] [https://discovery-center.cloud.sap](https://discovery-center.cloud.sap/card/3f60690c-8d7b-4ccb-aede-c6b343357457)
[5] [https://help.sap.com](https://help.sap.com/docs/conversational-ai/integration-with-sap-s-4hana/configure-sap-btp-subaccount-to-trust-ias-as-saml-idp-and-export-sp-metadata)
[6] [https://community.simplifier.io](https://community.simplifier.io/doc/installation-instructions/setup-external-identity-provider/configure-btp-identity-services-via-openid-connect/)
[7] [https://www.linkedin.com](https://www.linkedin.com/pulse/sap-btp-security-xsuaa-basics-anas-khan-carxc)
[8] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-members/obtaining-an-oauth-token-from-sap-xsuaa-on-sap-btp-trial-a-step-by-step/ba-p/14248017)
[9] [https://cap.cloud.sap](https://cap.cloud.sap/docs/guides/security/authentication)
[10] [https://developers.sap.com](https://developers.sap.com/tutorials/cp-cf-security-xsuaa-create..html)
[11] [https://www.mindsetconsulting.com](https://www.mindsetconsulting.com/sapui5-standalone-application-deployment-with-xsuaa-in-the-btp-cf-environment/)
[12] [https://developers.sap.com](https://developers.sap.com/tutorials/hana-cloud-cap-add-authentication..html)
[13] [https://developers.sap.com](https://developers.sap.com/tutorials/hana-cloud-cap-add-authentication..html)
[14] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-sap/demystifying-xsuaa-in-sap-cloud-foundry/ba-p/13468237)
[15] [https://developers.sap.com](https://developers.sap.com/tutorials/hana-cloud-cap-add-authentication..html)
[16] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-sap/from-xsuaa-to-ams-the-practical-path-to-stronger-authorization/ba-p/14320451)
[17] [https://www.youtube.com](https://www.youtube.com/watch?v=_m4lr5iNqi8)
[18] [https://www.linkedin.com](https://www.linkedin.com/pulse/securing-external-applications-sap-btp-xsuaa-using-oauth2-sanjay-pm-oii5c)
[19] [https://www.youtube.com](https://www.youtube.com/watch?v=mQ4eo8mwAwU)
[20] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-cloud-platform-backend-service-tutorial-25-understanding-app-router-2/ba-p/13409873)
[21] [https://community.sap.com](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-btp-security-how-to-use-rest-api-of-xsuaa-to-programmatically-manage/ba-p/13540720)

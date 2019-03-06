# API-Management-ASE-AppGateway
Deployment of Internal VNET API Management consuming APIs from Internal ASE. Accessing the APIM through Application gateway


This week we had a very interesting project to have internal business APIs hosted in App Service Environment to be consumed through an API Management and be available to Public devs (as well as Internal developers). The most interesting part was since both App Service Env and APIM services were hosted internally on the VNET, having them working took new challenges. We got it finally worked after we added HOST headers to APIM service and consume the APIs using ASE’s DNS.

For this project, we have a fairly self-explanatory Architecture diagram as below.

 ![alt](https://res.cloudinary.com/drlwgs1ub/image/upload/v1551912360/apim.jpg)

#####Challenges we faced

* The source control could be in Public Github or internal Source repo and should be accessible from anywhere.
* CI/CD pipelines depends on in built in Hosted Agents and needed to be accessible from public. Since the API Apps were hosted internally, there needed a infrastructure within the VNET to act like a Hosted Agent for Azure DevOps
 * Refer this to build your own Hosted Agent within the VNET: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops
* API Management service expects and explicit NSG for APIM management service to function properly
 * Refer section “__Ports required for API Management__” this : https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet#a-namenetwork-configuration-issues-acommon-network-configuration-issues 
* We tried to add the APIs to API Management service using the INTERNAL api app’s DNS name which didn’t work. After a hard struggle (its not mentioned anywhere in the Docs yet) we used the ASE’s DNS to consume the APIs and differentiated the Apps within ASE using HOST header as APIM Policy.
 * Look for how to configure Policies in APIM here: https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetHTTPheader 
* Even after specifying the HOST values, initially for Test Domain (that we didn’t own) the Portal Test features failed as the APIM service tries to resolve the DNS entries publicly. We were able to achieve this by using our Own Custom Domain and adding CNAME entries in our godaddy DNS settings.

#####Prereqs

1. Custom Domain Name purchased (like on godaddy)
2. Since we are using actual custom domain, procuring a SSL certificates (we used a wild card one from Azure Certificates Service) to use one for all our custom domains

#####Deployment flow for the resources and putting the pieces together  
The deployment process was in the sequence as below

1. VNET with the following configurations: ase-internal-vnet
  * Address space for VNET: 10.0.0.0/16
  * 4 Subnets
     * backendSubnet for DNS Service: 10.0.0.0/24
     * apimsubnet for Internal API Management Service: 10.0.1.0/28
     * asesubnet for ILB ASE: 10.0.2.0/24
     * VMSubnet for Test VMs and Internal DevOps Hosted Agent VM: 10.0.3.0/24
2. Private DNS service (Public Preview) since adding a DNS service requires the VNET to be empty. We used our custom domain purchased on godaddy: sarwascloud.com
  * Refer this for overview: https://docs.microsoft.com/en-us/azure/dns/private-dns-overview 
  * Refer this for deploying the service: https://docs.microsoft.com/en-us/azure/dns/private-dns-getstarted-cli 
  * Please note the service is in Public Preview.
  * We also tested this using our own custom DNS Server built on Windows Server 2016.
3. App Service Environment with Internal Load Balancer (ILB) option: aseinternal (DNS: aseinternal.sarwascloud.com)
4. Uploaded the wild car cert for the ILB
  * Created App Service Plan with ASE as location
  * Created two API Apps (App Services for simplicity)
     * srasprest (URL: https://srasprest.sarwascloud.com) – ASP.NET MVC based web API
     * srnoderest (URL: https://srnoderest.sarwascloud.com) – NodeJS based web API
     * configured the above web apps using the certificates
     * configured Application Insights to the above apps: api-insights
  * Created an Cosmos DB service for web APIs hosted internal to VNET: noderestapidb
  * Created DNS entries on the Private DNS zone created
5. Configured CI/CD pipelines for the above Apps using Github as source repo and Azure DevOps for pipelines
  * Configured a Build Agent VM internal to the VNET for the release pipeline for deploying the API Apps: asevm
6. Used a testVM to test the API Apps working internally on the VNET
7. Created API Management service: apim-internal
  * Configured the service to connect to intern VNET on Subnet: apimsubnet
  * Configured custom domains for APIM Services using SSL Cert
     * API portal (api.sarwascloud.com)
     * Dev Portal (portal.sarwascloud.com)
  * In the APIs section, configured the ASE Apps using ASE’s DNS name added Policy for HOST Header to differentiate the APIs (ASP.NET vs NodeJS)
8. Used the above created testVM to test the API Management service internally
  * Remember the testing the APIM APIs from Azure portal will still NOT work as we don’t have api.sarwascloud.com not be able to publicly resolve
9. Configured Application Gateway (WAF V1) to access the APU service: apim-gateway
  * Added a Frontend Public IP with DNS name to the App gateway: appgateway-pip (ase-appgtwy.westus.cloudapp.azure.com)
  * Added Backend Pool to point to Internal IP of API Management Service
  * Added two Http Settings for dev portal and API portal as custom probes
  * Added Health probes to point to (refer: https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway#create-application-gateway-configuration STEP: 6)
     * API Portal Probe path: /status-0123456789abcdef
     * Dev Portal Probe path: /signin
     * Configured Rules and Listeners using SSL Certificate pfx and cer
10. Configured the DNS entries in Godaddy CNAME entries of api.sarwascloud.com and portal.sarwascloud.com with App Gateway’s public DNS name: ase-appgtwy.westus.cloudapp.azure.com

Once we have the above resources configured, we were able to
==Reach the Dev Portal from Public==  
==Able to test the APIM services APIs using Azure Portal==

*Note that it is not a good practice to use same URL for Internal and External Devs for the APIM services (currently in the above demo, both URLs are same). If we want to choose to have different URLs for internal and external devs, we could make use of App Gateway WAF v2 which supports http redirection and much more.*

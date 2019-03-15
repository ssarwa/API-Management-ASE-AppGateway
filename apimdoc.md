---
title: Exposing internal Business API’s securely, reliably at scale to extranet
titleSuffix: Azure Example Scenarios
description: Use Azure API Management to modernize and expose intranet legacy web APIs.
author: ssarwa
ms.date: 03/12/2019
ms.topic: example-scenario
ms.service: architecture-center
ms.subservice: example-scenario
ms.custom: fasttrack
social_image_url: ./architecture.jpg
---

# Exposing internal Business API’s securely, reliably at scale to extranet

In this scenarios an organization has hosted multiple APIs using [App Service Environments][ase] and would like to make use of these APIs using [Azure API Management (APIM)][apim]. The API Management could also be exposed to external users (as well as Internal developers) to utlize the full potential of the APIs. This could be achieved using [Application Gateways][appgtwy] connected to Internal API Management service which inturn consumes apps in ASE. The most interesting part was since both App Service Env and APIM services were hosted internally on the VNET, having them working together took new challenges. This requires adding HOST headers to APIM service and consume the APIs using ASE’s DNS.

This example demonstrates how to configure the API Management Service consuming APIs hosted on App Service Environments on a secured private VNET and assumes
- The development team will modernize the APIs back end, which is hosted on WebApps in App Service Environments internally
- The API Management service is mainted by Administrator/s within the Organizations

## Architecture

![Architecture diagram][architecture]

The Web APIs will be hosted as a platform as a service (PaaS) applications on Azure with Hybrid Connectivity to onprem network. These APIs will be available to external and internal developers and can make use of streamlined API connections and documentation.

### Components

- [Azure Virtual Network][vnet] enables Azure resources to securely communicate with each other, the internet, and on-premises networks.
- [Azure Private DNS][dns] Azure DNS provides a reliable, secure DNS service to manage and resolve domain names in a virtual network without your needing to add a custom DNS solution.
- [Azure API Management][apim] helps organizations publish APIs to external, partner, and internal developers to unlock the potential of their data and services.
- [Application Gateway][appgtwy] Azure Application Gateway is a web traffic load balancer that enables you to manage traffic to your web applications.
- Internal Load Balancer [App Service Environment][ase] is an Azure App Service feature that provides a fully isolated and dedicated environment for securely running App Service apps at high scale.
- [Azure DevOps][devops] is a service for managing your development life cycle end-to-end &mdash; from planning and project management, to code management, and continuing to build and release.
- [Application Insights][appinsights] is a first-party, extensible Application Performance Management (APM) service for web developers on multiple platforms.
- [Azure Cosmos DB][cosmosdb] is Microsoft's globally distributed, multi-model database service.


### Alternatives

- In an [Azure lift and shift scenario][azure-vm-lift-shift] linked to their deployed Azure Virtual Network, the customer could directly address the back-end service through private IP addresses.
- In the on-premises scenario, the API Management instance could reach back to the internal service privately via an [Azure VPN gateway and site-to-site IPSec VPN connection][azure-vpn] or [ExpressRoute][azure-er] making this a [hybrid Azure and on-premises scenario][azure-hybrid].
- Customer could make use of there own DNS settings instead of using Azure based DNS Service.
- Customer could deploy the Internal APIs outside Azure's environment and could still benefit by feeding the APIs to API Management Service. 

## Considerations

1. The web apis are hosted over secured HTTPs protocol and will be using an [SSL Certificate][ssl].
2. The Application Gateway also is configured over port 443 for secured and reliable outband calls.
3. The API Management service is configured on custom domains using SSL certificates.
4. Consider the [Network configuration][ntwkcons] for App Service Environments
5. There needs to be a explicit mention about [port 3443 allowing API Management][apim-port-nsg] to manage via Portal or PowerShell.
6. The API Management accepts ASE's DNS entry for all the apps hosted under App Service Environments. Add [APIM policy][apim-policy] to set explicit HOST Header to differentiate between Apps under the App Service Environment.
7. Consider [Integrating with Azure Application Insights][azure-apim-ai], which also surfaces metrics through [Azure Monitor][azure-mon] for monitoring.
8. If using CI/CD pipelines for deploying Internal APIs, consider [build your own Hosted Agent on a VM][hosted-agent] in the VNET

### Availability

- Consider leveraging the [typical design patterns for availability][design-patterns-availability] when building your cloud application.
- Review the availability considerations in the appropriate [App Service web application reference architecture][app-service-reference-architecture]
- For additional considerations concerning availability, see the [availability checklist][availability] in the Azure Architecture Center.

### Scalability

- When building a cloud application be aware of the [typical design patterns for scalability][design-patterns-scalability].
- Review the scalability considerations in the appropriate [App Service web application reference architecture][app-service-reference-architecture]
- For other scalability topics, see the [scalability checklist][scalability] available in the Azure Architecture Center.

### Security

- Consider leveraging the [typical design patterns for security][design-patterns-security] where appropriate.
- Review the security considerations in the appropriate [App Service web application reference architecture][app-service-reference-architecture].
- Consider following a [secure development lifecycle][secure-development] process to help developers build more secure software and address security compliance requirements while reducing development cost.
- Review the blueprint architecture for [Azure PCI DSS compliance][pci-dss-blueprint].

### Resiliency

- Consider leveraging the [circuit breaker pattern][circuit-breaker] to provide graceful error handling should one part of the application not be available.
- Review the [typical design patterns for resiliency][design-patterns-resiliency] and consider implementing these where appropriate.
- You can find a number of [recommended practices for App Service][resiliency-app-service] in the Azure Architecture Center.
- Consider using active [geo-replication][sql-geo-replication] for the data tier and [geo-redundant][storage-geo-redudancy] storage for images and queues.
- For a deeper discussion on [resiliency][resiliency], see the relevant article in the Azure Architecture Center.

## Deployment scenario

### Prereqs and assumptions
1. Custom Domain Name purchased (like on godaddy)
2. Since we are using actual custom domain, procuring a SSL certificates (we used a wild card one from Azure Certificates Service) to use one for all our custom domains
3. This specific deployment uses domain name as contoso.org and a wild card certificate for the domain.
4. The deployment is using the resource names and address spaces mentioned in the deployment section which can be configured. 

### Deployment in sequence of resources and putting the pieces together  
The deployment process was in the sequence as below

1. VNET with the following configurations: ase-internal-vnet
  * Address space for VNET: 10.0.0.0/16
  * 4 Subnets
     * backendSubnet for DNS Service: 10.0.0.0/24
     * apimsubnet for Internal API Management Service: 10.0.1.0/28
     * asesubnet for ILB ASE: 10.0.2.0/24
     * VMSubnet for Test VMs and Internal DevOps Hosted Agent VM: 10.0.3.0/24
2. Private DNS service (Public Preview) since adding a DNS service requires the VNET to be empty.
  * Refer this for [deployment guidelines][dnsguide] 
3. App Service Environment with Internal Load Balancer (ILB) option: aseinternal (DNS: aseinternal.contoso.org)
4. Uploaded the wild car cert for the ILB
  * Created App Service Plan with ASE as location
  * Created two API Apps (App Services for simplicity)
     * srasprest (URL: https://srasprest.contoso.org) – ASP.NET MVC based web API
     * srnoderest (URL: https://srnoderest.contoso.org) – NodeJS based web API
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
     * API portal (api.contoso.org)
     * Dev Portal (portal.contoso.org)
  * In the APIs section, configured the ASE Apps using ASE’s DNS name added Policy for HOST Header to differentiate the APIs (ASP.NET vs NodeJS)
8. Used the above created testVM to test the API Management service internally
  * Remember the testing the APIM APIs from Azure portal will still NOT work as we don’t have api.contoso.org not be able to publicly resolve
9. Configured Application Gateway (WAF V1) to access the APU service: apim-gateway
  * Added a Frontend Public IP with DNS name to the App gateway: appgateway-pip (ase-appgtwy.westus.cloudapp.azure.com)
  * Added Backend Pool to point to Internal IP of API Management Service
  * Added two Http Settings for dev portal and API portal as custom probes
  * Added [Health probes to point][healthprobe] to refer:  STEP: 6)
     * API Portal Probe path: /status-0123456789abcdef
     * Dev Portal Probe path: /signin
     * Configured Rules and Listeners using SSL Certificate pfx and cer
10. Configured the DNS entries in Godaddy CNAME entries of api.contoso.org and portal.contoso.org with App Gateway’s public DNS name: ase-appgtwy.westus.cloudapp.azure.com

Once we have the above resources configured, we were able to
> Reach the Dev Portal from Public  
Able to test the APIM services APIs using Azure Portal

*Note that it is not a good practice to use same URL for Internal and External Devs for the APIM services (currently in the above demo, both URLs are same). If we want to choose to have different URLs for internal and external devs, we could make use of App Gateway WAF v2 which supports http redirection and much more.*

#### Deploy using ARM template

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/./azuredeploy.json" target="_blank">
    <img src="https://azuredeploy.net/deploybutton.png"/>
</a>


## Pricing

API Management is offered in four tiers: developer, basic, standard, and premium. You can find detailed guidance on the difference in these tiers at the [Azure API Management pricing guidance here.][apim-pricing]

Customers can scale API Management by adding and removing units. Each unit has capacity that depends on its tier.

> [!NOTE]
> The Developer tier can be used for evaluation of the API Management features. The Developer tier should not be used for production.

To view projected costs and customize to your deployment needs, you can modify the number of scale units and App Service instances in the [Azure Pricing Calculator][pricing-calculator].

## Related resources

Check out the related scenario on [Migrating legacy web APIs to API Management][related-scenario]

<!-- links -->

[architecture]: ./architecture.jpg
[ase]: https://docs.microsoft.com/en-us/azure/app-service/environment/intro
[apim]: https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts
[appgtwy]: https://docs.microsoft.com/en-us/azure/application-gateway/overview
[ssl]: https://docs.microsoft.com/en-us/azure/app-service/web-sites-purchase-ssl-web-site
[ntwkcons]: https://docs.microsoft.com/en-us/azure/app-service/environment/network-info
[apim-port-nsg]: https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet#a-namenetwork-configuration-issues-acommon-network-configuration-issues
[apim-policy]: https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetHTTPheader
[hosted-agent]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops
[vnet]: https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview
[devops]: https://docs.microsoft.com/en-us/azure/devops/index?view=azure-devops&viewFallbackFrom=vsts
[appinsights]: https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview
[cosmosdb]: https://docs.microsoft.com/en-us/azure/cosmos-db/introduction
[dnsguide]: https://docs.microsoft.com/en-us/azure/dns/private-dns-getstarted-cli
[healthprobe]: https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway#create-application-gateway-configuration
[related-scenario]: https://docs.microsoft.com/en-us/azure/architecture/example-scenario/apps/apim-api-scenario
[apim-pricing]: https://azure.microsoft.com/pricing/details/api-management/
[pricing-calculator]: https://azure.com/e/0e916a861fac464db61342d378cc0bd6
[azure-er]: https://docs.microsoft.com/en-us/azure/expressroute/expressroute-introduction
[azure-mon]: https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview

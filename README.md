# AZ-700 Hands-on Lab: Azure Front Door Private Link and Cross-Region Load Balancing

## Overview

This lab contains two independent exercises aligned with the AZ-700 Azure Network Engineer Associate curriculum.

- **Exercise 1:** Secure an Azure App Service origin behind Azure Front Door Premium using Private Link.
- **Exercise 2:** Deploy a cross-region Azure Load Balancer across Central India and Southeast Asia, with IIS virtual machines behind regional load balancers.

Both exercises use simple web applications so students can focus on networking, security, load balancing, health probes, and resilience.

**Estimated duration:** 2.5–3 hours, excluding lengthy provisioning delays.

**Deployment method:** Azure Portal

**Cleanup:** Required at the end of the lab.

---

## Lab objectives

By completing this lab, students will be able to:

1. Configure Azure Front Door Premium with a private App Service origin.
2. Approve and validate a Private Link connection.
3. Restrict direct public access to an App Service origin.
4. Differentiate regional and cross-region Azure Load Balancers.
5. Deploy IIS virtual machines in two Azure regions.
6. Configure backend pools, health probes, frontend IPs, and load-balancing rules.
7. Test backend health and demonstrate regional resilience.
8. Identify when Azure Front Door, regional Load Balancer, and cross-region Load Balancer are appropriate.

## Prerequisites

- An active Azure subscription with permission to create the resources in this lab.
- Access to the Azure Portal.
- A subscription quota sufficient for three small Windows Server VMs across the two regions.
- Permission to create App Service resources, Azure Front Door Premium, public IP addresses, virtual networks, NICs, and Standard Load Balancers.
- An RDP client for administering Windows VMs if required.

> **Cost warning:** Azure Front Door Premium, Windows VMs, managed disks, public IP addresses, and Standard Load Balancers can incur charges. Check current regional pricing before deployment and delete all lab resources afterward.

---

# Exercise 1 — Secure App Service Behind Azure Front Door Using Private Link

**Estimated time:** 45–60 minutes

## 1. Business scenario

A company hosts a customer-facing application on Azure App Service. Customers should access the application through Azure Front Door, which provides a global entry point and can apply web application security controls.

The company wants to prevent customers from bypassing Front Door and accessing the App Service directly through its public hostname.

The solution is to use Azure Front Door Premium with Private Link to the App Service origin.

## 2. Architecture

```text
                  Internet users
                        |
                        v
             Azure Front Door Premium
                 Public HTTPS endpoint
                        |
                        v
               Azure Private Link
             Front Door-managed endpoint
                        |
                        v
                  Azure App Service
                  Private origin access
```

Important: The private endpoint used for this integration is managed by Azure Front Door. Students do not need to create a separate private endpoint in their own VNet for this exercise.

## 3. Resources to create

| Resource | Suggested name | Region |
|---|---|---|
| Resource group | rg-az700-afd-pl | Central India |
| App Service | Unique name, e.g. az700app-uniqueid | Central India |
| App Service plan | asp-az700-afd | Central India |
| Front Door profile | afd-az700-lab | Global service |
| Front Door endpoint | afd-az700-uniqueid | Global service |
| Origin group | og-appservice | Front Door |
| Origin | origin-appservice | App Service region / supported Private Link region |

Use a globally unique App Service name and Front Door endpoint name. Azure Front Door Premium is required for Private Link origin support.

## 4. Deploy the App Service

### Step 1 — Create the resource group

1. Sign in to the Azure Portal.
2. Search for **Resource groups**.
3. Select **Create**.
4. Configure:
   - Subscription: Your lab subscription
   - Resource group: `rg-az700-afd-pl`
   - Region: `Central India`
5. Select **Review + create**, then **Create**.

### Step 2 — Create the App Service plan and web app

1. Search for **App Services**.
2. Select **Create** → **Web App**.
3. Configure:
   - Resource group: `rg-az700-afd-pl`
   - Name: A globally unique name
   - Publish: Code
   - Runtime stack: Choose a currently supported stack
   - Operating system: Windows
   - Region: Central India
   - Pricing plan: Choose a plan that supports the required Private Link configuration.
4. Review the plan and pricing before creating the app.
5. Select **Review + create** → **Create**.

### Step 3 — Verify application access

1. Open the deployed App Service.
2. Select **Overview**.
3. Open the **Default domain** link.
4. Confirm that the default application page is accessible.

For a more meaningful demonstration, deploy a simple HTML page containing:

```html
<!DOCTYPE html>
<html>
<head>
    <title>AZ-700 Private Link Lab</title>
</head>
<body>
    <h1>AZ-700 Private Link Lab</h1>
    <p>Application hosted on Azure App Service.</p>
    <p>Access should be through Azure Front Door.</p>
</body>
</html>
```

The default App Service page is also sufficient if you want to keep the lab short.

## 5. Create Azure Front Door Premium

### Step 4 — Create the Front Door profile

1. Search for **Front Door and CDN profiles**.
2. Select **Create**.
3. Choose the Azure Front Door service.
4. Select the **Premium** tier.
5. Create the profile using a name such as `afd-az700-lab`.
6. Configure an endpoint name that is globally unique.
7. Complete the deployment.

Portal labels may vary slightly with the current Azure interface.

### Step 5 — Create the origin group

1. Open the Front Door profile.
2. Navigate to **Settings → Origin groups**.
3. Select **Add**.
4. Name the origin group `og-appservice`.
5. Configure an appropriate health probe:
   - Protocol: HTTPS
   - Method: GET
   - Path: `/`
6. Save the origin group.

### Step 6 — Add the App Service origin

1. Open the origin group.
2. Select **Add an origin**.
3. Configure:
   - Origin type: App Services
   - Host name: Select the App Service created earlier.
   - Origin host header: The App Service default hostname, if not automatically populated.
   - HTTP port: 80
   - HTTPS port: 443
   - Priority: 1
   - Weight: Default value
4. Enable **Private Link**.
5. Select the appropriate supported Private Link region closest to the origin.
6. Select the `site` target subresource where requested.
7. Save the origin and update the origin group.

Wait for the origin configuration to be provisioned.

## 6. Approve the Private Link connection

### Step 7 — Approve the pending connection

1. Open the App Service.
2. Navigate to **Networking**.
3. Open the private endpoint connections configuration.
4. Locate the pending connection request associated with Azure Front Door.
5. Select the request and choose **Approve**.
6. Add a description if prompted.
7. Confirm the approval.

Wait for the connection to reach an approved and established state. Front Door may need several minutes before its origin becomes healthy.

### Step 8 — Create the Front Door route

1. Return to the Front Door profile.
2. Open **Front Door manager** or the endpoint and route configuration.
3. Create a route or edit the existing route.
4. Associate:
   - Endpoint: The Front Door endpoint
   - Origin group: `og-appservice`
   - Accepted protocols: HTTP and HTTPS, as appropriate
   - Redirect HTTP to HTTPS: Disabled
   - Forwarding protocol: HTTPS and HTTP
5. Save the route.

Wait for deployment and origin health checks to complete.

## 7. Validate Private Link security

### Step 9 — Test the Front Door endpoint

1. Open the Front Door endpoint hostname.
2. Use HTTPS.
3. Confirm that the application loads.

If it fails, verify the route, origin host header, origin health, Private Link approval, and protocol settings.

### Step 10 — Verify direct access is blocked

1. Copy the App Service default hostname.
2. Open it directly in a browser.
3. Confirm that direct public access is blocked once the private origin connection is established and public access has been disabled or restricted as required.

If necessary, open the App Service networking settings and explicitly disable public network access after verifying the Front Door path works.

**Expected result:**

- The application loads through the Front Door endpoint.
- Direct public access to the App Service is blocked.
- The origin uses the approved Private Link connection.

Do not disable public access before verifying the Front Door private connection, or you may temporarily make the application unavailable.

## 8. Troubleshooting checklist

| Problem | What to check |
|---|---|
| Front Door returns an error | Verify the origin hostname, route and forwarding protocol |
| Origin is unhealthy | Verify the health probe path and that HTTPS is working |
| Private endpoint request is pending | Approve the connection in App Service networking |
| Direct App Service access still works | Check App Service public network access and access restrictions |
| Front Door does not work immediately after approval | Allow time for Private Link and Front Door configuration to propagate |
| HTTPS fails | Verify origin HTTPS settings and the origin host header |

## 9. Discussion questions

1. Why is Front Door Premium required for this Private Link configuration?
2. Is the Private Link connection between the user's computer and Front Door, or between Front Door and App Service?
3. Why should direct access to the origin be restricted?
4. What happens if the Private Link request is not approved?
5. How does this design differ from using an App Service private endpoint inside a customer-managed VNet?
6. Would Azure Front Door Standard provide the same Private Link origin capability?

---

# Exercise 2 — Cross-Region Load Balancing with IIS

**Estimated time:** 90–120 minutes

## 1. Business scenario

A company operates a web application in two Azure regions:

- Central India: Two IIS virtual machines.
- Southeast Asia: One IIS virtual machine.

Each region has a regional Azure Standard Load Balancer. A cross-region Load Balancer provides a global frontend and directs traffic to the regional load balancers.

The objective is to demonstrate local load distribution and regional resilience using a simple IIS application.

## 2. Architecture

```text
                        Internet
                            |
                            v
              Cross-region Azure Load Balancer
                     Global public IP
                      /             \
                     /               \
                    v                 v
       Central India regional    Southeast Asia regional
          Standard LB               Standard LB
             |                          |
        +----+----+                     |
        |         |                     |
        v         v                     v
     IIS VM1   IIS VM2                IIS VM3
```

The cross-region Load Balancer uses the two regional Load Balancers as its backend resources. The regional Load Balancers distribute requests to their local IIS VMs.

## 3. Resources to create

| Resource | Suggested name | Region |
|---|---|---|
| Resource group | rg-az700-crlb | Central India |
| VNet | vnet-ci | Central India |
| Subnet | snet-web | Central India |
| Regional Load Balancer | lb-ci | Central India |
| Public IP | pip-lb-ci | Central India |
| IIS VM 1 | vm-ci-01 | Central India |
| IIS VM 2 | vm-ci-02 | Central India |
| VNet | vnet-sea | Southeast Asia |
| Subnet | snet-web | Southeast Asia |
| Regional Load Balancer | lb-sea | Southeast Asia |
| Public IP | pip-lb-sea | Southeast Asia |
| IIS VM 3 | vm-sea-01 | Southeast Asia |
| Cross-region Load Balancer | lb-global | Supported home region |
| Global public IP | pip-global | Global tier |

Use non-overlapping VNet address spaces, for example:

- Central India: `10.10.0.0/16`
- Southeast Asia: `10.20.0.0/16`

No VNet peering is required for this basic public cross-region load-balancing design.

## 4. Deploy the VNets

### Step 1 — Create the Central India VNet

1. Search for **Virtual networks**.
2. Select **Create**.
3. Configure:
   - Name: `vnet-ci`
   - Region: Central India
   - Address space: `10.10.0.0/16`
   - Subnet: `snet-web`
   - Subnet range: `10.10.1.0/24`
4. Create the VNet.

### Step 2 — Create the Southeast Asia VNet

1. Create another VNet.
2. Configure:
   - Name: `vnet-sea`
   - Region: Southeast Asia
   - Address space: `10.20.0.0/16`
   - Subnet: `snet-web`
   - Subnet range: `10.20.1.0/24`
3. Create the VNet.

## 5. Deploy three IIS virtual machines

### Step 3 — Create the Central India VMs

Create two Windows Server virtual machines:

- `vm-ci-01` in `vnet-ci`
- `vm-ci-02` in `vnet-ci`

Use a small VM size appropriate for the subscription quota and lab workload.

For both VMs:

1. Use the same Windows Server image.
2. Place the VM NIC in the `snet-web` subnet.
3. Do not assign individual public IP addresses unless specifically required for your chosen administration method.
4. Allow HTTP traffic from the intended client source and health probes.
5. Configure administration access using an appropriate secure method.

Repeat the process for `vm-sea-01` in Southeast Asia, using `vnet-sea`.

### Step 4 — Install IIS

Connect to each VM through an approved management method, such as Azure Bastion or a temporary secured RDP path.

Open PowerShell as Administrator and run:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

$server = $env:COMPUTERNAME
$body = @"
<!DOCTYPE html>
<html>
<head>
    <title>AZ-700 Load Balancing Lab</title>
</head>
<body>
    <h1>AZ-700 Load Balancing Lab</h1>
    <p>Server: $server</p>
</body>
</html>
"@

Set-Content -Path "C:\inetpub\wwwroot\iisstart.htm" -Value $body
Set-Content -Path "C:\inetpub\wwwroot\index.html" -Value $body
```

The default IIS welcome page may use `iisstart.htm`. The custom `index.html` page makes it easier to identify the responding server.

Verify that IIS is running:

```powershell
Get-Service W3SVC
```

Expected state: `Running`.

Ensure that the Windows Firewall permits inbound TCP 80. Also ensure that the Azure NSG allows the required HTTP traffic and Azure Load Balancer health probe traffic.

## 6. Create the Central India regional Load Balancer

### Step 5 — Create the Standard Load Balancer

1. Search for **Load balancers**.
2. Select **Create**.
3. Configure:
   - Name: `lb-ci`
   - Region: Central India
   - SKU: Standard
   - Type: Public
4. Create or select a Standard public IP address named `pip-lb-ci`.
5. Complete the deployment.

### Step 6 — Create the backend pool

1. Open `lb-ci`.
2. Select **Backend pools**.
3. Add a backend pool named `bp-ci-web`.
4. Select `vnet-ci`.
5. Add the NIC IP configurations for `vm-ci-01` and `vm-ci-02`.
6. Save the pool.

### Step 7 — Create the health probe

1. Open **Health probes**.
2. Add a probe named `probe-http`.
3. Protocol: TCP or HTTP.
4. Port: 80.
5. If using HTTP, configure the path `/`.
6. Set an appropriate interval and unhealthy threshold.
7. Save the probe.

An HTTP probe can help verify application-level availability, while a TCP probe checks whether the configured TCP port responds.

### Step 8 — Create the load-balancing rule

1. Open **Load balancing rules**.
2. Add a rule named `rule-http`.
3. Frontend IP: The `pip-lb-ci` frontend.
4. Backend pool: `bp-ci-web`.
5. Protocol: TCP.
6. Frontend port: 80.
7. Backend port: 80.
8. Health probe: `probe-http`.
9. Configure the required session persistence and idle timeout settings.
10. Save the rule.

## 7. Create the Southeast Asia regional Load Balancer

### Step 9 — Repeat the regional configuration

Create `lb-sea` in Southeast Asia.

Configure:

- Standard SKU
- Public frontend IP `pip-lb-sea`
- Backend pool `bp-sea-web` containing `vm-sea-01`
- Health probe on TCP or HTTP port 80
- Load-balancing rule from frontend port 80 to backend port 80

The secondary region has only one VM in this exercise. It provides regional redundancy, but it does not provide VM-level redundancy within Southeast Asia.

## 8. Validate the regional Load Balancers

### Step 10 — Test Central India

1. Open `lb-ci`.
2. Copy its frontend public IP address.
3. Open the IP address in a browser using HTTP.
4. Refresh several times.
5. Verify that the returned page identifies one of the Central India VMs.

Depending on the load-balancing distribution and client connections, responses may not alternate evenly between the two VMs.

### Step 11 — Test Southeast Asia

1. Open `lb-sea`.
2. Copy its frontend public IP address.
3. Open the IP address in a browser using HTTP.
4. Verify that the returned page identifies `vm-sea-01`.

If either test fails, verify the backend pool, health probe status, NSG rules, Windows Firewall and IIS service.

## 9. Create the cross-region Load Balancer

### Step 12 — Create the global Load Balancer

1. Search for **Load balancers**.
2. Select **Create**.
3. Choose the cross-region or global Load Balancer option supported by the current Azure Portal.
4. Configure the global frontend public IP using the supported Global tier and home region.
5. Name the resource `lb-global`.
6. Create the load balancer.

The global Load Balancer's home region is a control and configuration choice; it does not mean the application backend must be located in that region.

### Step 13 — Add regional Load Balancers as backends

1. Open `lb-global`.
2. Select **Backend pools**.
3. Create a backend pool containing the two regional Load Balancers:
   - `lb-ci`
   - `lb-sea`
4. Use the regional Load Balancer resources and frontend configurations supported by the portal.
5. Save the backend pool.

The cross-region Load Balancer requires compatible public regional Load Balancers. Do not attempt to add the individual IIS VMs directly to the global backend pool.

### Step 14 — Configure the global rule and health monitoring

1. Configure the global frontend to use TCP port 80.
2. Configure the backend port consistently with the regional Load Balancer rules.
3. Configure the required health probe and rule settings.
4. Save the configuration.
5. Wait for deployment and verify the status of the global and regional Load Balancers.

Follow the current portal's cross-region Load Balancer workflow for supported backend and probe settings. Global and regional load-balancing configuration options can differ from those of a standalone regional Load Balancer.

## 10. Test the global endpoint

### Step 15 — Validate application access

1. Open `lb-global`.
2. Copy the global public IP address.
3. Open the global IP address in a browser using HTTP.
4. Verify that the request reaches a healthy regional backend.
5. Repeat the test.

The global Load Balancer uses the Azure network to direct traffic to a participating healthy region. Do not expect a single client to receive alternating responses from both regions on every refresh. Regional selection depends on the global load-balancing behavior and health state.

## 11. Demonstrate regional resilience

### Step 16 — Simulate an application failure

Start with a single Central India VM:

1. Stop the IIS service on `vm-ci-01`:

```powershell
Stop-Service W3SVC
```

2. Wait for the regional health probe to detect the failure.
3. Access `lb-ci` again.
4. Confirm that the healthy Central India VM can still serve requests.

Restore IIS:

```powershell
Start-Service W3SVC
```

### Step 17 — Simulate a regional service failure

Because Southeast Asia contains only one VM, stopping IIS there makes its regional backend unhealthy.

1. Connect to `vm-sea-01`.
2. Stop IIS:

```powershell
Stop-Service W3SVC
```

3. Wait for the regional Load Balancer's health status to change.
4. Access the global endpoint and observe the result.
5. Restore IIS:

```powershell
Start-Service W3SVC
```

If necessary, allow time for health probes and global health status to converge before evaluating the result.

**Expected learning outcome:** Students understand that a global load-balancing design can route traffic away from an unhealthy region, provided health monitoring and backend configuration are correct.

This exercise demonstrates application failure and regional backend unavailability. It is not a full simulation of an Azure region outage.

## 12. Troubleshooting checklist

| Problem | What to check |
|---|---|
| IIS page cannot be reached | Verify IIS service, Windows Firewall and NSG rules |
| Regional Load Balancer has no healthy backend | Verify NIC membership and health probe configuration |
| Only one VM appears to respond | Use repeated new requests and understand load-balancing distribution |
| Global endpoint fails | Verify global backend pool, public frontend, rule and regional health |
| Global endpoint appears to use only one region | Check health status and client routing behavior |
| Failover is not immediate | Wait for the configured health probe thresholds and health state propagation |
| VM is reachable directly but not through LB | Check the backend pool, probe, rule and NSG configuration |

## 13. Discussion questions

1. Why do we need a regional Load Balancer in each region?
2. Why can't the cross-region Load Balancer use the IIS VMs directly as its regional backend resources?
3. What is the difference between a regional Load Balancer and a cross-region Load Balancer?
4. What does a health probe actually validate?
5. What happens if one VM fails but the region remains healthy?
6. What happens if all backends in one region become unhealthy?
7. What is the limitation of having only one VM in Southeast Asia?
8. When would Azure Front Door be a better choice than a cross-region Load Balancer?
9. Why is this a Layer 4 load-balancing design rather than a Layer 7 HTTP routing design?

---

# Service Endpoint Policies

Service Endpoint Policies are **not included in this lab**.

They are not required for Azure Front Door Private Link to App Service or for cross-region Load Balancer configuration. Introducing a Storage account solely to demonstrate a service endpoint policy would add resources and complexity without improving either exercise.

Students should understand the distinction:

- **Private Link:** Provides private connectivity to a supported service or origin.
- **Service endpoints:** Extend a VNet identity to supported Azure services over the Azure backbone.
- **Service Endpoint Policies:** Restrict access to selected supported Azure Storage resources through service endpoints.

These are different mechanisms and should not be treated as interchangeable.

---

# Cleanup

Delete the lab resources once validation and discussion are complete.

## Exercise 1 cleanup

1. Delete the Azure Front Door profile.
2. Delete the App Service.
3. Delete the App Service plan if it is no longer needed.
4. Delete `rg-az700-afd-pl` after confirming that it contains only lab resources.

## Exercise 2 cleanup

1. Delete `lb-global`.
2. Delete `lb-ci` and `lb-sea`.
3. Delete the associated public IP addresses if they are not automatically removed.
4. Delete the three virtual machines and their managed disks.
5. Delete `vnet-ci` and `vnet-sea`.
6. Delete `rg-az700-crlb` after verifying that it contains only lab resources.

Confirm that the resources have been removed from the subscription to prevent unnecessary charges.

---

# Microsoft Learn references

1. [Secure your Azure Front Door origin with Private Link](https://learn.microsoft.com/en-us/azure/frontdoor/private-link)
2. [Connect Azure Front Door Premium to an App Service origin](https://learn.microsoft.com/en-us/azure/frontdoor/standard-premium/how-to-enable-private-link-web-app)
3. [Secure traffic to Azure Front Door origins](https://learn.microsoft.com/azure/frontdoor/origin-security)
4. [Global Load Balancer overview](https://learn.microsoft.com/en-us/azure/load-balancer/cross-region-overview)
5. [Azure Load Balancer best practices](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-best-practices)

---

## Completion checklist

- [ ] App Service created and tested.
- [ ] Front Door Premium endpoint created.
- [ ] Private Link origin configured and approved.
- [ ] Direct public access to App Service restricted.
- [ ] Two Central India IIS VMs deployed.
- [ ] One Southeast Asia IIS VM deployed.
- [ ] Both regional Load Balancers tested.
- [ ] Cross-region Load Balancer deployed.
- [ ] Global endpoint tested.
- [ ] Backend health and failure behavior discussed.
- [ ] All lab resources deleted.

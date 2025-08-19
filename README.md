# Azure-Front-To-Private-End-pointed-App-WebService



Got it—here’s a clean, battle-tested path to put Azure Front Door (Standard/Premium) in front of an App Service that’s locked down with a Private Endpoint.

# Steps

1. Create / lock down the App Service

* Create (or use) your Web App.
* Networking ➜ **Public network access: Disabled**.
* (Optional but recommended) Create a **Private Endpoint** for `myapp.azurewebsites.net` (and another for `myapp.scm.azurewebsites.net` if you’ll deploy via Kudu).
* When the PE wizard asks for DNS, link the private DNS zone `privatelink.azurewebsites.net` (and `privatelink.scm.azurewebsites.net` if applicable) to the VNet that needs name resolution (e.g., build agents).
* Confirm the **Private Endpoint connection** shows *Approved*.

2. Provision Azure Front Door (Standard or Premium)

* Create an AFD **Profile** and an **Endpoint** (e.g., `myfrontdoor-xyz.azurefd.net`).
* (Optional) Attach a **WAF policy** now if you plan to use one.

3. Configure the origin (App Service) with Private Link from AFD

* In AFD ➜ **Origin groups** ➜ *+ Add origin group*.

  * Health probe: set a reachable path (e.g., `/healthz`) and method `GET`.
* Inside the origin group ➜ *+ Add an origin*:

  * **Origin type**: App Service.
  * **Origin hostname**: the app’s default hostname `myapp.azurewebsites.net`.
  * **Origin host header**: set to `myapp.azurewebsites.net` (don’t leave it as the AFD endpoint).
  * **Enable Private Link**: ✓
  * **Target sub-resource**: `sites`.
  * Pick the **App Service** resource; select the **region** that matches the app.
  * Save.

4. Approve the Private Link connection on the App Service side

* App Service ➜ Networking ➜ **Private endpoint connections** ➜ you’ll see the **Front Door (Private Link)** request in *Pending*.
* Select it and **Approve**.
* Wait until AFD origin shows **Private Link: Approved / Healthy**.

5. Create the route from AFD endpoint to the origin group

* AFD ➜ **Routes** ➜ *+ Add*.

  * **Domains**: select your `*.azurefd.net` endpoint (and later your custom domain).
  * **Origin group**: the one you created.
  * **Forwarding protocol**: HTTPS only.
  * **Caching**: as needed (often Disabled for apps).
  * **URL rewrite / rules**: as needed.
  * Ensure **Send host header** is `myapp.azurewebsites.net` (matches step 3).
  * Save.

6. Add a custom domain (optional but typical)

* AFD ➜ **Domains** ➜ *+ Add*.

  * Enter `www.contoso.com`.
  * Create a **CNAME** from `www.contoso.com` ➜ `myfrontdoor-xyz.azurefd.net`.
  * Complete DNS TXT validation if prompted.
  * Enable **HTTPS** (Managed certificate).
* Update your **Route** to include the custom domain.

7. Tighten access to only AFD (defense in depth)

* App Service ➜ Networking ➜ **Access restrictions**:

  * Add rule to **Allow traffic from “AzureFrontDoor.Backend” service tag** (for fallback), and keep public access disabled.
  * With AFD Private Link enabled, traffic from AFD to origin uses Private Link; leaving the service tag allow rule is a safe extra, but you can rely solely on Private Link + Public access disabled for strict lock-down.
* Verify there are **no wildcard allows** that re-open public access.

8. Validate end-to-end

* Hit `https://myfrontdoor-xyz.azurefd.net/healthz` and then the app path.
* Hit your **custom domain** once the certificate is active.
* Confirm in App Service **Diagnose and solve problems** that requests are arriving via **Private Link** (you’ll see `PrivateEndpoint` / AFD in connection details).
* Confirm **Public URL** `https://myapp.azurewebsites.net` is no longer reachable from the internet.

---

## Diagram (high-level)

```mermaid
flowchart LR
  user[User] -->|HTTPS| AFD[Azure Front Door Std/Premium]
  AFD -->|Private Link (Microsoft backbone)| Origin[App Service (Private Endpoint)]
  subgraph App Service VNet
    Origin
  end
  classDef c fill:#eef,stroke:#88f
  class AFD,Origin c
```

---

## CLI nuggets (optional)

> App Service: disable public & add PE

```bash
# Disable public access
az webapp update \
  --name myapp \
  --resource-group rg-app \
  --set publicNetworkAccess=Disabled

# Create Private Endpoint for the app (in a VNet/subnet you control)
az network private-endpoint create \
  -g rg-net \
  -n pe-app \
  --vnet-name vnet-app \
  --subnet snet-pe \
  --private-connection-resource-id $(az webapp show -g rg-app -n myapp --query id -o tsv) \
  --group-ids sites \
  --connection-name peconn-app
```

> Private DNS zone links (if your VNet needs name resolution to `*.privatelink.azurewebsites.net`)

```bash
az network private-dns zone create -g rg-net -n privatelink.azurewebsites.net
az network private-dns link vnet create -g rg-net -n link-app \
  -z privatelink.azurewebsites.net --virtual-network vnet-app --registration-enabled false
```

(Front Door Private Link setup is easiest in the portal; the approval flow is GUI-driven today.)

---

## Gotchas & best practices

* **SKU**: Private Link to origins requires **Front Door Standard/Premium** (not Classic).
* **Host header**: Must be the **app’s default hostname** (`*.azurewebsites.net`) unless you’ve added that custom host to the App Service and bound a cert.
* **Health probe path**: Ensure the probe path is **anonymous & returns 200**; otherwise AFD will mark the origin unhealthy.
* **SCM/Kudu**: If you disable public access and need Kudu, create a **second Private Endpoint** for `*.scm.azurewebsites.net`.
* **WAF**: Attach a WAF policy to the AFD endpoint for L7 protection (OWASP, bot mitigation, geo, rate limit).
* **No VNet in AFD**: Front Door’s Private Link is **not** routed through your VNet; it’s a managed, private Microsoft backbone connection to your origin.
* **End-to-end TLS**: Keep HTTPS-only at AFD and App Service; consider HSTS on your custom domain.

If you want, I can tailor this into a one-pager runbook for your team (with your resource names and a checklist).

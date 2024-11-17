+++
title = "Proxmox: Let's Encrypt Certificate with DNS Challenge"
date = 2024-10-30T15:00:00+02:00
description = "Proxmox let's encrypt certificate with DNS challenge Infomaniak"
tags = ['System']
draft = false
+++


## 📢 Introduction

Whether in your homelab or in a production environment, you've likely encountered this message on your first connection to your Proxmox host:

![Self signed alert](img/self_signed_alert.jpg)

By default, the Proxmox web interface comes with a self-signed certificate. Fortunately for us, the latest versions of Proxmox natively support ACME DNS challenges!

In this article, I will explain in detail all the steps necessary to set up a Let's Encrypt certificate using a DNS challenge.

Note that this procedure was carried out using an Infomaniak API token, as my domain is hosted with them. However, the procedure should work for other providers as well; you’ll just need to adapt the settings accordingly.

### 🔍 Prerequisites

3 things to check:

✅ An A record on your local DNS server(s) → Mine points to minastirith.gattabru.si\
✅ Public domain name → In my case, `gattabru.si`\
✅ Compatible registrar → Domain hosted with Infomaniak

### ➕ Pros
- No need to expose ports 80 and 443 to the Internet
- No need for A or CNAME records pointing to your public IP address
- Therefore, enhanced security

### ➖ Cons
- A public domain name is **required**
- Your registrar must support the DNS challenge ([list here](https://go-acme.github.io/lego/dns/))
- Complexity of setup (although, once you understand it, it's fairly straightforward)

## ⚙️ Setup

### 🛠️ ACME Accounts

We'll start by setting up not one but two ACME accounts on our host, here’s why:
- A `staging` account in the `Let's Encrypt V2 Staging` directory, which will be used to validate that our ACME configuration is correct. This allows for debugging and error correction without hitting the rate limit described below, which could otherwise result in account blocking.

- The other is a `produduction` account in the `Let's Encrypt V2` directory. Once configuration is validated, we’ll switch to this account to obtain our final certificate.

{{< alert >}}
The limit is 5 failures per account, per hostname, per hour in the `Let's Encrypt V2` directory.
{{< /alert >}}

To create the first account in the `Let's Encrypt V2 Staging` directory, go through the following menus:
1. Datacenter
2. ACME
3. Click `Add` to add a new account
4. Fill in the information and be sure to choose the `Let's Encrypt V2 Staging` directory. Don’t forget to accept the Terms of Service (TOS).

![Adding ACME stag. account](img/add_acme_account_stag.jpg)

Repeat steps 3 and 4 for the `Let's Encrypt V2` directory:
![Adding ACME prod. account](img/add_acme_account_prod.jpg)

With both accounts created, we’ll now create an API token from our registrar to enable the DNS challenge.

### 🪙 API Token (Infomaniak)

This step will vary depending on your domain provider. Since my domain is with Infomaniak, I’ll show you how it’s done on their platform. Generally, a quick Google search will provide guidance:
- [Gandi](https://api.gandi.net/docs/authentication/) 
- [OVH](https://help.ovhcloud.com/csm/en-api-getting-started-ovhcloud-api?id=kb_article_view&sysparm_article=KB0042777)

If your domain is with Infomaniak like mine, log into your manager and create a new token. Go to the top left in the `Users and Profile` section:
![Infomaniak API token](img/infomaniak_api_token1.jpg)

Next, select `My profile` > `Developer` > puis finalement `API Tokens` to access your token management console. In this console, we’ll create a new token:

![Infomaniak API token](img/infomaniak_api_token2.jpg)

Fill in the required information, then create your token. Be sure to limit the token scope to `domain - Produits Noms de domaine`. It’s strongly recommended not to set the scope to "All" for obvious security reasons. Think of the consequences if your token were to be compromised.

![Infomaniak API token](img/infomaniak_api_token3.jpg)

Your token is now created:

![Infomaniak API token](img/infomaniak_api_token4.jpg)

{{< alert >}}
Once created, copy your token and store it securely, such as in a password manager. 
{{< /alert >}}

As noted, you won’t be able to view the token again once you close this page. If you forget to copy it, you can always delete it and generate a new one.

Now that we have our token, limited to the required scope, we can return to our Proxmox host to continue configuration.

### 🔧 Challenge plugin

In the same `ACME` menu, go to the `Challenge Plugin` section and click `Add` to add a new plugin:
![Adding ACME dns plugin](img/add_acme_dns_plugin.jpg)

Some explanations:
- `Plugin ID` is the name you’d like to give to your plugin. I recommend keeping it consistent, without spaces.

- `Validation Delay` is the time in seconds between creating your DNS record via the API and when the ACME provider is asked to validate it. I recommend keeping this parameter default, adjusting it later if the validation period is too short.

- `DNS API` : Who your registrar is. Depending on your selection, the section below will request different information. *Example with Cloudflare:*
![ACME dns Cloudflare](img/acme_dns_cloudflare.jpg "We can clearly see that we don't have the same parameters as Infomaniak")

- `API DATA` : Your API parameters, such as your token, TTL, etc.

{{< alert >}}
The `API DATA` parameters depend on your registrar and vary between providers.
{{< /alert >}}

Since my registrar is Infomaniak, I’ll consult the documentation directly on the [go-acme.github.io website](https://go-acme.github.io/lego/dns/infomaniak/index.html):
![ACME Infomaniak config](img/acme_infomaniak_config.jpg)

The parameters listed under `Credentials` are mandatory. The others under `Additional Configuration` are optional but very helpful for validation (we’ll cover this below).

{{< alert >}}
**CAUTION:** If your domain is with Infomaniak, the `INFOMANIAK_ACCESS_TOKEN` parameter won’t work with Proxmox. 

Replace it with `INFOMANIAK_API_TOKEN` (as shown below); otherwise, you’ll encounter [the error described later in this article](#-infomaniak-api-token).
{{< /alert >}}

Now that we know all our parameters, we can enter them in the `API DATA` field on our Proxmox host:

![ACME Infomaniak good config](img/acme_dns_plugin_good_config.jpg)

As shown, I’ve replaced `INFOMANIAK_ACCESS_TOKEN` with `INFOMANIAK_API_TOKEN`. There’s no need to enclose your token in quotes.

I've also added `INFOMANIAK_PROPAGATION_TIMEOUT` and `INFOMANIAK_TTL`. both set to 300 seconds. This means validation will timeout after 5 minutes if it fails, and the TTL for our record will also be 5 minutes.

The DNS challenge configuration is complete, and we can now generate our certificates!

### 📜 Certificates creation

If you’re familiar with the DNS challenge, you can skip to the [production certificate generation](#-production-certificate).

Otherwise, I suggest testing with the **staging** account to avoid hitting the [the rate limits mentioned above](#-acme-accounts).\
Once the staging configuration is validated, we’ll switch to the **production** account.

#### 🧪 Staging certificate

For this, go to the `Certificates` tab of your host (`Hostname` > `System` > `Certificates`).\
Then switch the ACME account to Staging:

![Proxmox Certificates ACME account](img/proxmox_certificates_acme_account.jpg)

We can now create a new certificate via the `Add` button on this page:

![Proxmox Certificates ACME Staging](img/proxmox_certificates_acme_staging.jpg)

1. Specify a `Challenge Type` of DNS, select our previously created plugin, and enter the name of our DNS record.

2. Once the fields are filled out, finalize the certificate creation with the `Create` button, then order it with `Order Certificates Now`.

If everything goes well, you should see the following result:

![Proxmox ACME Staging Success](img/proxmox_certificates_staging_success.jpg)

The Proxmox web interface will then reload with the new certificate you generated:

{{< alert >}}
At this stage, it’s still normal to see a warning message. The `Let's Encrypt V2 Staging` directory does not issue certificates from a trusted authority recognized by your browser.
{{< /alert >}}

![Proxmox ACME Staging Certificate](img/proxmox_certificates_staging_check_certificate.jpg)

#### 🏭 Production certificate

With our configuration validated, we can generate our final certificate. Switch to your `ACME_PROD`account to use the `Let's Encrypt V2` directory, then order your certificate via the `Order Certificates Now` button:

![Proxmox ACME Prod](img/proxmox_certificates_acme_prod.jpg)

Again, if everything proceeds smoothly, a temporary `_acme-challenge` TXT record should appear in your DNS zone, with a TTL of 5 minutes as specified in the `INFOMANIAK_TTL` parameter of our `ACME DNS Plugin` :

![Proxmox ACME Prod](img/infomaniak_acme_dns_record.jpg)

The result of your request should be similar to that in the `Let's Encrypt V2 Staging` directory:

![Proxmox ACME Staging Success](img/proxmox_certificates_prod_success.jpg)

Finally, the Proxmox web interface should reload with your new Let's Encrypt certificate in place:

![Proxmox ACME Prod Certificate](img/proxmox_r3_certificate.jpg)

## 🎬 Conclusion

We’ve covered the steps to secure the Proxmox web interface with a signed certificate.

Certificate renewal will occur automatically by default, 30 days before expiration. In case of renewal issues, Let’s Encrypt will email you at the addresses you provided in your ACME accounts.

If you encounter any issues, feel free to check the [Troubleshooting](#problèmes-et-résolutions) section. Otherwise, there's only one thing left to do... Party tiiiime!!


![Celebrate](/img/posts/proxmox-dns-challenge/celebrate.webp)

## ❌ Troubleshooting

### 💡 Infomaniak API Token

You followed the documentation on the [go-acme.github.io website](https://go-acme.github.io/lego/dns/infomaniak/index.html) for your domain at Infomaniak but received the following error message:

```shell
Please provide a valid Infomaniak API token in variable INFOMANIAK_API_TOKEN
 ```

![Proxmox Certificate Staging Failed](img/proxmox_certificates_staging_failed.jpg)

And the configuration of your `ACME DNS Plugin` looks like this:

![ACME Infomaniak wrong config](img/acme_dns_plugin_wrong_config.jpg)

The answer is in the error message. It indicates that it expects a variable with the value `INFOMANIAK_API_TOKEN`. For some reason, on Proxmox, this parameter needs to be named `INFOMANIAK_API_TOKEN` instead of `INFOMANIAK_ACCESS_TOKEN`.

In the end, it’s a simple fix. Go back to your `ACME DNS Plugin` configuration and change the parameter name from `INFOMANIAK_ACCESS_TOKEN` to `INFOMANIAK_API_TOKEN`:

![ACME Infomaniak correct config](img/acme_dns_plugin_good_config.jpg)

### 💡 Incorrect Certificate

It may happen that during the [production certificate generation](#-production-certificate), your browser continues to display a warning regarding your certificate, even though it is indeed the Let's Encrypt certificate from the `Let's Encrypt V2` directory:

![Proxmox alert good certificate](img/alert_good_certificate.jpg)

First, check that your production certificate has indeed been applied.

If it has, there’s a high chance that your browser’s cache is causing the issue. Try restarting your browser, and if that doesn’t work, clear its cache and restart it again.

This should resolve the issue 😉!

### 💡 ACME Validation

I haven't encountered this with Proxmox yet, but it can happen.

Your `ACME DNS Plugin` configuration is correct since there are no specific errors during the certificate request, yet the ACME validation fails due to a timeout.

Here are a few troubleshooting tips:
- Adjust the `Validation Delay` and/or `INFOMANIAK_PROPAGATION_TIMEOUT` parameters in your `ACME DNS Plugin`:
  ![ACME DNS Plugin 60](img/acme_dns_plugin_60.jpg)
- Temporarily change your host’s DNS settings:
  ![Proxmox temp. DNS config.](img/proxmox_dns_config.jpg)

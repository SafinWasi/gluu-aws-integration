# SCIM support on Gluu Solo for external SSO

  

This article will guide you through setting up SCIM on your Gluu Solo instance, in order to create/modify/delete users that can seamlessly use Single-Sign-On (SSO) on an external application. For this example, we will be using Gluu Solo 4.4 with Amazon Web Services as the external application.

  

## Table of contents

1. [Requirements](#requirements)
2. [Preparing the Gluu Server](#preparing-the-gluu-server)
3. [Configuring AWS to accept SAML requests](#configuring-aws-to-accept-saml-requests)



  

## Requirements

1. A working Gluu installation with Shibboleth IDP and the SCIM API installed and enabled

2. An AWS account with administrative priviledges

3. An email address associated with an AWS account that you want to SSO with

4. A way to craft SCIM requests for an endpoint protected by [Oauth 2.0](https://datatracker.ietf.org/doc/html/rfc6749). If you are using Java, we recommend using [scim-client](https://github.com/GluuFederation/scim/tree/master/scim-client).

  

## Preparing the Gluu Server

To begin, ensure the Shibboleth IDP and SCIM are installed and enabled on your Gluu installation. During a fresh install, you can choose both of these to be installed. For an existing install, log onto your Gluu container and refer to the following commands.

```bash

cd /install/community-edition-setup/

```

For Shibboleth:

  

```bash

./setup.py --install-shib

```

For SCIM:

```bash

./setup.py --install-scim

```

Next, we need to enable the SCIM API. Log onto the oxTrust GUI, go to `Organization` > `Organization Configuration` and check the tick mark for `SCIM Support`

  

![oxTrust Panel](https://gluu.org/docs/gluu-server/4.4/img/scim/enable-scim.png)

After that, navigate to `Configuration` > `JSON Configuration` > `OxTrust Configuration`, locate the `Scim Properties` section (use Ctrl + F and search for `Scim`) and set the protection mode to `Oauth2`. Then save the configuration at the bottom of the page.

- This article assumes that we are using the Oauth2 protection scheme. For testing purposes, you may use the `TEST` scheme, which is unprotected and can be queried without any authentication; however, this is not recommended in a production environment.

![Protection scheme]()

Our Gluu server is now ready to accept authorized SCIM requests.

## Configuring AWS to accept SAML requests
1. Navigate to `https://<hostname>/idp/shibboleth` on your Gluu server and download the page as an XML file.
2. Log into the AWS Management Console with an administrator account. Find and navigate to the IAM module.
3. Create an IDP on your AWS account with the following steps:
    - On the left hand pane, choose `Identity Providers`
    - Click on `Add Provider`
    - Provider type: `SAML`
    - Provider name: `Shibboleth`
    - Metadata document: Upload the XML file you downloaded.
    - Click `Add Provider` at the bottom.

![AWS IAM]()

4. Create a role associated with the new IDP with the following steps:
    - On the left hand pane, choose `Roles`
    - Trusted Entity Type: `SAML 2.0 Federation`
    - SAML 2.0 Based Provider: `Shibboleth` (or what you chose for provider name previously) from the drop down menu.
    - Tick `Allow programmatic and AWS Management Console Access`
    - The rest of the fields will autofill.
    - Click `Next`. Here you may choose policies to associate with this role. Refer to [AWS documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_saml.html) for further information. For this example, we are not choosing any policies, and instead clicking `Next`.
    - Role name: `Shibboleth-Dev`
    - Role description: Choose a meaningful description. 
    - Verify that the Role Trust JSON looks like this (the X's will be some unique string):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRoleWithSAML",
            "Principal": {
                "Federated": "arn:aws:iam::xxxxxxxxxxxx:saml-provider/Shibboleth"
            },
            "Condition": {
                "StringEquals": {
                    "SAML:aud": [
                        "https://signin.aws.amazon.com/saml"
                    ]
                }
            }
        }
    ]
}
```
Finally, click on `Create Role`
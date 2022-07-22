# SCIM support on Gluu Solo for external SSO

  

This article will guide you through setting up SCIM on your Gluu Solo instance, in order to create/modify/delete users that can seamlessly use Single-Sign-On (SSO) on an external application. For this example, we will be using Gluu Solo 4.4 with Amazon Web Services as the external application.

  

## Table of contents

1. [Requirements](#requirements)

2. [Preparing the Gluu Server](#preparing-the-gluu-server)

  

## Requirements

1. A working Gluu installation with Shibboleth IDP and the SCIM API installed and enabled

2. An AWS account with administrative priviledges

3. An email address associated with an AWS account that you want to SSO with

4. A way to craft SCIM requests for an endpoint protected by [UMA](https://docs.kantarainitiative.org/uma/wg/rec-oauth-uma-grant-2.0.html). If you are using Java, we recommend using [scim-client](https://github.com/GluuFederation/scim/tree/master/scim-client).

  

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

After that, navigate to `Configuration` > `JSON Configuration` > `OxTrust Configuration`, locate the `Scim Properties` section (use Ctrl + F and search for `Scim`) and set the protection mode to `UMA`. Then save the configuration at the bottom of the page.

- This article assumes that we are using the UMA protection scheme. For testing purposes, you may use the `TEST` scheme; however, this is not recommended in a production environment.

![Protection scheme]()
Finally, we need to activate the UMA SCIM custom script. Navigate to `Configuration` > `Other Custom Scripts`, and in the tab for `UMA RPT policies` check `Enabled" for the script labeled "scim_access_policy`. Finally, click the `Update` button.
![UMA Custom](https://gluu.org/docs/gluu-server/4.4/img/scim/enable_uma.png)

Our Gluu server is now ready to accept authorized SCIM requests.
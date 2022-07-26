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

![Protection scheme](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/protection-mode.png?raw=true)

Our Gluu server is now ready to accept authorized SCIM requests.

Next, we need to add two custom attribute types to our Gluu LDAP.

1. SSH into your Gluu server, and open `/opt/opendj/config/schema/77-customAttributes.ldif` with your favorite editor.
2. Add the following `attributeTypes`, and then modify your `gluuCustomPerson objectClass` to add those two `attributeTypes`. Here is a part of our schema doc after modification:

```
...
attributeTypes: ( 1.3.6.1.4.1.48710.1.3.1003 NAME 'RoleEntitlement'
  EQUALITY caseIgnoreMatch
  SUBSTR caseIgnoreSubstringsMatch
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
  X-ORIGIN 'Gluu - AWS Assume Role' )
attributeTypes: ( 1.3.6.1.4.1.48710.1.3.1004 NAME 'RoleSessionName'
  EQUALITY caseIgnoreMatch
  SUBSTR caseIgnoreSubstringsMatch
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
  X-ORIGIN 'Gluu - AWS Assume Role Session Name' )
objectClasses: ( 1.3.6.1.4.1.48710.1.4.101 NAME 'gluuCustomPerson'
  SUP ( top )
  AUXILIARY
  MAY ( telephoneNumber $ mobile $ RoleEntitlement $ RoleSessionName )
  X-ORIGIN 'Gluu - Custom persom objectclass' )
```

**Warning**: Do NOT replace your `objectClasses` with the example above. Simply add `" $ RoleEntitlement "` and `" $ RoleSessionName "` without the quotes to the `MAY` field, as shown in the example. Be careful of spacing; there must be 2 spaces before and 1 after every entry (i.e. DESC), or your custom schema will fail to load properly because of a validation error. You cannot have line spaces between attributeTypes: or objectClasses:. This will cause failure in schema. Please check the error logs in `/opt/opendj/logs/errors` if you are experiencing issues with adding custom schema. In addition, make sure the attributetype LDAP ID number is unique. 

3. Save the file, and [restart](https://gluu.org/docs/gluu-server/4.4/operation/services/#restart) the opendj service.
4. Now, we need to register two attributes in oxTrust corresponding to the LDAP attributes we just added. Log onto the web GUI, navigate to `Configuration` > `Attributes` > `Register Attribute`.
5. For the first attribute:
    - Name: `RoleEntitlement`
    - SAML1 URI: `https://aws.amazon.com/SAML/Attributes/Role`
    - SAML2 URI: `https://aws.amazon.com/SAML/Attributes/Role`
    - Display Name: `RoleEntitlement`
    - Type: Text
    - Edit type: `admin`
    - View type: `admin`, `user` (use Ctrl+click to select multiple)
    - Usage type: Not Defined
    - Multivalued: False
    - Tick the box next to `Include in SCIM extension:`
    - Description: `Custom attribute for Amazon AWS SSO`
    - Status: Active
    - Click `Register`

    ![RoleEntitlement]()

6. For the second attribute:
    - Name: `RoleSessionName`
    - SAML1 URI: `https://aws.amazon.com/SAML/Attributes/RoleSessionName`
    - SAML2 URI: `https://aws.amazon.com/SAML/Attributes/RoleSessionName`
    - Display Name: `RoleSessionName`

    Every other field will have the same values as the first attribute.

    ![RoleSessionName]()

7. If everything is successful, these two attributes will successfully be saved and will show up under the `Attributes` tab. If you get an error saying the attributes don't exist, there was probably an error in your custom schema doc. Refer to the logs for more information.



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

    ![AWS IAM](https://raw.githubusercontent.com/SafinWasi/gluu-aws-integration/devel/assets/aws-iam.png)

4. Create a role associated with the new IDP with the following steps:
    - On the left hand pane, choose `Roles`
    - Trusted Entity Type: `SAML 2.0 Federation`
    - SAML 2.0 Based Provider: `Shibboleth` (or what you chose for provider name previously) from the drop down menu.
    - Tick `Allow programmatic and AWS Management Console Access`
    - The rest of the fields will autofill.

    ![AWS Role](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/aws-role.png?raw=true)

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
    - Finally, click on `Create Role`.

AWS is now ready to accept inbound SAML.

## Create Trust Relationship in Gluu Server

Now we need to create an outbound SAML trust relationship from the Gluu Server to AWS. 

1. Log onto the web GUI
2. Navigate to `SAML` > `Add Trust Relationship`
3. Use the following values:
    - Display Name: `Amazon AWS`
    - Description: `external SP / File method`
    - Entity Type: `Single SP` from the dropdown
    - Metadata location: `URI` from the dropdown
    - SP metadata URL: `https://signin.aws.amazon.com/static/saml-metadata.xml`
    - Check the box next to `Configure relying party` and an additional box will pop up.
        - Click on `SAML2SSO` and click Add; then expand the SAML2 SSO Profile menu.
        ![rp-config]()

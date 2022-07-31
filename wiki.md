# SCIM support on Gluu Solo for external SSO

  

This article will guide you through setting up SCIM on your Gluu Solo instance, in order to create/modify/delete users that can seamlessly use Single-Sign-On (SSO) on an external application. For this example, we will be using Gluu Solo 4.4 with Amazon Web Services as the external application.

  

## Table of contents

1. [Requirements](#requirements)
2. [Preparing the Gluu Server](#preparing-the-gluu-server)
3. [Configuring AWS to accept SAML requests](#configuring-aws-to-accept-saml-requests)
4. [Creating a Trust relationship in Gluu Server](#create-trust-relationship-in-gluu-server)
5. [Creating a test user manually](#create-the-test-user-manually)
6. [Setup for SCIM](#setup-for-scim)
7. [Setting up scim-client](#setting-up-scim-client)


  

## Requirements

1. A working Gluu installation with Shibboleth IDP and the SCIM API installed and enabled

2. An AWS account with administrative priviledges

3. An email address associated with an AWS account that you want to SSO with

4. A way to craft SCIM requests for an endpoint protected by [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749). If you are using Java, we recommend using [scim-client](https://github.com/GluuFederation/scim/tree/master/scim-client).

  

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

After that, navigate to `Configuration` > `JSON Configuration` > `OxTrust Configuration`, locate the `Scim Properties` section (use Ctrl + F and search for `Scim`) and set the protection mode to `OAUTH`. Then save the configuration at the bottom of the page.

- This article assumes that we are using the OAuth2 protection scheme. For testing purposes, you may use the `TEST` scheme, which is unprotected and can be queried without any authentication; however, this is not recommended in a production environment.

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

**Warning**: Do NOT replace your `objectClasses` with the example above. Simply add `" $ RoleEntitlement "` and `" $ RoleSessionName "` without the quotes to the `MAY` field, as shown in the example. Be careful of spacing; there must be 2 spaces before and 1 after every entry (i.e. DESC), or your custom schema will fail to load properly because of a validation error. You cannot have line spaces between attributeTypes: or objectClasses:. This will cause failure in schema. Please check the error logs in `/opt/opendj/logs/errors` if you are experiencing issues with adding custom schema. In addition, make sure the attributeTypes LDAP ID number is unique. 

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

        ![RoleEntitlement](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/roleentitlement.png?raw=true)

6. For the second attribute:
    - Name: `RoleSessionName`
    - SAML1 URI: `https://aws.amazon.com/SAML/Attributes/RoleSessionName`
    - SAML2 URI: `https://aws.amazon.com/SAML/Attributes/RoleSessionName`
    - Display Name: `RoleSessionName`

    Every other field will have the same values as the first attribute.

    ![RoleSessionName](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/roleSessionName.png?raw=true)

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
        ![rp-config](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/rp-config.png?raw=true)

        - Use the following values:
        - `includeAtributeStatemen`: yes
        - `assertionLifetime`: 300000
        - `signResponses`: conditional
        - `signAssertions`: never
        - `signRequests`: conditional
        - `encryptAssertions`: never
        - `encryptNameIds`: never
        - Click `Save`
    - On the right hand side, under `Release additional attributes`, there is a list of attributes that can be released to this relationship. From those, choose the following:
        - From `gluuPerson`, click on `Email` and `Username`
        - From `gluuCustomPerson`, click on `RoleEntitlement` and `RoleSessionName`
            ![aws-trust](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/aws-trust.png?raw=true)
4. Save this trust relationship. It will take some time to fully load, so please wait.

## Create the test user manually

Now we will create a test user on the Gluu server to check whether the outbound SAML works. Later, we will do this via SCIM requests. This user needs to have an email that is authorized to access the AWS account we want to sign on to. For this example, we will use dummy details. You will want to use your own details. 

First, we need to get some values from our AWS account. Log onto AWS.

1. For the first value, navigate to `Roles` and click on the role you created for the SAML relationship. Copy the ARN, which for us is in the format `arn:aws:iam::XXXXXXXXXXXX:role/Shibboleth-Dev`, with numbers replacing the Xs. Note this down.

    ![role-arn](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/role-arn.png?raw=true)

2. For the second value, navigate to `Identity providers` and click on the provider you created. Copy the ARN, which should be in a similar format. For us it is `arn:aws:iam::XXXXXXXXXXXX:saml-provider/Shibboleth`, with numbers replacing the Xs. Note this down as well.

    ![iam-arn](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/iam-arn.png?raw=true)

3. From the oxTrust GUI, navigate to `Users` > `Add person`
4. Our example values are as follows:
    - `Username`: Alice
    - `First Name`: Bob
    - `Display Name`: Bob
    - `Last Name`: Alice
    - `Email`: bob@alice.example
    - `Password`: any password
    - `Confirm Password`: repeat the password
    - `User Status`: Active
    - From the right hand side, click on the `gluuCustomPerson` class and click on the two custom attributes. They will show up as entry fields in the form to the left.
    - `RoleEntitlement`: The first value and the second value from steps 1 and 2, separated by a comma. For us it is `arn:aws:iam::XXXXXXXXXXXX:role/Shibboleth-Dev,arn:aws:iam::XXXXXXXXXXXX:saml-provider/Shibboleth`.
    - RoleSessionName: The email address that will be used to log onto AWS. We will use a dummy value, `bob@alice.example`.

    ![new-user](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/new-user.png?raw=true)

5. Click Add. Our new user should look like this:

    ![new-user-done](https://github.com/SafinWasi/gluu-aws-integration/blob/devel/assets/new-user-done.png?raw=true)

## Testing SSO

Before using SCIM, we recommend testing out the SAML flow to SSO onto Amazon. To do this, simply visit `https://<hostname>/idp/profile/SAML2/Unsolicited/SSO?providerId=urn:amazon:webservices` and use your new Gluu user. You should seamlessly be able to log onto AWS.

## Setup for SCIM
Communicating with a SCIM endpoint that is protected by OAuth is not a simple task. There are several registration and authentication steps involved because of security concerns. The [scim-client](https://github.com/GluuFederation/scim/tree/master/scim-client) Java library simplifies the actual request and response step, but some preparation is required. For this example, we will be using private key authentication using OpenID. In order to do this we need the following things:

1. An RSA key pair and an associated certificate for our client and the Gluu server to communicate over.
2. A [Java keystore](https://docs.oracle.com/cd/E19509-01/820-3503/ggffo/index.html) JKS file that contains the private key and the associated certificate (or chain).
3. The public key of the key pair in [JSON web key](https://datatracker.ietf.org/doc/html/rfc7517) format, preferably saved to a file. 
4. The SSL certificate of our Gluu server, which can be found at `/opt/gluu-server/etc/certs/httpd.crt`. You can obtain this by using `scp`.

### Steps involving a SCIM query
0. If this is the first time a query is being made, the requesting party registers a new client with the Gluu server using OpenID connect [dynamic client registration](https://openid.net/specs/openid-connect-registration-1_0.html). The public key for the client is uploaded to the server at this time. The requesting party obtains a Client ID, which will be used for subsequent steps.
1. The client requests an OAuth access token using the Client ID and its private key, and obtains an access token from the token endpoint.
2. This access token is used to make SCIM queries.

Steps 1 and 2 can be automated by `scim-client`. The manual preparation is needed for step 0.

### Generating keys
You will need Java 11 (and keytool) along with OpenSSL installed on your machine. Run the following command to generate a private key; enter a passcode when prompted and remember it.
```
openssl genrsa -des3 -out private-key.pem 2048
```
Next, generate the associated public key with the following command:
```
openssl rsa -in private-key.pem -pubout -out public-key.pem
```
It will ask you for the password for the private key; enter when prompted.
Finally, generate a self-signed certificate for the key pair:
```
openssl req -new -x509 -key private-key.pem -out cert.pem -days 360
```
You will be prompted for the private key password and some distinguished values necessary for generating a certificate. In a test environment you may use dummy values. However, in a production environment you will want to use appropriate values.

You should now have three files:
- private-key.pem
- public-key.pem
- cert.pem

### Creating new Java keystore
First we need to concatenate the private key and certificate to one file.
```
cat private-key.pem cert.pem > import.pem
```
Next, we need to convert this file to a PKCS12 keystore.
```
openssl pkcs12 -export -in import.pem -name bob > server.p12
```
Replace `bob` with some name you can remember. It will prompt you for the private key password, then a new password for the PKCS12 keystore.
Finally, we use keytool to create a new Java keystore and import:
```
keytool -importkeystore -srckeystore server.p12 -destkeystore client-key.jks -srcstoretype pkcs12 -alias bob
```
For `alias`, choose whatever name you originally chose for the PKCS12 keystore. For us it is still `bob` and so that's what we chose.

### Creating a JSON web key set
1. Visit https://russelldavies.github.io/jwk-creator/.
2. Fill in the following details:
    - Public Key Use: Signing
    - Algorithm: RS256
    - Key ID: The alias you chose. For us it is `bob`.
    - PEM encoded key: The public key you generated, from `public-key.pem`.
3. Click `Convert`,  and save the output on the right to a file. Name it `client-key.jwks`.

At this point you should have the following files:
- `client-key.jks`
- `client-key.jwks`

If you're ready with these two files, proceed to `Dynamic Client Registration`

### Dynamic Client Registration using OpenID connect
For your convenience, I have created a Python script for you which can automate this process. Visit https://github.com/SafinWasi/gluu_openid_python and follow the instructions in the README. Once it's done, you should have a `client.json` file that contains the Client ID and secret. Do not share this information.

If you wish to do this manually, please refer to [Dynamic Client Registration](https://openid.net/specs/openid-connect-registration-1_0.html).

### Allowing the new client access to SCIM
By default, the new client does not have access to SCIM scopes. To allow this:
- Login to oxTrust and locate the client created for interaction with SCIM. This should be easy to do by searching for the client name you provided.
- Click the "Add Scope" button.
- Tick the scopes prefixed by "https://gluu.org/scim". The kind of access every scope grants is displayed on the right column. It may be helpful to sort the scopes list by name.

You may add scopes to the client as you need. For a full list, visit `Scopes` in oxTrust. For our example, we will add `https://gluu.org/scim/users.read`, which is for querying and searching user resources.

## Setting up scim-client

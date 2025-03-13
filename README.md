# Flutter iOS app Deployment through Github Actions

# Action secrets

Need these following github actions secrets

- `APPSTORE_API_ISSUER_ID`
- `APPSTORE_API_KEY`
- `APPSTORE_API_KEY_BASE64`
- `BUILD_CERTIFICATE_BASE64`
- `P12_PASSWORD`
- `BUILD_PROVISIONING_PROFILE_BASE64`
- `KEYCHAIN_PASSWORD`

# Setup

### required tools
- openssl - v3.*

## Generate the following files - fileformat

### 1. Development/Distribution certificate - .p1

#### Certificate Signing Request (CSR)

Generating a Certificate Signing Request (CSR) to upload to app store to generate the certificate, [check this](https://www.geeksforgeeks.org/how-to-generate-a-csr-certificate-signing-request-in-linux/)

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout Private_Key.key -out My_Csr.csr
```
<details>
<summary>Explanation</summary>

 **Options**      | **Description**                                            
------------------|------------------------------------------------------------
 -new             | New request
 -newkey rsa:2048 | To create a 2048-bit RSA key
 -nodes           | This is used as it doesn’t encrypt the key  
 -keyout          | Specifying filename to send the key to the sample.com.csr
 -out             | Specifying file name to write to CSR

You will be prompted to enter information (in order) such as:
 **Requested Information** | **Description**                                                                                                 
 ---------------------------|--------------------------------------------------
 Country Name               | Two-letter abbreviation for the country you reside in
 State or Province Name     | Full name of the state from where your organization operates
 Locality Name              | Name of the city where the organization operates from             
 Organization Name          | Name of Organization. If registered as an individual, enter the name of the person requesting the certificate.
 Organizational Unit Name   | Section or sector in whichthe organization operates
 Common Name                | The domain name to whom you’re purchasing an SSL certificate.It should be a Fully Qualified Domain Name                                       
 Email Address **\***             | A email address associated with app store connect
 A challenge password       | Password to be sent with your certificate request
  **_\* - required_**
</details>

#### 1.1 Distribution Certificate (.cer)

Go to apple developer.
Navigate to the Certificates, Identifiers & Profiles and goto [Certificates](https://developer.apple.com/account/resources/certificates/list) section 

Click on the ![Add Icon](./icons/ic_add.svg) icons to Create a New Certificate. Select **Apple Distribution** and continue

Upload the Certificate Signing Request (CSR) you just generate and continue

Download and save the Distribution certificate _distribution.cer_

Notice that you have downloaded a .cer file, but we need a .p12 file to proceed. We'll now convert the .cer file to .p12 file using the **Private_Key.key** we generated with CSR

#### 1.2 Distribution Certificate .cer to .pem

First we need to convert the .cer file to a .pem file
```bash
openssl x509 -in distribution.cer -inform DER -out distribution.pem -outform PEM
```
#### 1.3 Apple's Worldwide developer cert

now that we got the _distribution.pem_ file we'll use the Private_Key.key file to generate the .p12 file. Found on [Simon Dalvai's blog](https://simondalvai.org/blog/dev-cert-linux/)

Download Apple's Worldwide developer cert from Apple's certificates website and convert it to pem:

_Note: Here Worldwide Developer Relations - G4 (expiring 12/10/2030) is used._

```bash
wget https://www.apple.com/certificateauthority/AppleWWDRCAG4.cer
```
and convert it to pem
```bash
openssl x509 -in AppleWWDRCAG4.cer -inform DER -out AppleWWDRCAG4.pem -outf
```

#### 1.4 Distribution Certificate .pem to .p12
Convert your cert plus Apple's cert to p12 format (choose a password for the .p12).
Note: use -legacy if using opensssl v3.x . Found on [StackOverflow](https://stackoverflow.com/questions/70431528/mac-verification-failed-during-pkcs12-import-wrong-password-azure-devops)

```bash
openssl pkcs12 -export -legacy -out distribution.p12 -inkey Private_Key.key -in distribution.pem -certfile AppleWWDRCAG4.pem
```
now we have the .p12 file, let's proceed to create the Provisioning profile


### 2. Provisioning profile - .mobileprovision

Go to apple developer. 
Navigate to the Certificates, Identifiers & Profiles and goto [Profiles](https://developer.apple.com/account/resources/profiles/list) section

Click on the ![Add Icon](./icons/ic_add.svg) icons to Register a New Provisioning Profile. Select **App Store Connect** under Distribution and continue

Select an App ID (BundleID) from the drop down and continue

Select the certificates you just created to include in this provisioning profile and continue.

Give a name to the profile to identify and continue

Download and save the Provisioning profile _<given_name>.mobileprovision_


### 3. App store connect api key - .p8

Go to app store connect.
Navigate to the Users and Access then to [Integrations](https://appstoreconnect.apple.com/access/integrations/api) section




## Convert the generated files to `base64` and upload to secrets

#### For Development/Distribution certificate - .p12
```bash
base64 dist_certificate.p12
```

save the file to secret `BUILD_CERTIFICATE_BASE64`

save the Certificate password to `P12_PASSWORD`

---

#### For Provisioning profile - .mobileprovision
```bash
base64 provisioning_profile.mobileprovision
```

save the file to secret `BUILD_PROVISIONING_PROFILE_BASE64`


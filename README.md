# Flutter iOS app Deployment through Github Actions
![Flutter](https://img.shields.io/badge/Flutter-%2302569B.svg?style=for-the-badge&logo=Flutter&logoColor=white)
![iOS](https://img.shields.io/badge/iOS-000000?style=for-the-badge&logo=ios&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)

## Table of Contents
- [Action Secrets](#action-secrets)
- [Key Generations](#key-generations)
  - [Generating Certificates](#generate-the-following-files---fileformat)
  - [Provisioning Profile](#2-provisioning-profile---mobileprovision)
  - [App Store Connect API Key](#3-app-store-connect-api-key---p8)
  - [Store GitHub Secrets](#convert-the-generated-files-to-base64-and-upload-to-secrets)
- [Xcode Project Setup](#xcode-project-setup)
- [Workflow](#workflow-file)


# Action secrets

Need these following github actions secrets

- `BUILD_CERTIFICATE_BASE64`
- `P12_PASSWORD`
- `BUILD_PROVISIONING_PROFILE_BASE64`
- `APPSTORE_API_KEY_BASE64`
- `APPSTORE_API_ISSUER_ID`
- `APPSTORE_API_KEY`
- `KEYCHAIN_PASSWORD`

# Key Generations

## Required Tools
- openssl - v3.*

## Generate the following files - fileformat

### 1. Development/Distribution certificate - .p12

#### Certificate Signing Request (CSR)

Generating a Certificate Signing Request (CSR) to upload to app store to generate the certificate, [check this](https://www.geeksforgeeks.org/how-to-generate-a-csr-certificate-signing-request-in-linux/)

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout Private_Key.key -out My_Csr.csr
```

>_NOTICE: Give your appstore email address and the Challenge password which will be used as **KEYCHAIN_PASSWORD** in next steps_

<details>
<summary>Explanation</summary>

 | **Options**      | **Description**                                           |
 | ---------------- | --------------------------------------------------------- |
 | -new             | New request                                               |
 | -newkey rsa:2048 | To create a 2048-bit RSA key                              |
 | -nodes           | This is used as it doesn't encrypt the key                |
 | -keyout          | Specifying filename to send the key to the sample.com.csr |
 | -out             | Specifying file name to write to CSR                      |

You will be prompted to enter information (in order) such as:
 | **Requested Information** | **Description**                                                                                                |
 | ------------------------- | -------------------------------------------------------------------------------------------------------------- |
 | Country Name              | Two-letter abbreviation for the country you reside in                                                          |
 | State or Province Name    | Full name of the state from where your organization operates                                                   |
 | Locality Name             | Name of the city where the organization operates from                                                          |
 | Organization Name         | Name of Organization. If registered as an individual, enter the name of the person requesting the certificate. |
 | Organizational Unit Name  | Section or sector in whichthe organization operates                                                            |
 | Common Name               | The domain name to whom you're purchasing an SSL certificate.It should be a Fully Qualified Domain Name        |
 | Email Address **\***      | A email address associated with app store connect                                                              |
 | Challenge password **\*** | Password to be sent with your certificate request                                                              |
  **_\* - required_**
</details>

#### 1.1 Distribution Certificate (.cer)

Go to apple developer.
Navigate to the Certificates, Identifiers & Profiles and go to [Certificates](https://developer.apple.com/account/resources/certificates/list) section 

Click on the ![Add Icon](./icons/ic_add.svg) icons to Create a New Certificate. Select **Apple Distribution** and continue

Upload the Certificate Signing Request (CSR) you just generate and continue

Download and save the Distribution certificate - _**distribution.cer**_

Notice that you have downloaded a .cer file, but we need a .p12 file to proceed. We'll now convert the .cer file to .p12 file using the **Private_Key.key** we generated with CSR

#### 1.2 Distribution Certificate .cer to .pem

First we need to convert the .cer file to a .pem file
```bash
openssl x509 -in distribution.cer -inform DER -out distribution.pem -outform PEM
```
#### 1.3 Apple's Worldwide developer cert

now that we got the _distribution.pem_ file we'll use the Private_Key.key file to generate the .p12 file. Found on [Simon Dalvai's blog](https://simondalvai.org/blog/dev-cert-linux/)

Download Apple's Worldwide developer cert from Apple's certificates website and convert it to pem:

_Note: Here Worldwide Developer Relations - G4 is used. **(expiring on 12/10/2030)**_

```bash
wget https://www.apple.com/certificateauthority/AppleWWDRCAG4.cer
```
and convert it to pem
```bash
openssl x509 -in AppleWWDRCAG4.cer -inform DER -out AppleWWDRCAG4.pem -outform PEM
```

#### 1.4 Distribution Certificate .pem to .p12
Convert your cert plus Apple's cert to p12 format (choose a password for the .p12).
Note: use -legacy if using opensssl v3.x . Found on [StackOverflow](https://stackoverflow.com/questions/70431528/mac-verification-failed-during-pkcs12-import-wrong-password-azure-devops)

```bash
openssl pkcs12 -export -legacy -out distribution.p12 -inkey Private_Key.key -in distribution.pem -certfile AppleWWDRCAG4.pem
```
>_NOTICE: save the password as it'll be used in later step to store in **P12_PASSWORD**_

now we have the .p12 file, let's proceed to create the Provisioning profile


### 2. Provisioning profile - .mobileprovision

Go to apple developer. 
Navigate to the Certificates, Identifiers & Profiles and go to [Profiles](https://developer.apple.com/account/resources/profiles/list) section

Click on the ![Add Icon](./icons/ic_add.svg) icons to Register a New Provisioning Profile. Select **App Store Connect** under Distribution and continue

Select an App ID (BundleID) from the drop down and continue

Select the certificates you just created to include in this provisioning profile and continue.

Give a name to the profile to identify and continue

Download and save the Provisioning profile _<given_name>.mobileprovision_


### 3. App store connect api key - .p8

Go to app store connect.
Navigate to the Users and Access then to [Integrations](https://appstoreconnect.apple.com/access/integrations/api) tab

Choose between Team Keys/Individual Keys and Click on the ![Add Icon](./icons/ic_add.svg) icons to create a key

Give a name and access type to create the key

Download and save the key

>_NOTICE: Note the IssuerID and KeyID for later_


## Convert the generated files to `base64` and upload to secrets

#### For Development/Distribution certificate - .p12
```bash
base64 distribution.p12
```
copy the output and add to the secret `BUILD_CERTIFICATE_BASE64`



---

#### For Provisioning profile - .mobileprovision
```bash
base64 provisioning_profile.mobileprovision
```

copy and add the output to secret `BUILD_PROVISIONING_PROFILE_BASE64`

#### For App store connect api key - .p8
```bash
base64 AuthKey_123456WXYZ.p8
```

copy and add the output to secret `APPSTORE_API_KEY_BASE64`

## Store other values to the github secrets

#### Distribution Certificate password

Copy and store the Distribution Certificate password to `P12_PASSWORD` 

#### IssuerID

Copy and store the App store connect api key IssuerID to `APPSTORE_API_ISSUER_ID` 

#### App store connect api key ID

Copy and store the App store connect api key ID which look like 123456WXYZ to `APPSTORE_API_KEY` 

#### Keychain password

Copy and store the Certificate Signing Request password which you gave at the time of generation to `KEYCHAIN_PASSWORD` 
_(hope you remember)_

# Project Setup

## Export Options
create a file **exportOptions.plist** in ios folder and use the below content,
make the appropriate changes as described

_Note: add the values without the square brackets [ ]_
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>destination</key>
        <string>export</string>
        <key>manageAppVersionAndBuildNumber</key>
        <true/>
        <key>method</key>
        <string>app-store</string>
        <key>signingCertificate</key>
        <!-- Distribution certificate with teamID -->
        <string>Apple Distribution: [team-name] ([teamID])</string>
        <key>provisioningProfiles</key>
        <dict>
            <!-- BundleID -->
            <key>com.example.app</key> 
            <!-- Provisioning Profile name -->
            <string>[provisioning profile name]</string> 
        </dict>
        <key>signingStyle</key>
        <string>manual</string>
        <key>stripSwiftSymbols</key>
        <true/>
        <key>teamID</key>
        <!-- teamID -->
        <string>[teamID]</string>
        <key>uploadBitcode</key>
        <false/>
        <key>uploadSymbols</key>
        <false/>
    </dict>
</plist>
```

## Xcode Project Setup
### Adding values to the xcode project (.pbxproj) file 

Just merge the values to the XCBuildConfiguration sections

```
249021D4217E4FDB00AE95B9 /* Profile */ = {
			isa = XCBuildConfiguration;
			baseConfigurationReference = 7AFA3C8E1D35360C0083082E /* Release.xcconfig */;
			buildSettings = {
				...
				CODE_SIGN_IDENTITY = "Apple Development";
				"CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Distribution";
				CODE_SIGN_STYLE = Manual;
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Profile;
		};
		331C8088294A63A400263BE5 /* Debug */ = {
			isa = XCBuildConfiguration;
			buildSettings = {
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app.RunnerTests;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Debug;
		};
		331C8089294A63A400263BE5 /* Release */ = {
			isa = XCBuildConfiguration;
			buildSettings = {
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app.RunnerTests;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Release;
		};
		331C808A294A63A400263BE5 /* Profile */ = {
			isa = XCBuildConfiguration;
			buildSettings = {
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app.RunnerTests;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Profile;
		};
        97C147061CF9000F007C117D /* Debug */ = {
			isa = XCBuildConfiguration;
			baseConfigurationReference = 9740EEB21CF90195004384FC /* Debug.xcconfig */;
			buildSettings = {
				...
				CODE_SIGN_IDENTITY = "Apple Development";
				"CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Distribution";
				CODE_SIGN_STYLE = Manual;
				CURRENT_PROJECT_VERSION = "$(FLUTTER_BUILD_NUMBER)";
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Debug;
		};
		97C147071CF9000F007C117D /* Release */ = {
			isa = XCBuildConfiguration;
			baseConfigurationReference = 7AFA3C8E1D35360C0083082E /* Release.xcconfig */;
			buildSettings = {
				...
				CODE_SIGN_IDENTITY = "Apple Development";
				"CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Distribution";
				CODE_SIGN_STYLE = Manual;
				...
				DEVELOPMENT_TEAM = "";
				"DEVELOPMENT_TEAM[sdk=iphoneos*]" = <teamID>;
				PRODUCT_BUNDLE_IDENTIFIER = com.example.app;
				PROVISIONING_PROFILE_SPECIFIER = "";
				"PROVISIONING_PROFILE_SPECIFIER[sdk=iphoneos*]" = "<provisioning profile name>";
				...
			};
			name = Release;
		};
```


## Workflow File

``` yml
name: iOS - Build and Release

on:
  push:
    branches: release

jobs:
  # Build jobs
  build-prod:
    name: Production build the flutter app and sign
    runs-on: macos-latest

    steps:
      # Check the Xcode version
      - name: Check Xcode version
        run: /usr/bin/xcodebuild -version

      # Checks-out our repository under $GITHUB_WORKSPACE, so our job can access it
      - name: Checkout
        uses: actions/checkout@v4

      # Install the Apple certificate and provisioning profile
      - name: Install the Apple certificate and provisioning profile
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.BUILD_PROVISIONING_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |

          # create variables
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

          # import certificate and provisioning profile from secrets
          echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

          # create temporary keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          # import certificate to keychain
          security import $CERTIFICATE_PATH \
          -P "$P12_PASSWORD" \
          -A -t cert -f pkcs12 \
          -k $KEYCHAIN_PATH

          security set-key-partition-list \
          -S apple-tool:,apple: \
          -k "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          security list-keychain -d user -s $KEYCHAIN_PATH

          # apply provisioning profile
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles
          ln -s ~/Library/Developer/Xcode/UserData/Provisioning\ Profiles \
          ~/Library/MobileDevice/Provisioning\ Profiles

      # Install flutter
      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: stable
          flutter-version: "3.29.1" # or the latest

      # Install your project's dependencies
      - run: flutter pub get

      # Build and sign the ipa using a single flutter command
      - name: Building IPA
        run: flutter build ipa --release --export-options-plist=ios/exportOptions.plist

      # Create and import app store connect api key
      - name: Import App store connect api key and upload app
        env:
          APPSTORE_API_KEY_BASE64: ${{ secrets.APPSTORE_API_KEY_BASE64 }}
          APPSTORE_API_ISSUER_ID: ${{ secrets.APPSTORE_API_ISSUER_ID }}
          APPSTORE_API_KEY: ${{ secrets.APPSTORE_API_KEY }}
        run: |

          # create app store connect api key filepath
          mkdir ~/private_keys
          APPSTORE_API_KEY_PATH=~/private_keys/AuthKey_$APPSTORE_API_KEY.p8

          # import App store connect api key from secrets
          echo -n "$APPSTORE_API_KEY_BASE64" | base64 --decode -o $APPSTORE_API_KEY_PATH

          # upload the ipa to app store
          xcrun altool --upload-app --type ios -f build/ios/ipa/*.ipa \
          --apiKey "$APPSTORE_API_KEY" --apiIssuer "$APPSTORE_API_ISSUER_ID"

```

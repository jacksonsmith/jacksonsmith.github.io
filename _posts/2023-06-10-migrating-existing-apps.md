---
layout: post
title: Migrating Existing Mobile App
---

Codebase Switch: A Step-by-Step Guide to Migrating an Android App that Uses React Native
Photo by Suzanne D. Williams on UnsplashIntroduction:
When faced with the need to migrate an existing app to another codebase, I found a lack of detailed materials outlining the process. As a result, I decided to share the simple steps I learned to efficiently carry out this migration. In this guide, I will show you how to switch the codebase of your app and direct it to another app in the Play Store.
Road Map
Make sure you have access to the codebase of both the existing and new apps before starting. Here are the steps to generate a binary from the codebase of an existing app to another existing app in the Play Store:
Make the necessary changes:
Generate a new binary.
Upload the newly generated binary to the Play Console.
Test the app in the beta version.
Release and notify users.

Necessary changes:
Identify the parts of the existing codebase that need to be transferred to the new codebase. In the Android app, the only files that needed modification were:
build.gradle
key.json
fastlane/fastfile
keystore's

# build.gradle
If you have a pipeline using Fastlane, you'll need to update the key.json file, which is used for authentication in the Play Console.Arquivo: key.json
# key.json
{
  "type": "service_account",
  "project_id": "api-xxxx",
  "private_key_id": "xxxxxxxxxxxxxxx",
  "private_key": "-----BEGIN PRIVATE KEY-----\xxxxx",
  "client_email": "app@xxxxx.iam.gserviceaccount.com",
  "client_id": "123456",
  "auth_uri": "https://accounts.google.com/....",
  "token_uri": "https://oauth2.googleapis.com/..",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/...",
  "client_x509_cert_url": "https://www.googleapis.com/...."
}
# Appfile
In the Fastlane Appfile, it will also be necessary.
json_key_file("pathToFolder/key.json")
package_name("br.com.oldApp")
# Keystore
The keystore file is a secure storage file used in Android app development to digitally sign the app. It contains encrypted information such as private keys and certificates, which are used to verify the authenticity and integrity of the app during the installation and update process.
The keystore is necessary to generate a signed APK or AAB file, which is the final version of the app that can be distributed to users. The digital signature ensures that the app has not been tampered with or modified by third parties and also allows for secure updates.
The keystore files should be updated with the keystore of the old app, and consequently, the passwords for this file also need to be updated.
Password:
release {
            storeFile file('release.keystore')
            storePassword 'password'
            keyAlias 'keyAliasName'
            keyPassword 'keyPassword'
            storePassword 'storePassword'
            keyAlias 'br.com.oldApp'
            keyPassword '12345'
        }
Change Package Name
To avoid invalid package name error you also need change the package name from older app to new app, there is this step by step answer on stackoverflow. (thanks Grazi Oliveira)
Change Version Code
VersionCode on app/build.gradle always should be greather then the last value submitted from older app on appstore.
Generating a new binary
After that, simply generate an AAB and submit it to the Play Store. I use this Fastlane pipeline for that.
desc "Deploy a new version to the Google Play"
  lane: deploy do
      bump_version_code #private lane to bump version code
      gradle( task: "bundle", build_type: "Release")
      upload_to_play_store(release_status: "draft", skip_upload_apk: true)
    end
Conclusion:
Migrating an existing app to another app in the Play Store may seem challenging, but by following the above steps, you'll be on the right track to achieve a smooth transition. Always remember to back up the source code, thoroughly test, and clearly communicate the changes to users. With patience and attention to detail, you'll be able to enjoy the benefits of a new app and provide an enhanced experience to your users.
# Requirements

- Jenkins Freestyle Project running on a Linux machine
- Java 8 (Tested on OpenJDK 8)	
- Veracode API ID and API Key (found in Veracode > Profile > API Credentials)
- Existing Veracode application profile with **no spaces** in the name

# Configure Parameters
In your Jenkins project *Configure* menu, select:
- [x] **This project is parameterized**

and create 4 *string parameters* with the following formats:

| Name          | API_ID                 |
|---------------|------------------------|
| Default Value | *your Veracode API ID* |
| Description   | Veracode API ID        |
---
| Name          | API_KEY                 |
|---------------|------------------------|
| Default Value | *your Veracode API Key* |
| Description   | Veracode API Key        |
---
| Name          | APP_NAME               |
|---------------|------------------------|
| Default Value | *your Veracode application profile name* |
| Description   | Veracode Application Profile Name (cannot include spaces)        |
---
| Name          | ZIP_FILE               |
|---------------|------------------------|
| Default Value | $WORKSPACE/upload.zip|
| Description   | Name of zip file from previous step      |

# Configure Build
1. Add a build step "Execute Shell" and place the following command inside:
	~~~
	zip -r upload.zip $WORKSPACE -x *node_modules/*
	~~~

3. Add another build step "Execute Shell" and copy/paste the below shell script:
	~~~
	#/bin/bash

        # Version argument
        if [ "$BUILD_VERSION" != "" ];
        then
             VERSION=$BUILD_VERSION
        else
             VERSION=`date "+%Y-%m-%d-%T"`    # Use date as default
        fi
        echo '[INFO] Scan-Name set to '$VERSION
        echo ""

        PRESCAN_SLEEP_TIME=60
        SCAN_SLEEP_TIME=120
        JAVA_WRAPPER_LOCATION="."
        OUTPUT_FILE_LOCATION="$WORKSPACE"
        OUTPUT_FILE_NAME=$APP_NAME'-'$VERSION'.txt'
        echo '[INFO] ------------------------------------------------------------------------'
        echo '[INFO] DOWNLOADING VERACODE JAVA WRAPPER'
        if `wget https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/17.11.4.9/vosp-api-wrappers-java-17.11.4.9.jar -O VeracodeJavaAPI.jar`; then
                chmod 755 VeracodeJavaAPI.jar
                echo '[INFO] SUCCESSFULLY DOWNLOADED WRAPPER'
        else
                echo '[ERROR] DOWNLOAD FAILED'
                exit 1
        fi


        echo '[INFO] ------------------------------------------------------------------------'
        echo '[INFO] VERACODE UPLOAD AND SCAN'

        app_ID=$(java -verbose -jar $JAVA_WRAPPER_LOCATION/VeracodeJavaAPI.jar -vid $API_ID -vkey $API_KEY -action GetAppList | grep -w "$APP_NAME" | sed -n 's/.* app_id=\"\([0-9]*\)\" .*/\1/p')

        if [ -z "$app_ID" ];
        then
             echo '[INFO] App does not exist'
             echo '[INFO] create app: ' $APP_NAME
             creat_addp=$(java -jar $JAVA_WRAPPER_LOCATION/VeracodeJavaAPI.jar -vid $API_ID -vkey $API_KEY -action createApp -appname "$APP_NAME" -criticality high)
             echo '[INFO]app created'
             app_ID=$(java -jar $JAVA_WRAPPER_LOCATION/VeracodeJavaAPI.jar -vid $API_ID -vkey $API_KEY -action GetAppList | grep -w "$APP_NAME" | sed -n 's/.* app_id=\"\([0-9]*\)\" .*/\1/p')
             echo '[INFO] new App-ID: ' $app_ID
             echo ""
        else
             echo '[INFO] App-IP: ' $app_ID
             echo ""
        fi



        echo ""
        echo '====== DEBUG START ======'
        echo 'API-ID: ' $API_ID
        echo 'API-Key: ' $API_KEY
        echo 'App-Name: ' $APP_NAME
        echo 'APP-ID: ' $app_ID
        echo 'File-Path: ' $ZIP_FILE
        echo 'Scan-Name: ' $BUILD_VERSION
        echo '====== DEBUG END ======'
        echo ""


        echo '[INFO] VERACODE scan pre-checks'
        echo '[INFO] directory checks'
        # Directory argument
        if [ "$ZIP_FILE" != "" ]; then
             UPLOAD_DIR="$ZIP_FILE"
        else
             echo "[ERROR] Directory not specified."
             exit 1
        fi

        # Check if directory exists
        if ! [ -f "$UPLOAD_DIR" ];
        then
             echo "[ERROR] File does not exist"
             exit 1
        else
             echo '[INFO] File set to '$UPLOAD_DIR
        fi

    

        #Upload files, start prescan and scan
        echo '[INFO] upload and scan'
        java -jar $JAVA_WRAPPER_LOCATION/VeracodeJavaAPI.jar -vid $API_ID -vkey $API_KEY -action uploadandscan -appname $APP_NAME -createprofile true -filepath $ZIP_FILE -version $VERSION > $OUTPUT_FILE_LOCATION$OUTPUT_FILE_NAME 2>&1
        echo ""

        upload_scan_results=$(cat $OUTPUT_FILE_LOCATION$OUTPUT_FILE_NAME)

        echo "*************************** upload_scan_results"
        echo $upload_scan_results

        if [ $upload_scan_results == *"already exists"* ];
        then
             echo ""
             echo '[ERROR] This scan name already exists'
             exit 1
        elif [ $upload_scan_results == *"in progress or has failed"* ];
        then
             echo ""
             echo '[ ERROR ] Something went wrong! A previous scan is in progress or has failed to complete successfully'
                exit 1
        else
             echo ""
             echo '[INFO] File(s) uploaded and PreScan started'
        fi

        exit 0
	~~~

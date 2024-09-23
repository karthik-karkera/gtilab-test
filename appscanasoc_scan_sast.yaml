#!/bin/bash
 
# Set variables
appId=$(cat appId.txt)
echo "Sast" > scanTech.txt
 
# Download and prepare SAClientUtil
curl -k "https://$serviceUrl/api/v4/Tools/SAClientUtil?os=linux" > $HOME/SAClientUtil.zip
unzip $HOME/SAClientUtil.zip -d $HOME > /dev/null
rm -f $HOME/SAClientUtil.zip
mv $HOME/SAClientUtil.* $HOME/SAClientUtil
export PATH="$HOME/SAClientUtil/bin:${PATH}"
 
# Fetch only updated files using GitLab CI environment
updated_files=$(git diff --name-only HEAD~1 HEAD)
 
# If there are no updated files, exit
if [ -z "$updated_files" ]; then
  echo "No files were updated. Exiting."
  exit 0
fi
 
echo "Preparing IRX file for updated files: $updated_files"
 
# Prepare the IRX file with only the updated files
appscan.sh prepare -d "$updated_files"
 
# Authenticate with AppScan on Cloud (ASoC)
asocToken=$(curl -k -s -X POST --header 'Content-Type:application/json' --header 'Accept:application/json' \
-d '{"KeyId":"'"$asocApiKeyId"'","KeySecret":"'"$asocApiKeySecret"'"}' \
"https://$serviceUrl/api/v4/Account/ApiKeyLogin" | grep -oP '(?<="Token":\ ")[^"]*')
 
# Validate the authentication
if [ -z "$asocToken" ]; then
  echo "Authentication failed. Exiting."
  exit 1
fi
 
# Upload the IRX file to AppScan
irxFile=$(ls -t *.irx | head -n1)
 
if [ -f "$irxFile" ]; then
  echo "Uploading IRX file $irxFile to ASoC."
  irxFileId=$(curl -k -s -X 'POST' "https://$serviceUrl/api/v4/FileUpload" \
  -H 'accept:application/json' \
  -H "Authorization:Bearer $asocToken" \
  -H 'Content-Type:multipart/form-data' \
  -F "uploadedFile=@$irxFile" | grep -oP '(?<="FileId":\ ")[^"]*')
  echo "IRX file uploaded with ID: $irxFileId."
else
  echo "No IRX file found. Exiting."
  exit 1
fi
 
# Start the scan in ASoC
scanId=$(curl -s -k -X 'POST' "https://$serviceUrl/api/v4/Scans/Sast" \
-H 'accept:application/json' \
-H "Authorization:Bearer $asocToken" \
-H 'Content-Type:application/json' \
-d '{"AppId":"'"$appId"'","ApplicationFileId":"'"$irxFileId"'","ClientType":"user-site","EnableMailNotification":false,"Execute":true,"Locale":"en","Personal":false,"ScanName":"'"SAST $scanName $irxFile"'","EnablementMessage":"","FullyAutomatic":true}' | jq -r '. | {Id} | join(" ")')
 
# Display scan status
echo "Scan started with Scan ID: $scanId"
echo "Scan Name: $scanName, Scan ID: $scanId"
echo $scanId > scanId.txt
 
# Monitor the scan until it's complete
scanStatus=$(curl -k -s -X 'GET' "https://$serviceUrl/api/v4/Scans/Sast/$scanId" \
-H 'accept:application/json' \
-H "Authorization:Bearer $asocToken" | jq -r '.LatestExecution | {Status} | join(" ")')
 
echo "Current scan status: $scanStatus"
 
while [ "$scanStatus" == "Running" ] || [ "$scanStatus" == "InQueue" ]; do
  echo "Scan is in progress: $scanStatus"
  sleep 60
  scanStatus=$(curl -k -s -X 'GET' "https://$serviceUrl/api/v4/Scans/Sast/$scanId" \
  -H 'accept:application/json' \
  -H "Authorization:Bearer $asocToken" | jq -r '.LatestExecution | {Status} | join(" ")')
done
 
# Check if scan failed
if [ "$scanStatus" == "Failed" ]; then
  echo "Scan failed. Check ASoC logs."
  exit 1
else
  echo "Scan completed successfully with status: $scanStatus"
fi

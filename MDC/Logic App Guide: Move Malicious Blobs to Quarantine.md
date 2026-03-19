# Logic App Guide: Move Malicious Blobs to Quarantine

## Purpose
This Logic App listens to Microsoft Defender for Cloud (MDC) malware alerts and automatically moves malicious blobs from their original container to the maliciousfiles container in the same storage account.

The move pattern is:
1. Copy source blob to maliciousfiles.
2. Delete the original source blob only after copy succeeds.

This provides safer behavior than direct delete.

## What This Workflow Does

### Trigger
- Trigger type: ApiConnectionWebhook
- Source: Microsoft Defender for Cloud alert subscription
- Trigger name: When_an_Microsoft_Defender_for_Cloud_Alert_is_created_or_triggered

### Main Logic
1. Log_Trigger_Body
- Stores full trigger payload for troubleshooting.

2. Check_Malicious_Alert
- Checks whether alert payload indicates malicious content.
- If not malicious, workflow exits through Skip_Non_Malicious.

3. GetBlobEntity
- Extracts blob entities from the alert payload.
- Supports both possible entity paths:
  - triggerBody().Entities
  - triggerBody().data.Entities

4. Check_Blob_Found
- Proceeds only if at least one blob entity is present.
- If none, exits through No_Blob_Found.

5. Compose_SourceBlobUrl
- Picks source blob URL from available fields in entity:
  - Url
  - url
  - blobUri
  - BlobUri

6. Compose_SourceBlobName
- Builds destination path from source URL path.
- Current behavior preserves source folder structure below source container.

7. Copy_Blob_To_Malicious
- Uses HTTP PUT to destination URL:
  - https://<same-storage-account>.blob.core.windows.net/maliciousfiles/<source-relative-path>
- Uses header x-ms-copy-source with the source blob URL.
- Uses Managed Identity with audience https://storage.azure.com/.

8. Delete_Blob
- Runs only after copy succeeded.
- Deletes the original source blob URL.

9. Log_Success / failure handlers
- Log success and failure states for operations and troubleshooting.

## Example
Source blob:
- https://xxxxxx.blob.core.windows.net/test/test/A/eicar.com

Destination blob after copy:
- https://xxxxxx.blob.core.windows.net/maliciousfiles/test/A/eicar.com

Original blob is then deleted.

## Prerequisites
Deploy the DeleteBlobLogicApp Azure Resource Manager (ARM) template by using the Azure portal.
https://github.com/Azure/Microsoft-Defender-for-Cloud/tree/main/Workflow%20automation/Delete%20Blob%20LogicApp%20Defender%20for%20Storage
Once deployed, use json file to replace logic app's code, and remember to put the right subscription id and the name of rg.

1. Logic App has system-assigned managed identity enabled.
2. Managed identity has required Storage Blob Data permissions on target storage account:
- Storage Blob Data Contributor (minimum for copy and delete)

3. MDC alert connection exists:
- Connection name in template: mdcalert-Remove-MalwareFile

4. Storage account has a container named maliciousfiles.
- If not, create it in advance.

## Recommended MDC Configuration
To avoid looped processing:
- Exclude maliciousfiles container from MDC malware scanning.

Reason:
- If MDC scans the copied blob in maliciousfiles and raises another alert, the Logic App can run again.

## Deployment and Use
1. Open Logic App code view.
2. Paste the JSON workflow definition from delefile.json.
3. Save and enable workflow.
4. Upload a known test malware sample in a monitored container (for example, EICAR test file in a non-maliciousfiles container).
5. Confirm run history shows:
- malicious alert detected
- blob entity found
- copy succeeded
- source delete succeeded

6. Verify storage results:
- Blob exists in maliciousfiles destination path.
- Original blob no longer exists.

## Troubleshooting

### Error: array index 0 from empty array
Cause:
- No blob entities found in alert payload.
Fix:
- Check Log_Trigger_Body and GetBlobEntity outputs to confirm payload shape and entity type.

### Error: copy failed
Cause candidates:
- Missing managed identity permissions on storage account.
- Source blob URL field mismatch in alert entity.
Fix:
- Validate managed identity role assignment.
- Inspect Compose_SourceBlobUrl output.

### Error: both source and destination missing
Cause candidates:
- Re-trigger loop and second run deleting quarantined blob.
Fix:
- Exclude maliciousfiles from MDC scanning.
- Keep guard logic to skip already-quarantined container (recommended).

## Operational Notes
- Keep Log_Trigger_Body for initial rollout and troubleshooting.
- For production hardening, reduce payload logging if data exposure is a concern.
- Keep copy-then-delete order to avoid accidental data loss.

# ROTATE STORAGE ACCOUNT KEYS USING AZ DEVOPS

Greetings to my fellow Technology Advocates and Specialists.

In this Session, I will demonstrate __How to Rotate Storage Account Keys (Primary & Secondary) and Store it in Key Vault Using Azure DevOps.__

| __LIVE RECORDED SESSION:-__ |
| --------- |
| __LIVE DEMO__ was Recorded as part of my Presentation in __JOURNEY TO THE CLOUD 9.0__ Forum/Platform |
| Duration of My Demo = __55 Mins 42 Secs__ |
| [![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/EGIOzEpOxzE/0.jpg)](https://www.youtube.com/watch?v=EGIOzEpOxzE) |

| __THIS IS HOW IT LOOKS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qmxxkmphb6ry6dxtgg1a.JPG) |


| __AUTOMATION OBJECTIVE:-__ |
| --------- |
| Validate If Resource Group Containing Key Vault Exists. If __No Resource Group Found__, Pipeline will __FAIL__. |
| Validate If Storage Account Exists inside the Specified Resource Group. If __No Storage Account Found__, Pipeline will __FAIL__. |
| Validate If Key Vault Exists inside the Specified Resource Group. If __No Key Vault Found__, Pipeline will __FAIL__. |
| If All of the above validation is __SUCCESSFUL__, Depending upon, which Key User wants to rotate (Primary or Secondary), Pipeline will then Rotate the Storage Account Key and  Store it in the Key Vault. |


| __IMPORTANT NOTE:-__ |
| --------- |
The YAML Pipeline is tested on __WINDOWS BUILD AGENT__ Only!!!


| __REQUIREMENTS:-__ |
| --------- |

1. Azure Subscription.
2. Azure DevOps Organisation and Project.
3. Service Principal with Required RBAC ( __Contributor__) applied on Subscription or Resource Group(s). 
5. Azure Resource Manager Service Connection in Azure DevOps.


| __HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ehjtot8ovf7pq91k50cl.png) |


| PIPELINE CODE SNIPPET:- | 
| --------- |

| AZURE DEVOPS YAML PIPELINE (azure-pipelines-storage-account-key-rotation-v1.0.yml):- | 
| --------- |

```
trigger:
  none

######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SUBSCRIPTIONID
  displayName: Subscription ID Details Follow Below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGNAME
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: STORAGEACCOUNTNAME
  displayName: Please Provide the Storage Account Name:-
  type: object
  default:

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: STORAGEACCOUNTKEYS
  displayName: Choose Storage Account Keys - Primary or Secondary:-
  type: string
  default: Primary
  values:
  - Primary
  - Secondary 

######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest

#########################
# Declare Build Agents:-
#########################
pool:
  vmImage: $(BuildAgent)

###################
# Declare Stages:-
###################

stages:

- stage: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV 
  jobs:
  - job: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV 
    displayName: VALIDATE RG STORAGE ACCOUNT & KV
    steps:
    - task: AzureCLI@2
      displayName: SET AZURE ACCOUNT
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SUBSCRIPTIONID }}
          az account show  
    - task: AzureCLI@2
      displayName: VALIDATE RG STORAGE ACCOUNT & RG
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |      
          $i = az group exists -n ${{ parameters.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ parameters.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "###################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} exists!!!"
                  echo "###################################################################"
                  $k = az keyvault list --resource-group ${{ parameters.RGNAME }} --query [].name -o tsv		
                    if ($k -eq "${{ parameters.KVNAME }}") {
                      echo "###################################################################"
                      echo "Key Vault ${{ parameters.KVNAME }} exists!!!"
                      echo "###################################################################"
                    }
                  else {
                    echo "###################################################################################################"
                    echo "Key Vault ${{ parameters.KVNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                    echo "###################################################################################################"
                    exit 1
                  }
                }  
                else {
                  echo "#######################################################################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                  echo "#######################################################################################################################"
                  exit 1
                }              
            }
            else {
              echo "#############################################################"
              echo "Resource Group ${{ parameters.RGNAME }} DOES NOT EXISTS!!!"
              echo "#############################################################"
              exit 1
            }

- stage: RENEW_STORAGE_ACCOUNT_PRIMARY_KEY
  condition: |
     and(succeeded(),
       eq('${{ parameters.STORAGEACCOUNTKEYS }}', 'Primary')
     ) 
  jobs:
  - job: RENEW_STORAGE_ACCOUNT_PRIMARY_KEY 
    displayName: ROTATE PRIMARY KEY & STORE IN KV
    steps:
    - task: AzureCLI@2
      displayName: ROTATE AND STORE
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $saprimary = az storage account keys renew -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --key ${{ parameters.STORAGEACCOUNTKEYS }} --query [0].value -o tsv
          az keyvault secret set --name "${{ parameters.STORAGEACCOUNTNAME }}-${{ parameters.STORAGEACCOUNTKEYS }}" --vault-name ${{ parameters.KVNAME }} --value $saprimary
          echo "#################################################################################################################################################################"
          echo "Storage Account Primary Key has been successfully Rotated and Stored in Key Vault ${{ parameters.KVNAME }} under the Resource Group ${{ parameters.RGNAME }}!!!"
          echo "#################################################################################################################################################################"

- stage: RENEW_STORAGE_ACCOUNT_SECONDARY_KEY
  condition: |
       eq('${{ parameters.STORAGEACCOUNTKEYS }}', 'Secondary')
  dependsOn: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV   
  jobs:
  - job: RENEW_STORAGE_ACCOUNT_SECONDARY_KEY 
    displayName: ROTATE SECONDARY KEY & STORE IN KV
    steps:
    - task: AzureCLI@2
      displayName: ROTATE AND STORE
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $sasecondary = az storage account keys renew -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --key ${{ parameters.STORAGEACCOUNTKEYS }} --query [1].value -o tsv
          az keyvault secret set --name "${{ parameters.STORAGEACCOUNTNAME }}-${{ parameters.STORAGEACCOUNTKEYS }}" --vault-name ${{ parameters.KVNAME }} --value $sasecondary
          echo "#################################################################################################################################################################"
          echo "Storage Account Secondary Key has been successfully Rotated and Stored in Key Vault ${{ parameters.KVNAME }} under the Resource Group ${{ parameters.RGNAME }}!!!"
          echo "#################################################################################################################################################################"

```

Now, let me explain each part of YAML Pipeline for better understanding.

| __PART #1:-__ | 
| --------- |

| __BELOW FOLLOWS PIPELINE RUNTIME VARIABLES CODE SNIPPET:-__ | 
| --------- |

```
######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SUBSCRIPTIONID
  displayName: Subscription ID Details Follow Below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGNAME
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: STORAGEACCOUNTNAME
  displayName: Please Provide the Storage Account Name:-
  type: object
  default:

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: STORAGEACCOUNTKEYS
  displayName: Choose Storage Account Keys - Primary or Secondary:-
  type: string
  default: Primary
  values:
  - Primary
  - Secondary 

```

| __PART #2:-__ | 
| --------- |

| __BELOW FOLLOWS PIPELINE VARIABLES CODE SNIPPET:-__ | 
| --------- |

```
######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest

```

| __NOTE:-__ | 
| --------- |
| Please change the values of the variables accordingly. |
| The entire YAML pipeline is build using __Runtime Parameters and Variables__. No Values are Hardcoded. |


| __PART #3:-__ | 
| --------- |

| __This is a 3 Stage Pipeline:-__ | 
| --------- |

| __STAGE #1 - VALIDATE_RG_STORAGE_ACCOUNT_AND_KV:-__ | 
| --------- |
| In this Stage, Pipeline will validate __Resource Group__, __Storage Account__ and __Key Vault__. If any one of the Azure Resource is __Not Available__, Pipeline will __FAIL__ and the other 2 Stages gets __SKIPPED__. |

```
- stage: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV 
  jobs:
  - job: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV 
    displayName: VALIDATE RG STORAGE ACCOUNT & KV
    steps:
    - task: AzureCLI@2
      displayName: SET AZURE ACCOUNT
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SUBSCRIPTIONID }}
          az account show  
    - task: AzureCLI@2
      displayName: VALIDATE RG STORAGE ACCOUNT & RG
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |      
          $i = az group exists -n ${{ parameters.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ parameters.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "###################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} exists!!!"
                  echo "###################################################################"
                  $k = az keyvault list --resource-group ${{ parameters.RGNAME }} --query [].name -o tsv		
                    if ($k -eq "${{ parameters.KVNAME }}") {
                      echo "###################################################################"
                      echo "Key Vault ${{ parameters.KVNAME }} exists!!!"
                      echo "###################################################################"
                    }
                  else {
                    echo "###################################################################################################"
                    echo "Key Vault ${{ parameters.KVNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                    echo "###################################################################################################"
                    exit 1
                  }
                }  
                else {
                  echo "#######################################################################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                  echo "#######################################################################################################################"
                  exit 1
                }              
            }
            else {
              echo "#############################################################"
              echo "Resource Group ${{ parameters.RGNAME }} DOES NOT EXISTS!!!"
              echo "#############################################################"
              exit 1
            }
```

| __STAGE #2 - RENEW_STORAGE_ACCOUNT_PRIMARY_KEY:-__ | 
| --------- |
| In this Stage, Pipeline has Conditions in Place. |
| __Condition #1: The Previous Stage has to be Successful.__ |
| __Condition #2: The User should Select option "Primary".__ |

```
- stage: RENEW_STORAGE_ACCOUNT_PRIMARY_KEY
  condition: |
     and(succeeded(),
       eq('${{ parameters.STORAGEACCOUNTKEYS }}', 'Primary')
     ) 
```

| __BELOW FOLLOWS THE LOGIC DEFINED TO RENEW/ROTATE STORAGE ACCOUNT PRIMARY KEY AND STORE IN THE MENTIONED KEYVAULT:-__ | 
| --------- |


```
jobs:
  - job: RENEW_STORAGE_ACCOUNT_PRIMARY_KEY 
    displayName: ROTATE PRIMARY KEY & STORE IN KV
    steps:
    - task: AzureCLI@2
      displayName: ROTATE AND STORE
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $saprimary = az storage account keys renew -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --key ${{ parameters.STORAGEACCOUNTKEYS }} --query [0].value -o tsv
          az keyvault secret set --name "${{ parameters.STORAGEACCOUNTNAME }}-${{ parameters.STORAGEACCOUNTKEYS }}" --vault-name ${{ parameters.KVNAME }} --value $saprimary
          echo "#################################################################################################################################################################"
          echo "Storage Account Primary Key has been successfully Rotated and Stored in Key Vault ${{ parameters.KVNAME }} under the Resource Group ${{ parameters.RGNAME }}!!!"
          echo "#################################################################################################################################################################"

```

| __STAGE #3 - RENEW_STORAGE_ACCOUNT_SECONDARY_KEY:-__ | 
| --------- |
| In this Stage, Pipeline has Conditions in Place. |
| __Condition #1: The User should Select option "Secondary".__ |
| __Condition #2: Depends upon Stage #1: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV  .__ |


```
- stage: RENEW_STORAGE_ACCOUNT_SECONDARY_KEY
  condition: |
       eq('${{ parameters.STORAGEACCOUNTKEYS }}', 'Secondary')
  dependsOn: VALIDATE_RG_STORAGE_ACCOUNT_AND_KV  

```

| __BELOW FOLLOWS THE LOGIC DEFINED TO RENEW/ROTATE STORAGE ACCOUNT SECONDARY KEY AND STORE IN THE MENTIONED KEYVAULT:-__ | 
| --------- |


```
jobs:
  - job: RENEW_STORAGE_ACCOUNT_SECONDARY_KEY 
    displayName: ROTATE SECONDARY KEY & STORE IN KV
    steps:
    - task: AzureCLI@2
      displayName: ROTATE AND STORE
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $sasecondary = az storage account keys renew -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --key ${{ parameters.STORAGEACCOUNTKEYS }} --query [1].value -o tsv
          az keyvault secret set --name "${{ parameters.STORAGEACCOUNTNAME }}-${{ parameters.STORAGEACCOUNTKEYS }}" --vault-name ${{ parameters.KVNAME }} --value $sasecondary
          echo "#################################################################################################################################################################"
          echo "Storage Account Secondary Key has been successfully Rotated and Stored in Key Vault ${{ parameters.KVNAME }} under the Resource Group ${{ parameters.RGNAME }}!!!"
          echo "#################################################################################################################################################################"

```

__NOW ITS TIME TO TEST !!!...__

| __TEST CASES:-__ | 
| --------- |

| __TEST CASE #1: VALIDATE RESOURCE GROUP, STORAGE ACCOUNT AND KEY VAULT:-__ | 
| --------- |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN RESOURCE GROUP DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w8g8zupd7yln5ig94bq5.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jbt4l8xz9ner1dk2x128.png) |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN STORAGE ACCOUNT DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zfqwvcs115bbd3nlwazf.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/76zw5pb3h5l1vnxm8xxr.png) |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN KEY VAULT DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h5f0er20l4gindwt0vrd.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uktkmbl0v7xr55961xpw.png) |
| __TEST CASE #2: ROTATE/RENEW STORAGE ACCOUNT PRIMARY KEY AND STORE IN KEY VAULT:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kxxn6jdfdxd4rwi8zgim.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fg575um8i1s460n2yy93.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0usyjk4pamlvkzfxai3r.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/65n3r162xhvj69w1zz1w.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vswx9q68jtrz7s8xow44.JPG) |
| __TEST CASE #3: ROTATE/RENEW STORAGE ACCOUNT SECONDARY KEY AND STORE IN KEY VAULT:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4u63hhfwmauqm0f0wkpb.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xhublqbxci7vx4sqwsm8.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0t3iuqgm9j4tzgnlttab.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5alxz2bqpc6qdysw0gau.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/78j8zpz1vcqypt3p2bod.JPG) |


__Hope You Enjoyed the Session!!!__

__Stay Safe | Keep Learning | Spread Knowledge__

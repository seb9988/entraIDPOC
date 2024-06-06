# ------------------------------------------------------------------------------------
# History: 
# ------------------------------------------------------------------------------------
#        Version Draft (POC like)
#        ==================
#        seb, 15.09.2023 - Initial version
#
#        Version 1
#        ==================
#        seb, 22.11.2023 - Renamed M365 to M365AVH
#	 seb, 06.06.2024 - git updated
# ------------------------------------------------------------------------------------


LogExecution ">source_M365AVH.ps1 sourced" "logfileonly"   

# ------------------------------------------------------------------------------------
# MAIN
# ------------------------------------------------------------------------------------
# 

function getM365AVHData { 

   LogExecution "      fct getM365AVHData - started " "logfileonly"

   defineglobalVariableSpecificToM365AVH
   
   retreiveAzureGraphAPICredentials
   retreiveAzureGraphAPIToken
   
   # if WriteMode is "new" (recreate the file at each run) then execute the query without any check on the date
   # if WriteMode is "append" (data will be appened to existing file) then we have to ensure we are not missing data or duplicating data
   switch ( $toto:tata.$source.WriteMode ) {
		'new' {
			#Retrieve M365AVH data for this source (Execute the query)
			executeM365AVHQuery 
		}
		'append' {
			# Get the last data date in the history file
			$global:lastDate = GetLastDataDate
   
			# Check if data has already been done for yesterday
			$tmpregexPattern = $global:dateYesterday
			#write-host "Yesterday is: $tmpregexPattern"
			#Write-Host "Last Date: $global:lastDate"
			if ($global:lastDate -eq $tmpregexPattern) {
				# yesterday data already in the history file, running the query is not required
				LogExecution "         >[WARNING] Query not performed. Data for $tmpregexPattern are already present in the file"
				$global:newDataAvailable = 'false'
			} else {
				# reinitialize the startDate to get missing days if any
				if ($null -ne $global:lastDate ) {
					# Set the M365AVH_APIQueryParameterStartDate variable to the correct date
					$toto:tata.$source.M365AVH_APIQueryParameterStartDate = "let startDate = startofday(datetime($global:lastDate)+1d);"	   
				}
				#Retrieve M365AVH data for this source (Execute the query)
				executeM365AVHQuery
			}
		}
    }
	
	   
   # final 
   removeglobalVariableSpecificToM365AVH
   
}

# ------------------------------------------------------------------------------------
# Azure GraphAPI interaction
# ------------------------------------------------------------------------------------
   
   
function retreiveAzureGraphAPICredentials() {
	
   #Retrieve AZGraphAPIApplicationID and AZGraphAPIAccessSecret
   $global:AZGraphAPIApplicationID = RetrieveCredentials $global:filenameCredentialFile $toto:tata.$source.AZGraphAPIApplicationID
   $global:AZGraphAPIAccessSecret = RetrieveCredentials $global:filenameCredentialFile $toto:tata.$source.AZGraphAPIAccessSecret
      
}

function retreiveAzureGraphAPIToken() {

   LogExecution "         fct retreiveAzureGraphAPIToken - started " "logfileonly"

	
	$APIclientID = "90994e4a-2dd4-4645-98c2-960cf76a4357"
	$APIclientSecret = "p3n7Q~XXXX"
	$APITenantDomain = "toto.onmicrosoft.com"
	$APIresource = "office.microsoft.com"
	$AZVA_TenantID = "99d4947b-4cb1-4119-9ace-abceab54bcd4"
			
	# ---------------------------------
    # Get token 
    # ---------------------------------
    $Body = @{    
       Grant_Type    = "client_credentials"
       resource      = $APIresource
       client_Id     = $APIclientID
       Client_Secret = $APIclientSecret
    } 

    $ConnectAzureManagement = Invoke-RestMethod -Uri "https://login.microsoftonline.com/$AZVA_TenantID/oauth2/token" `
       -Method POST -Body $Body

    $global:AzureAPIToken = $ConnectAzureManagement.access_token
	$global:AzureAPITokenType = $ConnectAzureManagement.token_type
	$global:AzureAPIHeader = @{Authorization = "$($global:AzureAPITokenType) $($global:AzureAPIToken)"}
	
	#suppress global value from cache
	$global:AZGraphAPIAccessSecret = "none - What are you trying to find dude ?"
		
}

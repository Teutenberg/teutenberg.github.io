
# Visual Studio Post Build Event Package Upload 

1. In visual Studio open the "Configuration Manager".
2. Add new "Active solution configuration" >
3. Copy settings from "Release" >
4. Tick "Create new project configurations"
5. Create Push2Octopus.ps1 <see code below>
6. Create file "C:\OctopusApiKey.secret" and save your API key here. 'https://octopus.com/docs/octopus-rest-api/how-to-create-an-api-key'
6. Add Post build event <see command below>
7. Set configuration to "Octopus" and build solution. 

When building the solution using the "Octopus" configuration, 
the database dacpac will be compressed into a zip file and uploaded to your Octopus package repository. 


Push2Octopus.ps1
```
PARAM(
    [string]$ReleaseName = 'MyRelease',
    [string]$ReleaseVersion = $(Get-Date -Format 'yyyy.mm.dd.hhmmss'),
    [string[]]$ReleaseFiles,
    [string]$OutputPath = "C:\Temp",
    [string]$OctopusRootUrl = 'https://octopus-os'
    [string]$OctopusSpaceId = '2'
    [string]$OctopusApiKey = $(Get-Content "C:\OctopusApiKey.secret"),
    [string]$OctopusReplaceExisting = $false,
    [string]$Configuration
)

if ($Configuration -ne 'Octopus') {
    Write-Output 'Configuration is not Octopus. Package has not been pushed to Octopus.'
    Return
}

$packageFilePath = Join-Path $OutputPath "$ReleaseName.$ReleaseVersion.zip"

$compress = @{
    Path= $ReleaseFiles
    CompressionLevel = 'Fastest'
    DestinationPath = $packageFilePath
}

if ($ReleaseFiles) {
    Compress-Archive @compress
}
else {
    Write-Output 'No files to release!'
    Return
}

$packageUrl = "$OctopusRootUrl/api/Spaces-$OctopusSpaceId/packages/raw?replace=$OctopusReplaceExisting";
Write-Output "Release build version: $ReleaseVersion"
Write-Output "Uploading $packageFilePath to $packageUrl"

$webRequest = [System.Net.HttpWebRequest]::Create($packageUrl);
$webRequest.AllowWriteStreamBuffering = $false
$webRequest.SendChunked = $true
$webRequest.Accept = "application/json";
$webRequest.ContentType = "application/json";
$webRequest.Method = "POST";
$webRequest.Headers["X-Octopus-ApiKey"] = $OctopusApiKey;
$packageFileStream = new-object IO.FileStream $packageFilePath,'Open','Read','Read'
$boundary = "----------------------------" + [System.DateTime]::Now.Ticks.ToString("x");
$boundarybytes = [System.Text.Encoding]::ASCII.GetBytes("`r`n--" + $boundary + "`r`n")
$webRequest.ContentType = "multipart/form-data; boundary=" + $boundary;
$webRequest.GetRequestStream().Write($boundarybytes, 0, $boundarybytes.Length);
$header = "Content-Disposition: form-data; filename="""+ [System.IO.Path]::GetFileName($packageFilePath) +"""`r`nContent-Type: application/octet-stream`r`n`r`n";
$headerbytes = [System.Text.Encoding]::ASCII.GetBytes($header);
$webRequest.GetRequestStream().Write($headerbytes, 0, $headerbytes.Length);
$packageFileStream.CopyTo($webRequest.GetRequestStream());
$webRequest.GetRequestStream().Write($boundarybytes, 0, $boundarybytes.Length);
$webRequest.GetRequestStream().Flush();
$webRequest.GetRequestStream().Close();
$packageFileStream.Close();
$packageFileStream.Dispose();
$webResponse = $webRequest.GetResponse();
Write-Host $webResponse.StatusCode $webResponse.StatusDescription;  
$webResponse.Dispose();
```

Post-build event command line:
```
PowerShell.exe 
  -NoProfile 
  -ExecutionPolicy unrestricted 
  -file "$(SolutionDir)Push2Octopus.ps1" 
  -ReleaseName $(TargetName)
  -ReleaseFiles "$(ProjectDir)$(OutputPath)$(TargetName).dacpac" 
  -OutputPath $(ProjectDir)$(OutputPath) 
  -Configuration $(Configuration)
```

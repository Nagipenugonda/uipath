# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  name: SandboxBuild   # adjust to your agent pool

variables:
- group: UiPath_Dev
- name: ProjectJson
  value: '$(Build.SourcesDirectory)\project.json'
- name: OutputDir
  value: '$(Build.SourcesDirectory)\output'
- name: PackagePattern
  value: '*.nupkg'

steps:
# PACK
- powershell: |
    uipcli.exe package pack "$(ProjectJson)" --output "$(OutputDir)"
  displayName: "UiPath: Pack project"

# FIND PACKAGE
- powershell: |
    $pkg = Get-ChildItem -Path "$(OutputDir)" -Filter "$(PackagePattern)" -File | Sort-Object LastWriteTime -Descending | Select-Object -First 1
    if (-not $pkg) { throw "No package found under $(OutputDir)" }
    Write-Host "##vso[task.setvariable variable=BuiltPackagePath]$($pkg.FullName)"
    Write-Host "Package ready: $($pkg.Name)"
  displayName: "Find latest .nupkg"

# DEPLOY
- powershell: |
    uipcli.exe package deploy "$(BuiltPackagePath)" `
      --hostURL    "$(UIP_HOST_URL)" `
      --username   "$(UIP_USERNAME)" `
      --password   "$(UIP_PASSWORD)" `
      --tenant     "$(UIP_TENANT)" `
      --environment "$(UIP_ENVIRONMENT)" `
      --folder      "$(UIP_FOLDER)"
  displayName: "UiPath: Deploy package"

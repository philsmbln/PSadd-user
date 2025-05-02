# PSadd-user
# Add-UserWithRole.ps1
# Script to add a new user and assign a specified role
# Usage: .\Add-UserWithRole.ps1 -Username "johndoe" -FullName "John Doe" -Password "SecureP@ss123" -Email "john.doe@example.com" -Role "Administrators"

param(
    [Parameter(Mandatory=$true)]
    [string]$Username,
    
    [Parameter(Mandatory=$true)]
    [string]$FullName,
    
    [Parameter(Mandatory=$true)]
    [string]$Password,
    
    [Parameter(Mandatory=$true)]
    [string]$Email,
    
    [Parameter(Mandatory=$true)]
    [string]$Role
)

# Function to validate parameters
function Validate-Parameters {
    # Check username format
    if ($Username -notmatch "^[a-zA-Z0-9._-]+$") {
        Write-Error "Username contains invalid characters. Use only letters, numbers, dots, underscores, and hyphens."
        exit 1
    }
    
    # Check email format
    if ($Email -notmatch "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$") {
        Write-Error "Invalid email format. Please provide a valid email address."
        exit 1
    }
    
    # Check if the role exists
    try {
        $roleExists = Get-LocalGroup -Name $Role -ErrorAction SilentlyContinue
        if (-not $roleExists) {
            Write-Error "The specified role '$Role' does not exist."
            Write-Host "Available roles:"
            Get-LocalGroup | ForEach-Object { Write-Host "- $($_.Name)" }
            exit 1
        }
    }
    catch {
        Write-Error "Error checking if role exists: $_"
        exit 1
    }
}

# Function to create a new local user
function Create-LocalUser {
    try {
        # Convert password to secure string
        $securePassword = ConvertTo-SecureString $Password -AsPlainText -Force
        
        # Check if user already exists
        $userExists = Get-LocalUser -Name $Username -ErrorAction SilentlyContinue
        
        if ($userExists) {
            Write-Warning "User '$Username' already exists. Skipping user creation."
            return $true
        }
        
        # Create the new user
        New-LocalUser -Name $Username -FullName $FullName -Password $securePassword -Description "Created by automated script" -AccountNeverExpires -PasswordNeverExpires $true -UserMayNotChangePassword $false
        
        # Set additional properties
        Set-LocalUser -Name $Username -EmailAddress $Email
        
        Write-Host "User '$Username' created successfully." -ForegroundColor Green
        return $true
    }
    catch {
        Write-Error "Failed to create user: $_"
        return $false
    }
}

# Function to add user to specified role/group
function Add-UserToRole {
    try {
        # Check if user is already a member of the group
        $groupMembers = Get-LocalGroupMember -Group $Role -ErrorAction SilentlyContinue
        $isAlreadyMember = $groupMembers | Where-Object { $_.Name -like "*\$Username" }
        
        if ($isAlreadyMember) {
            Write-Warning "User '$Username' is already a member of the '$Role' group."
            return $true
        }
        
        # Add user to the specified group
        Add-LocalGroupMember -Group $Role -Member $Username
        
        Write-Host "User '$Username' added to role '$Role' successfully." -ForegroundColor Green
        return $true
    }
    catch {
        Write-Error "Failed to add user to role: $_"
        return $false
    }
}

# Main script execution
try {
    # Ensure the script is running with administrative privileges
    $currentPrincipal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
    if (-not $currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
        Write-Error "This script requires administrative privileges. Please run PowerShell as Administrator."
        exit 1
    }
    
    # Validate input parameters
    Validate-Parameters
    
    # Create the user
    $userCreated = Create-LocalUser
    
    # Add user to role if user was created successfully or already exists
    if ($userCreated) {
        Add-UserToRole
    }
    
    Write-Host "Script completed." -ForegroundColor Cyan
}
catch {
    Write-Error "An unexpected error occurred: $_"
    exit 1
}

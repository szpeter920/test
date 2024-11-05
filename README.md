# Function to calculate the integer representation of an IP address for comparison
function ConvertTo-IpInt {
    param ($ip)
    $bytes = $ip.Split('.') | ForEach-Object {[int]$_}
    return ($bytes[0] -shl 24) -bor ($bytes[1] -shl 16) -bor ($bytes[2] -shl 8) -bor $bytes[3]
}

# Function to get available subnets by subtracting used subnets from an address space
function GetAvailableSubnets {
    param (
        [string]$addressSpace,   # The base address space
        [array]$usedSubnets      # Array of used subnets within that address space
    )

    $availableSubnets = @()
    $addressSpaceParts = $addressSpace.Split('/')
    $baseIp = $addressSpaceParts[0]
    $baseCidr = [int]$addressSpaceParts[1]

    # Calculate total addresses in the Address Space
    $numAddresses = [math]::Pow(2, (32 - $baseCidr))

    # Loop through potential /28 subnets
    for ($offset = 0; $offset -lt $numAddresses; $offset += 16) {
        $subnetIp = [System.Net.IPAddress]::new((ConvertTo-IpInt $baseIp) + $offset)
        $subnetCidr = "$subnetIp/28"
        
        # Check if this /28 subnet overlaps with any used subnet
        $isUsed = $false
        foreach ($usedSubnet in $usedSubnets) {
            if (DoesSubnetOverlap -subnetPrefix $subnetCidr -usedPrefix $usedSubnet) {
                $isUsed = $true
                break
            }
        }
        
        # If not used, add to available subnet list
        if (-not $isUsed) {
            $availableSubnets += $subnetCidr
        }
    }
    
    return $availableSubnets
}

# Retrieve all VNets in the subscription
$vnetList = Get-AzVirtualNetwork

foreach ($vnet in $vnetList) {
    Write-Output "Virtual Network: $($vnet.Name)"
    Write-Output "Location: $($vnet.Location)"
    Write-Output "Address Spaces: $($vnet.AddressSpace.AddressPrefixes -join ', ')"

    # Retrieve used subnet ranges
    $usedSubnets = $vnet.Subnets | ForEach-Object { $_.AddressPrefix }

    # Calculate available /28 ranges within each Address Space
    foreach ($addressSpace in $vnet.AddressSpace.AddressPrefixes) {
        Write-Output "Checking Address Space: ${addressSpace}"
        
        # Get available /28 subnets by subtracting used ranges
        $availableSubnets = GetAvailableSubnets -addressSpace $addressSpace -usedSubnets $usedSubnets
        
        # Display available /28 subnets
        if ($availableSubnets.Count -gt 0) {
            Write-Output "Available /28 Subnets in Address Space ${addressSpace}:"
            foreach ($subnet in $availableSubnets) {
                Write-Output "`t$subnet"
            }
        }
        else {
            Write-Output "No available /28 subnets in Address Space ${addressSpace}."
        }
    }
    
    Write-Output "-------------------------"
}
 
 

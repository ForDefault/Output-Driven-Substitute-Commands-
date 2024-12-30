# Output-Driven-Substitute-Commands- example using Dynamic Interface Variable Aggregation (DIVA)
##### DIVA is designed to dynamically create variables for all ETH# interfaces, including
><ETH#>_IP: Stores the interface's IP.
>
><ETH#>_MAC: Stores the interface's MAC.
>
><ETH#>_ALL: Aggregates both.

Template Design:
The purpose is both automation and scripting with the intent of using the outputs.

Step-by-Step Explanation
Tier 1 - the pieces
Tier 2 - the variable naming convention
Tier 3 - the aggregate

In many ways its best to start with the last step first. 
```
"${iface^^}_ALL=\"IP:$ip_address,MAC:$mac_address\"
```
When you see the output you want broken down into its pieces the rest is just putting it together. 



Fetch MAC Address (fetch_mac_address):

Input: Name of the interface (e.g., eth0).
```
cat /sys/class/net/"$interface_name"/address
```
Reads the MAC address directly from the system's /sys/class/net directory.

### Why start here? The MAC address is static and always available, making it a reliable first step.
Fetch IP Address (fetch_ip_address):

Input: Name of the interface (e.g., eth0).
```
ip -o -4 addr show "$interface_name" | awk '{print $4}' | cut -d/ -f1
```
ip -o -4 addr show: Lists IPv4 addresses for the specified interface.
awk '{print $4}': Extracts the relevant column with the address.
cut -d/ -f1: Removes the subnet mask, leaving just the IP.

Dynamic Interface Discovery:
```
interfaces=$(ls /sys/class/net | grep -E '^eth[0-9]+$')
```
ls /sys/class/net: Lists all network interfaces.
grep -E '^eth[0-9]+$': Filters for interfaces matching the eth# naming pattern.
Loop Through Interfaces:
```
for iface in $interfaces; do
```
Iterates over each detected ETH# interface.



Dynamic Variables:

```
echo "${iface^^}_IP=\"$ip_address\""
echo "${iface^^}_MAC=\"$mac_address\""
echo "${iface^^}_ALL=\"IP:$ip_address,MAC:$mac_address\""
```
${iface^^}: Converts the interface name
(e.g., eth0) to uppercase
(e.g., ETH0).

Error Handling:
```
[ -z "$ip_address" ] && ip_address="UNASSIGNED"
```
Ensures that interfaces without an IP are marked as "UNASSIGNED", maintaining consistent output.


```
# Fetch the MAC address for a specific interface
fetch_mac_address() {
    local interface_name="$1"
    cat /sys/class/net/"$interface_name"/address
}

# Fetch the IP address for a specific interface
fetch_ip_address() {
    local interface_name="$1"
    ip -o -4 addr show "$interface_name" | awk '{print $4}' | cut -d/ -f1
}

# Generate dynamic variables for all ETH# interfaces
generate_eth_definitions() {
    echo "Generating definitions for all ETH# interfaces..."
    local interfaces
    interfaces=$(ls /sys/class/net | grep -E '^eth[0-9]+$')

    for iface in $interfaces; do
        # Fetch the MAC and IP addresses
        local mac_address
        local ip_address
        mac_address=$(fetch_mac_address "$iface")
        ip_address=$(fetch_ip_address "$iface")

        # Handle missing IP addresses
        [ -z "$ip_address" ] && ip_address="UNASSIGNED"

        # Output individual definitions
        echo "${iface^^}_IP=\"$ip_address\""
        echo "${iface^^}_MAC=\"$mac_address\""

        # Aggregate the IP and MAC into ETH#_ALL
        echo "${iface^^}_ALL=\"IP:$ip_address,MAC:$mac_address\""
    done
}
```
```
# Call the function
generate_eth_definitions
```




# A far more comprehensive example:


```

get_ports_for_containers() {
    local container_name="$1"
    local container_var_prefix="$2"

    echo "Fetching ports for $container_name container..."

    # Host-facing ports
    local host_ports
    host_ports=$(docker inspect -f '{{range $p, $conf := .NetworkSettings.Ports}}{{$p}}:{{(index $conf 0).HostPort}}{{"\n"}}{{end}}' "$container_name" | awk -F: '{print $2}')

    # Container-internal ports
    local container_ports
    container_ports=$(docker inspect -f '{{range $p, $conf := .NetworkSettings.Ports}}{{$p}}{{"\n"}}{{end}}' "$container_name" | awk -F/ '{print $1}')

    # Assign host-facing ports and container-internal ports to variables
    eval "${container_var_prefix}_FORHOST_PORTS=($host_ports)"
    eval "${container_var_prefix}_FORCONTAINER_PORTS=($container_ports)"

    # Display for verification (optional)
    echo "${container_var_prefix}_FORHOST_PORTS: ${host_ports[*]}"
    echo "${container_var_prefix}_FORCONTAINER_PORTS: ${container_ports[*]}"
}

format_ports() {
    local container_var_prefix="$1"
    shift
    local ports=("$@")
    local formatted_ports="${container_var_prefix}_PORTS=\n"  # Initialize top-level variable

    for i in "${!ports[@]}"; do
        formatted_ports+="${container_var_prefix}_PORT_$((i + 1))=\"${ports[i]}\"\n"
    done

    echo -e "$formatted_ports"
}

capture_ports_for_both_containers() {
    # Capture host-facing and container-internal ports for CONTAINER1
    get_ports_for_containers "$CONTAINER1_NAME" "CONTAINER1"
    local container1_host_ports=("${CONTAINER1_FORHOST_PORTS[@]}")
    local container1_internal_ports=("${CONTAINER1_FORCONTAINER_PORTS[@]}")

    # Format ports for CONTAINER1
    CONTAINER1_HOST_PORTS=$(format_ports "CONTAINER1_HOST" "${container1_host_ports[@]}")
    CONTAINER1_CONTAINER_PORTS=$(format_ports "CONTAINER1_INTERNAL" "${container1_internal_ports[@]}")

    # Capture host-facing and container-internal ports for CONTAINER2
    get_ports_for_containers "$CONTAINER2_NAME" "CONTAINER2"
    local container2_host_ports=("${CONTAINER2_FORHOST_PORTS[@]}")
    local container2_internal_ports=("${CONTAINER2_FORCONTAINER_PORTS[@]}")

    # Format ports for CONTAINER2
    CONTAINER2_HOST_PORTS=$(format_ports "CONTAINER2_HOST" "${container2_host_ports[@]}")
    CONTAINER2_CONTAINER_PORTS=$(format_ports "CONTAINER2_INTERNAL" "${container2_internal_ports[@]}")

    # Output for appending to .env
        echo -e "$CONTAINER1_HOST_PORTS"
        echo -e "$CONTAINER1_CONTAINER_PORTS"
        echo -e "$CONTAINER2_HOST_PORTS"
        echo -e "$CONTAINER2_CONTAINER_PORTS"
}
```

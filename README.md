This tutorial outlines how to establish a testing and deployment environment for smart contracts, with a script that sets up accounts, performs tests, and executes multicalls.

![image](https://github.com/palmerchelsea/starknet-guide/assets/149751913/2afe0fad-a8ed-4808-b3ae-7c16d4e0b3d4)


Disclaimer: This script serves as a foundational example. Adapt it to fit your project's specific requirements.

Setup
1. Prepare the Script File

In the root folder of your project, create a file named script.sh. This will contain your script.
To make the script executable, adjust the file permissions:
bash
Copy code
chmod +x script.sh
2. Insert the Script

The script below is crafted to embody best practices for clarity, error handling, and maintainability.
Security Note: Employ environment variables instead of hardcoding private keys in your scripts. While safer, remember that environment variables are still exposed to any process on your machine and could be inadvertently leaked through logs or error messages.

#!/usr/bin/env bash

# Ensure the script stops on first error
set -e

# Global variables
file_path="$HOME/.starknet_accounts/starknet_open_zeppelin_accounts.json"
CONTRACT_NAME="HelloStarknet"
PROFILE_NAME="account1"
MULTICALL_FILE="multicall.toml"
FAILED_TESTS=false

# Addresses and Private keys as environment variables
ACCOUNT1_ADDRESS=${ACCOUNT1_ADDRESS:-"0x3f61fa3893ad0637b2ff66fed23ebbb91835aacd4f743c2347716f856438423"}
ACCOUNT2_ADDRESS=${ACCOUNT2_ADDRESS:-"0x53c615080d35defd55564488bc48c1a91d82f2d2ce6199463e095b4a4ead553"}
ACCOUNT1_PRIVATE_KEY=${ACCOUNT1_PRIVATE_KEY:-"CHANGE_ME"}
ACCOUNT2_PRIVATE_KEY=${ACCOUNT2_PRIVATE_KEY:-"CHANGE_ME"}

# Utility function to log messages
function log_message() {
    echo -e "\n$1"
}

# Step 1: Clean previous environment
if [ -e "$file_path" ]; then
    log_message "Removing existing accounts file..."
    rm -rf "$file_path"
fi

# Step 2: Define accounts for the smart contract
accounts_json=$(cat <<EOF
[
    {
        "name": "account1",
        "address": "$ACCOUNT1_ADDRESS",
        "private_key": "$ACCOUNT1_PRIVATE_KEY"
    },
    {
        "name": "account2",
        "address": "$ACCOUNT2_ADDRESS",
        "private_key": "$ACCOUNT2_PRIVATE_KEY"
    }
]
EOF
)

# Step 3: Run contract tests
echo -e "\nTesting the contract..."
testing_result=$(snforge 2>&1)
if echo "$testing_result" | grep -q "Failure"; then
    echo -e "Tests failed!\n"
    snforge
    echo -e "\nEnsure that your tests are passing before proceeding.\n"
    FAILED_TESTS=true
fi

if [ "$FAILED_TESTS" != "true" ]; then
    echo "Tests passed successfully."

    # Step 4: Create new account(s)
    echo -e "\nCreating account(s)..."
    for row in $(echo "${accounts_json}" | jq -c '.[]'); do
        name=$(echo "${row}" | jq -r '.name')
        address=$(echo "${row}" | jq -r '.address')
        private_key=$(echo "${row}" | jq -r '.private_key')

        account_creation_result=$(sncast --url http://localhost:5050/rpc account add --name "$name" --address "$address" --private-key "$private_key" --add-profile 2>&1)
        if echo "$account_creation_result" | grep -q "error:"; then
            echo "Account $name already exists."
        else
            echo "Account $name created successfully."
        fi
    done

    # Step 5: Build, declare, and deploy the contract
    echo -e "\nBuilding the contract..."
    scarb build

    echo -e "\nDeclaring the contract..."
    declaration_output=$(sncast --profile "$PROFILE_NAME" --wait declare --contract-name "$CONTRACT_NAME" 2>&1)

    if echo "$declaration_output" | grep -q "error: Class with hash"; then
        echo "Class hash already declared."
        CLASS_HASH=$(echo "$declaration_output" | sed -n 's/.*Class with hash \([^ ]*\).*/\1/p')
    else
        echo "New class hash declaration."
        CLASS_HASH=$(echo "$declaration_output" | grep -o 'class_hash: 0x[^ ]*' | sed 's/class_hash: //')
    fi

    echo "Class Hash: $CLASS_HASH"

    echo -e "\nDeploying the contract..."
    deployment_result=$(sncast --profile "$PROFILE_NAME" deploy --class-hash "$CLASS_HASH")
    CONTRACT_ADDRESS=$(echo "$deployment_result" | grep -o "contract_address: 0x[^ ]*" | awk '{print $2}')
    echo "Contract address: $CONTRACT_ADDRESS"

    # Step 6: Create and execute multicalls
    echo -e "\nSetting up multicall..."
    cat >"$MULTICALL_FILE" <<-EOM
[[call]]
call_type = 'invoke'
contract_address = '$CONTRACT_ADDRESS'
function = 'increase_balance'
inputs = ['0x1']

[[call]]
call_type = 'invoke'
contract_address = '$CONTRACT_ADDRESS'
function = 'increase_balance'
inputs = ['0x2']
EOM

    echo "Executing multicall..."
    sncast --profile "$PROFILE_NAME" multicall run --path "$MULTICALL_FILE"

    # Step 7: Query the contract state
    echo -e "\nChecking balance..."
    sncast --profile "$PROFILE_NAME" call --contract-address "$CONTRACT_ADDRESS" --function get_balance

    # Step 8: Clean up temporary files
    echo -e "\nCleaning up..."
    [ -e "$MULTICALL_FILE" ] && rm "$MULTICALL_FILE"

    echo -e "\nScript completed successfully.\n"
fi

3. Adjust the Bash Path

The shebang #!/usr/bin/env bash specifies the bash interpreter's path. If your environment requires a different version or location of bash, find its path with which bash and update the shebang accordingly.
Execution
Run the script by providing the necessary environment variables for ACCOUNT1_PRIVATE_KEY and ACCOUNT2_PRIVATE_KEY.

Example:

bash
Copy code
ACCOUNT1_PRIVATE_KEY="0x259f4329..." ACCOUNT2_PRIVATE_KEY="0xb4862b23..." ./script.sh
Considerations
The set -e directive ensures the script exits upon any command failure, bolstering the dependability of the deployment and testing procedure.
Always protect private keys and sensitive details. Prevent their exposure in logs or visible outputs.
For enhanced adaptability, consider relocating hardcoded values like account or contract names to a configuration file, streamlining future modifications and general upkeep.


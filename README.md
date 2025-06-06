# POKT Node Allocation YAML Generator

This repository contains a set of scripts to manage POKT node allocations, from account creation to configuration generation. The scripts work together in a specific sequence to set up and manage POKT nodes.

## Scripts Overview

The repository contains the following scripts that should be used in sequence:

1. `create_accounts.py`: Creates new POKT accounts with mnemonics
2. `import_operator_to_keyring.py`: Imports operator accounts into the keyring
3. `stake_operator_wallet.py`: Stakes the operator wallets
4. `fund_operator_wallets.py`: Transfers funds from owner to operator wallets
5. `generate_supplier_config.py`: Generates YAML configuration files for suppliers
6. `stake_from_supplier_config.py`: Executes the final staking step using the generated supplier configurations

## Install Dependencies
```bash
pip install -r requirements.txt
```

## Recommended Workflow
## Owner
Follow these steps in order to set up your POKT nodes:

### 1. Create Accounts
```bash
python create_accounts.py
```
This script will:
- Prompt for the number of accounts to create
- Prompt for a customer ID prefix
- Generate accounts with mnemonics
- Save account details to `pocket_accounts.csv`

### 2. Import Operator Accounts
```bash
python import_operator_to_keyring.py
```
This script will:
- Read accounts from `pocket_accounts.csv`
- Import each operator account into the keyring using pocketd
- Use the test keyring backend

### 3. Stake Operator Wallets

### Owner address needs to be in the local keyring, the account has to be funded as well
```bash
pocketd keys add owner --recover --keyring-backend=test
```

```bash
python stake_operator_wallet.py
```
This script will:
- Read wallet information from `supplier_stake_info.csv`
- Generate stake configuration files
- Execute stake commands for each operator wallet
- Requires a `.env` file with `NETWORK` variable set

### 4. Fund Operator Wallets
```bash
python fund_operator_wallets.py
```
This script will:
- Read owner and operator addresses from a CSV file
- Prompt for the amount of POKT to send to each operator
- Execute fund transfers from owner to operator addresses
- Requires a `.env` file with `NETWORK` variable set
- Uses the test keyring backend
- The CSV file should have columns: `owner_address` and `operator_address`

Note: Make sure the owner accounts have sufficient funds before running this script.

## Operator
### 5. Generate Supplier Configurations
```bash
python generate_supplier_config.py
```
This script will:
- Process `NodeAllocation.csv` for node allocations
- Use `morse_to_shannon_service_mapping.csv` for service ID mapping
- Generate YAML configuration files in the `output` directory

### 6. Stake Using Supplier Configurations
```bash
python stake_from_supplier_config.py
```
This script will:
- Prompt whether you are the owner or operator
- Read all YAML configuration files from the `output` directory
- Execute stake commands using the appropriate address (owner or operator)
- Use the test keyring backend
- Requires a `.env` file with `NETWORK` variable set

Note: This final step can be performed by either the owner or operator:
- If run as owner: Uses the owner address for staking
- If run as operator: Uses the operator address for staking
- The script will automatically select the correct address based on your role


## Required Files

### 1. Main Allocation CSV (`NodeAllocation.csv`)

The main CSV file should have the following structure:
- First 3 columns: Service ID (in format "Chain Name (Morse_Chain_Id)"), Node Type, Stake Nodes
- Remaining columns: Owner addresses (column headers should be the owner addresses)
- Values in the matrix: Number of nodes allocated (0 means no allocation)

Example:
```csv
service_id,node_type,stake_nodes,1,2,3
Avalanche (F003),HTC,1,1,0,1
Avalanche-DFK (F004),HTC,1,0,1,1
```

### 2. Service ID Mapping CSV (`morse_to_shannon_service_mapping.csv`)

Required file that maps Morse Chain IDs to Shannon Service IDs:

Required columns:
- `Morse_Chain_Id`: The 4-character Morse Chain ID (e.g., "F003")
- `Shannon_Service_id`: The corresponding Shannon Service ID (e.g., "avax")

Example:
```csv
Morse_Chain_Id,Shannon_Service_id
F003,avax
F004,avax
```

### 3. Environment Variables (`.env`)
Create a `.env` file with:
```
NETWORK=testnet  # acceptable values are (alpha, beta, main)
```

## Requirements

- Python 3.x
- Required packages (install via `pip install -r requirements.txt`):
  - pandas>=2.0.0
  - pyyaml>=6.0.1
  - cosmpy>=0.8.0
  - mnemonic>=0.20
  - hdwallet>=2.0.0
  - dotenv>=0.9.9

## File Structure

The scripts expect the following files to be present:
- `NodeAllocation.csv`: Main allocation matrix
- `morse_to_shannon_service_mapping.csv`: Service ID mapping
- `supplier_stake_info.csv`: Wallet information (generated by create_accounts.py)
- `pocket_accounts.csv`: Account details (generated by create_accounts.py)
- `.env`: Environment variables
- `sample.yml`: Template for stake configuration
- `output/*.yml`: Generated supplier configuration files (required for final staking step)

## Output

The script generates YAML files in the `output` directory, one for each owner address. Each YAML file follows this structure:

```yaml
owner_address: "1"
operator_address: "1_operator_test"  # From CSV if available, otherwise default
stake_amount: "1000000upokt"
default_rev_share_percent:
  "1": 50
  "1_operator": 50
services:
  - service_id: "avax"  # Shannon Service ID from mapping
    endpoints:
      - publicly_exposed_url: "https://relayminer1.example.com"  # From CSV if available
        rpc_type: "JSON_RPC"  # From CSV if available
    # rev_share_percent will be added for non-HTC nodes
```

## Usage

1. Prepare your CSV files:
   - Create `NodeAllocation.csv` with the main allocation matrix
   - Ensure `morse_to_shannon_service_mapping.csv` exists with the service ID mapping
   - Optionally create service detail CSV files for each owner (e.g., `1.csv`, `2.csv`, etc.)

2. Run the scripts in sequence:
```bash
python create_accounts.py
python import_operator_to_keyring.py
python stake_operator_wallet.py
python fund_operator_wallets.py
python generate_supplier_config.py
python stake_from_supplier_config.py  # Run as either owner or operator
```

3. Check the `output` directory for generated YAML files

## Notes

- The script will convert Morse Chain IDs to Shannon Service IDs in the output YAML files
- If a service detail CSV file is not provided for an operator, default values will be used
- For non-HTC node types, a custom rev_share_percent will be set (0% owner, 100% operator)
- The script will create an `output` directory if it doesn't exist
- Any errors reading service detail CSV files will be reported but won't stop the script
- If the service mapping file cannot be loaded, the script will use original service IDs
- Make sure to have `pocketd` installed and accessible in your PATH
- The scripts use the test keyring backend by default
- All generated files are saved in the current directory
- The supplier configuration files are generated in the `output` directory
- Any errors during execution will be reported but won't stop the script unless critical
- The final staking step (`stake_from_supplier_config.py`) can be run by either the owner or operator, with the script automatically using the appropriate address
- Make sure to have `pocketd` installed and accessible in your PATH 
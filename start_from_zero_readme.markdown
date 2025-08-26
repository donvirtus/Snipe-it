# Snipe-IT Setup Guide for Gatekeeper Prototype

This guide provides step-by-step instructions to set up and configure the Snipe-IT asset management system for the Gatekeeper Prototype project from scratch. It assumes you are working in the project directory `/home/donvirtus/Documents/MILYAR/pt_internusa_food/gatekeeper_prototype` and have already run `docker compose down -v` to clear all data.

## Prerequisites
- Docker and Docker Compose installed.
- Python 3 and required dependencies (`pip install -r requirements.txt`).
- RFID reader configured with IP (e.g., `30.1.1.12`).
- Access to a web browser for Snipe-IT UI.

## Step-by-Step Setup

### 1. Clean Up (Optional, If Needed)
Ensure all containers are stopped and volumes are cleared (already done if you ran `docker compose down -v`).

```bash
# Navigate to docker folder
cd /home/donvirtus/Documents/MILYAR/pt_internusa_food/gatekeeper_prototype/docker

# Stop containers and remove volumes (data loss is permanent)
docker compose down -v

# Clean local uploads/db folders if they exist
rm -rf ./dbdata ./uploads || true

# Return to project root
cd ..
```

### 2. Configure Environment Files
Set up the `.env` files for both the Docker setup and the Gatekeeper scripts.

```bash
# Set APP_URL in docker/.env to localhost
sed -i "s|^APP_URL=.*|APP_URL=http://localhost:8000|" docker/.env

# Set SNIPE_API_URL in root .env for Gatekeeper scripts
sed -i "s|^SNIPE_API_URL=.*|SNIPE_API_URL=http://localhost:8000/api/v1|" .env

# Set READER_IP to your RFID reader IP (e.g., 30.1.1.12)
sed -i "s|^READER_IP=.*|READER_IP=30.1.1.12|" .env
```

### 3. Start Snipe-IT with Docker
Pull and start the Docker containers.

```bash
# Navigate to docker folder
cd docker

# Pull latest images
docker compose pull

# Start containers in detached mode
docker compose up -d

# Monitor logs to ensure services are ready
docker compose logs -f app
```

### 4. Set Up Snipe-IT via UI
1. Open `http://localhost:8000` in your browser.
2. Follow the setup wizard to create an admin account if prompted.
3. Log in as admin, go to **Profile → API Tokens → New Token**, and copy the generated token.
4. Save the token in the root `.env` file (do not commit to git):

```bash
# Manually edit .env or use:
sed -i "s|^API_TOKEN=.*|API_TOKEN=PASTE_YOUR_ADMIN_TOKEN_HERE|" .env
```

### 5. Configure Custom Fields, Fieldsets, and Models
You can configure these manually via the UI or automate them using a script (if available).

#### Manual Setup (via UI)
1. **Custom Fields** (Admin → Custom Fields):
   - **RFID EPC**: Text Box, Database Column: `rfid_epc`, Required/Unique as needed, check "Show in list".
   - **Gatekeeper Allow**: Checkbox, Database Column: `gatekeeper_allow`, value `1` = Allow (optional `0` = Deny).
   - **ES Item Code**: Text, Database Column: `es_item_code` (optional).
2. **Fieldsets** (Admin → Fieldsets):
   - Create a fieldset named "Gatekeeper RFID" and add the three fields above.
3. **Manufacturers, Categories, Status Labels, Models** (Admin):
   - Create "Generic" (Manufacturer), "Gatekeeper" (Category), "Ready to Deploy" (Status Label), and "RFID Generic" (Model).
   - Associate the "Gatekeeper RFID" fieldset with the "RFID Generic" model.

#### Automated Setup (if script available)
If you have `setup_internusa_food.py` in your repository, run it to automate the setup (requires `API_TOKEN` in `.env`):

```bash
python3 setup_internusa_food.py --mode setup
```

### 6. Verify API Configuration
Check the environment variables and test the API connection.

```bash
# Check environment variables used by scripts
python3 - <<'PY'
from env_utils import load_env, os
load_env()
print("SNIPE_API_URL=", os.getenv("SNIPE_API_URL"))
print("API_TOKEN set? ", bool(os.getenv("API_TOKEN")))
PY

# Test API endpoint
curl -s -H "Authorization: Bearer $(grep ^API_TOKEN .env | cut -d= -f2-)" "http://localhost:8000/api/v1/hardware?limit=1" | jq .
```

If the `curl` command fails, verify container status, port availability, or token validity.

### 7. Seed or Import Assets
#### Option 1: Seed Dummy Assets
Create dummy assets for testing.

```bash
# Set environment variables
export SNIPE_API_URL=http://localhost:8000/api/v1
export API_TOKEN="PASTE_TOKEN"

# Run seeder (e.g., create 30 dummy assets)
python3 setup_internusa_food.py --mode setup --assets 30
# OR use dedicated seeder tool
python3 tools/snipeit_seed_dummy_assets.py --count 30
```

#### Option 2: Import Real EPCs via CSV
Use a CSV file to import real EPC data.

```bash
# Import from CSV
python3 tools/snipeit_update_epc_allow.py --csv tools/assets_epc_for_import.csv
```

#### Option 3: Update Single Asset
Update a specific asset's EPC and Gatekeeper flag.

```bash
python3 tools/snipeit_update_epc_allow.py --asset_tag GK-00001 --epc E2801170000002190F942343 --allow 1
```

### 8. Verify EPC and Gatekeeper Flag
Check if an EPC is correctly configured.

```bash
python3 tools/gatekeeper_quickcheck.py E2801170000002190F942343
```

### 9. Run Gatekeeper Bridge or GUI
Ensure `READER_IP` is set in `.env` before running.

```bash
# Run headless bridge
python3 gatekeeper_bridge.py

# Run GUI (optional, requires PySide6)
pip install PySide6
python3 gatekeeper_gui.py
```

### 10. Troubleshooting
- **Check container status**:
  ```bash
  cd docker
  docker compose ps
  ```
- **View logs**:
  ```bash
  docker compose logs app
  docker compose logs db
  ```
- **API errors (401/403)**: Generate a new token in the UI and update `.env`.
- **Reader connectivity issues**:
  ```bash
  nc -vz 30.1.1.12 2022
  ```
  Ensure the reader IP and port are reachable.

## Important Notes
- Running `docker compose down -v` permanently deletes the database volume. Ensure you have an SQL backup if needed.
- Store the API token in `.env` and avoid committing it to git.
- For CSV imports or automated setups, ensure `API_TOKEN` and `SNIPE_API_URL` are set in `.env`.

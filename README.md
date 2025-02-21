# ArgoCD-test

import requests
import json
import datetime
import sys
import logging
import os

# Logging setup
logging.basicConfig(
    filename="cymulate_to_splunk.log",  # Log file name
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s"
)

# API and Splunk configurations
CYMULATE_API_URL = "https://api.cymulate.com/v1/logs/activity"
CYMULATE_API_KEY = "YOUR_CYMULATE_API_KEY"

SPLUNK_HEC_URL = "https://splunk-server:8088/services/collector"
SPLUNK_HEC_TOKEN = "YOUR_SPLUNK_HEC_TOKEN"

# State file to track processed logs
STATE_FILE = "processed_logs.json"


def load_processed_logs():
    """Load processed log IDs from a local file."""
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, "r") as file:
            return json.load(file)
    return []


def save_processed_logs(log_ids):
    """Save processed log IDs to a local file."""
    with open(STATE_FILE, "w") as file:
        json.dump(log_ids, file)


def get_cymulate_logs():
    """Fetch activity logs from Cymulate API."""
    headers = {
        "Authorization": f"Bearer {CYMULATE_API_KEY}",
        "Content-Type": "application/json"
    }

    try:
        response = requests.get(CYMULATE_API_URL, headers=headers)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        logging.error(f"Error fetching logs from Cymulate: {e}")
        sys.exit(1)


def send_log_to_splunk(log):
    """Send a single log entry to Splunk."""
    headers = {
        "Authorization": f"Splunk {SPLUNK_HEC_TOKEN}",
        "Content-Type": "application/json"
    }

    # Use original timestamp if available, otherwise current UTC time
    log_time = log.get("timestamp", datetime.datetime.utcnow().isoformat())

    payload = {
        "event": log,
        "sourcetype": "cymulate:activity",
        "time": log_time
    }

    try:
        response = requests.post(SPLUNK_HEC_URL, headers=headers, data=json.dumps(payload), verify=False)
        response.raise_for_status()
        logging.info(f"Log sent to Splunk successfully: {log.get('id', 'N/A')}")
    except requests.exceptions.RequestException as e:
        logging.error(f"Error sending log to Splunk: {e}")


def main():
    logging.info("Starting log ingestion from Cymulate to Splunk...")

    # Load processed log IDs
    processed_logs = load_processed_logs()

    # Fetch logs from Cymulate
    logs = get_cymulate_logs()
    if logs:
        logging.info(f"Fetched {len(logs)} logs from Cymulate.")

        # Process only new logs
        new_logs = [log for log in logs if log.get("id") not in processed_logs]
        if new_logs:
            logging.info(f"Processing {len(new_logs)} new logs.")
            for log in new_logs:
                send_log_to_splunk(log)
                processed_logs.append(log.get("id"))

            # Update state file
            save_processed_logs(processed_logs)
        else:
            logging.info("No new logs to process.")
    else:
        logging.warning("No logs fetched from Cymulate.")


if __name__ == "__main__":
    main()

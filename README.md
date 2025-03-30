# full_db_backup_tool

echo "# Database Backup Tool" > README.md
nano db_backup_tool.py

import argparse
import subprocess
import os
import tarfile
import logging
import boto3
import requests
import mysql.connector
import psycopg2
import pymongo
import sqlite3

# Logging Configuration
logging.basicConfig(filename="backup.log", level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# Database Connection
def get_connection(db_type, host, port, user, password, db_name):
    try:
        if db_type == "mysql":
            return mysql.connector.connect(host=host, user=user, password=password, database=db_name, port=port)
        elif db_type == "postgres":
            return psycopg2.connect(host=host, user=user, password=password, dbname=db_name, port=port)
        elif db_type == "mongodb":
            return pymongo.MongoClient(host, port)
        elif db_type == "sqlite":
            return sqlite3.connect(db_name)
        else:
            raise ValueError("Unsupported database type")
    except Exception as e:
        logging.error(f"Database connection error: {e}")
        return None

# Backup Function
def perform_backup(args):
    backup_file = f"{args.name}_backup.sql"

    try:
        if args.db == "mysql":
            cmd = f"mysqldump -h {args.host} -P {args.port} -u {args.user} -p{args.password} {args.name} > {backup_file}"
        elif args.db == "postgres":
            cmd = f"PGPASSWORD={args.password} pg_dump -h {args.host} -p {args.port} -U {args.user} -d {args.name} -F c -f {backup_file}"
        elif args.db == "mongodb":
            cmd = f"mongodump --host {args.host} --port {args.port} --db {args.name} --out {backup_file}"
        elif args.db == "sqlite":
            cmd = f"sqlite3 {args.name} .dump > {backup_file}"
        else:
            print("Unsupported database type")
            return

        subprocess.run(cmd, shell=True, check=True)

        # Compress the backup
        compressed_file = f"{backup_file}.tar.gz"
        with tarfile.open(compressed_file, "w:gz") as tar:
            tar.add(backup_file)
        os.remove(backup_file)

        log_backup(args.db, compressed_file, "SUCCESS")
        print(f"Backup successful: {compressed_file}")

        if args.storage == "s3":
            upload_to_s3(compressed_file, "your-s3-bucket-name")

    except Exception as e:
        log_backup(args.db, None, f"FAILED: {e}")
        print(f"Backup failed: {e}")

# Restore Function
def perform_restore(args):
    try:
        with tarfile.open(args.file, "r:gz") as tar:
            tar.extractall()
        backup_file = args.file.replace(".tar.gz", "")

        if args.db == "mysql":
            cmd = f"mysql -u root -p < {backup_file}"
        elif args.db == "postgres":
            cmd = f"PGPASSWORD=root psql -U postgres -d {backup_file} -f {backup_file}"
        elif args.db == "mongodb":
            cmd = f"mongorestore --db {backup_file} {backup_file}"
        elif args.db == "sqlite":
            print("SQLite does not support direct restore from dump file.")
            return

        subprocess.run(cmd, shell=True, check=True)
        print("Restore successful.")

    except Exception as e:
        print(f"Restore failed: {e}")

# Logging Function
def log_backup(db_type, file_name, status):
    logging.info(f"Database: {db_type}, File: {file_name}, Status: {status}")

# Slack Notification Function
def send_slack_notification(message):
    try:
        webhook_url = "YOUR_SLACK_WEBHOOK_URL"
        payload = {"text": message}
        requests.post(webhook_url, json=payload)
    except Exception as e:
        print(f"Slack notification failed: {e}")

# AWS S3 Upload Function
def upload_to_s3(file_name, bucket_name):
    try:
        s3 = boto3.client("s3")
        s3.upload_file(file_name, bucket_name, file_name)
        print(f"Uploaded {file_name} to S3 bucket {bucket_name}")
    except Exception as e:
        print(f"Failed to upload to S3: {e}")

# CLI Setup
def main():
    parser = argparse.ArgumentParser(description="Database Backup and Restore CLI Tool")
    subparsers = parser.add_subparsers(dest="command", help="Available commands")

    # Backup Command
    backup_parser = subparsers.add_parser("backup", help="Perform database backup")
    backup_parser.add_argument("--db", required=True, choices=["mysql", "postgres", "mongodb", "sqlite"], help="Database type")
    backup_parser.add_argument("--host", default="localhost", help="Database host")
    backup_parser.add_argument("--port", type=int, help="Database port")
    backup_parser.add_argument("--user", help="Database username")
    backup_parser.add_argument("--password", help="Database password")
    backup_parser.add_argument("--name", required=True, help="Database name")
    backup_parser.add_argument("--storage", choices=["local", "s3"], default="local", help="Storage option")

    # Restore Command
    restore_parser = subparsers.add_parser("restore", help="Restore a database from a backup")
    restore_parser.add_argument("--db", required=True, choices=["mysql", "postgres", "mongodb", "sqlite"], help="Database type")
    restore_parser.add_argument("--file", required=True, help="Backup file to restore from")

    args = parser.parse_args()

    if args.command == "backup":
        perform_backup(args)
    elif args.command == "restore":
        perform_restore(args)
    else:
        parser.print_help()

if __name__ == "__main__":
    main()

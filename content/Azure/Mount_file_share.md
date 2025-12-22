---
title: Montagem do Azure Fileshare no Linux
tags:
  - Linux
  - Azure
enableToc: true
---
```shell
#!/bin/bash

# Configuration variables (these should be set in a separate config file or environment variables)
STORAGE_ACCOUNT_NAME="${STORAGE_ACCOUNT_NAME:-your_storage_account_name}"
STORAGE_ACCOUNT_PASS="${STORAGE_ACCOUNT_PASS:-your_storage_account_key}"
STORAGE_SHARE_URI="devsftppocunmanageddisk.file.core.windows.net/sftp"
STORAGE_ACCOUNT_MOUNTPOINT="/mnt/$STORAGE_ACCOUNT_NAME"
STORAGE_CREDENCIAL_FILE="/etc/$STORAGE_ACCOUNT_NAME.cred"

# Function to display usage
usage() {
    echo "Usage: $0 {mount|unmount|cleanup}"
    echo "  mount   : Mount the Azure file share"
    echo "  unmount : Unmount the Azure file share"
    echo "  cleanup : Remove credentials and fstab entry"
    exit 1
}

# Function to check if running as root
check_root() {
    if [ "$EUID" -ne 0 ]; then
        echo "Error: This script must be run as root."
        exit 1
    fi
}

# Function to validate required variables
validate_vars() {
    if [ -z "$STORAGE_ACCOUNT_NAME" ] || [ -z "$STORAGE_ACCOUNT_PASS" ]; then
        echo "Error: STORAGE_ACCOUNT_NAME and STORAGE_ACCOUNT_PASS must be set."
        echo "Set them as environment variables or edit the script."
        exit 1
    fi
}

# Function to create credentials file
create_credentials() {
    echo "Creating credentials file at $STORAGE_CREDENCIAL_FILE..."
    echo -e "username=$STORAGE_ACCOUNT_NAME\npassword=$STORAGE_ACCOUNT_PASS" > "$STORAGE_CREDENCIAL_FILE"
    if [ $? -ne 0 ]; then
        echo "Error: Failed to create credentials file."
        exit 1
    fi
    chmod 600 "$STORAGE_CREDENCIAL_FILE"
    echo "Credentials file created and permissions set."
}

# Function to add fstab entry
add_fstab_entry() {
    # Check if entry already exists in /etc/fstab
    if grep -q "$STORAGE_SHARE_URI" /etc/fstab; then
        echo "fstab entry already exists for $STORAGE_SHARE_URI."
        return 0
    fi

    echo "Adding fstab entry for $STORAGE_SHARE_URI..."
    echo "//$STORAGE_SHARE_URI $STORAGE_ACCOUNT_MOUNTPOINT cifs nofail,vers=3.0,credentials=$STORAGE_CREDENCIAL_FILE,dir_mode=0777,file_mode=0777,serverino" >> /etc/fstab
    if [ $? -ne 0 ]; then
        echo "Error: Failed to add fstab entry."
        exit 1
    fi
    echo "fstab entry added."
}

# Function to create mount point
create_mountpoint() {
    if [ ! -d "$STORAGE_ACCOUNT_MOUNTPOINT" ]; then
        echo "Creating mount point at $STORAGE_ACCOUNT_MOUNTPOINT..."
        mkdir -p "$STORAGE_ACCOUNT_MOUNTPOINT"
        if [ $? -ne 0 ]; then
            echo "Error: Failed to create mount point."
            exit 1
        fi
        echo "Mount point created."
    else
        echo "Mount point $STORAGE_ACCOUNT_MOUNTPOINT already exists."
    fi
}

# Function to mount the file share
mount_fileshare() {
    # Check if already mounted
    if mountpoint -q "$STORAGE_ACCOUNT_MOUNTPOINT"; then
        echo "File share is already mounted at $STORAGE_ACCOUNT_MOUNTPOINT."
        return 0
    fi

    echo "Mounting file share at $STORAGE_ACCOUNT_MOUNTPOINT..."
    mount -t cifs "//$STORAGE_SHARE_URI" "$STORAGE_ACCOUNT_MOUNTPOINT" -o vers=3.0,credentials="$STORAGE_CREDENCIAL_FILE",dir_mode=0777,file_mode=0777,serverino
    if [ $? -ne 0 ]; then
        echo "Error: Failed to mount file share."
        exit 1
    fi
    echo "File share mounted successfully."
}

# Function to unmount the file share
unmount_fileshare() {
    if ! mountpoint -q "$STORAGE_ACCOUNT_MOUNTPOINT"; then
        echo "File share is not mounted at $STORAGE_ACCOUNT_MOUNTPOINT."
        return 0
    fi

    echo "Unmounting file share from $STORAGE_ACCOUNT_MOUNTPOINT..."
    umount "$STORAGE_ACCOUNT_MOUNTPOINT"
    if [ $? -ne 0 ]; then
        echo "Error: Failed to unmount file share."
        exit 1
    fi
    echo "File share unmounted successfully."
}

# Function to clean up (remove credentials and fstab entry)
cleanup() {
    echo "Cleaning up configuration..."

    # Unmount if mounted
    unmount_fileshare

    # Remove credentials file
    if [ -f "$STORAGE_CREDENCIAL_FILE" ]; then
        echo "Removing credentials file $STORAGE_CREDENCIAL_FILE..."
        rm -f "$STORAGE_CREDENCIAL_FILE"
        if [ $? -ne 0 ]; then
            echo "Error: Failed to remove credentials file."
            exit 1
        fi
        echo "Credentials file removed."
    fi

    # Remove fstab entry
    if grep -q "$STORAGE_SHARE_URI" /etc/fstab; then
        echo "Removing fstab entry for $STORAGE_SHARE_URI..."
        sed -i "\|$STORAGE_SHARE_URI|d" /etc/fstab
        if [ $? -ne 0 ]; then
            echo "Error: Failed to remove fstab entry."
            exit 1
        fi
        echo "fstab entry removed."
    fi

    # Remove mount point if empty
    if [ -d "$STORAGE_ACCOUNT_MOUNTPOINT" ]; then
        rmdir "$STORAGE_ACCOUNT_MOUNTPOINT" 2>/dev/null
        if [ $? -eq 0 ]; then
            echo "Mount point $STORAGE_ACCOUNT_MOUNTPOINT removed."
        else
            echo "Mount point $STORAGE_ACCOUNT_MOUNTPOINT not removed (directory not empty or error occurred)."
        fi
    fi

    echo "Cleanup completed."
}

# Main logic
if [ $# -ne 1 ]; then
    usage
fi

check_root
validate_vars

case "$1" in
    mount)
        create_credentials
        create_mountpoint
        add_fstab_entry
        mount_fileshare
        ;;
    unmount)
        unmount_fileshare
        ;;
    cleanup)
        cleanup
        ;;
    *)
        usage
        ;;
esac

exit 0
```
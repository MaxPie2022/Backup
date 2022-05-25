set -eu
set -o pipefail

#!/bin/sh

# Download the latest backup from
# Heroku and gzip it

heroku pg:backups:download --output=/tmp/pg_backup.dump --app $APP_NAME
gzip /tmp/pg_backup.dump

# Encrypt the gzipped backup file
# using GPG passphrase

gpg --yes --batch --passphrase=$PG_BACKUP_PASSWORD -c /tmp/pg_backup.dump.gz

# Remove the plaintext backup file

rm /tmp/pg_backup.dump.gz

# Generate backup filename based
# With the timestamp on the current date

BACKUP_FILE_NAME="heroku-backup-$(date '+%Y-%m-%d_%H.%M').gpg"
# BACKUP_FILE_NAME="heroku-backup-$(date '+%Y-%m-%d_%H.%M').dump"

# Make sure to use the UTC
# date for S3 signature!

DATE=`date -R -u`

S3_PATH="${BACKUP_S3_BUCKET}/${BACKUP_FILE_NAME}"

# Generate S3 signature needed
# to upload file to the bucket

MD5="$(openssl md5 -binary < "/tmp/pg_backup.dump.gz.gpg" | base64)"
CONTENT_TYPE="application/octet-stream"
S3_STRING="PUT\n$MD5\napplication/octet-stream\n${DATE}\n${S3_PATH}"

S3_SIGNATURE="$(printf "PUT\n$MD5\n$CONTENT_TYPE\n$DATE\n/$S3_PATH" | openssl sha1 -binary -hmac "$BACKUP_S3_SECRET" | base64)"
# Upload the file to S3 using
# the signature auth header

curl -X PUT -T "/tmp/pg_backup.dump.gz.gpg" \
  -H "Host: ${BACKUP_S3_BUCKET}.s3-ap-us-east-1.amazonaws.com" \
  -H "Date: ${DATE}" \
  -H "Content-Type: application/octet-stream" \
  -H "Content-MD5: $MD5" \
  -H "Authorization: AWS ${BACKUP_S3_KEY}:${S3_SIGNATURE}" \
  https://${BACKUP_S3_BUCKET}.s3-ap-southeast-1.amazonaws.com/${BACKUP_FILE_NAME}

# Remove the encrypted backup file

rm /tmp/pg_backup.dump.gz.gpg

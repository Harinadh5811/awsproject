# CREATING SRT FILES USING AWS TRANSCRIBE AND INTEGRATION WITH TELEGRAM

## Telegram Bot Python Code (Python anywhere)

```python
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters
import requests
import os

TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN'  # Replace with your actual bot token

def start(update, context):
    context.bot.send_message(chat_id=update.effective_chat.id, text="Hello! Send me an audio file in WAV format to upload.")

def handle_audio(update, context):
    if update.message.document and update.message.document.mime_type == "audio/vnd.wave":
        try:
            # Get the file ID of the audio
            file_id = update.message.document.file_id
            # Get the file path from Telegram servers
            file_path = context.bot.get_file(file_id).file_path

            # Download the audio file
            audio_url = file_path
            filename = str(update.effective_chat.id) + '---' + os.path.basename(audio_url)
            audio_file = requests.get(audio_url)
            audio_file.raise_for_status()  # Raise an exception for non-2xx status codes from Telegram API

            # Prepare the PUT request to upload to S3
            bucket = 'input-bucket-name'
            api_url = f'apigatewayinvokeurl/{bucket}/{filename}'

            try:
                response = requests.put(api_url, data=audio_file.content)
                response.raise_for_status()  # Raise an exception for non-2xx status codes

                # Handle successful upload
                context.bot.send_message(chat_id=update.effective_chat.id, text='Audio uploaded successfully!\nPlease wait while it gets processed....')
            except requests.exceptions.RequestException as e:
                # Handle errors from API request
                context.bot.send_message(chat_id=update.effective_chat.id, text=f'An error occurred while uploading the audio: {e}')

        except Exception as e:  
            context.bot.send_message(chat_id=update.effective_chat.id, text=f'An unexpected error occurred: {e}')
    else:
        context.bot.send_message(chat_id=update.effective_chat.id, text='Please send an audio file in WAV format.')
        

def main():
    updater = Updater(token=TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(MessageHandler(Filters.all, handle_audio))

    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()

```


## AWS Lambda Python Code to create a transcription job
```python
import boto3
import os
import uuid

def lambda_handler(event, context):
    # Retrieve the S3 bucket and object information from the event
    s3_bucket = event['Records'][0]['s3']['bucket']['name']
    s3_key = event['Records'][0]['s3']['object']['key']

    # Extract the desired filename without modification at the beginning
    original_filename_prefix = s3_key.split("---")[0]  # Get the part before "---"
    desired_filename = f"{original_filename_prefix}---result"

    print(f"{desired_filename} and {s3_bucket}")

    # Generate a unique identifier for the transcription job (optional)
    job_name = str(uuid.uuid4())

    # Create an Amazon Transcribe client
    transcribe_client = boto3.client('transcribe')

    # Set the S3 URI for the input audio file
    s3_uri = f"s3://{s3_bucket}/{s3_key}"

    # Start the transcription job using the desired filename with .srt extension
    response = transcribe_client.start_transcription_job(
        TranscriptionJobName=job_name,  # Optional job name
        LanguageCode='en-US',
        Media={
            'MediaFileUri': s3_uri
        },
        OutputBucketName='output-bucket-name',
        OutputKey=f"{desired_filename}.srt",  # Use desired filename with .srt extension
        Subtitles={
            'Formats': ['srt']  # Enable SRT output
        }
    )

    print(f"Transcription job started with name: {job_name}")  # Print job name if used

    return {
        'statusCode': 200,
        'body': response
    }

```
## AWS Lambda Python Code to send srt file to telegram
```python
import json
import boto3
import http.client
import time

s3_client = boto3.client('s3')
sent_files = {}

def get_s3_presigned_url(bucket_name, object_key):
    expiration = 3600  
    response = s3_client.generate_presigned_url(
        ClientMethod='get_object',
        Params={'Bucket': bucket_name, 'Key': object_key},
        ExpiresIn=expiration
    )
    return response

def send_telegram_message(bot_token, chat_id, message_text):
    host = "api.telegram.org"
    path = f"/bot{bot_token}/sendMessage"

    connection = http.client.HTTPSConnection(host)
    headers = {"Content-type": "application/json"}
    payload = {
        "chat_id": chat_id,
        "text": message_text,
        "parse_mode": "HTML"  
    }

    try:
        connection.request("POST", path, body=json.dumps(payload), headers=headers)
        response = connection.getresponse()
        response_data = response.read().decode("utf-8")
        print(response_data)
    except Exception as e:
        print(f"Error sending message: {e}")
    finally:
        connection.close()

def lambda_handler(event, context):
    s3_key = event['Records'][0]['s3']['object']['key']
    telegram_chat_id = s3_key.split("---")[0]  
    telegram_bot_token = "YOUR TELEGRAM BOT TOKEN"  

    for record in event['Records']:
        bucket_name = record['s3']['bucket']['name']
        object_key = record['s3']['object']['key']

        if object_key.endswith('.srt'):
            if object_key in sent_files:
                print("File already sent, skipping.")
                continue

            object_size = s3_client.head_object(Bucket=bucket_name, Key=object_key)['ContentLength']
            if object_size < 5120:
                presigned_url = get_s3_presigned_url(bucket_name, object_key)
                message_text = f"Yahooooo......! Your file is converted into SRT Click the following Hyperlink...😁😁\n\n<a href='{presigned_url}'>Result.srt</a>"
                send_telegram_message(telegram_bot_token, telegram_chat_id, message_text)
                sent_files[object_key] = True

                return "SRT file pre-signed URL sent to Telegram."

    return "No suitable SRT file found."

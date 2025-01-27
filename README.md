# WhatsBot

<div align="left">
      <a href="https://www.youtube.com/shorts/zAwtykOesns">
         <img src="logo.png" height=300 width=300>
      </a>
</div>

> How DALL-E 2 imagines the project

### Important note:
This is to explain how to use the verification token in Whatsapp.

1- This service must be running on a https server (secure). So, you can not run it in local. An alternative is to use ngrok service https://dashboard.ngrok.com/ 
2- Once you get your application running in localhost (http://localhost:5000) --> then using ngrok, the IP will be something like https://721c-95-16-18-23.ngrok-free.app
3- You go you the configurations of your Whatsapp app, and put the link under the callback URL. IMPORTANT: add /webhooks to the URL. The URL will be: https://721c-95-16-18-23.ngrok-free.app/webhooks
4- In the verification token, put any secure token, but it MUST be the same token as your VERIFY_TOKEN in the .env file
5- Then, you click on verify and save. In this case, Whatsapp will send a GET request to your server to validate the toke. If all Ok, the link will be validated.
6- Activate sending messages to be able to send and receive whatapp messages.

![img.png](img.png)

## Getting-Started
To create your own WhatsApp chatbot you all you need to do is:
1. Creata an [OpenAI API](https://openai.com/product) account and generate an API key
2. Create a [Meta developer](https://developers.facebook.com/) account
3. Create a [Whatsapp Business app](https://developers.facebook.com/apps) and generate a Token
4. [Remix the app on Glitch](https://glitch.com/~whatsapp-openai-webhook-python) and set environmental variables
5. Link your glitch app as [webhook](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks) to your WhatsApp app
6. Last but not least: star [this project](https://github.com/gustavz/whatsbot) on GitHub ❤️


## TL;DR

> How ChatGPT describes the project

This program is a webhook implemented in python and flask. It receives incoming messages from WhatsApp, processes and responds with the help of OpenAI API. The messages can be either text or audio, and the response is generated with OpenAI's GPT-3.5 model. The critical functions of the program include handling incoming messages of different types, converting audio files to text and sending responses back to the user.

## Learnables

- Python programming language
- OpenAI API
- WhatsApp Cloud API
- Webhooks
- `Flask` web framework
- `PyDub` for audio manipulation
- `SpeechRecognition` for speech to text conversion
- `Soundfile` for working with audio files

## Code Deep-Dive

The program consists of several functions to handle different aspects of the process.

As first step `verify` verifies that the webhook connection is trustworthy.<br>
It does so by checking that the verfifcation token and correct mode are set in the request.

```python
def verify(request):
    # Parse params from the webhook verification request
    mode = request.args.get("hub.mode")
    token = request.args.get("hub.verify_token")
    challenge = request.args.get("hub.challenge")
    # Check if a token and mode were sent
    if mode and token:
        # Check the mode and token sent are correct
        if mode == "subscribe" and token == verify_token:
            # Respond with 200 OK and challenge token from the request
            print("WEBHOOK_VERIFIED")
            return challenge, 200
        else:
            # Responds with '403 Forbidden' if verify tokens do not match
            print("VERIFICATION_FAILED")
            return jsonify({"status": "error", "message": "Verification failed"}), 403
    else:
        # Responds with '400 Bad Request' if verify tokens do not match
        print("MISSING_PARAMETER")
        return jsonify({"status": "error", "message": "Missing parameters"}), 400
```

`handle_message` processes incoming WhatsApp messages and does error handling

```python
def handle_message(request):
    # Parse Request body in json format
    body = request.get_json()
    print(f"request body: {body}")

    try:
        # info on WhatsApp text message payload:
        # https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/payload-examples#text-messages
        if body.get("object"):
            if (
                body.get("entry")
                and body["entry"][0].get("changes")
                and body["entry"][0]["changes"][0].get("value")
                and body["entry"][0]["changes"][0]["value"].get("messages")
                and body["entry"][0]["changes"][0]["value"]["messages"][0]
            ):
                handle_whatsapp_message(body)
            return jsonify({"status": "ok"}), 200
        else:
            # if the request is not a WhatsApp API event, return an error
            return (
                jsonify({"status": "error", "message": "Not a WhatsApp API event"}),
                404,
            )
    # catch all other errors and return an internal server error
    except Exception as e:
        print(f"unknown error: {e}")
        return jsonify({"status": "error", "message": str(e)}), 500
```

`handle_whatsapp_message` is the main function that handles incoming messages of different types, and based on the type of the message, it calls the relevant function.

```python
def handle_whatsapp_message(body):
    message = body["entry"][0]["changes"][0]["value"]["messages"][0]
    if message["type"] == "text":
        message_body = message["text"]["body"]
    elif message["type"] == "audio":
        audio_id = message["audio"]["id"]
        message_body = handle_audio_message(audio_id)
    response = make_openai_request(message_body, message["from"])
    send_whatsapp_message(body, response)
```

For example, if the message is an audio file, it calls `handle_audio_message` to convert the file to text.

```python
def handle_audio_message(audio_id):
    audio_url = get_media_url(audio_id)
    audio_bytes = download_media_file(audio_url)
    audio_data = convert_audio_bytes(audio_bytes)
    audio_text = recognize_audio(audio_data)
    message = (
      "Please summarize the following message in its original language "
      f"as a list of bullet-points: {audio_text}"
    )
    return message
```

`convert_audio_bytes` is another critical function that converts the audio file into a format that can be processed by the `speech_recognition` library.<br>
The function first converts the OGG file into WAV format and then converts it into an AudioData object.

```python
def convert_audio_bytes(audio_bytes):
    ogg_audio = pydub.AudioSegment.from_ogg(io.BytesIO(audio_bytes))
    ogg_audio = ogg_audio.set_sample_width(4)
    wav_bytes = ogg_audio.export(format="wav").read()
    audio_data, sample_rate = sf.read(io.BytesIO(wav_bytes), dtype="int32")
    sample_width = audio_data.dtype.itemsize
    print(f"audio sample_rate:{sample_rate}, sample_width:{sample_width}")
    audio = sr.AudioData(audio_data, sample_rate, sample_width)
    return audio
```

The `make_openai_request` function makes a request to the OpenAI API and generates a response based on the user's message and the previous conversation log.<br> 
It uses the `openai.ChatCompletion.create` method to generate the response.

```python
def make_openai_request(message, from_number):
    try:
        message_log = update_message_log(message, from_number, "user")
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=message_log,
            temperature=0.7,
        )
        response_message = response.choices[0].message.content
        print(f"openai response: {response_message}")
        update_message_log(response_message, from_number, "assistant")
    except Exception as e:
        print(f"openai error: {e}")
        response_message = "Sorry, the OpenAI API is currently overloaded or offline. Please try again later."
        remove_last_message_from_log(from_number)
    return response_message
```

The `send_whatsapp_message` function creates the Cloud API request sends the response back to the user via WhatsApp.

```python
def send_whatsapp_message(body, message):
    value = body["entry"][0]["changes"][0]["value"]
    phone_number_id = value["metadata"]["phone_number_id"]
    from_number = value["messages"][0]["from"]
    headers = {
        "Authorization": f"Bearer {whatsapp_token}",
        "Content-Type": "application/json",
    }
    url = "https://graph.facebook.com/v15.0/" + phone_number_id + "/messages"
    data = {
        "messaging_product": "whatsapp",
        "to": from_number,
        "type": "text",
        "text": {"body": message},
    }
    response = requests.post(url, json=data, headers=headers)
    print(f"whatsapp message response: {response.json()}")
    response.raise_for_status()
```
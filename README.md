# Alexa Voice Service Client

[![code-climate-image]][code-climate]
[![circle-ci-image]][circle-ci]
[![codecov-image]][codecov]

**Python Client for Alexa Voice Service (AVS)**

---

## Installation
```sh
pip install avs_client
```

## Usage ##

### File audio ###
```py
from avs_client import AlexaVoiceServiceClient

alexa_client = AlexaVoiceServiceClient(
    client_id='my-client-id',
    secret='my-secret',
    refresh_token='my-refresh-token',
)
alexa_client.connect()  # authenticate and other handshaking steps
with open('./tests/resources/alexa_what_time_is_it.wav', 'rb') as f:
    for i, directive in enumerate(alexa_client.send_audio_file(f)):
        if directive.name in ['Speak', 'Play']:
            with open(f'./output_{i}.mp3', 'wb') as f:
                f.write(directive.audio_attachment)
```

Now listen to `output.wav` and Alexa should tell you the time.

### Microphone audio
```py
from io import BytesIO

from avs_client import AlexaVoiceServiceClient
import pyaudio


def callback(in_data, frame_count, time_info, status):
    buffer.write(in_data)
    return (in_data, pyaudio.paContinue)

p = pyaudio.PyAudio()
stream = p.open(
    format=pyaudio.paInt16,
    channels=1,
    rate=16000,
    input=True,
    stream_callback=callback,
)

alexa_client = AlexaVoiceServiceClient(
    client_id='my-client-id',
    secret='my-secret',
    refresh_token='my-refresh-token',
)

buffer = BytesIO()
try:
    stream.start_stream()
    print('listening. Press CTRL + C to exit.')
    alexa_client.connect()
    for i, directive in enumerate(alexa_client.send_audio_file(buffer)):
        if directive.name in ['Speak', 'Play']:
            with open(f'./output_{i}.mp3', 'wb') as f:
                f.write(directive.audio_attachment)
finally:
    stream.stop_stream()
    stream.close()
    p.terminate()
```

### Voice Request Lifecycle

An Alexa command may relate to a previous command e.g,

[you] "Alexa, open the magic door"
[Alexa] "...do you want to go to the dark forest or turn back?"
[you] "Go to the dark forest"
[Alexa] "...it's creepy. Keep going or turn back?"
[you] "Keep going"
[Alexa] "...it's getting darker."

This request lifecycle can be managed by passing in a dialog request id:

```py
import io

from avs_client import AlexaVoiceServiceClient
from avs_client.avs_client import helpers
from pydub import AudioSegment
from pydub.playback import play


alexa_client = AlexaVoiceServiceClient(
    client_id='my-client-id',
    secret='my-secret',
    refresh_token='my-refresh-token',
)
alexa_client.connect()  # authenticate and other handshaking steps

input_file_names = [
    './tests/resources/alexa_open_magic_door.wav',
    './tests/resources/alexa_go_to_dark_forest.wav',
    './tests/resources/alexa_keep_going.wav',
]

dialog_request_id = helpers.generate_unique_id()
for input_file_name in input_file_names:
    with open(input_file_name, 'rb') as f:
        for directive in alexa_client.send_audio_file(f, dialog_request_id=dialog_request_id):
            if directive.name in ['Speak', 'Play']:
                track = AudioSegment.from_mp3(io.BytesIO(directive.audio_attachment))
                play(track)

```


## Authentication

To use AVS you must first have a [developer account](http://developer.amazon.com). Then register your product [here](https://developer.amazon.com/avs/home.html#/avs/products/new). Choose "Application" under "Is your product an app or a device"?

The client requires your `client_id`, `secret` and `refresh_token`:

| client kwarg    | Notes |
| --------------- | ------------------------------------- |
| `client_id`     | Retrieve by clicking on the your product listed [here](https://developer.amazon.com/avs/home.html#/avs/home) |
| `secret`        | Retrieve by clicking on the your product listed [here](https://developer.amazon.com/avs/home.html#/avs/home) |
| `refresh_token` | You must generate this. [See below](#refresh-token) |

### Refresh token ###

You will need to login to Amazon via a web browser to get your refresh token.

To enable this first go [here](https://developer.amazon.com/avs/home.html#/avs/home) and click on your product to set some security settings under `Security Profile`:

| setting             | value                            |
| ------------------- | ---------------------------------|
| Allowed Origins     | http://localhost:9000           |
| Allowed Return URLs | http://localhost:9000/callback/ |

Note what you entered for Product ID under Product Information, as this will be used as the device-type-id (case sensitive!)

Then run:

```sh
python ./avs_client/refreshtoken/serve.py \
    --device-type-id=enter-device-type-id-here \
    --client-id=enter-client-id-here \
    --client-secret=enter-client-secret-here
```

Follow the on-screen instructions shown at `http://localhost:9000` in your web browser. 
On completion Amazon will return your `refresh_token` - which you will require to [send audio](#file-audio) or [recorded voice](#microphone-audio).

## Steaming audio to AVS
`alexa_client.send_audio_file` streaming uploads a file-like object to AVS for great latency. The file-like object can be an actual file on your filesystem, an in-memory BytesIo buffer containing audio from your microphone, or even audio streaming from [your browser over a websocket in real-time](https://github.com/richtier/alexa-browser-client).

AVS requires the audio data to be 16bit Linear PCM (LPCM16), 16kHz sample rate, single-channel, and little endian.

## Persistent AVS connection

Calling `alexa_client.connect()` creates a persistent connection to AVS. The connection may get forcefully closed due to inactivity. Keep open by calling `alexa_client.conditional_ping()`:

```py
import threading


def ping_avs():
    while True:
        alexa_client.conditional_ping()

ping_thread = threading.Thread(target=ping_avs)
ping_thread.start()
```

You will only need this if you intend to run the process for more than five minutes. [More information](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/docs/managing-an-http-2-connection).

## Unit test ##

To run the unit tests, call the following commands:

```sh
make test_requirements
make test
```

## Other projects ##

This library is used by [alexa-browser-client](https://github.com/richtier/alexa-browser-client), which allows you to talk to Alexa from your browser.

[code-climate-image]: https://codeclimate.com/github/richtier/alexa-voice-service-client/badges/gpa.svg
[code-climate]: https://codeclimate.com/github/richtier/alexa-voice-service-client

[circle-ci-image]: https://circleci.com/gh/richtier/alexa-voice-service-client/tree/master.svg?style=svg
[circle-ci]: https://circleci.com/gh/richtier/alexa-voice-service-client/tree/master

[codecov-image]: https://codecov.io/gh/richtier/alexa-voice-service-client/branch/master/graph/badge.svg
[codecov]: https://codecov.io/gh/richtier/alexa-voice-service-client


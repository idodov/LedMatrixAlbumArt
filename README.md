# LED Matrix Album Art Display 
EspHoMaTriXv2 with AppDaemon

## Introduction

This script takes the album art of the currently playing song and transforms it into a vibrant 8x8 pixel art display on your LED matrix screen. It's a fun and unique way to visualize your music and add a touch of color to your listening experience.

## Functionality

**Based on AppDaemon (add-on)** the script listens to the `entity_picture` attribute of a specified media player entity in Home Assistant. When the attribute changes (indicating a new song is playing), the script fetches the new album art image from the provided URL.

The image is then processed in the following way:

1. The image is converted to RGB format and resized to fit within a 64x64 pixel bounding box, maintaining its aspect ratio.
2. If the image is not already in RGB format, it is converted.
3. The image is resized again to an 8x8 pixel resolution, effectively transforming it into pixel art.
4. The RGB values of the image data are converted to a numpy array.
5. The RGB values are mapped to 16-bit color values, suitable for display on an LED matrix that supports a color depth of 65535 colors.
6. The 2D numpy array is flattened into a 1D list, creating a linear sequence of pixel color values.

The resulting pixel color values are then set as an attribute of a sensor entity in Home Assistant, ready to be displayed on your LED matrix screen.

This script is designed to be robust and efficient, with error handling to prevent crashes if there's an issue with the URL or image processing.

While there are multiple methods to transmit the image information to the LED matrix in Home Assistant, I personally found that creating a **dedicated sensor** that updates each time a music track is played is the most straightforward approach. This makes the information readily accessible whenever needed.

## Configuration
To configure AppDaemon for this script, you need to ensure certain system packages and Python packages are installed. Here's the configuration you should have:

```yaml
system_packages: []
python_packages:
  - requests
  - numpy
  - pillow
init_commands: []
```

In this configuration: `python_packages` includes the `requests`, `numpy`, and `pillow` packages. `requests` is used for making HTTP requests to fetch the album art image, `numpy` is used for processing the image data, and `pillow` is used for opening and manipulating the image.
Please ensure that this configuration is correctly set in your AppDaemon configuration file. 


In the AppDaemon apps directory, **create a new file named** `imageprocessor.py` and paste the script into it.

### Modify the script
- **Match the name of your media player entity.** In my case, the name is `media_player.era300`.
- Confirm that you can access `http://homeassistant.local:8123`. If you're unable to do so, you'll need to replace it with your local IP address in the script. 
Once you've made the necessary changes, save the script.
```yaml
import numpy as np
from PIL import Image
import requests
from io import BytesIO
import appdaemon.plugins.hass.hassapi as hass

class ImageProcessor(hass.Hass):

 def initialize(self):
        # Define the entity and attribute names
        self.entity_name = "media_player.era300" #CHANGE THE VALUE TO YOUR ENTITY NAME!
        self.attribute_name = "entity_picture"
        self.ha_url = "http://homeassistant.local:8123"

        # Listen to the attribute of the entity
        self.listen_state(self.process_image, self.entity_name, attribute=self.attribute_name)

    def process_image(self, entity, attribute, old, new, kwargs):
        if new is not None:
            try:
                # Get the image from the URL
                response = requests.get(f"{self.ha_url}{new}")
                img = Image.open(BytesIO(response.content))

                # Convert the image to RGB and resize it
                img = img.convert("RGB")
                img.thumbnail((64, 64), Image.Resampling.LANCZOS)

                # Ensure the image is in RGB format
                if img.mode != "RGB":
                    img = img.convert("RGB")

                # Resize the image to 8x8
                img = img.resize((8, 8))

                # Convert the image data to a numpy array
                img_data = np.array(img)

                # Map the RGB values to 16-bit color values
                img_data = img_data * 257
                led_matrix = ((img_data[:, :, 0] >> 3) << 11) | ((img_data[:, :, 1] >> 2) << 5) | (img_data[:, :, 2] >> 3)

                # Flatten the numpy array to a 1D list
                led_matrix = led_matrix.flatten().tolist()

                # Log the LED matrix
                new_attributes = {
                    "album_art": led_matrix,
                }
                # Create sensor.8x8_pic in HA to store the data. 
                # Use: {{ strate_attr('sensor.8x8_pic', 'album_art') }} 
                self.set_state("sensor.8x8_pic", state="on", attributes=new_attributes)
                self.log(f"Album Art: {led_matrix}")

            except Exception as e:
                self.log(f"Error processing image: {e}")
```
Next, navigate to `apps.yaml` and append the following lines:
```yaml
imageprocessor:
  module: imageprocessor
  class: ImageProcessor
```
Example Data
```log
Album Art: [22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 18785, 20801, 22849, 22881, 22849, 26977, 7556, 28005, 60130, 39297, 18753, 39618, 39618, 47842, 7620, 28133, 15974, 54982, 63431, 27105, 22849, 31201, 39618, 35490, 22977, 18785, 43746, 22849, 22849, 22849, 22881, 22881, 22849, 22849, 22881, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849]
```
Example Ulanzi Service
```yaml
service: esphome.ulanzi_rainbow_bitmap_small
data:
  default_font: true
  lifetime: >-
    {{ (state_attr('media_player.era300', 'media_duration') | float(default=0) / 60) | int(default=1) if state_attr('media_player.era300', 'media_duration') is not none else 4 }}
  screen_time: 40
  text: >-
    {{ state_attr('media_player.era300', 'media_artist') }} - {{
    state_attr('media_player.era300', 'media_title') }}
  icon: "{{ state_attr('sensor.8x8_pic', 'album_art') }}"
enabled: true
```
Exmaple Result type: string (This template listens for the following state changed events: Entity: `media_player.era300` and Entity: `sensor.8x8_pic`)
```log
service: esphome.ulanzi_rainbow_bitmap_small
data:
  default_font: true
  lifetime: >-
    3
  screen_time: 40
  text: >-
    Pink Floyd - Brain Damage
  icon: "[22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 18785, 20801, 22849, 22881, 22849, 26977, 7556, 28005, 60130, 39297, 18753, 39618, 39618, 47842, 7620, 28133, 15974, 54982, 63431, 27105, 22849, 31201, 39618, 35490, 22977, 18785, 43746, 22849, 22849, 22849, 22881, 22881, 22849, 22849, 22881, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849, 22849]"
enabled: true
```

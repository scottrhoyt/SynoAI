# SynoAI
A Synology Surveillance Station notification system utilising DeepStack AI, inspired by Christopher Adams' [sssAI](https://github.com/Christofo/sssAI) implementation.

The aim of the solution is to reduce the noise generated by Synology Surveillance Station's motion detection by routing all motion events via a [Deepstack](https://deepstack.cc/) docker image to look for particular objects, e.g. people.

While sssAI is a great solution, it is hamstrung by the Synology notification system to send motion alerts. Due to the delay between fetching the snapshot, processing the image using the AI and requesting the alert, it means that the image attached to the Synology notification is sometimes 5-10 seconds after the motion alert was originally triggered.

SynoAI aims to solve this problem by side-stepping the Synology notifications entirely by allowing other notification systems to be used.

## Buy Me A Coffee! :coffee:

I made this application mostly for myself in order to improve upon Christopher Adams' original idea and don't expect anything in return. However, if you find it useful and would like to buy me a coffee, feel free to do it at [__Buy me a coffee! :coffee:__](https://buymeacoff.ee/djdd87). This is entirely optional, but would be appreciated! Or even better, help supported this project by contributing changes such as expanding the supported notification systems (or even AIs).

## Features
* Triggered via an Action Rule from Synology Surveillance Station
* Works using the camera name and requires no technical knowledge of the Surveillance Station API in order to retrieve the unique camera ID
* Uses an AI for object/person detection
* Produces an output image with highlighted objects using the original image at the point of motion detection
* Sends notification(s) at the point of notification with the processed image attached.

## Supported AIs
* [Deepstack](https://deepstack.cc/)

## Supported Notifications
### [Pushbullet](https://www.pushbullet.com/)
```json
{
  "Type": "Pushbullet",
  "ApiKey": "0.123456789"
}
```

### Webhook 
The webhook notification will POST an image to the specified URL with a specified field name.

```json
{
  "Url": "http://servername/resource",
  "Method": "POST",
  "Field": "image"
}
```
* Url [required]: The URL to send the image to
* Method [optional] (Default: ```POST```): The HTTP method to use, e.g. POST, PUT
* Field [optional] (Default: ```image```): The field name of the image in the POST data.

### HomeAssistant
Integration with HomeAssistant can be achieved using the [Push](https://www.home-assistant.io/integrations/push/) integration by following the instructions on that page by calling the HomeAssistant webhook using the SynoAI Webhook notification. 

## Caveats
* SynoAI still relies on Surveillance Station triggering the motion alerts
* Looking for an object, such as a car on a driveway, will continually trigger alerts if that object is in view of the camera when Surveillance Station detects movement, e.g. a tree blowing in the wind.

## Configuration
The configuration instructions below are primarily aimed at running SynoAI in a docker container on DSM (Synology's operating system). Docker will be required anyway as Deepstack is assumed to be setup inside a Docker container. It is entirely possible to run SynoAI on a webserver instead, or to install it on a Docker instance that's not running on your Synology NAS, however that is outside the scope of these instructions. Additionally, the configuration of the third party notification systems (e.g. generating a Pushbullet API Key) is outside the scope of these instructions and can be found on the respective applications help guides.

The top level steps are:
* Setup the Deepstack Docker image on DSM
* Setup the SynoAI image on DSM
* Add Action Rules to Synology Surveillance Station's motion alerts in order to trigger the SynoAI API.

### Configure Deepstack
The following instructions explain how to set up the Deepstack image using the Docker app built into DSM. Before continuing, you'll need to obtain a *free* API key from [Deepstack](https://deepstack.css). 

* Download the deepquestai/deepstack image by either;
  * Searching the registry for deepquestai/deepstack
  * Choose the tag cpu-x6-beta, or noavx; this is dependent on the capabilities of your NAS.
* Run the image
* Enter a name for the image, e.g. deepstack
* Edit the advanced settings
* Enable auto restarts
* On the port settings tab;
  * Enter a port mapping to port 5000 from an available port on your NAS, e.g. 83
* On the Environment tab;
  * Set MODE: Low
  * Set VISION-DETECTION: True
* Accept the advnaced settings and then run the image
* Open a webbrowser and go to the Deepstack page by navigating to http://{YourIP}:{YourDeepstackPort}
* If you've set everything up successfully then you will be able to enter your API key in here and move onto the next step.
   
### Configure SynoAI
The following instructions explain how to set up the SynoAI image using the Docker app built into DSM. For docker-compose, see the example file in the src, or in the documentation below.

* Create a folder called synoai (this will contain your Captures directory and appsettings.json)
* Put your appsettings.json file in the folder
* Create a folder called Captures 
* Open Docker in DSM
* Download the djdd87/synoai:latest image by either;
  * Searching the registry for djdd87/synoai
  * Going to the image tab and;
    * Add > Add from URL
    * Enter https://hub.docker.com/r/djdd87/synoai
* Run the image
* Enter a name for the image, e.g. synoai
* Edit the advanced settings
* Enable auto restarts
* On the volumes tab;
   * Add a file mapping from your appsettings.json to /app/appsettings.json
   * Add a folder mapping from your captures directory to /app/Captures (optional)
* On the port settings tab;
   * Enter a port mapping to port 80 from an available port on your NAS, e.g. 8080

### Create Action Rules
The next step is to configure actions inside Surveillance Station that will call the SynoAI API. 

* Open up Surveillance Station
* Open Action Rules
* Create a new rule and enter;
  * Name: A name for the action e.g. Trigger SynoAI - Driveway
  * Rule type: Triggered (Default)
  * Action type: Interruptible (Default)
* Click next to open the Event tab and enter;
  * Event source: Camera
  * Device: Your camera, e.g. Driveway
  * Event: Motion Detected
* Click next to open the Action tab and enter;
  * Action device: Webhook
  * URL: http://{YourIP}:{YourPort}/Camera/{CameraName}, e.g. http://10.0.0.10:8080/Camera/Driveway, where
    * YourIP: Is the IP of your NAS, or the Docker server where SynoAI is deployed
    * YourPort: The port that the SynoAI image is listening on as you configured above.
    * CameraName: The name of the camera, e.g. Driveway
  * Username: Blank
  * Password: Blank
  * Method: GET
  * Times: 1
* Click test send and if everything is set up correctly, then you'll get a green tick
* Click next and the action will now be created.

### Summary

Congratulations, you should now have a trigger calling the SynoAI API for your camera every time Surveillance Station detects motion. In order to set up multiple cameras, just create a new Action Rule for each camera.

Note that SynoAI is still reliant on Surveillance Station detecting the motion, so this will need some tuning on your part. However, it's now possible to up the sensitivity and avoid false-positives as SynoAI will only notify you (via your preferred notification system/app) when an object is detected, e.g. a Person.

## Docker
SynoAI can be installed as a docker image, which is [available from DockerHub](https://hub.docker.com/r/djdd87/synoai).

### Docker Configuration
The image can be pulled using the Docker cli by calling:
```
docker pull djdd87/synoai:latest
```
To run the image a volume must be specified to map your appsettings.json file. Additionally a port needs mapping to port 80 in order to trigger the API. Optionally, the Captures directory can also be mapped to easily expose all the images output from SynoAI.

```
docker run 
  -v /path/appsettings.json:/app/appsettings.json 
  -v /path/captures:/app/Captures 
  -p 8080:80 djdd87/synoai:latest
```

### Docker-Compose
```yaml
version: '3.4'

services:
  synoai:
    image: djdd87/synoai:latest
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8080:80"
    volumes:
      - /docker/synoai/captures:/app/Captures
      - /docker/synoai/appsettings.json:/app/appsettings.json
```

## Example appsettings.json
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },

  "Url": "http://10.0.0.10:5000",
  "User": "SynologyUser",
  "Password": "SynologyPassword",

  "AI": {
    "Type": "DeepStack",
    "Url": "http://10.0.0.10:83",
    "MinSizeX": 100,
    "MinSizeY": 100
  },

  "Notifiers": [{
    "Type": "Pushbullet",
    "ApiKey": "0.123456789"
  }],

  "Cameras": [
    {
      "Name": "Driveway",
      "Types": [ "Person", "Car" ],
      "Threshold": 45
    },
    {
      "Name": "BackDoor",
      "Types": [ "Person" ],
      "Threshold": 30
    }
  ]
}
```

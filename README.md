# POC - Azure IOT Hub with Device Update installing custom package

Proof of Concept - how to use custom debian/ubuntu packages with Azure IOT Device Update

## Prerequisites

- Machine/VM/Device with Ubuntu 18.04 (at least 2GB RAM).
- Azure subscription.
- Azure IOT Hub. [Steps](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-through-portal)
- Enable Device Update for Azure IOT Hub. [Steps](https://learn.microsoft.com/en-us/azure/iot-hub-device-update/create-device-update-account?tabs=portal)
- Register an Azure Edge Device in Azure IOT Hub. [steps](https://learn.microsoft.com/en-us/azure/iot-edge/quickstart-linux?view=iotedge-1.4#register-an-iot-edge-device)
- Get familiar with Azure IOT Device Update. [Docs](https://docs.microsoft.com/en-us/azure/iot-hub-device-update/overview)
- Get familiar with how a sample .NET 7.0. application is packed for Ubuntu 18.04 [Ref](https://github.com/UlisesGascon/poc-packaging-dot-net-core-service-for-ubuntu)
- Get familiar with how the custom packages are distributed using a Github Repository. [Ref](https://github.com/UlisesGascon/poc-custom-package-debian-repository)

### Notes

- Ubuntu 18.04 is used as a base OS for this POC. It is possible to use other OS, but it is not covered in this POC. Higher versions of Ubuntu are not supported by Azure IOT Device Update at [the moment of writing this POC](https://github.com/Azure/iot-hub-device-update/discussions/408).
- In this poc, we will use a custom package that is hosted on a public github repository. It is possible to use a private repository, but it is not covered in this POC.
- The Ubuntu 18.04 machine should have internet access to be able to download packages from the internet.
- The Ubuntu 18.04 machine should have access to Azure IOT Hub to be able to download packages from Azure IOT Device Update.
- The Ubuntu 18.04 used as Operating System is the server version. It is possible to use the desktop version, but it is not covered in this POC.


## Steps

### Configure Azure IOT Edge Runtime

1. Install IoT Edge runtime on device
```bash
wget https://packages.microsoft.com/config/ubuntu/18.04/multiarch/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```

2. Install Container Runtime
```bash
sudo apt-get update; \
  sudo apt-get install moby-engine
```

3. Use local logging driver
```bash
sudo nano /etc/docker/daemon.json
```

Replace the content with:
```json
{
  "registry-mirrors": ["http://localhost:5000"]
}
```

```bash
sudo systemctl restart docker
```

4. Install IoT Edge runtime
```bash
sudo apt-get update; \
   sudo apt-get install aziot-edge
```

5. Provision the device with its cloud identity
```bash
sudo iotedge config mp --connection-string 'PASTE_DEVICE_CONNECTION_STRING_HERE'
```
```bash
sudo iotedge config apply
```
```bash
sudo cat /etc/aziot/config.toml
```

### Deploy Azure IOT Edge Modules

Follow this [steps](https://learn.microsoft.com/en-US/azure/iot-edge/how-to-provision-single-device-linux-symmetric?view=iotedge-1.4&preserve-view=true&tabs=azure-portal%2Cubuntu#deploy-modules)

> To deploy your IoT Edge modules, go to your IoT hub in the Azure portal, then:
> 1. Select **Devices** from the IoT Hub menu.
> 2. Select your device to open its page.
> 3. Select the **Set Modules** tab.
> 4. Since we want to deploy the IoT Edge default modules (edgeAgent and edgeHub), we don't need to add any modules to this pane, so select **Review + create** at the bottom.
> 5. You see the JSON confirmation of your modules. Select **Create** to deploy the modules.

It will generate a deployment manifest similar to this one:

```json
{
    "modulesContent": {
        "$edgeAgent": {
            "properties.desired": {
                "schemaVersion": "1.1",
                "runtime": {
                    "type": "docker",
                    "settings": {}
                },
                "systemModules": {
                    "edgeAgent": {
                        "settings": {
                            "image": "mcr.microsoft.com/azureiotedge-agent:1.4"
                        },
                        "type": "docker"
                    },
                    "edgeHub": {
                        "restartPolicy": "always",
                        "settings": {
                            "image": "mcr.microsoft.com/azureiotedge-hub:1.4",
                            "createOptions": "{\"HostConfig\":{\"PortBindings\":{\"443/tcp\":[{\"HostPort\":\"443\"}],\"5671/tcp\":[{\"HostPort\":\"5671\"}],\"8883/tcp\":[{\"HostPort\":\"8883\"}]}}}"
                        },
                        "status": "running",
                        "type": "docker"
                    }
                },
                "modules": {}
            }
        },
        "$edgeHub": {
            "properties.desired": {
                "schemaVersion": "1.1",
                "storeAndForwardConfiguration": {
                    "timeToLiveSecs": 7200
                },
                "routes": {}
            }
        }
    }
}
```

Note: It might take a few seconds (up to 2 mins) for the portal to reflect the deployments status correctly. Sometimes one of the modules seems to be in status error.

### Verify Modules deployment

**IOT Edge Runtime**
```bash
sudo iotedge system status
```
```bash
sudo iotedge system logs
```
```bash
sudo iotedge check
```

**Docker containers**
```bash
docker ps
```

### Install device update agent on Ubuntu

1. Install device update agent
```bash
sudo apt-get install deviceupdate-agent 
```

2. Configure device update agent
```bash
sudo nano /etc/adu/du-config.json
```

Replace the content with:

```json
{
  "schemaVersion": "1.1",
  "aduShellTrustedUsers": [
    "adu",
    "do"
  ],
  "iotHubProtocol": "mqtt",
  "manufacturer": "Contoso",
  "model": "Video",
  "agents": [
    {
      "name": "corretto-machine-edge-01",
      "runas": "adu",
      "connectionSource": {
        "connectionType": "string",
        "connectionData": "PASTE_DEVICE_CONNECTION_STRING_HERE"
      },
      "manufacturer": "Contoso",
      "model": "Video"
    }
  ]
}
```

3. Restart the device update agent
```
sudo systemctl restart deviceupdate-agent
```

4. Check the status of the device update agent
```
sudo systemctl status deviceupdate-agent
```

### Add a tag to your device

Follow [this steps](https://learn.microsoft.com/en-US/azure/iot-hub-device-update/device-update-ubuntu-agent#add-a-tag-to-your-device)

> 1. Sign in to the Azure portal and go to the IoT hub.
> 2. On the left pane, under **Devices**, find your IoT Edge device and go to the **Device twin** or module twin.
> 3. In the module twin of the Device Update agent module, delete any existing Device Update tag values by setting them to null. If you're using Device identity with Device Update agent, make these changes on the device twin.
> 4. Add a new Device Update tag value.


In our case the tag is:

```json
"tags": {
    "ADUGroup": "test-corretto"
},
```

### Add Custom Package Debian Repository Origin

1. Add [POC - Custom Package Debian Repository](https://github.com/UlisesGascon/poc-custom-package-debian-repository) public key to your trusted keys

```bash
wget -qO - https://raw.githubusercontent.com/UlisesGascon/poc-custom-package-debian-repository/main/PUBLIC.KEY | sudo apt-key add -
```

2. Add [POC - Custom Package Debian Repository](https://github.com/UlisesGascon/poc-custom-package-debian-repository) repository as source list

```bash
echo "deb https://raw.githubusercontent.com/UlisesGascon/poc-custom-package-debian-repository/main/ bionic main" | sudo tee /etc/apt/sources.list.d/ulisesgascon.list
```

3. Update the package list

```bash
sudo apt-get update
```

Note: this step can be automated using ADU

### Import and deploy the update 

1. Download the files to your machine:
- [corretto-1.0.2-manifest.json](corretto-1.0.2-manifest.json)
- [corretto-initialization-2.0.edge.importmanifest.json](corretto-initialization-2.0.edge.importmanifest.json)

2. Import the update [Following this steps](https://learn.microsoft.com/en-US/azure/iot-hub-device-update/device-update-ubuntu-agent#import-the-update), but use the files downloaded in the previous step and not the ones mentioned in the example.

3. Check that the group `test-corretto` is listed, and that the update is available. [Following this steps](https://learn.microsoft.com/en-US/azure/iot-hub-device-update/device-update-ubuntu-agent#view-device-groups)

4. Deploy the update [Following this steps](https://learn.microsoft.com/en-US/azure/iot-hub-device-update/device-update-ubuntu-agent#deploy-the-update)

### Check that the update was applied


1. Check that `demoapi` is running.

```bash
systemctl -l status demoapi.service
```

2. Check that `demoapi` services is responsive.

```bash
curl http://localhost:5000/WeatherForecast
```

The expected response is similar to:

```json
[{"date":"2023-08-04","temperatureC":4,"temperatureF":39,"summary":"Warm"},{"date":"2023-08-05","temperatureC":-14,"temperatureF":7,"summary":"Scorching"},{"date":"2023-08-06","temperatureC":-14,"temperatureF":7,"summary":"Warm"},{"date":"2023-08-07","temperatureC":-4,"temperatureF":25,"summary":"Chilly"},{"date":"2023-08-08","temperatureC":7,"temperatureF":44,"summary":"Cool"}]
```


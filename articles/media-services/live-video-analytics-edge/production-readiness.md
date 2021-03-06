---
title: Production readiness and best practices - Azure
description: This article provides guidance on how to configure and deploy the Live Video Analytics on IoT Edge module in production environments.
ms.topic: conceptual
ms.date: 04/27/2020

---
# Production readiness and best practices

This article provides guidance on how to configure and deploy the Live Video Analytics on IoT Edge module in production environments. You should also review [Prepare to deploy your IoT Edge solution in production](../../iot-edge/production-checklist.md) article on preparing your IoT Edge solution. 

> [!NOTE]
> You should consult your organizations’ IT departments on aspects related to security.

## Running the module as a local user

When you deploy the Live Video Analytics on IoT Edge module to an edge device, by default it runs with elevated privileges. When you do this, if you check the logs on the module (`sudo iotedge logs {name-of-module}`), you will see the following:

```
!! production readiness: user accounts – Warning
       LOCAL_USER_ID and LOCAL_GROUP_ID environment variables are not set. The program will run as root!
       For optimum security, make sure to set LOCAL_USER_ID and LOCAL_GROUP_ID environment variables to a non-root user and group.
       See https://aka.ms/lva-iot-edge-prod-checklists-user-accounts for more information.
```

The sections below discuss how you can address the above warning.

### Creating and using a local user account

You can and should run the Live Video Analytics on IoT Edge module in production using an account with as few privileges as possible. The following commands, for example, show how you can create a local user account on a Linux VM:

```
sudo groupadd -g 1010 localuser
sudo adduser --home /home/edgeuser --uid 1010 -gid 1010 edgeuser
```

Next, in the deployment manifest, you can set the LOCAL_USER_ID and LOCAL_GROUP_ID environment variables to that non-root user and group:

```
"lvaEdge": {
"version": "1.0",
…
"env": {
    "LOCAL_USER_ID": 
    {
        "value": "1010"
    },
    "LOCAL_GROUP_ID": {
        "value": "1010"
    }
}
},
…
```

### Granting permissions to device storage

The Live Video Analytics on IoT Edge module requires the ability to write files to the local file system when:

* Using a module twin property [[applicationDataDirectory](module-twin-configuration-schema.md#module-twin-properties)], where you should specify a directory on the local file system for storing configuration data.
* Using a media graph to record video to the cloud, the module requires the use of a directory on the edge device as a cache (see [Continuous video recording](continuous-video-recording-concept.md) article for more information).
* [Recording to a local file](event-based-video-recording-concept.md#video-recording-based-on-events-from-other-sources),where you should specify a file path for the recorded video.

If you intend to make use of any of the above, you should ensure that the above user account has access to the relevant directory. 
Consider applicationDataDirectory for example. You can create a directory on the edge device and link device storage to module storage. 

```
sudo mkdir /var/local/mediaservices
sudo chown -R edgeuser /var/local/mediaservices
```

Next, in the create options for the edge module in the deployment manifest, you can add a binds setting mapping the directory (“var/local/mediaservices/”) above to a directory in the module (such as “/var/lib/azuremediaservices/”). And you would use the latter directory as the value for the applicationDataDirectory.

```
"lvaEdge": {
    "version": "1.0",
    "type": "docker",
    "status": "running",
    "restartPolicy": "always",
    "settings": {
        "image": "mcr.microsoft.com/media/live-video-analytics:1.0",
        "createOptions": "{\"HostConfig\":{\"LogConfig\":{\"Type\":\"\",\"Config\":{\"max-size\":\"10m\",\"max-file\":\"10\"}},\"Binds\":[\"/var/local/mediaservices/:/var/lib/azuremediaservices/\"]}}"
    },
    "env": {
        "LOCAL_USER_ID": 
        {
            "value": "1010"
        },
        "LOCAL_GROUP_ID": {
            "value": "1010"
        }
    }
    },
    …
    
    "lvaEdge": {
    "properties.desired": {
    "applicationDataDirectory": "/var/lib/azuremediaservices",
    …
    }
}
```

If you look at the sample media graphs for the quickstart and tutorials, such as [continuous video recording](continuous-video-recording-tutorial.md), you will note that the media cache directory (localMediaCachePath) uses a subdirectory under applicationDataDirectory. This is the recommended approach, since the cache contains transient data.

### Naming video assets or files

Media graphs allows for creation of assets in the cloud or mp4 files on the edge. Media assets can be generated by [continuous video recording](continuous-video-recording-tutorial.md) or by [event-based video recording](event-based-video-recording-tutorial.md). While these assets and files can be named as you want, the recommended naming structure for continuous video recording-based media asset is "&lt;anytext&gt;-${System.GraphTopologyName}-${System.GraphInstanceName}". As an example, you can set the assetNamePattern on the asset sink as follows:

```
"assetNamePattern": "sampleAsset-${System.GraphTopologyName}-${System.GraphInstanceName}
```

For event-based video recording-generated assets, the recommended naming pattern is "&lt;anytext&gt;-${System.DateTime}". The system variable ensures that assets don not get overwritten if events happen at same time. As an example, you can set the assetNamePattern on the Asset Sink as follows:

```
"assetNamePattern": "sampleAssetFromEVR-LVAEdge-${System.DateTime}"
```

If you are running multiple instances of the same graph, you can use the graph topology name and instance name to differentiate. As an example, you can set the assetNamePattern on the asset sink as follows:

```
"assetNamePattern": "sampleAssetFromEVR-${System.GraphTopologyName}-${System.GraphInstanceName} -${System.DateTime}"
```

For event-based video recording-generated mp4 video clips on the edge, the recommended naming pattern should include DateTime and for multiple instances of the same graph recommend using the system variables GraphTopologyName and GraphInstanceName. As an example, you can set filePathPattern on file sink as follows: 

```
"filePathPattern": "/var/media/sampleFilesFromEVR-${fileSinkOutputName}-${System.DateTime}"
```

Or 

```
"filePathPattern": "/var/media/sampleFilesFromEVR-${fileSinkOutputName}--${System.GraphTopologyName}-${System.GraphInstanceName} ${System.DateTime}"
```

### Keeping your VM clean

The Linux VM that you are using as an edge device can become unresponsive if it is not managed on a periodic basis. It is essential to keep the caches clean, eliminate unnecessary packages and remove unused containers from the VM as well. To do this here is a set of recommended commands, you can use on your edge VM.

1. `sudo apt-get clean`

    The apt-get clean command clears the local repository of retrieved package files that are left in /var/cache. The directories it cleans out are /var/cache/apt/archives/ and /var/cache/apt/archives/partial/. The only files it leaves in /var/cache/apt/archives are the lock file and the partial subdirectory. The apt-get clean command is generally used to clear disk space as needed, generally as part of regularly scheduled maintenance. For more information, refer to [Cleaning up with apt-get](https://www.networkworld.com/article/3453032/cleaning-up-with-apt-get.html).
1. `sudo apt-get autoclean`

    The apt-get autoclean option, like apt-get clean, clears the local repository of retrieved package files, but it only removes files that can no longer be downloaded and are not useful. It helps to keep your cache from growing too large.
1. `sudo apt-get autoremove1`

    The auto remove option removes packages that were automatically installed because some other package required them but, with those other packages removed, they are no longer needed
1. `sudo docker image ls` – Provides a list of Docker images on your edge system
1. `sudo docker system prune `

    Docker takes a conservative approach to cleaning up unused objects (often referred to as “garbage collection”), such as images, containers, volumes, and networks: these objects are generally not removed unless you explicitly ask Docker to do so. This can cause Docker to use extra disk space. For each type of object, Docker provides a prune command. In addition, you can use docker system prune to clean up multiple types of objects at once. For more information, refer to [Prune unused Docker objects](https://docs.docker.com/config/pruning/).
1. `sudo docker rmi REPOSITORY:TAG`

    As updates happen on the edge module, your docker can have older versions of the edge module still present. In such a case, it is advisable to use the docker rmi command to remove specific images identified by the image version tag.

## Next steps

[Quickstart: Get started - Live Video Analytics on IoT Edge](get-started-detect-motion-emit-events-quickstart.md)

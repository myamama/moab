---
layout: layouts/page.njk
title: "Project Moab - Tutorials 1: Train a Brain to Balance a Ball"
class: md
---

# Moab Tutorial 1: Train AI to Balance a Ball

Teach an AI brain to balance a ball in the center of a plate using a custom simulator and sample code.

**Total time to complete**: 45 minutes\
**Active time**: 25 minutes\
**Machine training time**: 20 minutes

**Prerequisites**: To complete this tutorial, you must have a Bonsai workspace provisioned on Azure. If you do not have one, follow the [the account setup guide](https://docs.microsoft.com/en-us/bonsai/guides/account-setup).

## Outline


<!-- TOC depthfrom:2 depthto:2 updateonsave:false -->

- [Outline](#outline)
- [Step 1: Define the problem](#step-1-define-the-problem)
- [Step 2: Create a new brain](#step-2-create-a-new-brain)
- [Step 3: Inspect the brain](#step-3-inspect-the-brain)
- [Step 4: Understand goals](#step-4-understand-goals)
- [Step 5. Train the brain](#step-5-train-the-brain)
- [Step 6: Evaluate training progress](#step-6-evaluate-training-progress)
- [Step 7: Assess the trained brain](#step-7-assess-the-trained-brain)
- [Step 8: Export and Deploy](#step-8-export-and-deploy)
- [Next steps](#next-steps)
- [Feedback and discussion](#feedback-and-discussion)

<!-- /TOC -->

<a name="step-1-define-the-problem"></a>

## Step 1: Define the problem

Imagine you are holding a plate and trying to keep a ball balanced in the center.

How would you do it?

First, you observe the ball:

- You could track the movement visually.
- You could track the vibrations in the plate by feel.
- You could track the rolling sound.

In observing the ball, you intuitively determine its **current location and speed**.

Next, you act on the gathered information:

- If the ball is already in the center, hold the plate flat to keep it there.
- If the ball moves away from the center, adjust the plate angle to move the ball back.

In adjusting the plate angle, you alter its **pitch and roll**.

### Ball Balance using the Moab Device

![A photo of the Moab device](../../img/tutorials/1/moab-photo.png)

Microsoft Project Moab is a fully integrated system for users of all levels to learn and explore building autonomous intelligent controls using reinforcement learning through Project Bonsai's Machine Teaching platform. The device (shown in the previous image) has three arms powered by servo motors. These arms work in tandem to control the angle of the transparent plate to keep the ball balanced.

<img alt="Diagram of the Z-Up right hand coordinate system used by the Moab device." src="../../img/tutorials/1/moab-frame-of-reference.png" width="400">


The Moab device tracks and maps the the ball movement onto a standard 2D coordinate system. Looking at the front the of the device, the **x-axis** runs left-to-right, and the **y-axis** runs front-to-back, with the plate center at location (0, 0), and a radius of r.

The same coordinate system is also used to define the two different tilt angles. Pitch is the plate angle about the x-axis, roll is the plate angle about the y-axis. A perfectly level plate would have pitch and roll of (0, 0).

The trained AI must learn how to adjust the plate pitch and roll to balance a ball using the following objectives:

1. The ball position (x, y) will reach the plate center at (0, 0) and stay there.
2. The ball position will not get near the plate edge at ( | (x, y) - (0, 0) | << r).

Now that you have identified the problem and defined the objectives, use machine teaching to train an AI to balance a ball.

<a name="step-2-create-a-new-brain"></a>

## Step 2: Create a new brain

<img alt="A screenshot of the Bonsai home screen" src="../../img/tutorials/1/bonsai-home.png" width="800">

To start a new Moab brain:

1. Create an account or sign into Bonsai.
2. Click the **Moab** icon in the **Getting started** panel, or **Create Brain** and **Moab demo**, as in the previous illustration.
3. Name your new brain (e.g., "Moab Tutorial 1").
4. Click **Create**

<a name="step-3-inspect-the-brain"></a>

## Step 3: Inspect the brain

After creating the Moab sample brain and simulator sample, Bonsai automatically opens the teaching interface, which is prepopulated with everything you need to get started.

<img alt="Screenshot of the Bonsai teaching interface" src="../../img/tutorials/1/tutorial1-ui-sections.png" width="800">

The teaching interface has three areas, as in the previous illustration:

- The **Navigation sidebar** lists all your brains and simulators.
- The **Coding panel** is your Inkling code editor. Inkling is a machine teaching proprietary language, designed to focus on what you want to teach while handling the AI details for you.
- The **Graphing panel** displays an interactive, graphical representation of the observable state, concept, and `SimAction` currently defined in the coding panel.

### Inspect the state node: ObservableState

<img alt="screenshot of ObservableState in the Graphing panel" src="../../img/tutorials/1/concept-graph-observablestate-highlight.png" width="200">

 Click the `ObservableState`node to jump to the relevant part of your Inkling code, as in the previous illustration.

`ObservableState` defines what information the brain is sent during every simulation iteration. For your ball balancing problem, the Moab device tracks the ball position and velocity. So, your simulation `ObservableState` is:

- `ball_x`, `ball_y`: the (x, y) ball position.
- `ball_vel_x`, `ball_vel_y`: the x and y ball velocity components.

```
# State received from the simulator after each iteration
type ObservableState {
    # Ball X,Y position
    ball_x: number<-RadiusOfPlate .. RadiusOfPlate>,
    ball_y: number<-RadiusOfPlate .. RadiusOfPlate>,

    # Ball X,Y velocity
    ball_vel_x: number<-MaxVelocity .. MaxVelocity>,
    ball_vel_y: number<-MaxVelocity .. MaxVelocity>,
}
```

Each state has an associated expected range. In the previous code snippet, the `ball_x` and `ball_y` locations are bounded by the plate radius. If provided, ranges can reduce AI training time.

### Inspect the actions node: SimAction

<img alt="screenshot of SimAction in the Graphing panel" src="../../img/tutorials/1/concept-graph-simaction-highlight.png" width="200">

Click on the `SimAction` node to jump to the next part of your Inkling code.

`SimAction` defines the ways that the brain interacts with the simulated environment. In your simulation, this is reflected by the following variables:

- `input_pitch`: a value that sets the plate angle along the X axis
  - -1 means tilt all the way forwards (away from the joystick), +1 means tilt all the way backwards (toward the joystick)
- `input_roll`: a value that sets the target plate angle along the Y axis
  - -1 means tilt all the way to the left, +1 means tilt all the way to the right

As highlighted before, the Moab device tilts the plate with two orthogonal angles (`SimAction`). An onboard algorithm translates this action to the three servo-powered arms, achieving the desired tilt.

```
# Action provided as output by policy and sent as
# input to the simulator
type SimAction {
    # Range -1 to 1 is a scaled value that represents
    # the full plate rotation range supported by the hardware.
    input_pitch: number<-1 .. 1>, # rotate about x-axis
    input_roll: number<-1 .. 1>, # rotate about y-axis
}
```

### Inspect the concept node: MoveToCenter

Click on the `MoveToCenter`concept node to define what the AI should learn, as in the following example:

A `goal` describes what you want the brain to learn using one or more objectives (previously defined), as in the following example:

```
# Define a concept graph with a single concept
graph (input: ObservableState) {
    concept MoveToCenter(input): SimAction {
        curriculum {
            # The source of training for this concept is a simulator that
            #  - can be configured for each episode using fields defined in SimConfig,
            #  - accepts per-iteration actions defined in SimAction, and
            #  - outputs states with the fields defined in SimState.
            source simulator (Action: SimAction, Config: SimConfig): ObservableState {
            }
...
```

A `concept` defines what the AI needs to learn and the `curriculum` is how it learns. Your `MoveToCenter`concept will receive simulator states and respond with actions.  

<a name="step-4-understand-goals"></a>

## Step 4: Understand goals

We previously defined two objectives for our brain:

- The ball position will not get near the plate edge at ( | (x, y) - (0, 0) | << r).
- The ball position (x, y) will reach the plate center at (0, 0) and stay there.

A `goal` describes what you want the brain to learn using one or more objectives, as in the following example:

```
# The training goal has two objectives:
#   - don't let the ball fall off the plate 
#   - drive the ball to the center of the plate
goal (State: ObservableState) {
    
    avoid `Fall Off Plate`:
        Math.Hypot(State.ball_x, State.ball_y) in Goal.RangeAbove(RadiusOfPlate * 0.8)

    drive `Center Of Plate`:
        [State.ball_x, State.ball_y] in Goal.Sphere([0, 0], CloseEnough)
}
```

As the brain trains, it attempts to simultaneously meet all the defined objectives during each episode.

Available goal objectives include:

- `avoid`: Avoid a defined region.
- `drive`: Get to a target as quickly as possible and stay in that place.
- `reach`: Get to a target as quickly as possible.

For your objectives, use `avoid` and `drive`, as follows:

### Objective: Avoid falling off the plate

To teach the brain to keep the ball on the plate, use `avoid` to define an objective called `Fall Off Plate`, as in the following code snippet:

```
avoid `Fall Off Plate`:
    Math.Hypot(State.ball_x, State.ball_y) in Goal.RangeAbove(RadiusOfPlate * 0.8)
```

An `avoid` objective asks the brain to learn to avoid a certain region of states. Your objective states that the ball's distance from the center must not reach values above 80% of the plate radius. This will teach the brain to keep the ball on the plate.

### Objective: Move the ball to the center of the plate

To teach the brain to move the ball to a specific spot and keep it there, use `drive` to define a objective called `Center Of Plate`, as in the following code snippet:

```
drive `Center Of Plate`:
    [State.ball_x, State.ball_y] in Goal.Sphere([0, 0], CloseEnough)
```

A `drive` objective asks the brain to learn to reach to a target as soon as possible and stay there. In this case, the target is for the ball's X and Y coordinates (`state.ball_x` and `state.ball_y`) to stay within `CloseEnough` radial distance from the plate center.

<a name="step-5-train-the-brain"></a>

## Step 5. Train the brain

 After defining the brain objectives, Click the **Train** button to start training and open the Train UI, as seen in the following screenshot:

<img alt="Screenshot of Bonsai UI starting training" src="../../img/tutorials/1/tutorial1-starting-training.png" width="800"/>

When training starts, Bonsai launches multiple Moab simulator instances in the cloud. The training progress is reflected in several UI sections, as seen in the following screenshot:

<img alt="Screenshot of Bonsai UI training" src="../../img/tutorials/1/tutorial1-training-started.png" width="800"/>

### Section 1: Training performance plot

Start with the chart at the top of the data panel that displays the **performance plot**, as seen in the following screenshot:

<img alt="Screenshot of the goal satisfaction performance chart" src="../../img/tutorials/1/tutorial1-training-started-perf.png" width="400">

This shows the average performance of the brain from test episodes that are regularly run during training. (A test episode evaluates the brain's performance without the exploratory actions used during training to help the brain learn.)

Next, inspect the goal satisfaction plots. Goal satisfaction plots display the brains' achievement progress for each objective. For example, 100% Goal Satisfaction for `Fall Off Plate` indicates that the brain has learned to consistently keep the ball on the plate. The overall **Goal Satisfaction** line is the average goal satisfactions across all the objectives. Select several other performance metrics using the Y-axis selector.

As it trains, the brain improves at accomplishing the goals you defined. The goal satisfaction values should eventually reach close to 100%.

### Section 2: Simulator node

The teaching graph has a new **Simulator** node in the graphing panel, as seen in the following screenshot:

<img alt="Screenshot of teaching graph with a simulator node" src="../../img/tutorials/1/tutorial1-concept-graph-training-mode.png" width="300"/>

The simulator node displays the following:

- the number of simulation iterations running in parallel.
- the overall iterations per second.

During training, the concept node displays the latest goal satisfaction.

### Section 3: Moab Device Visualization

There is a live visualization of the Moab simulator below the performance plot, as seen in the following screenshot:

<img alt="3D visualization of Moab simulator" src="../../img/tutorials/1/tutorial1-visualizer.png" width="800"/>

In addition to the 3D ball and hardware, the visualization displays the following:

- The ball velocity (the blue arrow projected onto the plate)
- The estimated ball position (blue circle projected on to the plate under the ball).

Click and drag to rotate the visualization view angle.

Below the visualization is an interactive graph that plots training values, as seen in the following screenshot:

<img alt="Screenshot of live-streaming state and action chart" src="../../img/tutorials/1/tutorial1-visualizer-and-sim-chart.png" width="800"/>

Click `ball_x` and `ball_y` to display the ball X and Y coordinates. The vertical dashed lines show the stop and start of new episodes.

As the brain learns to keep the ball from falling off the plate, each episode will get longer. As the brain learns to center the ball,  `ball_x` and `ball_y` will reach zero every episode.

While you wait for the brain to train, try charting other values.

<a name="step-6-evaluate-training-progress"></a>

## Step 6: Evaluate training progress

### When should we stop training?

The **Goal Satisfaction %** trends upwards during the first 100k – 200k iterations. Afterwards, the various performance lines will converge and flatten. After training for an hour or so, you should see a plot similar to the following screenshot:

<img alt="Screenshot of converged goal satisfaction chart" src="../../img/tutorials/1/tutorial1-training-converged.png" width="800"/>

Here, performance has reached a peak level and additional training time does not seem to yield any improvement.

**Training can be stopped when you notice that the Goal Satisfaction has not made any meaningful progress, which should occur before 500k iterations.** Hitting the Train button afterwards will resume training.

Congratulations! You have successfully trained a brain to balance the ball!

<a name="step-7-assess-the-trained-brain"></a>

## Step 7: Assess the trained brain

Click the **Start Assessment** button on the Train page, and the following sub-window should appear.

<img alt="Screenshot of assessment mode loading state" src="../../img/tutorials/1/tutorial1-starting-assessment.png" width="800"/>

After the simulator starts, the visualizer and streaming charts from training are displayed. In assessment mode, the brain is tested using the same random scenarios defined in the Inkling lesson.

<img alt="Screenshot of assessment mode loading state" src="../../img/tutorials/1/tutorial1-assessment.png" width="800"/>

<a name="step-8-export-and-deploy"></a>

## Step 8: Export and Deploy

Here is a video of a trained brain from this tutorial, deployed on the Moab hardware:

<iframe width="800" height="450" src="https://www.youtube.com/embed/iMDVCL7W9xs" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Once Moab kits ship, look here for instructions on deploying the trained brain onto your bot.

<a name="next-steps"></a>

## Step 9: Export and Deploy to Azure IoT Edge (Bonus Step)
---
title: Quickstart create an Azure IoT Edge device on Linux | Microsoft Docs 
description: In this quickstart, learn how to create an IoT Edge device on Linux and then deploy prebuilt code remotely from the Azure portal. 
author: kgremban
manager: philmea
ms.author: kgremban
ms.date: 04/07/2021
ms.topic: quickstart
ms.service: iot-edge
services: iot-edge
ms.custom: mvc, devx-track-azurecli
---

# Quickstart: Deploy your first IoT Edge module to a virtual Linux device

[!INCLUDE [iot-edge-version-201806-or-202011](../../includes/iot-edge-version-201806-or-202011.md)]

Test out Azure IoT Edge in this quickstart by deploying containerized code to a virtual Linux IoT Edge device. IoT Edge allows you to remotely manage code on your devices so that you can send more of your workloads to the edge. For this quickstart, we recommend using an Azure virtual machine for your IoT Edge device, which allows you to quickly create a test machine and then delete it when you're finished.

In this quickstart you learn how to:

* Create an IoT Hub.
* Register an IoT Edge device to your IoT hub.
* Install and start the IoT Edge runtime on a virtual device.
* Remotely deploy a module to an IoT Edge device.

![install-edge-full](https://user-images.githubusercontent.com/4113533/115604450-ae448700-a2e1-11eb-8410-c0f8f488fc86.png)

This quickstart walks you through creating a Linux virtual machine that's configured to be an IoT Edge device. Then, you deploy a module from the Azure portal to your device. The module used in this quickstart is a simulated sensor that generates temperature, humidity, and pressure data. The other Azure IoT Edge tutorials build upon the work you do here by deploying additional modules that analyze the simulated data for business insights.

If you don't have an active Azure subscription, create a [free account](https://azure.microsoft.com/free) before you begin.

## Prerequisites

Prepare your environment for the Azure CLI.

[!INCLUDE [azure-cli-prepare-your-environment-no-header.md](../../includes/azure-cli-prepare-your-environment-no-header.md)]

Cloud resources:

* A resource group to manage all the resources you use in this quickstart. We use the example resource group name **IoTEdgeResources** throughout this quickstart and the following tutorials.

   ```azurecli-interactive
   az group create --name IoTEdgeResources --location westus2
   ```

## Create an IoT hub

Start the quickstart by creating an IoT hub with Azure CLI.

![create-iot-hub](https://user-images.githubusercontent.com/4113533/115604541-cae0bf00-a2e1-11eb-84f4-652262c1ec17.png)


The free level of IoT Hub works for this quickstart. If you've used IoT Hub in the past and already have a hub created, you can use that IoT hub.

The following code creates a free **F1** hub in the resource group **IoTEdgeResources**. Replace `{hub_name}` with a unique name for your IoT hub. It might take a few minutes to create an IoT Hub.

   ```azurecli-interactive
   az iot hub create --resource-group IoTEdgeResources --name {hub_name} --sku F1 --partition-count 2
   ```

   If you get an error because there's already one free hub in your subscription, change the SKU to **S1**. Each subscription can only have one free IoT hub. If you get an error that the IoT Hub name isn't available, it means that someone else already has a hub with that name. Try a new name.

## Register an IoT Edge device

Register an IoT Edge device with your newly created IoT hub.

![register-device](https://user-images.githubusercontent.com/4113533/115604567-d03e0980-a2e1-11eb-92c0-db51ad78e476.png)

Create a device identity for your IoT Edge device so that it can communicate with your IoT hub. The device identity lives in the cloud, and you use a unique device connection string to associate a physical device to a device identity.

Since IoT Edge devices behave and can be managed differently than typical IoT devices, declare this identity to be for an IoT Edge device with the `--edge-enabled` flag.

1. In the Azure Cloud Shell, enter the following command to create a device named **myEdgeDevice** in your hub.

   ```azurecli-interactive
   az iot hub device-identity create --device-id myEdgeDevice --edge-enabled --hub-name {hub_name}
   ```

   If you get an error about iothubowner policy keys, make sure that your Cloud Shell is running the latest version of the azure-iot extension.

2. View the connection string for your device, which links your physical device with its identity in IoT Hub. It contains the name of your IoT hub, the name of your device, and then a shared key that authenticates connections between the two. We'll refer to this connection string again in the next section when you set up your IoT Edge device.

   ```azurecli-interactive
   az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name {hub_name}
   ```
<img width="516" alt="retrieve-connection-string" src="https://user-images.githubusercontent.com/4113533/115604631-e2b84300-a2e1-11eb-9593-c70f54572cbe.png">

## Configure your IoT Edge device

Create a virtual machine with the Azure IoT Edge runtime on it.

![start-runtime](https://user-images.githubusercontent.com/4113533/115604657-e8ae2400-a2e1-11eb-9e49-946c3a6025cc.png)

The IoT Edge runtime is deployed on all IoT Edge devices. It has three components. The *IoT Edge security daemon* starts each time an IoT Edge device boots and bootstraps the device by starting the IoT Edge agent. The *IoT Edge agent* facilitates deployment and monitoring of modules on the IoT Edge device, including the IoT Edge hub. The *IoT Edge hub* manages communications between modules on the IoT Edge device, and between the device and IoT Hub.

During the runtime configuration, you provide a device connection string. This is the string that you retrieved from the Azure CLI. This string associates your physical device with the IoT Edge device identity in Azure.

### Deploy the IoT Edge device

This section uses an Azure Resource Manager template to create a new virtual machine and install the IoT Edge runtime on it. If you want to use your own Linux device instead, you can follow the installation steps in [Install the Azure IoT Edge runtime](how-to-install-iot-edge.md), then return to this quickstart.

<!-- 1.1 -->
:::moniker range="iotedge-2018-06"

Use the following CLI command to create your IoT Edge device based on the prebuilt [iotedge-vm-deploy](https://github.com/Azure/iotedge-vm-deploy) template.

* For bash or Cloud Shell users, copy the following command into a text editor, replace the placeholder text with your information, then copy into your bash or Cloud Shell window:

   ```azurecli-interactive
   az deployment group create \
   --resource-group IoTEdgeResources \
   --template-uri "https://aka.ms/iotedge-vm-deploy" \
   --parameters dnsLabelPrefix='<REPLACE_WITH_VM_NAME>' \
   --parameters adminUsername='azureUser' \
   --parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name <REPLACE_WITH_HUB_NAME> -o tsv) \
   --parameters authenticationType='password' \
   --parameters adminPasswordOrKey="<REPLACE_WITH_PASSWORD>"
   ```

* For PowerShell users, copy the following command into your PowerShell window, then replace the placeholder text with your own information:

   ```azurecli
   az deployment group create `
   --resource-group IoTEdgeResources `
   --template-uri "https://aka.ms/iotedge-vm-deploy" `
   --parameters dnsLabelPrefix='<REPLACE_WITH_VM_NAME>' `
   --parameters adminUsername='azureUser' `
   --parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name <REPLACE_WITH_HUB_NAME> -o tsv) `
   --parameters authenticationType='password' `
   --parameters adminPasswordOrKey="<REPLACE_WITH_PASSWORD>"
   ```

:::moniker-end
<!-- end 1.1 -->

<!-- 1.2 -->
:::moniker range=">=iotedge-2020-11"

Use the following CLI command to create your IoT Edge device based on the prebuilt [iotedge-vm-deploy](https://github.com/Azure/iotedge-vm-deploy/tree/1.2.0) template.

* For bash or Cloud Shell users, copy the following command into a text editor, replace the placeholder text with your information, then copy into your bash or Cloud Shell window:

   ```azurecli-interactive
   az deployment group create \
   --resource-group IoTEdgeResources \
   --template-uri "https://raw.githubusercontent.com/Azure/iotedge-vm-deploy/1.2.0/edgeDeploy.json" \
   --parameters dnsLabelPrefix='<REPLACE_WITH_VM_NAME>' \
   --parameters adminUsername='azureUser' \
   --parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name <REPLACE_WITH_HUB_NAME> -o tsv) \
   --parameters authenticationType='password' \
   --parameters adminPasswordOrKey="<REPLACE_WITH_PASSWORD>"
   ```

* For PowerShell users, copy the following command into your PowerShell window, then replace the placeholder text with your own information:

   ```azurecli
   az deployment group create `
   --resource-group IoTEdgeResources `
   --template-uri "https://raw.githubusercontent.com/Azure/iotedge-vm-deploy/1.2.0/edgeDeploy.json" `
   --parameters dnsLabelPrefix='<REPLACE_WITH_VM_NAME>' `
   --parameters adminUsername='azureUser' `
   --parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name <REPLACE_WITH_HUB_NAME> -o tsv) `
   --parameters authenticationType='password' `
   --parameters adminPasswordOrKey="<REPLACE_WITH_PASSWORD>"
   ```
:::moniker-end
<!-- end 1.2 -->

This template takes the following parameters:

| Parameter | Description |
| --------- | ----------- |
| **resource-group** | The resource group in which the resources will be created. Use the default **IoTEdgeResources** that we've been using throughout this article or provide the name of an existing resource group in your subscription. |
| **template-uri** | A pointer to the Resource Manager template that we're using. |
| **dnsLabelPrefix** | A string that will be used to create the virtual machine's hostname. Replace the placeholder text with a name for your virtual machine. |
| **adminUsername** | A username for the admin account of the virtual machine. Use the example **azureUser** or provide a new username. |
| **deviceConnectionString** | The connection string from the device identity in IoT Hub, which is used to configure the IoT Edge runtime on the virtual machine. The CLI command within this parameter grabs the connection string for you. Replace the placeholder text with your IoT hub name. |
| **authenticationType** | The authentication method for the admin account. This quickstart uses **password** authentication, but you can also set this parameter to **sshPublicKey**. |
| **adminPasswordOrKey** | The password or value of the SSH key for the admin account. Replace the placeholder text with a secure password. Your password must be at least 12 characters long and have three of four of the following: lowercase characters, uppercase characters, digits, and special characters. |

Once the deployment is complete, you should receive JSON-formatted output in the CLI that contains the SSH information to connect to the virtual machine. Copy the value of the **public SSH** entry of the **outputs** section:

![outputs-public-ssh](https://user-images.githubusercontent.com/4113533/115604709-f8c60380-a2e1-11eb-895b-4893f72b1d83.png)

### View the IoT Edge runtime status

The rest of the commands in this quickstart take place on your IoT Edge device itself, so that you can see what's happening on the device. If you're using a virtual machine, connect to that machine now using the admin username that you set up and the DNS name that was output by the deployment command. You can also find the DNS name on your virtual machine's overview page in the Azure portal. Use the following command to connect to your virtual machine. Replace `{admin username}` and `{DNS name}` with your own values.

   ```console
   ssh {admin username}@{DNS name}
   ```

Once connected to your virtual machine, verify that the runtime was successfully installed and configured on your IoT Edge device.

<!--1.1 -->
:::moniker range="iotedge-2018-06"

1. Check to see that the IoT Edge security daemon is running as a system service.

   ```bash
   sudo systemctl status iotedge
   ```

<img width="463" alt="iotedged-running" src="https://user-images.githubusercontent.com/4113533/115604752-054a5c00-a2e2-11eb-8abd-b2d4d4cf07d9.png">

   >[!TIP]
   >You need elevated privileges to run `iotedge` commands. Once you sign out of your machine and sign back in the first time after installing the IoT Edge runtime, your permissions are automatically updated. Until then, use `sudo` in front of the commands.

2. If you need to troubleshoot the service, retrieve the service logs.

   ```bash
   journalctl -u iotedge
   ```

3. View all the modules running on your IoT Edge device. Since the service just started for the first time, you should only see the **edgeAgent** module running. The edgeAgent module runs by default and helps to install and start any additional modules that you deploy to your device.

   ```bash
   sudo iotedge list
   ```

<img width="463" alt="iotedge-list-1" src="https://user-images.githubusercontent.com/4113533/115604787-0f6c5a80-a2e2-11eb-9257-25afd2161da9.png">:::moniker-end
<!-- end 1.1 -->

<!-- 1.2 -->
:::moniker range=">=iotedge-2020-11"

1. Check to see that IoT Edge is running. The following command should return a status of **Ok** if IoT Edge is running, or provide any service errors.

   ```bash
   sudo iotedge system status
   ```

   >[!TIP]
   >You need elevated privileges to run `iotedge` commands. Once you sign out of your machine and sign back in the first time after installing the IoT Edge runtime, your permissions are automatically updated. Until then, use `sudo` in front of the commands.

2. If you need to troubleshoot the service, retrieve the service logs.

   ```bash
   sudo iotedge system logs
   ```

3. View all the modules running on your IoT Edge device. Since the service just started for the first time, you should only see the **edgeAgent** module running. The edgeAgent module runs by default and helps to install and start any additional modules that you deploy to your device.

   ```bash
   sudo iotedge list
   ```

:::moniker-end
<!-- end 1.2 -->

Your IoT Edge device is now configured. It's ready to run cloud-deployed modules.

## Deploy a module

Manage your Azure IoT Edge device from the cloud to deploy a module that will send telemetry data to IoT Hub.

![deploy-module](https://user-images.githubusercontent.com/4113533/115604811-18f5c280-a2e2-11eb-9e90-2c483f98b82a.png)
<!-- [!INCLUDE [iot-edge-deploy-module](../../includes/iot-edge-deploy-module.md)]

Include content included below to support versioned steps in Linux quickstart. Can update include file once Windows quickstart supports v1.2 -->

One of the key capabilities of Azure IoT Edge is deploying code to your IoT Edge devices from the cloud. *IoT Edge modules* are executable packages implemented as containers. In this section, you'll deploy a pre-built module from the [IoT Edge Modules section of Azure Marketplace](https://azuremarketplace.microsoft.com/marketplace/apps/category/internet-of-things?page=1&subcategories=iot-edge-modules) directly from Azure IoT Hub.

The module that you deploy in this section simulates a sensor and sends generated data. This module is a useful piece of code when you're getting started with IoT Edge because you can use the simulated data for development and testing. If you want to see exactly what this module does, you can view the [simulated temperature sensor source code](https://github.com/Azure/iotedge/blob/027a509549a248647ed41ca7fe1dc508771c8123/edge-modules/SimulatedTemperatureSensor/src/Program.cs).

Follow these steps to start the **Set Modules** wizard to deploy your first module from Azure Marketplace.

1. Sign in to the [Azure portal](https://portal.azure.com) and go to your IoT hub.

1. From the menu on the left, under **Automatic Device Management**, select **IoT Edge**.

1. Select the device ID of the target device from the list of devices.

1. On the upper bar, select **Set Modules**.

   ![Screenshot that shows selecting Set Modules.](./media/quickstart/select-set-modules.png)

### Modules
#### Select device and add modules

1. Sign in to the [Azure portal](https://portal.azure.com) and navigate to your IoT hub.
1. On the left pane, select **IoT Edge** from the menu.
1. Click on the ID of the target device from the list of devices.
1. On the upper bar, select **Set Modules**.
1. In the **Container Registry Settings** section of the page, provide the credentials to access any private container registries that contain your module images. 
![image](https://user-images.githubusercontent.com/4113533/115598389-a9300980-a2da-11eb-85b1-c87a3e0857bd.png)

1. In the **IoT Edge Modules** section of the page, select **Add** -> **IoT Edge Module** - You provide the module name and container image URI. For example, the image URI for the exported Brain module is `meabonsai.azurecr.io/XXXXXXXXXXXXXXXXX/moab_demo_18-4-2021:1-linux-amd64`. If the module image is stored in a private container registry, add the credentials on this page to access the image.

![image](https://user-images.githubusercontent.com/4113533/115599267-b39ed300-a2db-11eb-8c9a-87e09833d2b6.png)



1. After adding a module, select the module name from the list to open the module settings. Fill out the optional fields if necessary.

   For more information about the available module settings, see [Module configuration and management](module-composition.md#module-configuration-and-management).

   For more information about the module twin see [Define or update desired properties](module-composition.md#define-or-update-desired-properties).

1. Go to **CreateOptions** tab and paste the following, this will allow API calls from outside the Edge Runtime, note we used port 5000 as this is the default one to call Brain APIs:
```json
  {
    "ExposedPorts": {
        "5000/tcp": {}
    },
    "HostConfig": {
        "PortBindings": {
            "5000/tcp": [
                {
                    "HostPort": "5000"
                }
            ]
        }
    }
}
```
![image](https://user-images.githubusercontent.com/4113533/115600060-8dc5fe00-a2dc-11eb-87d9-9405dae39b5d.png)
3. Select **Review + Create **.

### Review and create

Review the JSON file, and then select **Create**. The JSON file defines all of the modules that you deploy to your IoT Edge device. You'll see the **moabBrainDemo** module and the two runtime modules, **edgeAgent** and **edgeHub**.

   >[!Note]
   >When you submit a new deployment to an IoT Edge device, nothing is pushed to your device. Instead, the device queries IoT Hub regularly for any new instructions. If the device finds an updated deployment manifest, it uses the information about the new deployment to pull the module images from the cloud then starts running the modules locally. This process can take a few minutes.

After you create the module deployment details, the wizard returns you to the device details page. View the deployment status on the **Modules** tab.

You should see three modules: **$edgeAgent**, **$edgeHub**, and **moabBrainDemo**. If one or more of the modules has **YES** under **SPECIFIED IN DEPLOYMENT** but not under **REPORTED BY DEVICE**, your IoT Edge device is still starting them. Wait a few minutes, and then refresh the page.

   ![image](https://user-images.githubusercontent.com/4113533/115601216-cf0add80-a2dd-11eb-9383-c42e02c614f6.png)

## View generated data

In this quickstart, you created a new IoT Edge device and installed the IoT Edge runtime on it. Then, you used the Azure portal to deploy an IoT Edge module to run on the device without having to make changes to the device itself.

Open the command prompt on your IoT Edge device, or use the SSH connection from Azure CLI. Confirm that the module deployed from the cloud is running on your IoT Edge device:

   ```bash
   sudo iotedge list
   ```

![image](https://user-images.githubusercontent.com/4113533/115601413-16916980-a2de-11eb-9394-8dcbfd706d7b.png)

You can make an API call to your Brain, for example:
   ```bash
curl -X POST "http://localhost:5000/v1/prediction" -H "accept: application/json" -H "Content-Type: application/json" -d "{\"ball_x\":0.2,\"ball_y\":0.2,\"ball_vel_x\":2,\"ball_vel_y\":2}"
   ```
![image](https://user-images.githubusercontent.com/4113533/115602064-cf57a880-a2de-11eb-9e01-1df1af4ad054.png)

If you have a HW device, you would just pass the returned values to adjust the plate with the Brain output

## Next steps

Congratulations! You trained a brain that can balance a ball on a plate, and can be deployed on real hardware. In [Tutorial 2](../2-robustness/index.html), you will learn how to use domain randomization to make the deployed brain more robust to differences between simulation and real life.

<a name="feedback-and-discussion"></a>
## Feedback and discussion

Discuss this tutorial and ask questions in the [Bonsai community forums](https://aka.ms/as/forums).

We would appreciate your feedback! [Submit feedback and product feature requests](https://aka.ms/as/productfeedback).

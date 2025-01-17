# Quickstart

In this guide we'll take Airbotics Cloud for a spin, we will:
- Build a full system image using Yocto and push it to Airbotics.
- We'll then flash the first image onto an emulated device and run it.
- We'll then make a new version of the image and push it again.
- Finally, we'll use the dashboard to update the device to the new version "over-the-air".


## Prerequisites
- A Linux-based machine with plenty of storage, compute and memory.
    - ~100GB disk space
    - At least 8GB of RAM
- An account with Airbotics, you can create one [here](https://dashboard.airbotics.io/registration).
- The following packages which include:
    - The QEMU emulator
    - Androids [repo](https://source.android.com/docs/setup/download#installing-repo) tool.  If an official package is not available for your Linux distribution, follow the install instructions from the link.
    - Yocto dependencies 

```

sudo apt install qemu

sudo apt install repo

sudo apt install gawk wget git diffstat unzip texinfo gcc-multilib build-essential chrpath socat cpio python python3 python3-pip python3-pexpect python-dev xz-utils debianutils iputils-ping cpu-checker default-jre parted

```

Finally you might want to get ready to make a large coffee. Let's go! 🚀



## 1. Download your provisioning credentials

The first thing to do after creating your account is download your provisioning credentials from the [dashboard](https://dashboard.airbotics.io/team/provisioning-credentials). Choose an sensible expiry date for them, it may take a few seconds for them to be issued.

Provisioning credentials are used to upload software images to Airbotics and to provision your robots, they come in a `credentials.zip` file.

> More: You can read more about managing credentials [here](../platform/provisioning-credentials.md).


## 2. Build an image

Now we'll build a full system image using [Yocto](https://www.yoctoproject.org/) - a set of tools for building embedded images. You'll need a relatively powerful computer to do this, the requirements for running Yocto can be found [here](https://docs.yoctoproject.org/3.2.3/ref-manual/ref-system-requirements.html).

For this guide we'll build a basic image on top of the reference distribution [Poky](https://www.yoctoproject.org/software-item/poky/), we will build it without vim and later create a new version that does contain vim.

In the interest of moving quickly we'll make use of the [updater-repo](https://github.com/advancedtelematic/updater-repo) from Advanced Telematic. This repo can be used to gather all of the Yocto layers we'll need to build an image for Airbotics.

Set up a project directory and then sync the required yocto layers.

```
mkdir airbotics-quickstart
cd airbotics-quickstart
repo init -u https://github.com/advancedtelematic/updater-repo.git -m dunfell.xml
repo sync -j8
```

At this point you should have all the required yocto layers on your host machine, including [meta-updater](https://github.com/uptane/meta-updater) which contains the agent, OSTree, and other magic.  After this guide, you can add more layers with everything your robot needs by creating your own or importing [pre-built](https://layers.openembedded.org/layerindex/branch/master/layers/) ones.

Meta-updater includes a helper script to set up the environment. From your project root run: 
```
source meta-updater/scripts/envsetup.sh qemux86-64
``` 

You should now find yourself in the `airbotics-quickstart/build` directory.

From here we will need to edit `airbotics-quickstart/build/conf/local.conf` and tell yocto where to look for our `credentials.zip` we downloaded in step 1. 

Open the conf file and set the `SOTA_PACKED_CREDENTIALS` variable to be the absolute path pointing to wherever you download your credentials.

For example: 

```
SOTA_PACKED_CREDENTIALS=<absolute-path-to-credentials>/credentials.zip
```

Save the file.

At this point we are finally ready to build the image with:

```
bitbake core-image-minimal
```

> Note: the first time you do this it can take as long as an hour. This could be a good time to get that coffee.


When the image finishes building meta-updater will sign and upload it to Airbotics. You'll then be able to see it in the [images](https://dashboard.airbotics.io/images) page on the dashboard. Click into the image and change its description to something like _Quickstart image v1 - does not contain vim_.

> More: You can read a more detailed tutorial about building images [here](../platform/images.md) including how to pin images to certain compatible boards.


## 3. Flash your image to a robot

Now that we have a shiny new image with everything our robot needs we'll flash it to our "robot". To simplify things, we're using the [QEMU](https://www.qemu.org/) emulator instead of a physical board.

We'll "flash" and boot the "robot" by running the following command:

```
../meta-updater/scripts/run-qemu-ota --overlay quickstart-robot.cow
```

Once it boots the agent will automatically provision itself with Airbotics and genreate a random ID. You can log in with the password `root`. From the command line in QEMU you can manually check which version is running with the command below:

```
ostree admin status
```

You can also confirm that vim is not found by running `vim` which should produce a nice error message.

If you head over to the [robots](https://dashboard.airbotics.io/robots) page of the dashboard you can see information about it your robot and confirm the correct version is running.


<!-- > More: You can read more about different boards [here](#), ostree commands, agent status. -->

----------------------------
## 4. Build a new version of the image

Great! Now we have a robot that's running an image we've built and is provisioned with Airbotics. Let's improve our software by adding vim to it. In the `airbotics-quickstart/build/conf/local.conf` of the meta-updater layer add the following line:

```
IMAGE_INSTALL_append = " vim "
```

and build it again using:

```
bitbake core-image-minimal
```

> Note: This should take a fraction of the time that it did on the first build.

After it builds we can see it in the [images](https://dashboard.airbotics.io/images) page of the dashboard, let's click into it and give it a description like: _Quickstart image v2 - contains vim_.

## 5. Roll out an update

So our engineers have been busy at work improving our codebase and we now have a brand new image that includes vim. Let's roll it out to our fleet.

We'll head to the [rollouts page](https://dashboard.airbotics.io/rollouts) on the dashboard and create a [rollout](../platform/rollouts.md). We'll target the rollout at selected robots (just one for now) and we'll specify we want to update ECUs with the `qemu86-64` hardware id to update to the update to our new image with vim. You should see a summary showing how our robot will be affected by the rollout, then you can confirm the creation of the rollout.

The final step to get the robot to update is to launch the rollout. You can do that from the Rollout Details page. At this point our robot should begin to download and install the new update. We can monitor the status of the rollout from the dashboard.

With Airbotics, **updates only take place after a reboot**, you can confirm that although the update has been accepted by the robot it hasn't yet been applied by running this command on the board:

```
ostree admin status
```

We can also confirm vim isn't on the robot yet.

Now Let's reboot qemu and confirm the update was applied: 
```
sudo reboot
```
Once it boots you should now be able to use vim. You'll see the ECU, Robot and Rollout status has changed. Nice 😎

## 7. Cleaning up

If you don't intend to continue to use your credentials we recommend you revoke them in Airbotics dashboard and delete them from your computer. You can also delete the images and robot.

<!-- ## Next steps

Now that you have completed a basic workflow here are some cool things you can do:

- [Group your robots](../components/groups.md).

- [Learn about the core concepts of Airbotics](../introduction/core-concepts.md).

- [Learn about the technologies Airbotics is built on](../introduction/supporting-technologies.md). -->
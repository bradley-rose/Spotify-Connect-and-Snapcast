# Faux-Sonos Multi-room Audio with Snapcast + Spotify Connect
I mentioned this setup elsewhere in a few forums, and I've been asked for a tutorial on how to set this up, so here we are. I don't own my own home, and thus I haven't yet invested in a high-quality receiver and multiple sets of speakers yet. It's simply impractical for those of us that move on the semi-frequent for work/education, but Sonos also doesn't make a setup like this affordable in the slightest for a wireless solution. Thankfully, Snapcast is free software (though I've donated to the developer simply for it being surprisingly stable, latency checks included), RaspberryPis aren't hard to come by, and there are various unofficial implementations of the Spotify library out there. Low and behold, here's a decent setup that's been working in my apartment for several months without a hitch. 

I know there are operating systems dedicated to this exact setup like Volumio or BalenaAudio, but I wanted something that was extensible (Volumio is a dedicated OS on Debian Jessie), local (BalenaAudio is a cloud setup. I'm a privacy snob from time to time,) and not expensive (aka. not Sonos). This satisfies all of that criteria, and I can still install a few other services like Unifi Controller for my home wireless access points, as well as Home Assistant all on the same RaspberryPi 4. The device doesn't break a sweat whatsoever even with all of these stacked services, so don't worry about it being overloaded.

I do have the appropriate installers here as referenced, but I would suggest still grabbing them from the original BadAix repository.

# Prerequisites
- One RaspberryPi (or other Linux client, i.e. laptop/desktop running Linux will work) for each set of speakers/rooms/hosts you want to deploy. I recommend the RaspberryPi Zero WH + a [DAC from HifiBerry](https://www.hifiberry.com/) that works with the Pi. Personally, I wanted a multi-purpose RaspberryPi, so I have a RaspberryPi 4 (standalone, no DAC) plugged directly into my network via Ethernet beside my router. This is not configured to output audio, but simply act as the main server that other RaspberryPis connect to in this setup.
- Good wireless connectivity should you choose to utilize the Pi Zero WH. Snapcast does a good job, but you don't want your wireless network to be intermittent while music is playing. Make sure you've got decent coverage in the area in which your client Pis will be living, or connect them all directly to your Ethernet network if possible. Again, this isn't usually achievable in apartment spaces without significant modifications, so I've just invested in good wireless access points.
- Speakers
- As with every Linux project, some time to learn the hurdles when setting it up.

# Primary RaspberryPi
This RaspberryPi will be acting as the server. This is the one that I've got sitting by my home router. This device *can* also be configured to act as a RaspberryPi client to output audio to a connected speaker or set of speakers, but since I've got my computer in my office (also coincidentally where my router lives) operating on Ubuntu full time, I just configured this device as a Snapcast client instead of relying on the DAC from the nearby Pi. 

1. Install Raspotify on the Pi that you've identified as your "server". 

    `curl -sL https://dtcooper.github.io/raspotify/install.sh | sh`

2. Install Snapcast Server. Visit the [Snapcast releases page](https://github.com/badaix/snapcast/releases/) and identify the current version number. It was on 0.25 last I checked, so I will be referencing 0.25 from hereon. In this below command, you'll notice I pull v0.25.0 of the **ARMHF** debian file. If this version number has changed, modify the below `wget` command to reflect the location of the new version.

    ```bash
    wget https://github.com/badaix/snapcast/releases/download/v0.25.0/snapserver_0.25.0-1_armhf.deb
    apt-get install ./snapserver_0.25.0-1_armhf.dev
    ```

3. Configure Snapserver:  
    `sudo nano /etc/snapserver.conf`  
Modify the contents to mimic the following:

    ```
    [http]
    enabled = true
        
    [stream]
    source = spotify:///librespot?name=Spotify&bitrate=320

    [audio]
    output = audioresample ! audio/x-raw,rate=44100,channels=2,format=S16LE ! audioconvert ! wavenc ! filesink location=/tmp/snapfifo
    ```

    This configuration makes several assumptions. If any of the following assumptions do not apply to you, then you will have to do some research through the Snapserver configurations to work for your specific setup:

* You pay for Spotify Premium and want to accept the overhead that comes with streaming at a bitrate of 320Kbps. This is a higher quality file so it will sound nicer, especially when being output of decent speakers.
* You are outputting only two channels, meaning that in no part of your setup, do you have a "5 channel" setup, or even a "mono" (single speaker) setup. 

4. Disable Raspotify! This might seem counterintuitive as we just installed this! What we actually did was installed the Librespot library with the Raspotify installer. Raspotify makes the installation of Librespot very easy. I had some troubles previously installing this library, and they may have since been fixed, but this Raspotify installation is super simple.

    `sudo systemctl disable raspotify`

5. Start and enable the Snapcast Server service: 

```bash
    sudo systemctl start snapserver
    sudo systemctl enable snapserver
```

By this point in the process, if you open Spotify on a device that's on the same network as this RaspberryPi, you should be able to see it showing up in the available Spotify Connect devices.  You just don't have any output devices configured, so you won't be able to hear any audio output from it.

# Snapcast Clients

6. Connect your Snapcast Clients! Repeat this step for all clients in your setup. Similar to what you did for Step 2, navigate to the [Snapcast releases page](https://github.com/badaix/snapcast/releases/) and identify the most recent version number for the Snapcast client library. SSH to the RaspberryPi client, and perform the following commands. Again, I will reference v0.25.0 here.

    ```
    wget https://github.com/badaix/snapcast/releases/download/v0.25.0/snapclient_v0.25.0-1_armhf.deb
    sudo dpkg -i snapclient_0.25.0-1_armhf.deb
    sudo apt -f install
    ```

    Edit the `/etc/default/snapclient` file with the following entry. I'm going to use a placeholder IPv4 Address of 192.168.0.100, but this IP address will have to reflect the IP address of your RaspberryPi server that we configured before this step. I would recommend setting a reservation on your router's DHCP service to ensure that this doesn't change:  
    ```
    SNAPCLIENT_OPTS="--host 192.168.0.100"
    ```

Just as a reminder, that 192.168.0.100 address **is a placeholder** and inserting this directly into your setup will not work. This needs to be replaced by the IPv4 address of your RaspberryPi that we configured in steps 1-5. SSH into the device and run `ip addr show` to find the address. I would highly recommend either setting this as a static IP address, or reserving this in your DHCP server's reservation list. 

All that's left is to restart the Snapclient service, and your clients should be able to play music that you've sent it from Spotify Connect!

    sudo systemctl restart snapclient
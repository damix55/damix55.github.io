---
layout: post
title: Creating the perfect Kindle Clock
date: 2023-11-22T18:16:31
categories:
  - python
  - modding
tags: []
---
[GitHub](https://github.com/damix55/kindle-clock)

![xkcd Kindle](/assets/2023-11-22-kindle-clock/dashboard-old.png) 

I had this little Kindle gathering dust in one of my drawers, and if there's one thing that I hate about old tech, is throwing them away. I can't help but try to find some useful uses for that little piece of silicon and plastic laying around. Moreover, the Kindle has always been one of my favorite devices, mostly because of his display technology: **e-ink**.
E-ink is phenomenal. It can displays images and text without glare, like it was on paper.  It's also low-consumption, since it draws power only when it has to change what's showing.
This technology is perfect for a creating a desk dashboard!

The idea of repurposing an old Kindle into something else is nothing new: there are many tutorials on the web on how to convert your old Kindle into a clock or a weather station.
The problem with these solution is that many of them rely on the integrated browser to display contents. This is a big limitation in my opinion, since you need an external device where you have to host a website that will displayed on the Kindle. Moreover, the browser cannot be set on fullscreen mode and therefore the top bar will always be visible.

The Kindle is essentially a device powered by Linux, and thanks to the jailbreak, we can have access to the full system via SSH. So why don't we take advantage of the (little) computational power of the e-reader to generate all the contents to display locally?

The MobileRead forum has a lot of mods to install on a jailbroken Kindle, in this case the Python package comes really handy. With Python installed, we can run virtually any script we want (while keeping in mind the limitation of the e-reader). Combine that with a little of custom commands and you can display images generated locally. You can display data about weather informations, your current song playing on Spotify, what events are planned today and more! The potential is endless.

## About this project
I will admit that i had this project *almost* ready for more than 2 years now (even though I've been using my Kindle as a desk clock since then). It also went through various iteration, I implemented different features and then scratched others, i reworked the whole codebase at least two times.
But being the perfectionist I am, it took me this long to finally write an article about this project and finalize it.

| ![xkcd Kindle](/assets/2023-11-22-kindle-clock/xkcd.png) | 
|:--:| 
| *A Kindle displaying random comics from xkcd!* |

Even though I tried to explain the process in details, the solution isn't plug and play and requires a little bit of tinkering and familiarity with Linux and the command line. My final goal was to realize a modular solution in the most easily installable "package". But I've decided that the work I had to put for reaching my perfect goal wasn't really worth, so I opted for creating an exhaustive guide on how to realize something like this yourself, using my code as a starting point.
Keep in mind that this is still work in progress! I would like to develop this project further, since there is a lot of room for improvements and features I'd love to add.
Let's dive into it!

## The perfect dashboard

| ![Wait, its 19.09? Always has been](/assets/2023-11-22-kindle-clock/main.png) | 
|:--:| 
| *I know I have a bad sense of humor, but this is how I use it daily.* |

### How I built it
Explain briefly how I got the idea, how i resolved the various problem I faced and so on.

### Disabling screensaver and framework
The first thing we are going to do before running our program, is to disable the screensaver,. to avoid the device to go in sleep mode, and stopping the Kindle user interface, since we don't need it and takes unnecessary resources. To do so, it is necessary to run these two commands:
``` bash
lipc-set-prop -i com.lab126.powerd preventScreenSaver 1
/etc/init.d/framework stop
```

### Getting the display to show images
Thanks to the [MobileRead Wiki](https://wiki.mobileread.com/wiki/Eips), it was very easy to get images to display on the Kindle.
We need essentially three commands:
* `eips -g [path_of_image]` to display an image given its path.
* `eips -f -g [path_of_image]` displays an image like the previous command and forces a full refresh of the e-ink display. This is particularly useful to prevent ghosting.
* `eips -c` clears the screen. Helps even more with the ghosting issue.

### Generating the images to display
All the images are created or converted using the [Pillow](https://pillow.readthedocs.io/en/stable/) library. To display images correctly, there are a few steps we have to take:
1. Convert the image to greyscale
2. Crop it (if necessary)
3. Resize it to 800x600 (the screen resolution)

The library is also used to generate the images that are then shown: in fact, the library is also capable of drawing rectangles and text, which is all we are gonna need for generating the clock interface. 

### Scheduling the tasks
Using the `schedule` library, we can easily schedule functions to run at certain time. This is perfect for the generation of the image of the clock for the current time. 

```Python
schedule.every().minute.at(':59').do(run_threaded, kindle.update_clock)
```

The clock update is scheduled every minute at the 59th second, since it needs around 2 seconds to generate the image, refresh the screen and show the image. 
Then, we have to create a main loop that check's if there's any pending tasks to run.

```Python
    while True:
        schedule.run_pending()
        sleep(1)
```

### Getting the buttons input
This was a tricky part. Even though i managed to get the buttons working using Python libraries, i was getting some lag and inconsistent inputs. So I opted to go a little lower level.
Inputs events are written to `/dev/input/event0` and `/dev/input/event1`. We can access them and read them like a stream of bytes, similar to how you would read from a file. When a button is pressed, we end up with a sequence of bytes, which represents information about the input.
You can check yourself the keys monitored by each device and their code by running: `evtest info /dev/input/event0` and `evtest info /dev/input/event1`. Here's how they are mapped:

| **Code** | **Internal Name** | **Description** | **Device** |
| ---- | ---- | ---- | ---- |
| 193 | F23 | Left Side Down | `/dev/input/event0` |
| 104 | PageUp | Left Side Up | `/dev/input/event0` |
| 191 | F21 | Right Side Down | `/dev/input/event0` |
| 109 | PageDown | Right Side Up | `/dev/input/event0` |
| 158 | Back |  | `/dev/input/event0` |
| 29 | LeftControl |  | `/dev/input/event0` |
| 139 | Menu |  | `/dev/input/event0` |
| 102 | Home |  | `/dev/input/event0` |
| 105 | Left |  | `/dev/input/event1` |
| 108 | Down |  | `/dev/input/event1` |
| 106 | Right |  | `/dev/input/event1` |
| 103 | Up |  | `/dev/input/event1` |
| 194 | F24 | Center Button | `/dev/input/event1` |

#### /dev/input/event0
Each keypress is a 24 bytes chunk: `data = f.read(24)`. Each chunk is then interpreted as an array of 12 unsigned short integers: `upk = unpack('12H', data)`.
Of these 12 integers we need only two of them: `upk[2]` , which represent the two different kind of keypresses (push and release), and `upk[1]`, which represents the code of the key. Since we don't need to use long presses, we can filter them out by checking for `upk[2] == 1`. The key code in `upk[1]` is then passed to the `map_key` function to handle the keypress event.

``` python
f = open('/dev/input/event0', 'rb')
while True:
	data = f.read(24)
	upk = unpack('12H', data)
	if upk[2] == 1:
		code = upk[1]
		map_key(code)
```

#### /dev/input/event1
The process here is very similar. Instead of a 24 bytes, we have only 16. These bytes represent an array of 8 unsigned short integers. `upk[6]` contains the type of keypress and `upk[5]` contains the code of the button.

``` python
def get_keypress_2():
	f = open('/dev/input/event1', 'rb')
	while True:
		data = f.read(16)
		upk = unpack('8H', data)
		if upk[6] == 1:
			code = upk[5]
			map_key(code)
```

#### Mapping the keys
The `map_key` function utilizes a dictionary to link key codes with specific functions. When a key code is passed to `map_key`, it checks this dictionary to find a corresponding function and if there's one associated, then it calls it. Now when you press a key on the Kindle, the action preprogrammed in this function gets activated instantly. Pretty cool uh?

``` python
def map_key(code):
	keys = {
		193: kindle.submode_back,	# F23
		104: kindle.submode_up,	    # PageUp
		191: kindle.submode_back,   # F21
		109: kindle.submode_up,     # PageDown
		158: kindle.refresh,        # Back
		29:  kindle.show_picture,   # LeftControl
		139: kindle.show_clock,     # Menu
		102: None,				    # Home
		105: None,				    # Left
		108: None,				    # Down
		106: None,				    # Right
		103: None,				    # Up
		194: None,				    # F24
	}
	
	action = keys[code]
	if action is not None:
		action()
```


### Displaying photos using a Telegram bot
Although I genuinely think that using the Kindle as a clock/dashboard is far more useful, there's something fascinating about displaying pictures on a e-ink monitor. Don't get me wrong, the display is only capable of showing 16 levels of grayscale and the contrast isn't great. But I love how the matte look makes it look like the photo was printed. So I decided to put the capability to also act as a digital frame. The idea was to have some way to send a photo to the Kindle and have it to be shown on the device.
The easiest way I could think of was to build a Telegram bot. To create one, all you have to do is to use the [BotFather](https://t.me/BotFather), which will guide you into the creation of a new bot. You can choose a name and a username and after that the BotFather will provide you the API token. This will be used to receive the messages sent to the bot and then take an appropriate action on the Kindle. 
Now every time we send a picture to the bot, the Kindle will receive it, crop it, rotate it counter-clockwise, convert it to 4 bits and then display it on the Kindle. And *voilà*, we can display any photo we want with just a message!
Note that to avoid anyone messing with the bot, we can filter message by username, so that we are the only one allowed to send message to the bot.

### The angled cable
For the charging cable, i wanted something that didn't get in the way. I opted for [this](https://aliexpress.com/item/1005003216764469.html) cable from AliExpress, which was perfect for the job. The cable has an angled Micro-USB connector, so that when you plug the Kindle in, the cable goes right behind the back of the device, providing a clean look to the project.

### The little details
* The cardboard mounts

## How to setup
### 1. What you will need
This project is based on the **4th generation Kindle**, with **firmware 4.1.4**. I haven't tested this on any other firmware version or device. You also need a PC for jaibreaking the device and to connect via SSH.

### 2. Jailbreaking your Kindle
Warning! This procedure can brick permanently your device, so follow at your own risk. I'm not responsible if your Kindle doesn't turn on anymore. All the necessary files are available in the `bin.zip` file in my repository.

* Plug in the Kindle and copy the content of the `jailbreak` folder in `bin.zip` to the Kindle's USB drive's root 
* Safely remove the USB cable and restart the Kindle (Menu button -> Settings -> Menu button -> Restart)
* Once the device restarts into diagnostics mode, select `D) Exit, Reboot or Disable Diags` (using the 5-way keypad) 
* Select `R) Reboot System` and `Q) To continue` (following on-screen instructions, when it tells you to use 'FW Left' to select an option, it means left on the 5-way keypad) 
* Wait about 20 seconds: you should see the Jailbreak screen for a while, and the device should then restart normally 
* After the Kindle restarts, you should see a new book titled "You are Jailbroken", if you see this, the jailbreak has been successful

### 3. Enabling SSH
Congratulation! You have successfully jailbroken your device. Now we have to enable SSH to take full control of the system.
* Plug in the Kindle and copy the content of the `usbnetwork` folder in `bin.zip` to the Kindle's USB drive's root
* Safely remove the USB cable and update the Kindle (Menu button -> Settings -> Menu button -> Update Your Kindle)
* Once the device restarts, plug in the Kindle and setup your public key in `usbnet/etc/authorized_keys`
* Edit `usbnet/etc/config` and set `K3_WIFI="true"` and `USE_OPENSSH="true"`
* Safely remove the USB cable
* Test if the connection via SSH works
    * Press the keyboard key on the Kindle and write `;debugOn` followed by enter
    * Press the keyboard key on the Kindle and write `~usbNetwork` followed by enter
    * Now try connecting to the Kindle via SSH (with root user)
* If SSH connection works, enable USBnetwork at system startup by creating a blank `auto` file in the `usbnet` folder 
    * You can use the command `touch /mnt/us/usbnet/auto`
    * Warning: this will disable USB mass storage, so be sure that everything is working as it should!
* Reboot the device and test if SSH works at startup

### 4. Installing Python and screen
Now that we can access the device in SSH, we have to install Python and screen in order in order to make the software work. Python is used to run the program, while screen is used to run it detached from the SSH session.
* Plug in the Kindle and copy the content of the `python` folder in `bin.zip` to the Kindle's USB drive's root (`/mnt/us/`)
* Update the Kindle (Menu button -> Settings -> Menu button -> Update Your Kindle)
* Once the device restarts, install pip by running `python3 -m ensurepip --upgrade`
* Copy the content of the `screen` folder in `bin.zip` to `/mnt/us/bin`
* Run `mntroot rw` to make the root filesystem writeable
* Create a symlink for using screen system-wide `ln -s /mnt/us/bin/screen /usr/bin/screen`

### 5. Installing and running the software
* Clone the repository in `/mnt/us/dashboard`
* Install requirements by running `python3 -m pip install -r /mnt/us/dashboard/requirements.txt`
* Run `/mnt/us/dashboard/start.sh`

Wait a few seconds and everything should hopefully work as expected!

### 6. Updating the time
* To syncronize clock: `ntpdate 0.it.pool.ntp.org`
* If you want to change the timezone, you must edit `/etc/localtime`
    * Copy the timezone file you want from a Linux system (for example: `/usr/share/zoneinfo/Europe/Rome`) to /mnt/us/
    * Remove the old symlink `rm /etc/localtime`
    * Create a symlink to the new timezone: `ln -s /mnt/us/Rome /etc/localtime`

### 7. Start the script at boot
* Run `mntroot rw` to make the root filesystem writeable
* Edit `/etc/init.d/boot_finished` and add `/mnt/us/dashboard/start.sh` before `return 0` of the `after_framework_start()` function


## Possible improvements
* WiFi seems to stop working after some weeks but I currently don't know why. It may be a solution to schedule a device reboot every once in a while (or maybe just the network)
* Make it possible to set timezone from configuration
* I don't like that the battery is still inside, since it's always charging


## Some old iterations and concepts
![xkcd Kindle](/assets/2023-11-22-kindle-clock/calendar.png) 


## Sources
* Kindle jailbreak: https://wiki.mobileread.com/wiki/Kindle4NTHacking#Jailbreak
* USBNetwork: https://www.mobileread.com/forums/showthread.php?t=88004
* Python: https://www.mobileread.com/forums/showthread.php?t=225030
* screen: https://www.fabiszewski.net/kindle-terminal/
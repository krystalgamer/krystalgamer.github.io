---
layout: post
title: "Kanye West's Stem Player - An engineering disaster"
description: "This post cover the history of the Stem Player, the development of the emulator, how badly the product was handled and some behind-the-scenes interactions with the creators"
created: 2022-08-05
modified: 2022-08-05
tags: [later]
comments: false
---



This blog post will cover the story of the Stem Player (since release), how the emulator was developed, how badly the creators have designed and handled the product and some behind the scenes conversations that led me to lose all faith on the company and the people managing it.

*Writer's Note: Kanye West has legally changed his name to ye. Excepting the title, I'll be referring to him as ye throughout the post.*

# What is a Stem Player

It is a 200$ device created by Kano Tech in collaboration with ye that allows to playback song stems and individually control their volume. A song stem is an isolated component of the song, they can be individual components (e.g vocals, drums, piano,..) or multiple components (e.g vocals and instrumental). The Stem Player only has the capacity for 4 stems per song, which expose the following elements: vocals, bass, drums and other(everything that doesn't fit the other categories).


A common misconception is that it splits any song into stems. NO! As the name implies, it is a stem **player**. The splitting process is done off the device. In fact, they use the open-source Deezer's splitter called [Spleeter](https://github.com/deezer/spleeter). It is possible to get the files off the device, when connected via USB, it can be mounted as mass storage.

The device was announced a few days before the album Donda was released and, for the most part, was deemed a hypebeast-y gadget. Music production enthusiasts noted it was interesting since it introduces the mixing concept to a broader audience. Still, it's hard to justify the price point.

## The Web Portal


All actions on the device are done through the web portal and it uses [WebUSB](https://developer.mozilla.org/en-US/docs/Web/API/USB) for communication. You're out of luck if you don't use a Chrome-based browser. Once the connection is established, you can modify some device settings or upload songs. The songs can come from links, local files or the existing library, which contains all ye albums since Jesus is King and a few singles. Songs from links or local files are sent to the back-end, split and then the stems are uploaded to the device. With ye's catalog, stems are simply sent to the device.



![Web Portal currently](images/stemplayer/web_portal.png)



## Known Limitations/Problems

- 8GB of space
- Extremely slow USB speeds, it takes hours to backup the whole device
- Stems are stored uncompressed, WAV format. All stems of a song take the same amount of space regardless of how much actual data they contain
- No visual output
- Limited amount of buttons. Play/Pause, next song and volume control. If you want to go to the previous song, you must go through your library.
- Official guide is just doodles. A community guide has been made and is recommended over the official.
- Novelty wears rapidly thus it is mainly used as a speaker
- Shipping takes too long, usually months
- Some ye songs seem to be AI splitted
- Even though the Web Portal has higher quality stems, lower quality ones are the ones sent to the user's device
- Easy to brick, if the configuration files contain any error

Having only 8GB is a blessing in disguise, or else people with larger libraries would struggle to find their favorite song.


From the get-go, one can see the device is an expensive lackluster gadget. So, why develop an emulator?


# Birth of the emulator

The Donda album rollout counted with 3 listening parties, followed by the album release. Initially, the Stem Player was even called DONDA Stem Player. ye announced he was having a listening party on the 22nd of February of 2022 for his next album, Donda 2. On the morning of the 18th, I wake up and check Instagram, there's a post from ye. It was a new song being played on the Stem Player and the announcement that the album was to be released exclusively on the Stem Player after the listening party. Some fans panicked since there wasn't enough time to get a device, even if they bought it that day, it could take months to arrive at their homes. It was known Stem Player users would rip it, but it would also mean other users would buy it just for the album, which didn't sit right with me.

This ordeal is not new for ye, when he released his album "The Life of Pablo" exclusively on Tidal, most people consumed the album illegally, which he mentions in his song "Saint Pablo" - `A million illegally downloaded my truth over the drums`.


![Donda 2 Announcement](images/stemplayer/donda2_announcement.png)

At first, ye's announcement confused me because I knew nothing about the device and it made no sense how he would ship the album directly to it. After learning how the device works from YouTube videos and reviews, I knew I had to trick the Web Portal into believing I had an actual Stem Player. My initial idea was to develop a device driver, but that wouldn't be compatible with all OSes and most listeners wouldn't accept it. That's when the idea for a browser plugin came into mind, it can be easily installed and would keep all the logic inside the JavaScript world. The problem was that it was not compatible with all browsers. Even though Stem Player users were forced to use Chrome-based browsers (because of WebUSB), the emulator didn't need to be under the same restrictions. The solution was to use [Tampermonkey](https://www.tampermonkey.net/), it is compatible with multiple browsers and allows me to execute arbitrary JavaScript, which is precisely what I needed.


## Basics of hacking in JavaScript

Hooking/Redirecting the flow of a program in JavaScript is extremely simple. WebUSB methods can be accessed through the `navigator.usb` object. E.g to hook `requestDevice` I just need to do.

```js
navigator.usb.requestDevice = (argument) => { /*hook code goes here :)*/ }; 
```

JavaScript also offers a handy utility called [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy). It allows the user to capture read and write accesses to JavaScript objects. This makes developing hooks much easier since I can send a proxy object and log all the accessed members.


## Faking WebUSB calls

On the Web Portal, if the user presses "CONNECT"(previously PLATFORM) the function `navigator.usb.requestDevice` is called. It can take an argument to filter USB devices by the vendor or product ID, class codes or serial number, that's how the website knows the Stem Player is connected. At first, I hooked this function and returned a Proxy [USBDevice](https://developer.mozilla.org/en-US/docs/Web/API/USBDevice) object to see the accessed fields. Having logged all the accessed fields and read the documentation of the expected result, the `FakeUSB` class was born.

```js
    class FakeUSB{
        constructor(){
            this.opened = true;
            this.emulator = new StemEmulator();
        }

        open(){
            return emptyPromise();
        }

        selectConfiguration(config){
            //console.log(config);
            return emptyPromise();
        }

        claimInterface(interfaceNumber){
            //console.log(interfaceNumber);
            return emptyPromise();
        }

        transferOut(endpointNumber, data){
            this.emulator.handleOut(data);
            return new Promise((res,_) => { res(new FakeUSBOutTransferResult(data.length)); });
        }

        transferIn(endpointNumber, length){
            //console.log('reading ' + length);
            return new Promise((res, _) => { res(this.emulator.handleIn()); } );
        }

    };

```

As you can see, most functions can be replaced with empty promises (promises that return nothing), the two most important methods are `transferOut` and `transferIn`, which are used to write data to the device and read from the device, respectively. Under normal circumstances, it is not guaranteed that all data is available, so these functions might need to be called multiple times. Since I'm faking the device, I can always have all the data available, which considerably simplifies the logic. Because of that, fake response objects were created that always have the `status` field set to `"ok"`.


## Mimicking the real device


The Web Portal code is webpacked which adds a layer of obfuscation, luckily function names and strings are preserved. There are some tools to un-webpack the code, but they yielded underwhelming results. My solution was to use [Fiddler](https://www.telerik.com/fiddler) and serve the JavaScript code locally. I could prettyify, change it and test what emulator modifications could do ahead of time. One of the said modifications was to re-enable a logging function that had an early exit condition. I also found a function to dump all message exchanges, which would be fantastic if I had a Stem Player. Figuring out how the messages are encoded was trivial, simply place a breakpoint on `transferIn` and climb the callstack.

The messages follow the Length-Type-Value pattern:

- First two bytes are small-endian and contain the length of the value
- Third byte is the type of message
- Buffer with the respective length



The existing message types are as follows:


```js

class MessageType{
	static ACK = 0;
	static NAK = 1;
	static CONNECT = 2;
	static DISCONNECT = 3;
	static CONTROL = 4;
	static RESPONSE = 5;
	static FILE_HEADER = 6;
	static FILE_BODY = 7;
	static ABORT = 8;
};

class ControlType{
	static REBOOT = 0;
	static VERSION = 1;
	static GET_STORAGE_INFO = 2;
	static GET_TRACKS_INFO = 3;
	static GET_DEVICE_CONFIG = 4;
	static GET_ALBUM_CONFIG = 5;
	static GET_TRACK_CONFIG = 6;
	static GET_ALBUM_COVER = 7;
	static ADD_ALBUM = 8;
	static DELETE_ALBUM = 9;
	static DELETE_TRACK = 10;
	static GET_MUSIC_FILE = 11;
	static GET_RECORDING_SLOTS = 12;
	static GET_RECORDING = 13;
	static DELETE_RECORDING = 14;
	static RENAME_ALBUM = 15;
	static MOVE_TRACK = 16;
	static GET_STATE_OF_CHARGE = 17;
	static CHALLENGE = 18;
};

```


The `CONTROL` type contains in the value buffer a sub-type of messages. No idea why they didn't simply expand the message types. The difference is that a regular message usually requires an `ACK` while a `CONTROL` message requires a `RESPONSE` message back.


Deciphering the messages was easy since the value buffer contained JSON data. Figuring out the required responses was also simple but tedious. At first, I would send an `ACK`, if the code crashed, complained or re-issued the same message, I would deep-dive the decoder function. Following webpacked code is annoying since most variables are one-letter words and function calls are not linear. Slowly but surely, messages were all decoded. Some of them were harder to figure out since invalid results would cause the page to go all blank, while others would tell exactly what the problem was.



## Accessing the Portal


After pressing "CONNECT", the Web Portal expects the device to send quite a bit of information before displaying the page. First, a `CONNECT` message is sent, followed by most of the control `GET_*` messages. I got stuck during this part for a while figuring out the correct JSON fields, but what took me the most was the `GET_TRACKS_INFO`. Figuring out how albums and tracks were organized was mind-boggling. The problem was that the more information I sent, the more I was requested to send. Then, it dawned on me. What if I simply tell the Portal I have no songs on my device? BAM! Problem solved. No tracks on the device makes the Portal not ask anything else.

I also had to ensure I was sending high enough firmware versions or the Portal would tell me to update the device.

## Accessing the stems

At this point in the emulator development, I can reach their Web Portal. If I try to use the link/local file feature, the page loads the songs stems and allows playback, but when I press the upload button, it tells me I don't have enough space on the device; if I access a ye track both the playback and upload do nothing. First, I focused on the low space problem. Turns out I was sending an empty response to the `GET_STORAGE_INFO` control message, what it required was a JSON object with two fields, `free` and `size`. Setting this to high values causes the emulator to receive a `ADD_ALBUM` message, which means the Portal was trying to initiate a file transfer!


Regarding the ye tracks not loading, the Network tab showed the cause. I was getting 403'd because the serial number was invalid. How could I get a valid number? I tried to ask on public ye servers but got no response. I was stuck, if Donda 2 was released, I wouldn't be able to get the album. That was when I got an idea, what if the serial number is printed on the box? It's not uncommon for that to happen. I google around for unboxing videos and marketplace listings with the original box. It didn't take long to find a valid serial number. I add it to the code, and it does solve the issue! This was not the end, I was trying to avoid real serial numbers. I start testing and for some reason, all the serial numbers yield a valid response. In fact, as long the serial has 24 numbers, it will work. The best keygen was then born:

```js

generateValidSerialNumber(){

    let res = '';
    for(let i = 0; i<24; i++){
	res += parseInt(Math.random()*10);
    }
    return res;
}
```


## Downloading the first stems

 I now had a big decision to take. How would I download the stems? To upload to the device, the Portal issues a `fetch` request to get the stems' temporary download links. I could hook the function, get the result and download them separately, but that would make the emulator flakier towards updates. E.g changing the update or encrypting the data would be enough. It was already 10PM and I really didn't want to add a new message handler to the emulator. Still, implementing it would make it much harder for Kano to fix the emulator since it would require a firmware update. I opted to implement the file transfer messages.


 Once the upload button is pressed, the Portal sends commands to create an album and a track on the Player. It immediately requests data to confirm it was successfully created. This implies data needs to be persisted on the emulator during the transfer process. Classes `Album`, `Track` and `Stem` were born to fulfill this need. At first, the download was slow that's why I had a console log with `console.log('Content is at ' + this.written+'/'+this.size);`, the reason was that I was manually expanding the `Uint8Array` with each chunk of data. Since the stem size was communicated before sending it, I could simply preallocate the array, which turned a process that lasted seconds instantaneous.


Finally, triggering the browser to download the stem data was done with HTML magic. Create a hidden anchor object with an `href` that is the base64 encoded stem content, add it to the `<body>`, `.click()` on it and then remove it. The click will trigger the download.

```js

   function downloadContent(content, name){
        base64_arraybuffer(content).then( (data) => {
            const element = document.createElement('a');
            element.setAttribute('href', data);
            element.setAttribute('download', name);
            element.style.display = 'none';
            document.body.appendChild(element);
            element.click();
            document.body.removeChild(element)
        });

    }
```

It was now 2AM and I had a working emulator that could download any of the tracks. It was time to wait for the album release to make this public!


## Releasing to the public

I took a day off to work on other projects and on the 20th I recorded the tutorial and set the video to unlisted. Many news publications think I made this after the album release due to confusing this with another project. Still, as seen on the video, it was recorded on the 20th.

{.video https://www.youtube.com/embed/QqBiKZmr5rw}

The listening party came and went, but the album wasn't released. The next day I was furiously refreshing social media to see if it would release. It was almost 6PM and I was tired, so I decided to release and close the computer for a few hours. It was the best timing, a few moments later, the album was released. My original post was done on /r/WestSubEver and got some attention, someone posted it on /r/Kanye and it blew up. The comments were overwhelmingly positive and praised the work. I had never had such a successful product.

## Making the Emulator better than the actual device

I was thrilled with the feedback, so I desired to take the emulator to the next level. The first thing I did was allow the users to download higher-quality stems from the Portal. Sadly, the Portal was coded in such a way that only the lower-quality ones are accessible. The solution was to intercept the `fetch` request and replace the query parameter with a higher quality one, `mp3->wav`. Users think they have high-quality stems because of the uncompressed audio, but in reality, when the device receives an MP3 file, it converts it to WAV. Replacing the query parameters makes the Portal download the higher quality ones.


The next significant feature was to expand the emulator to all browsers. I didn't notice at first, but the Web Portal doesn't use feature detection to find whether `WebUSB` is available, it uses browser detection. Their browser detection is straight from [stack-overflow](https://stackoverflow.com/questions/9847580/how-to-detect-safari-chrome-ie-firefox-and-opera-browsers):


```js
// Opera 8.0+
var isOpera = (!!window.opr && !!opr.addons) || !!window.opera || navigator.userAgent.indexOf(' OPR/') >= 0;

// Firefox 1.0+
var isFirefox = typeof InstallTrigger !== 'undefined';

// Safari 3.0+ "[object HTMLElementConstructor]" 
var isSafari = /constructor/i.test(window.HTMLElement) || (function (p) { return p.toString() === "[object SafariRemoteNotification]"; })(!window['safari'] || (typeof safari !== 'undefined' && window['safari'].pushNotification));

// Internet Explorer 6-11
var isIE = /*@cc_on!@*/false || !!document.documentMode;

// Edge 20+
var isEdge = !isIE && !!window.StyleMedia;

// Chrome 1 - 79
var isChrome = !!window.chrome && (!!window.chrome.webstore || !!window.chrome.runtime);

// Edge (based on chromium) detection
var isEdgeChromium = isChrome && (navigator.userAgent.indexOf("Edg") != -1);

// Blink engine detection
var isBlink = (isChrome || isOpera) && !!window.CSS;

```

If a user tries to connect using a non-Chrome-Browser, the button will change to "USE CHROME OR EDGE". Superb engineering, Kano!
There is a reason why browser detection is frowned upon, it relies upon internals which are subject to change leading to hurting more users than helping.

Making the emulator work on all browsers was quite simple, just fulfill the requirements:

```js
if(!!window.InstallTrigger){
	window.InstallTrigger = undefined;
	}

if(!!window['safari']){
	window['safari'] = {};
}

if(!!window.opr){
	window.opr = undefined;
}

if(!!window.opera){
	window.opera = undefined;
}

Object.defineProperty(navigator, 'userAgent', {
	value: navigator.userAgent.replaceAll(' OPR/', '').replaceAll('SamsungBrowser', ''),
	configurable: false
});
```


At this point, I had created an alternative to the physical device, which was better and free. I was waiting for Kano to take any action.


## Serial Number Enforcement

One of the very first forks of my work was from a Kano employee. It took them quite a while to take any action to prevent the emulator from working. After some time, they did add Serial Number checks. At this point, I had already built a collection with hundreds of valid serial numbers. A few users have also contributed some working ones. I had enough to allow daily changes of serials over 2 years. The problem with blocking serial numbers is that it is not possible to do so without hurting real clients, it was not an option. Kano was defeated in this regard.




## New Listening method added


Stem Player buyers were complaining that the extraction process was annoying and slow. They had to have the device connected to listen to the mixed tracks. Even if they extracted the stems from the device, they had to mix them in a program such as Audacity. This was not the 200$ experience they signed for. Kano added to the Portal the possibility to listen to the Donda 2 album without connecting the device. The sole requirement was to insert the buyer's shipping email.

Emails started to be brute-forced almost immediately and collections were circulating around. Some dummy emails, such as `kanye@gmail.com` and `p@gmail.com`, worked for some reason. My first guess was to test with Kano employees, starting with Alex Klein, the CEO and founder of Kano. I insert `alex@kano.me` and BAM! It works. The login is also persistent through local storage, so I quickly add it to the emulator.



```js
window.localStorage.setItem('production_sp_basic_session', JSON.stringify({"AuthToken":btoa(atob('YWxleEBrYW5vLm1l')+':'+Date.now())}))
```

Once again, the emulator has surpassed the actual device in terms of features! You could now listen to the fully mixed tracks or download the tracks. I was curious how they would deal with the email situation due to how badly it was planned. First, resellers would have a single email associated with multiple devices. If you bought your Stem Player on eBay or any second-hand marketplace, you wouldn't have access to this feature. The same is true for those who purchased the Stem Player at the listening party or pop-up events organized by ye and Kano. Pop-ups are sporadic events that last less than a day(usually announced on the same day) in random places and allow fans to buy merch and the Stem Player. This was quite funny because if you're a big fan and got the device in-person from an event, you'd get a worse experience. I was curious to see how they would deal with this.



## Email + password update

Their solution to the created clusterfuck was even dumber. Kano emails and dummy emails were disabled first. When I fell back to a semi-public list of valid emails, they added a password requirement to listen to the tracks on the website. How did they solve the problem for people with a secondhand devices or bought in person? Simple, just message their Instagram account or send an email to `help@stemplayer.com` and pray to get a response.

I tried to find any vulnerabilities but their use of AWS Cognito seemed to be done correctly. Congratulations, it only took Kano 3 weeks!!!! Emulator users still had the upper hand because they could access the same tracks without needing to have a device connected ;)


## Locking access to ye tracks

A firmware update was released at the end of March (1 month + 1 week after the emulator release). The device had to answer a challenge if you wanted to download a track from a ye album. Access would only be granted if answered correctly. Link/Local file uploads still worked fine since they had no challenge. Firmware update files were encrypted, I knew some users with the device had already reverse engineered the operating system but were unwilling to associate with the project, which is understandable. I didn't want to spend 200$ on a Stem Player, so I added a notice to the README telling users that if they wanted to support me, they could through [PayPal](https://www.paypal.com/donate/?hosted_button_id=N6QZL2267QYLU). No one was forced to donate, if you can't or don't want, then I recommended people to find mirrors of the album. Additionally, suppose someone has a Stem Player and wishes to donate one. In that case, I'll kindly take it or simply a decrypted firmware, anything that will help the emulator development.


The emulator still offered features the Stem Player didn't, downloading stems through the Web Portal and access to higher quality stems.



## Kano adds the ability to download stems off the Web Portal (but wait)


It was already May 16th, so 3 months after the release of the emulator. Kano announced that it was now possible to download stems for user-split tracks only in low quality. User-split tracks are link/locally uploaded files split by their back-end.

Once again, the emulator is still the best option. To be honest, I don't know why it took them so long and why they didn't add the download option for ye tracks.


We are already in August, and no update has been done to the website since then. The following section will cover how badly the Kano team and ye promoted and handled the product on social media.


## Tricking Kano

To prevent Kano employees from noticing I had updated the emulator, I resorted to not announcing updates. There were two components for the trick to work:

- The git history, when an update was pushed, I'd squash the commits and change the date so that it would like I haven't updated the repository
- If a user sent me a message that it was not working and I had already created a fix, I would tell them it was working for me and that they should reinstall the emulator

The idea was to make it seem like I hadn't touched the emulator and confuse the employees tracking the repository.







# Stem Player or hype as a product

According to several sources, ye has been trying to develop a product similar to the Stem Player for years. Before working with Kano, he worked with Teenage Engineering, which repurposed the developed product as an actual mixing device. When it was released, unsurprisingly, only die-hard fans actually got it. Nothing came out of it until the days before the Donda 2 listening party, as if ye was unhappy with the sales and wanted to force people to buy a Stem Player. Anywho, ye isn't the only one at fault. The following sections will cover several bits of information to showcase Kano Tech's incompetence, especially of Alex Klein, the founder and CEO.


## Kano Tech doesn't manage their Discord server

On the Web Portal, there's a button called `FORUM`(previously called `COMMUNITY`), which contains an invite to a Discord Server, the official Stem Player Discord Server. The server was created by a fan and managed by fans, who are all volunteers. Some Kano employees are on the server, but they rarely talk. What do people do there then? Mostly complain about defects and malfunctions. It was exceptionally active near the Donda 2 release, with people constantly asking whether the album was out.

The lack of updates and engagement turned the server into memes and complaints about the device.


![People complaining on discord](images/stemplayer/complaint.jpg)

That's right, the device can't handle Donda and Donda 2 at the same time.


## Screwing up their users


If someone bought a Stem Player on a second-hand market, during the Donda 2 listening party or during a pop-up event, they wouldn't have access to the album playback feature due to not having a purchaser email. This means that the biggest fans willing to spend 200$ in person would have a worse experience. The only solution was to shoot Kano an email and pray they wouldn't ignore it.

The user-guide? The best we can offer are some doodles. If you want a real user guide, hop into our Discord and check out the community user guide. Leaving the work of figuring out the device to the community is such a 200$ experience.


Want to disassemble the device? Be careful when opening because the speaker cables are so poorly soldered you can rip them off by accident.


## Lack of identity

The device has some primitive remixing functionality, limited to muting stems, speed up, slow down and reverse. It didn't take long for users to notice those features would be much easier on any audio editing software. If you watch Kano promoted remixes, you'll notice something interesting. Most of the work is done off the Stem Player, the device is mainly used as a speaker. Marvelous!


The number of different ventures Kano and ye try to throw the Stem Player into is astonishing. My favorite is the "STEMWEAR", clothes with a pocket for the Stem Player.


![Stemwear](images/stemplayer/stemwear.png)

## Flexing the revenue

After announcing that Donda 2 would be Stem Player exclusive, ye decided to open about the product sales. Since his post (less than 24 hours had passed), $1.3M of net sales had been done, totaling 15% of total net sales. This only motivated me more to work on the emulator.

![Flexing the revenue post](images/stemplayer/revenue.png)

## No artists besides ye

It's been a year since the product's release and to this day, their official catalog only offers ye albums since Jesus is King. Why? Only Alex Klein knows. On the day before the listening party, he asked people to message their favorite artists to bring them to the Stem Player platform, but nothing came out of it. Artists, such as Rich The Kid, have previewed some songs on their Instagram page using the Stem Player and publicly showed interest in joining the platform. Other artists, such as Jack Harlow, have also shown admiration for the product. All these conditions were met and still, nothing came out of it.

![Alex asks people to bother artists](images/stemplayer/alex_email.png)
![Rich the kid](images/stemplayer/rich_the_kid.jpg)
![Jack harlow with stemplayer](images/stemplayer/jack_harlow.png)

So, the CEO asks fans to bother the artists, the artists show interest, but still nothing. Their catalog has been stagnant since February.



## No business model

One of the points ye shouted the most on the social for the creation of the Stem Player was that streaming took a big chunk of his revenue, so the Stem Player would allow him to get direct access to it, no middle-man. There's only a tiny problem, HOW!? Research and Development isn't cheap and Kano employees have salaries. This takes a chunk out of your revenue. Also, besides the initial cost of 200$, how is ye expecting to earn more money? It's not like if he releases new songs on the platform, people have to pay for them, right? RIGHT!?

A couple of hours before the release of the Donda 2 album, users scrapping Kano APIs noticed a new field on the JSON response `paid_content`. All of the songs were set to `false`. So besides the 200$ device, ye and Kano expect users to buy future songs/albums? We can already do that with other services, what does a 200$ piece of plastic change?

I suppose the artists interested in joining the platform were prevented by their labels/managers because there's no way in hell they'd see the money. How would the revenue split work? Usually, users only download the song to the device once. The device is not programmed to track how many times a track was listened to; if it was, it has no Wi-Fi for this kind of telemetry. It's a complete wreck.

## Deceptive marketing

Their marketing consists of the following: they're changing the music industry completely and nothing will be the same. Alex Klein said the following `It's time to overthrow the oppressive system And build our own`. Do you mean the system that allows me to access any song without a 200$ entry fee?


![Alex Klein delusional messages](images/stemplayer/oppressive.png)



Out of all the posts, the one that really got under my nerves was the following [Instagram post](https://www.Instagram.com/p/CaK8vwhjNxh/):

![Alex Klein delusional marketing](images/stemplayer/bs_insta_post.png)




It contains a quote from Alex where he mentions ye said he liked his work and a video with a historical overview of music distribution. The first step is the "LP ALBUM" where the process is linear, curated and physical, so no EPs or singles, I guess. Also, what's the problem with curation? If a label is putting money on an artist, of course, they'd like to make sure it's not going to waste. The next phase is "MP3 and STREAMING", marked as 2003, everyone remembers streaming music in the early 2000s, right? It is described as shuffle, incomplete and digital. First of all, what the hell do they mean with shuffle? Digital music formats made it easier to rearrange listening orders, but is that really important? INCOMPLETE!? The early 2000s was the time of peer-to-peer file sharing, torrents, Limewire, Napster,... It was the best time to access entire artists' discography if something, it was far more complete. Before, if the record store didn't have the record, tough luck. Now it was just a few clicks away. The last stage is the "STEM PLAYER" where music distribution is interactive, creative and decentralized. I'd like to find a more interactive system than the play button. TikTok is a far more creative music distribution system than the stem player. The part that annoyed me was the "decentralized" while the animation shows lines connecting different Stem Players. Are they implying Stem Players can send songs between each other? Because they can't, it's one of the most centralized services ever. High entry fee, proprietary gadget and proprietary Web Portal, nothing about it decentralized.

## Donda 2 stems were terribly split

After release, users quickly noticed that most tracks were not correctly split and had weird production artifacts that should never reach the public, e.g On Louie Bags the drums were clipping. Someone on Twitter tagged Mike Dean, a long-time ye collaborator and music legend, and I hoped on the thread to show him how bad it was, to which he replied:

*ThTs how some songs are. Sometimes u don't get stems from the producer. And it is what it is.*

Alex Klein liked that post (he removed it later, but I took a screenshot :D) which prompted my response asking him if that was the 200$ experience. Why would the founder of a self-proclaimed platform that is freeing music side with such a stance?


![Alex Klein liking dumb post](images/stemplayer/alex_dumb_like.png)

## Donda 2 lack of updates killed the hype


Donda 2 was released in an unfinished state. The idea was for ye to keep updating the songs as they were finished. The kicker was that the album stopped getting updates long before the challenge firmware update was released. Everyone who downloaded the stems before that update already had access to the latest ones in the best quality.

It was a turbulent time in ye's life with the divorce, but it was Kano's fault for not onboarding more artists on the platform. Nowadays, I can find Donda 2 mirrors and there's no difference from the one on the Stem Player. Relying on a single artist was their crux.



## Taking credit for the splitter algorithm

Alex Klein likes to retweet people showcasing the device, especially when a new album is out and people are playing back the isolated vocals on the device. According to Spleeter's disclaimer, one should get proper authorization before using the software on copyrighted music. But who cares?

When someone says something along the lines of "the splitting algorithm used for the Stem Player is great!!!!", you bet you'll get a retweet, but zero mention of the fact they are using open-source software.


![Alex Klein taking credit marketing](images/stemplayer/taking_credit.png)

On May 16th, they also announced they're moving towards a new splitting AI with fewer artifacts because people complained Spleeter wasn't good enough. Do you know which splitting algorithm they are using now? That's right [Demucs](https://github.com/facebookresearch/demucs), another open-source software while giving no attribution!

Who'd guess the 200$ experience wasn't more than a clunky piece of plastic and open-source software?

![New Splitting algo](images/stemplayer/new_split.png)




## "Visionary" mentality

Suppose you do a quick check-up on Kano tech social media, including the employees with public pages. In that case, you'll notice they have a "visionary" mentality and are full of themselves. If you don't like the product, it's because you lack vision or can't understand how the future will unfold. The CEO has tweeted that the only true way to listen to Donda 2 is through the player. Every positive comment about the gadget gets an immediate like and retweet. Call the creator a genius, retweet; say the product is revolutionary, retweet.


![Alex copium](images/stemplayer/alex_copium.png)

Added to this are the numerous posts with self-quotes or ye calling Alex a genius. Isn't this the 200$ experience?




## Broken promises

If someone requests something for the Stem Player and gets a response, it will likely be the good old soon™.

- "Can we get an SDK to modify the device's contents without the need for an internet connection?" -> soon
- "Mashup update?" -> soon
- "Can we upload our own stems?" -> soon
- "When can we have another Q&A session?" -> soon, in fact, we'll announce next week
- "When are orders going to start to be shipped?" -> soon...actually it's the covid fault
- "When is the black Stem Player going to release?" -> soon


As you might guess, absolutely nothing materialized out of these. Regarding the SDK, it wasn't done because they're doing "foundational work" that will only be shown when "ready" - here's a [tweet](https://twitter.com/alexnklein/status/1472049149797445633) of Alex Klein from December 18th 2021 saying it's coming soon and him again on March 18th 2022.

![SDK soon](images/stemplayer/alex_sdk_when.png)
![SDK soon](images/stemplayer/alex_sdk_when_2.png)

Regarding the mashup update, uploading own stems and the Q&A, Alex promised them and hasn't sent a message on the Discord server since March (5 months ago).


![Mashup update](images/stemplayer/alex_mashup.png)
![Upload own stems !!!](images/stemplayer/alex_own_stems.png)
![QnA next week!!!](images/stemplayer/alex_qna_next_week.png)


Regarding orders not being shipped, Alex claimed to talk with the courier and that they should lift Covid restrictions and resume as normal after "next" week... orders were still delayed.

![Covid lockdown fault](images/stemplayer/alex_covid.png)


Regarding the black Stem Player, it was shown on the official Instagram page and nothing came out of it.


## NFTs

Alex Klein is [into crypto](https://twitter.com/alexnklein/status/1362931574946344962) and NFTs. Somehow he thinks that's the future for artists. He has pitched the idea of NFTs on the Stem Player multiple times and received a negative reaction.

![Stem NFTS](images/stemplayer/stem_nfts.png)

The final nail in the coffin was a ye Instagram post that he did during the Donda 2 recording sessions that said the following:

"Do not ask me to do a fucking NFT

                ye


        ASK ME LATER"

Initially, there was no "ASK ME LATER", someone on his team suggested that closing all future NFT money-making related ventures could be harmful, so he added the last line. Thank god ye, a billionaire, makes sure not to close any possible future money-making avenues.

Alex even liked someone that claims the Stem Player has more community than most NFTs. If by more community you mean an inactive Discord server full of complaints, then yeah! At least Stem Player users know how shitty the product they bought is.



![More community post](images/stemplayer/more_comm.png)


## Should've been an APP

This was a common concern from the fans. A 4-channel mixing device isn't quite the 200$ experience. Lots of fans gladly mentioned that for 5$, they would buy a ye exclusive app to listen to his album. An old video with Mike Dean surfaced where he was mixing a ye song on his iPhone.


{.video https://www.youtube.com/embed/dAjD6o9dm-A}

Not only is an app more versatile in terms of features, but it would also help reduce the cost of the product.

Kano has hinted multiple times at the fact they're developing an APP, but nothing has come out of that.


## Unfullfilled(-lable) job listings

Shortly after the album came out, Kano announced some job openings. In April, they created a channel on the Discord server dedicated to sharing their openings. If you visit the links today, they all seem to still be open:

- Junior QA Engineer (Web/Consumer Devices)
- Senior Social Media Manager
- QA Engineer
- Software Development Engineer in Test
- Full Stack Developer
- Mid Backend Engineer
- Tech Lead / Front End Engineer
- Mid Android Native Application Developer
- Senior Android System Developer

The last two job openings gave the impression to the users that they are working on a mobile app, but 4 months have passed and no updates on any of the openings. These were posted on Otta, but if you visit their website, the job openings link to [Workable](https://apply.workable.com/kano-computing-limited/). The top banner doesn't even say "Kano", but "STEMPLAYER".

![Workable page](images/stemplayer/workable.png)

Their description is top-notch. The first bullet point contains the following quote - `biggest thing to happen to music since the iPod` - wow! It came from a [medium article](https://medium.com/@liammaccormack/the-stem-player-is-the-biggest-thing-to-happen-to-music-since-the-ipod-and-heres-why-19d71ddff29e) from a guy with 4 followers and 3 total articles. Great start! The second bullet point mentions how their technology is made for creation and not consumption... the only way to consume Donda 2 is by owning your device. The third point mentions how they formed in 2014 and since then have sold more than a million units worldwide, starting first with computer coding kits(more on this later). The fourth point talks about how they won multiple awards while omitting the year and product.. "TIME Maganize's Invention of the Year" is awarded yearly to 100 products, and their DYI PCs was one of the listed products in 2019; For "Fast Company's Top 50 Most Innovative Companies", Kano also made the list for DYI PC in 2019; The "Golden Lion at Canne" I couldn't find any information, but if they actually did win something it was for their DYI PCs. The fifth point is composed of quotes about DYI tech and the sixth is about when they started to work with ye. Overall, a lot of showing off of a product they don't produce anymore.


## Kano isn't Kano

From the latest section, you were probably confused about DYI tech. Kano started as a company that built computer kits for children to learn to code. They mainly were custom casings that were disassemblable and some peripherals such as mice, keyboards and headphones. You can easily find articles and videos on YouTube where Alex Klein is showing off the product to news outlets. You can also find famous people such as [Mark Zuckerberg](https://twitter.com/alexnklein/status/1400739467921694720) and Djokovic talking positively about the product. So.... what happened to the custom DYI PC kits? If you access their website, it's out of stock, and a few people are asking for a restock. You can find very few kits on eBay, which makes me believe they probably never sold any and just distributed when visiting schools. *Speculation*: After linking up with ye in 2019, they completely halted their ventures in educational technology and focused on the Stem Player.


If you visit their GitHub page, you can see a lot of old repositories. If you check the contributors to those repos, you'll quickly realize most people are not working at Kano anymore. Nothing against someone who saw an opportunity, but it sets a record of a company that gets a lot of hype in a short span of time and then massively underdelivers and ignores the past. Also, according to [this](https://www.cnbc.com/2020/07/14/kano-microsoft-backs-start-up-looking-to-challenge-google-in-edtech.html) news article Kano was founded in 2013, which is backed up by their [Wikipedia page](https://en.wikipedia.org/wiki/Kano_Computing) where it listed when they launched their products.


One quick look on Twitter, one can find other products that never materialized such as:

- [VR kit](https://twitter.com/teamkano/status/715970504264384512) - link to their website is dead and from what I could find, none were produced
- [Code with dazzling lights](https://twitter.com/alexnklein/status/1367256062391246854) - posted 5 months before the release of the Stem Player to the public, as other DYI kits none seems to ever been produced or sold

Are they still on the DYI and teach children to code market? Who knows? I don't think they know.




## ye collaborators don't like the Stem Player

Decided to include this part because it is interesting, but the circumstances around it are still a little muddy. So, this starts with a collaborator of ye, Theophilus London, accidentally leaking the phone numbers of many people working with ye, including Mike Dean. Mike Dean makes some posts threatening the dude for being careless. Meanwhile, on [this post](https://www.Instagram.com/reel/CbNB2D6jtYj/) he leaves the following comment:


*We all hate Alex. Stem players a joke. No royalty. Plz stop Dick riding ye*


![Mike dean sends shots at Stem Player and Alex](images/stemplayer/mike_dean_shots.jpg)


Some users were quick to point out he might've not been talking about Alex Klein since the post tags a producer called Alex. The thing is that he said, "We all hate Alex", implying the group of people he hangs out with doesn't like Alex, which would be weird not to be Alex Klein since he works closely with ye. Also, as mentioned before, Alex Klein doesn't miss a chance to praise ye or praise his work by proxy of ye, aka is a colossal dick rider. Regardless, it highlights an interesting part, that there are no royalties. Guess the revolution was to take more from artists :)


A few hours later, Mike Dean tweeted that his Instagram was hacked. A few fans noted that it was indeed a possibility since his phone number leaked. Still, most fans quickly pointed out that he was posting normally on Instagram even after the comment. The only bad thing the hacker did was leave a comment talking down on the Stem Player on a random Instagram post; something was fishy. The consensus is that he was not hacked and in the heat of the moment(after his phone number leaked) he was pulling no punches.


An important thing to know is that people who work with ye are constantly being threatened by hackers trying to access ye's material, such as unreleased songs, original project files and stems. Earlier in June, a previous collaborator of Ye, Allan Kingdom, made a [tweet calling out ye](https://mobile.twitter.com/AllanKingdom/status/1532362311599546368) for asking him for stems and not paying him. Turns out it was an impersonator. Mike Dean, in the past, even doxxed a known leaker for trying to get access to his cloud accounts. This is to say, a hacker wouldn't waste his time throwing shots at random people while having access to such a valuable account. Mike Dean has been mixing ye for a really long time. If someone would have the best unreleased material and access to stems would be him.


Finally, let's not forget that Mike Dean didn't export the Donda 2 stems himself and used the Stem Player's back-end, showing that he does not give a shit about the product.





## Alex Klein's weird social media

To the general audience, Alex Klein is Kano's CEO and developed the Stem Player with ye. As previously mentioned, he doesn't miss a chance to tweet a quote from ye claiming how smart he is. One trip to his public profiles is enough to understand everything that has been said until now. He believes he's a visionary, comparing himself to others such as ye and Elon Musk(I won't even start on dumb this is). In fact, he tries too hard to be them. There's a photo of him, ye and Musk talking at SpaceX. He made sure to post this on his Twitter, Instagram and set it as a LinkedIn cover; cool, isn't it? During a [Complex interview](https://www.complex.com/music/kanye-west-stem-player-alex-klein-interview/interview-3) he was asked about their meetup, to which he said the following:

*I want to be very respectful of the privacy of everyone involved. It was a meeting between Elon and Ye, first and foremost, who I know have been friends for many years. And all I'll say about that is, I'm very grateful to Ye for including me, because it meant a lot to me.*



![Alex Klein, ye and Elon](images/stemplayer/alex_elon.png)

If you want to respect their privacy, then why post it on all of your social media?


Unsurprisingly he is a ye fanboy and doesn't lose a chance to share with the world when he's hanging with him or the contributions he has made to ye's albums. He wrote what is arguably one of the worst verses in ye's discography for the song `Water`. He's so proud of that verse that he responds to random Twitter threads about [underrated ye songs](https://twitter.com/alexnklein/status/1374524408430170114) and [lyrics that belong in a museum](https://twitter.com/alexnklein/status/1374768537567440900) with that exact song. Funnily enough, someone back in 2019 called him out for being such a [ye meat rider](https://twitter.com/alexnklein/status/1193008832093536256).



What really triggers the fanbase are the posts where he showcases upcoming products/releases while not indicating whether they'll be released. e.g right after the Donda 2 listening party, he shared that Mike Dean had sent him the stems to post on the Web Portal. Somehow it took them 10 extra hours to actually upload them. Another popular instance of that happening was with the custom-colored Stem Players, showed on their Instagram page but absolutely no information regarding if they would be made available.

![Mike sends stems to Alex](images/stemplayer/mike_stems.png)


After months of inactivity, Alex in the middle of July decided to post a [joke tweet](https://twitter.com/alexnklein/status/1546913843414269962) with the list of Stem Player "competitors". As expected, people didn't take it too kindly and started to clown him. Here are my favorite responses:


![Donda 2 is the real opp](images/stemplayer/alex_clowned/alex_donda2_comp.png)
![Saner list](images/stemplayer/alex_clowned/alex_lmao.png)
![Kano's fault](images/stemplayer/alex_clowned/alex_real_competition.png)
![Best of them all](images/stemplayer/alex_clowned/alex_best_clown.png)


My favorite is the last one because a valid criticism and a user is explaining to him that he lacks the vision.


Finally, when confronted with delayed packages, he answers to users that he is [checking](https://twitter.com/alexnklein/status/1504483526242045969), but gives absolutely no follow-up.





## Misunderstanding the technology

What happens when you don't understand the technology you're working with and it doesn't work? It's the other's fault, of course! Earlier this year, Jen Simmons, a Safari developer, made a tweet asking people to be honest on why they dislike Safari, to which one of Kano developers [said the following](https://twitter.com/neuemodern/status/1491460921465749515):

*Lack of WebUSB/WebBT support severely hinders people trying to create open and free hardware technology.*

There's a lot to unpack here. Most browsers don't support WebUSB/WebBluetooth due to valid [security concerns](https://mozilla.github.io/standards-positions/#webusb). How about instead of a web app, you develop an actual application? It would remove the internet connection requirement and could be turned into an electron app.

"hinders people trying to create open and free technology" shows their lack of self-awareness. The Stem Player is not open - no SDK, proprietary firmware and no schematics - and is not free, there's a 200$ price tag, for god's sake.


Checking their profile, there's also this [banger](https://twitter.com/neuemodern/status/1124037737991028736):


*It's time for a node based replacement for Photoshop that actually uses the power of a modern computer.*

"node based" and "uses the power of a modern computer" aren't expressions I expected to hear together. In fact, there's an urban myth that if you say "node is close to metal" three times in front of a mirror at 3AM, Dennis Ritchie will come out of it and punch you in the throat.

Guess when you only have a hammer, everything looks like a nail?



# Alex Klein's fake job offer


TL;DR Drama. Alex Klein and Kano employees led me on with the premise of a job offer but went cold right after I showed interest. This section mostly shows how slimy they are.

Until now, I only talked about the publicly accessible information. Now I'll give you an insight into the behind-the-scenes between Kano and me, more precisely Alex Klein. Alex DM'd me almost instantly after I tagged him on the Mike Dean thread. He wanted my phone number to speak but wouldn't divulge more information. Tried to give him my email, but he insisted on the phone call and gave me his number. It was Sunday, and I couldn't get another SIM because the stores were closed, so I did not give my number and agreed to call the next day.

![Alex first DM](images/stemplayer/alex_dm.png)

I got a brand new SIM card early in the morning and messaged him. He immediately called me back and we talked a bit, but the connection was crap, so we moved to Discord. During the call, I explained to him how the emulator works. He told me they were trying to add access to the Web Portal without requiring the emulator and was impressed I did it in a single day. The most important thing he said was that ye knew about the project and would like to take the project down, but also hire me as a "Community Development Lead", whatever that means. To start things on a good note, he asked me to take the project down. The call ended abruptly since he was getting a call from ye. Later, I got an email from HR asking me some questions and a message on Discord from another HR employee telling me they'd like to offer me a job.

![Alex whatsapp call](images/stemplayer/alex_first_zap_call.png)
![Alex discord call](images/stemplayer/alex_discord.png)

I sent a message to Alex telling him the repo was cleared and issues disabled to prevent people from asking what happened. Alex was happy and gave me two contacts, the HR employee from the email and the lead engineer and asked me to contact them.

![GitHub repo cleared](images/stemplayer/zap_clear_repo.png)



After this point, it's where the shitstorm begins, three parallel conversations ensue with Alex, HR and the lead engineer. In summary, they ignore me as soon as I show any interest, which I don't take too kindly.



## Talking to the lead engineer


Since our call ended abruptly, I asked Alex to have a call the next day, to which he agreed and suggested a time. Meeting time comes and I ask him if we're hopping on Discord, to which he asks me, `Are people still able to use the emulator?`. I tell him the truth, even though the repository is gone, people who have it installed can still use it. He tells me he won't be able to meet me and asks me to create a one-page development proposal and send it to a group chat with the lead engineer. It's important to note I haven't signed anything yet.


![Alex requests](images/stemplayer/alex_cant_hotel.png)


I created a group chat with the lead engineer and asked to set up some time with him, but there was no answer. The next day I ping him again and he tells me he's been too busy with the release of the album and that he'll be on vacation until Monday, note that we're already on Wednesday.

![Engineer vacay](images/stemplayer/zap_group_holiday.png)

It felt like a slap on the face, Alex asked me to remove the emulator as an act of goodwill, but his team could not arrange a meeting with me or answer me in time. If I wanted the job offer to move forward, I'd have to keep my project down for a week. What a nice thing for Alex! I asked the engineer if he was up to meeting the next day. Alex decided to intervene, telling me to be patient and that they'd review my one-pager when I sent it. So you're asking me to work before signing anything or talking with the man who architectured the back-end?


![Alex decides to defend the engineer](images/stemplayer/alex_intervene.png)

I did not enjoy that kind of response. I don't blame the engineer but Alex for throwing me into that awkward situation. I was mad since he seemed so excited about my agreement to work together and now was telling me to take it slow. I send him a message telling him to step up and show me a contract the next day. Suddenly, the engineer replies to me with what I can only say is the best response ever. Sorry, but "we're shipping magic with Stemplayer".



![Shipping magic](images/stemplayer/shipping_magic.png)

As expected, the engineer didn't say anything until Monday morning, asking if I was ready to meet... We meet, and I ask the engineer about Kano's tech stack and workflows. It was a decent meeting and I learned a lot about how disorganized they really are. Fun fact: the firmware is outsourced. Fun fact 2: the serial numbers were not verified at first because they didn't know which products were actually shipped.




## Talking to Alex Klein

Rewinding a bit to when I sent the first message to the engineer. After being ignored by the engineer, I tell Alex I have a first draft of the proposal and ask for the purpose of it, to which he gives me a vague answer. I then ask if everything is okay with the engineer since he hasn't read any of my messages. Alex's response? "We are super busy. Did you send the proposal?"


![Alex dumbass](images/stemplayer/alex_dumbass.png)


"Oh? The person I asked you to contact, don't bother, just send the one-pager I asked for. Also, can you sign an NDA with us too?" The NDA came up because one of the HR Discord contact started to ask personal questions about my thesis work, to which I replied it was under NDA. Alex's follow-up was even more confusing.


![Alex lacking braincells](images/stemplayer/alex_nda.png)


Annoyed by Alex's lack of direction and his sudden "please stay chill" attitude, I send an angry thread of messages telling him that I've taken time off my work to help them and that if he was as serious as he seemed on our first and only call then the following message better be a contract. No response from him...


*NOTE: This was on the exact same day as the "shipping magic" message*


![Alex step up](images/stemplayer/alex_contract.png)


I put the emulator back online since there was no chance I'd keep the project down for a week with no signed contract and for the opportunity to speak with a person from the development team.


## Talking with HR

During the process, I spoke with two HR individuals, one through email and another through Discord. The one on Discord was not super into the process. In contrast, the one on the email was the person actually handling the hire. My last email exchange was about salary expectations and happened right after the Alex Klein phone call.


Two days go by, so we're on the day the engineer tells me he can only talk Monday. I got an email saying that it's "manic at the moment" but that they'll "get back to [me] **ASAP today**", the response didn't come that day, shocker! Through Discord, I was informed that the email person hasn't been available for the past few days.... what the hell? Since the Discord contact was replying swiftly, I ask him about the engineer, to which I got the following response:

*X is a member of our software team, doesn't usually speak to anyone external. When did you speak to them?*

*They are best on email and in person :) you two will be great together. No, it's no joke*

I ask them to notify Alex and the engineer to read my messages, to which I get the "they're busy people" response, but that they'd do their utmost to notify them. That's when the engineer and Alex started to reply to me in the group chat.

The day after telling Alex Klein to step up, I got an email from HR asking for some bank details and to fill out a new hire form.

![New Hire Form email](images/stemplayer/new_hire_form.png)


It was March 3rd, I had officially filled up the form and was ready to move forward with Kano. Well... The weekend goes by and on March 7th, Monday, I get a message from a random employee telling me that they're "freezing this workstream" and to remove an IG account(?).


![Freezing workstream](images/stemplayer/random_hr.png)


I waste no time and swiftly thank Alex for the experience.

![Thanks Alex](images/stemplayer/stole_ye.png)

His response was great, that my language was abusive and that I stole ye's content. Also, what I wrote constitutes a breach of terms of service and I'm harassing him. I never agreed to any terms of service, nor I'm stealing anything. You even retweeted a Google Drive link for Donda 2. Announcing the exclusivity so close to the album release made it impossible to people to acquire the device on time. Finally, harassment? It's not my fault you cannot coordinate your employees to onboard a possible new hire.


This all got me thinking... were they trying to kill two birds with one stone? Keep me hanging with the "job offer" while adding new features/fixing the emulator. Seemed like it. As you know, it only took them 2 more months to partially fix the emulator...


## HR Discord contact messages me later

This happened on the 8th, so 1 day after being noticed of the "freezing" of my hire process. The HR contact that uses Discord messages me surprised that the emulator is back up. I explained that my process was frozen, so there was no reason to keep the emulator down. Hilarity ensued.


The subsequent messages were about how much they wanted to work with me and that the emulator was illegal and if I didn't take it down, they'd take the "legal route". I explained to them that their lack of security wasn't my fault, and if they wanted to sue, go ahead. It then escalated to my favorite message exchange ever:


*HR: We are requesting you remove the Github account.*

*me: my account has a decade of work there, don't mix 1 project 1 with 1 decade of projects*

*HR: Jose, This is formal request to remove the emulator as software contravenes copyright and Trade Marked Material. Thank You*

*me: Dear **employee's name**, It doesn't contravene nothing. Thank you*




## Kicked from the Discord server

A few days later, on the 10th of March, I woke up to a message on the official Discord server from Alex:


*If you want to work on stemplayer and have engineering experience, also feel free to DM me directly*

Some users reacted to his message with letters spelling "EMULATOR", which I didn't contribute. Amused by the situation, I [tweeted it](https://twitter.com/krystalgamer_/status/1502014167112470528). A day later, I got the boot of the server "per quest of Kano", the other users were allowed to stay, though.


![EMULATOR](images/stemplayer/alex_emulator.jpg)


## Stem Player missing feature

From talking to Alex and the engineer, I learned that one key feature was left out due to lack of time. To contextualize, when working on a track, ye asks multiple artists to record verses for it and for multiple producers to work on it, so he ends up with hundreds of versions of the same song. To capitalize on this, he thought about offering multiple versions of the stems, e.g with a different bass line, different hook,... Kano never got around to implementing it and hasn't mentioned anything about it since the release of the album.


# Conclusion

The Stem Player is a terrible unfinished product that barely fulfills its alleged role. The product was rushed and its high price point makes it more apparent. It was also clear it was never intended to be a device to handle exclusive releases. Engineering wise, it's a total wreck. Even though it's praised for its novelty, that aura lasts for a very brief moment.

Alex Klein isn't prepared to handle a company. He has no direction and solely relies on hype to move forward. When the hype train stops, he resolves to silence and cryptic posts. Kano's incompetence is not the employees' fault, but Alex's lack of vision and over-reliance on ye. I genuinely believe the Stem Player will be out of everyone's mind in a few months and will be remembered as that weird gadget ye shoved the fans' throat to access the album.


To end on a high note, I hope this post highlights the reality of being in the tech industry. Connections speak louder than skills, as demonstrated by Alex by being friends with one of the most famous artists of our age. He could throw himself into an industry he knew nothing about, make no plans and still get millions in revenue. The trick is to keep your charade running long enough to get the maximum gain.


To ye, you've been down this road with TLoP and Tidal. When Donda 2 eventually comes to streaming, you'll be sued again for tricking people into a "exclusive" platform and [end up settling](https://www.theverge.com/2022/2/18/22940748/donda-2-stem-player-kanye-west-exclusive-music). Also, you're a great artist and terrible at envisioning the future of technology.




# Source code

Feel free to check out the project [here](https://github.com/krystalgamer/stem-player-emulator).

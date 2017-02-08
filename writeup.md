SANS Holiday Hack Challenge 2016
==============================

## Part 1

> 1) What is the secret message in Santa's tweets?
> 2) What is inside the ZIP file distributed by Santa's team?

First we find Santa's business card, with his Twitter and Instagram handles.

![Business Card](/img/business_card.png)

### Tweets

His Twitter had a bunch of tweets that looked to be some sort of crypto composed of words like SANTA, HOHOHO, CHRISTMAS, ELF, etc. I held down the end button to scroll through all the tweets, copy-pasted them into a text editor, and removed metadata lines. But before I could do further analysis, it became apparent that the answer was in plain sight, written in ASCII art with the tweets.

![](/img/tweets.png)

Answer:
`BUG BOUNTY`

### Instagram and ZIP File

The Instagram pictures weren't too special, but one of them had a laptop and SANS workshop materials at the side. Time for OS recon!

![Instagram Image](/img/instagram.jpg)

The laptop had the NORAD Santa tracker pulled up, and but the edge of screen had a cut off window with the visible text: `Path ./ -DestinationPath SantaGram_v4.2.zip`., and from what some of the NPCs in game said, this is evidently the North Pole social media platform. The `.\` and `DestinationPath` were suspicious and sounded relevant to web servers. There's a lot else in the picture, including an nmap report for `www.northpolewonderland.com`. When poking around the game, some NPC gave a download URL on the same host, so it immediately stood out as an official Holiday Hack domain. Putting the two together, I guessed the zip file was right in the root directory and tried `northpolewonderland.com/SantaGram_v4.2.zip`, which got a password protected zip file containing the SantaGram APK.

Answer:
`SantaGram_4.2.apk`

## Part 2

> 3) What username and password are embedded in the APK file?
> 4) What is the name of the audible component (audio file) in the SantaGram APK file?

From NPC hints, I probably had to use John the Ripper and with a password list "rock you" to crack the zip. Community version 1.7.9 quickly found the password, except it had control characters that interrupted the terminal: `*7¡Vamos!`. Several workarounds were attempted such as `unzip -P "somepassword" SantaGram_4.2.apk` but didn't work.

Digging further and finding a [reddit thread](https://www.reddit.com/r/netsec/comments/5hmlkj/the_2016_sans_holiday_hack_challenge/), I discovered that JtR wasn't the intended solution. Some comments hinted about previous levels, and trying with names and phrases relevant to previous questions, I found `bug bounty` worked and extracted the compressed APK.

![SantaGram Icon](/img/santagram_icon.png)

### Audio File

As APKs are simply zip files, I extracted SantaGram and used `tree` to get an idea of the contents and directory structure. The audio file, being in the last folder `res/raw/`, was immediately visible. It sounded very distorted and almost horrific, by the way.

![](/img/tree.png)

Answer:
`discombobulatedaudio1.mp3`

### Embedded Credentials

While there's many options for decompilers, I took the hints from the elves. Apktool might come in handy later, but I thought smali would be harder to parse than java, so I got jadx up and running. I skimmed through the java code, and while there was a `Login` class, I didn't find much. Next option was the search function, so I queried for "password", and checked "code" under what definitions to search for.

The OS interface and colors were buggy but you can see the highlighted line shows the relevant search result.

![jadx search results](/img/jadxsearch.png)

A line came up with potential password, I navigated to the file, and found the username with it. Seems to be used for authenticating web requests containing analytics data, though I thought elves would be developing apps and not the reindeer? (hahaha...)

The relevant snipped of decompiled code is below.

```java
private void postDeviceAnalyticsData() {
    final JSONObject jSONObject = new JSONObject();
    try {
        jSONObject.put("username", "guest");
        jSONObject.put("password", "busyreindeer78");
        jSONObject.put("type", "launch");
        // ...
        new Thread(new Runnable(this) {
            final /* synthetic */ SplashScreen f2615b;

            public void run() {
                C0987b.m4776a(this.f2615b.getString(R.string.analytics_launch_url), jSONObject);
            }
        }).start();
    } catch (JSONException e) {
        Log.e(TAG, "Error in postDeviceAnalyticsData: " + e.getMessage());
    }
}
```

Answer:
user: `guest`, pass: `busyreindeer78`

## Part 3

> 5) What is the password for the "cranpi" account on the Cranberry Pi system?
> 6) How did you open each terminal door and where had the villain imprisoned Santa?

Unfortunately my account was somehow deleted and I had to re-register and make up all my progress. The cranpi pieces are scattered throughout the area, and not behind doors I'm yet to unlock. Speaking to the elf Holly, I was rewarded with the cranbian image file that goes on the device.

### Cranpi Password

I went through a somewhat convoluted process before gaining access to the img contents. The default system's disk mount utility had read errors, and I eventually converted the img to a vmdk [(reference)](http://stackoverflow.com/questions/454899/how-to-convert-flat-raw-disk-image-to-vmdk-for-virtualbox-or-vmplayer) and mounted it with VMware. Home directory didn't have anything interesting (other than cranberry recipes), so as per one of the elf's hints, I tried using John the Ripper to crack the shadow file with the rockyou wordlist. A few minutes later I had it.

```bash
$ ./john --wordlist=../../rockyou.txt ../cranpi_shadow 
Warning: detected hash type "sha512crypt", but the string is also recognized as "crypt"
Use the "--format=crypt" option to force loading these as that type instead
Loaded 1 password hash (sha512crypt [64/64])
guesses: 0  time: 0:00:00:13 0.15% (ETA: Mon Jan  2 04:15:46 2017)  c/s: 2105  trying: gandhi - PURPLE1
guesses: 0  time: 0:00:00:17 0.21% (ETA: Mon Jan  2 04:06:16 2017)  c/s: 2113  trying: 102100 - tinker14
guesses: 0  time: 0:00:00:50 0.57% (ETA: Mon Jan  2 04:17:31 2017)  c/s: 1984  trying: 05mustang - 020180
guesses: 0  time: 0:00:03:51 2.40% (ETA: Mon Jan  2 04:31:45 2017)  c/s: 1739  trying: mice123 - mexico1992
yummycookies     (cranpi)
guesses: 1  time: 0:00:04:21 DONE (Mon Jan  2 01:55:42 2017)  c/s: 1740  trying: yves69 - yoyojojo
Use the "--show" option to display all of the cracked passwords reliably
```
Answer:
`yummycookies`

### Terminal Doors and Santa

#### Elf House #2

This was the house near the starting point, with a small kitchen and second floor.

The pcap file contains the door's key in two parts, but is only readable by user `itchy`, and I'm logged in as user `scratchy`. Files on the system aren't interesting, but `sudo -l` prints out the commands the I can use.

```bash
scratchy@1c9fab44b702:/$ sudo -l
sudo: unable to resolve host 1c9fab44b702
# Matching Defaults entries for scratchy on 1c9fab44b702:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
User scratchy may run the following commands on 1c9fab44b702:
    (itchy) NOPASSWD: /usr/sbin/tcpdump
    (itchy) NOPASSWD: /usr/bin/strings
scratchy@1c9fab44b702:/$ 
```

Notably, I can run tcpdump and strings. Using strings just floods me with results. I instead use tcpdump's `-r` flag to read the pcap. Near the start is the first part of the key, with a web request containing html and the the string `santasli`.

```html
<head></head>
<body>
<form>
<input type="hidden" name="part1" value="santasli" />
</form>
</body>
</html>
```

Second part was a binary file, and tcpdump showed a bunch of garbled data. Checking through the `man` pages didn't anything that couldn't helped. Resorting to strings, there was a flag that was interesting, `--encoding`. Although tcpdump's output mentioned the content was an octet stream, the 8 bits encoding option returned garbage data. So I went down the list and `l` nicely printed out part 2.

```bash
scratchy@1c9fab44b702:/$ sudo -u itchy strings --encoding=b /out.pcap 
sudo: unable to resolve host 1c9fab44b702
scratchy@1c9fab44b702:/$ sudo -u itchy strings --encoding=l /out.pcap 
sudo: unable to resolve host 1c9fab44b702
part2:ttlehelper
```

Putting the two together, I got the correct key. The door brought me to "Room 2" in the elf house.

Key: `santaslittlehelper`

#### Workshop 1

This terminal requires a successful playthrough of the classic game wumpus. Not much to it: a draft means I'm near a bottomless pit, a stench means I'm near a monster called the wumpus. Both kill me. I simply shot arrows at each room when a wumpus was nearby and killed it, revealing the passphrase.

The door leads to a room called "DFER" with reindeer and a netwars coin, but there's nothing notable. And you can't even pet the reindeer.

Key: `WUMPUS IS MISUNDERSTOOD`

#### Workshop 2

The **workshop** is the structure on the cliff, above the clouds and treetops.

The key is in a deeply nested directory. Keyboard commands are limited (i.e. arrow keys and tab autocompletion), and the directories have confusing names that either disappear or require a lot of escaping. Fortunately, `find` reveals all files in the current directory and spoiled a lot of the surprise. The file `./.doormat/. / /\/\\/Don't Look Here!/You are persistent, aren't you?/'/key_for_the_door.txt` stands out.

![](/img/find.png)

Slowly navigating eventually led to the file, and `cat key_for_the_door.txt` revealed the key.

Key: `open_sesame`

#### Santa's Office

Going through the door leads to **Santa's Office**. The terminal on the wall doesn't show a regular shell, instead prompting `GREETINGS PROFESSOR FALKEN`.

![Wargames reference, anyone?](/img/greetings.png)

I pulled up the scene from Wargames via YouTube and entered in the responses the main character gave. Exact responses only, or I had to enter them again.

At this point I was considering if the game was about to explode

![Mutually Assured Destruction](/img/world_destruction.png)

Eventually I got the key and used it to access the secret bookshelf doorway. It led to a corridor and locked door, but no hints on how to get through.

![](/img/falken_key.png)

Key: `LOOK AT THE PRETTY LIGHTS`

#### Train

The last terminal is at the train station. It's some sort of shell that only accepts limited commands. I can start the train, enable/disable brakes, check status, show help text, or exit. Starting requires a password, though. But evidently from the help page, the help text is shown using less.

```
**HELP** brings you to this file.  If it's not here, this console cannot do it, unLESS
 you know something I don't.
```

From the man pages for less, `!` can be used to run shell commands, and `!ls -al` revealed an executable `ActivateTrain`. `!./ActivateTrain` starts the train, and I end up at the starting point, in a 1978 north pole.

Looking around, there's not much to note. NPC messages are different, but provide nothing notable in terms of clues. There's also netwars coin here and there. Most importantly, the DFER (Detention For Errant Reindeer) room now contains Santa, who thanks you for rescuing him.

Answer:
`Santa was imprisoned in the DFER room.`

## Part 4

> 7) ONCE YOU GET APPROVAL OF GIVEN IN-SCOPE TARGET IP ADDRESSES FROM TOM HESSMAN AT THE NORTH POLE, ATTEMPT TO REMOTELY EXPLOIT EACH OF THE FOLLOWING TARGETS:
> 
> The Mobile Analytics Server (via credentialed login access)
> The Dungeon Game
> The Debug Server
> The Banner Ad Server
> The Uncaught Exception Handler Server
> The Mobile Analytics Server (post authentication)
>
> For each of those six items, which vulnerabilities did you discover and exploit?
> 
> 8) What are the names of the audio files you discovered from each system above? There are a total of SEVEN audio files (one from the original APK in Question 4, plus one for each of the six items in the bullet list above.)

Now returning to the APK, there's systems that can be exploited for more audio files like in Part 2. The problem now was finding the URLs for each of the servers. First place I thought of was the embedded credentials in Part 2, as it sent the data to a server. However, jadx only showed the url as `R.string.somethingURL`. It (and other URLs) was actually stored in an XML file, see [What is the concept behind R.java?](http://stackoverflow.com/questions/10004906/what-is-the-concept-behind-r-java). Further searching showed that apktool can dump `strings.xml`, so I installed it and ran `apktools d SantaGram_4.2.apk`. Running a recursive grep showed the exception handler url was in `res/values/strings.xml`, which contained several other URLs.

```xml
<string name="analytics_launch_url">https://analytics.northpolewonderland.com/report.php?type=launch</string>
<string name="analytics_usage_url">https://analytics.northpolewonderland.com/report.php?type=usage</string>
<string name="banner_ad_url">http://ads.northpolewonderland.com/affiliate/C9E380C8-2244-41E3-93A3-D6C6700156A5</string>
<string name="debug_data_collection_url">http://dev.northpolewonderland.com/index.php</string>
<string name="dungeon_url">http://dungeon.northpolewonderland.com/</string>
<string name="exhandler_url">http://ex.northpolewonderland.com/exception.php</string>
```

I ran dig on each domain. I added `northpolewonderland.com` for reference.

```bash
$ dig +noall +answer {northpolewonderland.com,analytics.northpolewonderland.com,analytics.northpolewonderland.com,ads.northpolewonderland.com,dev.northpolewonderland.com,dungeon.northpolewonderland.com,ex.northpolewonderland.com}
northpolewonderland.com. 1800   IN  A   130.211.124.143
analytics.northpolewonderland.com. 1693 IN A    104.198.252.157
analytics.northpolewonderland.com. 1693 IN A    104.198.252.157
ads.northpolewonderland.com. 1693 IN    A   104.198.221.240
dev.northpolewonderland.com. 1693 IN    A   35.184.63.245
dungeon.northpolewonderland.com. 1693 IN A  35.184.47.139
ex.northpolewonderland.com. 1693 IN A   104.154.196.33
```

Each of the IPs were confirmed to be in scope, except the main `northpolewonderland.com` domain.

### Dungeon Game

The dungeon URL shows only an instruction sheet, but a simple nmap reveals a few ports open.

```
$ nmap dungeon.northpolewonderland.com

Starting Nmap 7.01 ( https://nmap.org ) at 2017-01-09 17:58 PST
Nmap scan report for dungeon.northpolewonderland.com (35.184.47.139)
Host is up (0.076s latency).
rDNS record for 35.184.47.139: 139.47.184.35.bc.googleusercontent.com
Not shown: 997 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
11111/tcp open  vce

Nmap done: 1 IP address (1 host up) scanned in 9.16 seconds
```

Port 11111 is in fact the dungeon game, connecting via netcat launches the game.

I forgot where, but somewhere in the Holiday Hack game is revealed that secret rooms have been added to the dungeon game, with an elf in the room. This was probably what I was supposed to look for. Some Google searching later revealed a tool called GDT to debug the dungeon game. This guide helps: [Zork hints](http://gunkies.org/wiki/Zork_hints). Trying several commands reveals some of interest:

> AH- Alter HERE
> DL- Display lengths
> DT- Display text

`DL` to display the "lengths" of the game, meaning numbers like how many rooms there are, in this case 192. `AH` teleports you between rooms, and the last room is the secret one.

```
DT>ah
Old=      2      New= 192
GDT>exit
>l
You have mysteriously reached the North Pole. 
In the distance you detect the busy sounds of Santa's elves in full 
production. 

You are in a warm room, lit by both the fireplace but also the glow of 
centuries old trophies.
On the wall is a sign: 
        Songs of the seasons are in many parts 
        To solve a puzzle is in our hearts
        Ask not what what the answer be,
        Without a trinket to satisfy me.
The elf is facing you keeping his back warmed by the fire.
```

But I couldn't talk to the elf for some reason, so I used `DT` to extract the conversation. Again `DL` revealed 1027 messages in the game, so walking back from 1027, I got the following:

```
GDT>dt
Entry:    1024
The elf, satisified with the trade says - 
send email to "peppermint@northpolewonderland.com" for that which you seek.
```

Sending an email to that address gets a reply with the audio attached.

### Debug Server

The website for this was blank, and kept returning code 200 with no contents even when using Burp. So I changed the request to POST, Content-Type to application/json, and tried to emulate the request in the APK code. Turns out it's also very picky about the contents, primarily with the debug value. I eventually got the correct string by looked through the APK code and searching `getCanonicalName()` and `getSimpleName()`.

I ended up with the following request:

```json
{
    "date": "201701041000",
    "udid": "device",
    "debug": "com.northpolewonderland.santagram.EditProfile, EditProfile",
    "freemem": 500
}
```

... and finally got back a response:

```json
{"date":"20170105012837","status":"OK","filename":"debug-20170105012837-0.txt","request":{"date":"201701041000","udid":"device","debug":"com.northpolewonderland.santagram.EditProfile, EditProfile","freemem":500,"verbose":false}}
```

Didn't appear too special, but wait, there's a `"verbose":false`. I sent the original request again, but with `"verbose": true`. Burp hangs for a while, but returns all the debug reports. Searching for audio returns nothing, but mp3 shows the audio file name. Finally I used wget on the url and downloaded the audio file.

![](/img/debug.png)

### Ad Server

Navigating to the site and pulling up Burp, I got a bunch of binary data. It was similar to what I read about Meteor from what one of the NPCs said. I installed the tools they talked about on the blog post: [Mining Meteor](https://pen-testing.sans.org/blog/2016/12/06/mining-meteor). It revealed the routing rules, and when I went to the hidden quotes page, there were 5 quotes instead of the 4 on the main page. Retrieving it with `HomeQuotes.find().fetch()` revealed the "just ad it" entry.

![](/img/admeteor_quotes.png)


![](/img/admeteor_audio.png)

It contained the audio's url, and I downloaded it with wget

### Exception Handler

Navigating to the url in a browser showed the initial response: `Request method must be POST`, and the code showed a JSON object being created with some device data. Time to fire up Burp Proxy!

Initially, from the error messages and trying out various inputs, I figured out the following:

- Request type has to be POST, and `Content-Type` set to `application/json` (server receiving JSON data)
- `operation` can either be `WriteCrashDump` or `ReadCrashDump`


I ended up with the following request for writing a crash dump.

```
POST /exception.php HTTP/1.1
Host: ex.northpolewonderland.com
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:48.0) Gecko/20100101 Firefox/48.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/json
Content-Length: 603

{
    "operation": "WriteCrashDump",
    "data": {
        "message": "content",
        "lmessage": "content",
        "strace": "content",
        "model": "content",
        "sdkint": "content",
        "device": "content",
        "product": "content",
        "lversion": "content",
        "vmheapsz": "content",
        "vmallocmem": "content",
        "vmheapszlimit": "content",
        "natallocmem": "content",
        "cpuusage": "content",
        "totalstor": "content",
        "freestor": "content",
        "busystor": "content",
        "udid": "content"
    }
}
```

The server replies the write is succesful to a file "crashdump-ujJ8id.php"

```json
{
    "success" : true,
    "folder" : "docs",
    "crashdump" : "crashdump-ujJ8id.php"
}
```

After playing around with the JSON structure and contents, I figured out the following to read crashdumps

```json
{
    "operation": "ReadCrashDump",
    "data": {
        "crashdump": "crashdump-ujJ8id"
    }
}
```

But where's the discombobulated audio file? Attempting to read `exception.php` returns a 500 error, but an NPC in game hints at the following blog post: [Getting MOAR Value out of PHP Local File Include Vulnerabilities](https://pen-testing.sans.org/blog/2016/12/07/getting-moar-value-out-of-php-local-file-include-vulnerabilities).

Using their example, I tested it out by reading crash dumps.

![](/img/exhandler_readcrash.png)

I guess the location of `exception.php`, and it successfully returns the source!

![](/img/exhandler_exception.png)

I pipe it to `base64 -d` which decodes it to the original file, with the URL for the audio file at the top.

![](/img/exhandler_decode.png)


### Analytics

#### Part 1

Initially I logged in with the credentials back in Part 2. The audio file was right in the nav bar at the top, and it downloaded without a problem. I wasn't aware of any vulnerability other than that, so I guess that's all I needed to do.

#### Part 2

The second portion was a bit frustrating. I ran nmap on it with the `-sC` flag hinted by an NPC but revealed nothing. The POST requests like with debug and exception server didn't do anything notable. Eventually later I ran it again and it showed a git repository! I downloaded it with recursive wget, but a lot of the website files were missing. `git checkout --force HEAD` resorted all the files.

There were a lot of interesting files but nothing notable. There was some protection against SQLi, the sql file had the relevant data stripped out, but checking the git history did reveal some interesting commits.

```
$ git log-flow
* 16ae0cb - (5 weeks ago) Finishing touches (style, css, etc) - me (HEAD -> master)
* 106079e - (7 weeks ago) Got rid of mysqli_fetch_all(), which isn't widely supported - me
* e46b41e - (7 weeks ago) HTML escape more output values on the test page - me
* 935d797 - (7 weeks ago) HTML escape an output value on the test page - me
* 6254786 - (7 weeks ago) Fix database dump - me
* 85a4207 - (7 weeks ago) Saved queries now save the query object instead of the results - me
* 45edadc - (7 weeks ago) Update README.md to reflect the actual current state - me
* 885ec6a - (7 weeks ago) Update report.php to log actual data to the database instead of static strings - me
* 58c900f - (7 weeks ago) Add view.php - me
* 4397009 - (7 weeks ago) Remove unnecessary data from the database dump - me
* 1908b71 - (7 weeks ago) Update the database dump - me
* 0778ac7 - (7 weeks ago) Reports can now be saved - me
* 1562064 - (7 weeks ago) Add a fairly complex query page for looking up records - me
* 259d406 - (7 weeks ago) Add a header, a footer, and a logout page - me
* 2689a45 - (8 weeks ago) Fix the database dump - me
* cf5f27b - (8 weeks ago) Change the database and application/test script to use the real field names instead of fake names - me
* 6ab9fe6 - (8 weeks ago) Add login to the HTML side of things - me
* 02e8d14 - (8 weeks ago) Add a HTML login page, and refactor a little to make check_user() usable by both JSON and HTML - me
* f0d28ed - (8 weeks ago) Move some functions into this_is_json.php - me
* d9636a3 - (8 weeks ago) Small authentication fix - me
* 5f0c135 - (8 weeks ago) Add authentication - me
* 420f433 - (8 weeks ago) Add some basic write-to-the-database functionality - me
* bb26466 - (8 weeks ago) Add a bit of database functionality - me
* 1057b70 - (8 weeks ago) Add a script to test the API - me
* d63a7e0 - (8 weeks ago) Added the start of a reporting page - me
$ git show 4397009
...
-INSERT INTO `users` VALUES (0,'administrator','KeepWatchingTheSkies'),(1,'guest','busyllama67');
...
```

One of the commits removed admin credentials from the DB, but since it was committed previously in git and I had the repository, it can be retrieved from the history. Trying it on the website, it allowed me in! It was essentially the same except the audio on the nav bar was "Edit" instead. It apparently edited saved queries, but notably it looks for a query. checking the parameters in the URL, there's id, name, and description as show on the web page.

![](/img/analytics_edit.png)

So I manually add a simple sql query `https://analytics.northpolewonderland.com/edit.php?id=f89b455f-1401-4967-9504-37994553049d&name=stuff&description=stuff&query=select * from audio` and checked the under "view".

![](/img/analytics_results.png)

So I can access DB, and from the schema, the file is also right there. Going off of the previous parts, maybe I can extract the file as base64. Also note that an incorrect UUID reveals the DB as MariaDB, so a quick search reveals [TO_BASE64](https://mariadb.com/kb/en/mariadb/to_base64/).

I end up with the query:

`https://analytics.northpolewonderland.com/edit.php?id=f89b455f-1401-4967-9504-37994553049d&name=stuff&description=stuff&query=SELECT *, TO_BASE64(mp3) FROM audio`

![](/img/analytics_audio.png)

It's pretty big, so I copied it into a file, removed the spaces, and it decoded successfully

## Part 5

> 9) Who is the villain behind the nefarious plot.
> 10) Why had the villain abducted Santa?

Now with all the audio files, it's time to put them together. I first just concatenated them in order with audacity, and sped up the tempo. Note that speed also increases pitch, which makes the audio incomprehensible. I eventually arrived at something like "Oh Christmas Santa Clause, or as I've known him, Jeff". Fortunately it happens that the quote is findable via Google.

![](/img/quote.png)

The quote successfully opens the last door in the secret hallway behind Santa's Office, revealing Dr. Who as the perpetrator, as the quote's origin indicated.

The game ends, and Christmas is saved! Thank you SANS for putting together this challenge. I should finally get some sleep.

Answer:
`Dr. Who`

Answer:
`He apparently didn't like the Christmas special episode of Star Wars or something, and the North Pole's train/time machine could prevent it.`

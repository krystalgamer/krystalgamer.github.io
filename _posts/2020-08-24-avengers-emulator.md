---
layout: post
title: "Reverse Engineering: Marvel's Avengers - Developing a Server Emulator"
description: "Documentation of reversal and building of a Marvel's Avenger server emulator"
created: 2020-08-24
modified: 2020-08-24
comments: false
tags: [marvel, avengers, protection, protection, spiderman 2000, reverse engineering, ida pro, curl, osdk, exploiting, buffer overflow, x64, tls, wireshark, fiddler, packet dumper, packet, denuvo, wireshark, shakes and fidget, fiddler]
---

# Context
{: .center}

During these past two weeks I had the chance to play the Marvel's Avengers Beta. The game allows the player to player solo or hop into matchmaking to find some squad-mates.
Even playing solo it was clear that the game needed internet connectivity. Having already played last week during the closed-beta I decided to use the new open-beta to explore more about the game's networking.


# The process

One of the reasons I like reversing and understanding client/server communications is that the process is much more clear than other types of reversing, such as custom archives and formats.
The process goes like this:

1. Understand the type, the sender and destination of the traffic - who is responsible of sending it, whether it's UDP/TCP, the endpoints, is it encrypted? ...
2. Acquire readibility and instrumentation - develop/use tools to dump the traffic to later analyze it
3. Learn the protocol details - crucial for debugging, this can be done by omitting requests/responses, messing with the contents,...


If the goal is to develop an emulator then there's extra steps:

4. Start by parroting the server responses
5. Slowly build the backend and start making your own responses


## Understanding the traffic
{: .center}

I started by opening `Wireshark` and checking how my actions impact the traffic. During the game's boot there's nothing happening, pressing the `Start` button in the main menu the game performs a DNS query of `cry-trmv6-beta.os.eidos.com` followed by setting up a TLS connection to the return IP. This indicates that the protocol being HTTPS - it's nice since it well known but it's encrypted which hurts readibility and tamperability.


Having some experience with games that use HTTPS, such as [Shakes and Fidget](https://github.com/krystalgamer/shakes_fidget_server_emulator), it's common for the client not to care about the server being HTTP(useful for dumping packets) or even the server allowing HTTP requests. I added an entry to the `HOSTS` file to redirect the traffic to my local server sadly the connection couldn't be established and Square Enix servers only allowed HTTPS. This meant I had to dug deeper to acquire reading power of the packets.


This part got me scared since the game uses Denuvo which contributes to its 400MB of executable size. I wanted to avoid messing its anti-tamper measures.
To verify where the traffic comes from within the game using `x64dbg` I placed a breakpoint at `getaddrinfo` in `ws2_32.dll`. When the breakpoint was hit reading the callstack showed where it came from, it was `osdk.dll`. This module was shipped with the game and until that moment I was not aware of its existence.

The next step was to see what it exports:

```shell


$ dumpbin -exports osdk.dll


    ordinal hint RVA      name

         25   18 000DB510 osdk_HTTPClient_SetDefaultLoggingFlags
         26   19 000DB520 osdk_HTTPClient_addBackFilter
         29   1C 000DB5F0 osdk_HTTPClient_enableBodyCompression
         30   1D 000DB600 osdk_HTTPClient_enableWebLogging
         33   20 000DB7A0 osdk_HTTPClient_setLoggingFlags
         34   21 000DBBF0 osdk_HTTPFilter_Create
         35   22 000DBF60 osdk_HTTPHeaderNames_Get
         36   23 000DBF70 osdk_HTTPHeaders_getKeyAt
         48   2F 000DC8A0 osdk_HTTPRequestBuilder_setConnectTimeout
         49   30 000DC8B0 osdk_HTTPRequestBuilder_setHeader
         50   31 000DC8C0 osdk_HTTPRequestBuilder_setParameter
         51   32 000DC8D0 osdk_HTTPRequestBuilder_setSerializationMilliseconds
         52   33 000A9B00 osdk_HTTPRequestBuilder_structSize
         53   34 000DC0B0 osdk_HTTPRequest_setHeader
         54   35 000DCA70 osdk_HTTPResponse_Create
         63   3E 000DCE10 osdk_HTTPResponse_getReceivedAt
         64   3F 000DCE40 osdk_HTTPResponse_getStatusCode
         65   40 000DCE50 osdk_HTTPResponse_setStatus
         66   41 000DCEE0 osdk_HTTPUserAgentBuilder_build
         67   42 000DCF50 osdk_HTTPUserAgentBuilder_construct
         68   43 000DCF60 osdk_HTTPUserAgentBuilder_destruct
         69   44 000DCF70 osdk_HTTPUserAgentBuilder_setApplicationDetail
         75   4A 000DD000 osdk_HTTPUserAgentBuilder_setFrameworkVersion
         76   4B 000B5E20 osdk_IdentityProviders_Get
         83   52 000DD280 osdk_Identity_assignMove
         84   53 000DD290 osdk_Identity_construct
         85   54 000DD2B0 osdk_Identity_constructMove
        107   6A 000DD7A0 osdk_JSONArray_begin
        108   6B 000DD7C0 osdk_JSONArray_end
        109   6C 000DD7F0 osdk_JSONArray_get
        110   6D 000DD820 osdk_JSONArray_isEmpty
        111   6E 000DD840 osdk_JSONArray_pushBack
        112   6F 000DD860 osdk_JSONArray_pushBackBool
        147   92 000E5A20 osdk_JSONValue_asFloat
        148   93 000E5A30 osdk_JSONValue_asInt16
        149   94 000E5A30 osdk_JSONValue_asInt32
        150   95 000E5A40 osdk_JSONValue_asInt64
        208   CF 000C0F90 osdk_ManualSignal_cleanup
        209   D0 000C0FC0 osdk_ManualSignal_initialize
        217   D8 000E6220 osdk_MediaTypeNames_Get
        230   E5 000C0A60 osdk_ObjectId_Parse
        231   E6 000C0A70 osdk_ObjectId_structSize
        232   E7 000C0A80 osdk_ObjectId_toString
        233   E8 000A9B00 osdk_OsdkInfo_structSize
        234   E9 000C0AF0 osdk_ProfileId_structSize
        305  130 000BFF50 osdk_UInt32ToAscii
        306  131 000BFF70 osdk_UInt32ToAsciiLength
        307  132 000BFFB0 osdk_UInt64ToAscii
        308  133 000BFFD0 osdk_UInt64ToAsciiLength
        313  138 000C2B40 osdk_UserAccountType_Get
        314  139 000A57C0 osdk_UserId_structSize
        315  13A 000A5840 osdk_UserInfo_constructCopy
        316  13B 000A5850 osdk_UserInfo_move
        317  13C 000A58B0 osdk_UserInfo_structSize
        329  148 000C2E00 osdk_Variant_ConstructUInt16
        336  14F 000C3210 osdk_VersionValues_Get
        337  150 000C3220 osdk_Version_Parse
        338  151 000C32E0 osdk_Version_fromParts
        339  152 000C3320 osdk_Version_toString
        340  153 000A71A0 osdk_WebServiceClientAttributeKeys_Get
        341  154 000A97A0 osdk_WebServiceClientBuilder_Create
        342  155 000A97C0 osdk_WebServiceClientBuilder_build
        360  167 000A6A10 osdk_WebServiceClient_addStatisticsDeserializationMilliseconds
        387  182 000A3C90 osdk_addProcessCallback
        388  183 000A3CA0 osdk_beginNewGameSession



```

Some of the entries were removed due to being too long. This was my breakthrough, this library is responsible for the communications and it appears to be done in JSON! 

Digging around the **strings** I noticed it was using `libcurl` which is also awesome, but also `libuv` which could pose serious problems for debugging and tracing. IDA wasn't picking cURL's function names so I had to use one of the community FLIRT databases, [FLIRTDB](<https://github.com/Maktm/FLIRTDB), which was key in finding functions such as `curl_easy_setopt`.
cURL has two interfaces the `easy` and the `multi`, the latter being focused in asynchronous programming. Even though it uses `libuv` all the connections are done via the **easy** interface.



Having the targets isolated it was time to add some logging. In this case I resorted to dll proxying which consists of substituting the original module with a one created by me and redirecting all the calls to the original one. Microsoft Visual C compiler makes it super easy with a simple pragma directive.

```c
 __pragma (comment (linker, "/export:" export_name "=" original_dll_name.export_name ",@" ordinal))
```

In this case I renamed `osdk.dll` to `osdk_orig.dll`, the new module exported the exact same functions which just forwarded to the original.

```c
#define FORWARDED_EXPORT_WITH_ORDINAL(exp_name, ordinal, target_name) __pragma (comment (linker, "/export:" #exp_name "=" #target_name ",@" #ordinal))

FORWARDED_EXPORT_WITH_ORDINAL(osdk_Allocator_Free, 1, osdk_orig.osdk_Allocator_Free)
FORWARDED_EXPORT_WITH_ORDINAL(osdk_Allocator_Init, 2, osdk_orig.osdk_Allocator_Init)

(...)
```


DLL Proxying has also another big advantage, since `DllMain` runs before the main executable(the ones of the modules that are explicitly imported!), it's the safest place to perform any hooks/patches. Other methods such as DLL Injection require creating an extra thread which might cause some racing issues.


Here I run into another problem, for some reason `GetModuleHandle` was always returning 0, except when the argument was `NULL`. Calling `ntdll!LdrGetDllHandle` also yielded 0 and even walking the LDR table from PEB(Process Environment Block), yield no modules which was extremely suspicious.

I was eager to to get my hooks running and debugging Windows internal structures was not on my best interest, because it might be caused by Denuvo! Being desperate I decided to try something that even [Microsoft discourages](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-best-practices) from doing, calling `LoadLibrary` in `Dllmain`, the reason behind it is that it during the `DllMain` routine the loader lock is acquired and is not free'd until it's over(also the loaded modules list must not change inside), thus loading new modules might cause crashes or deadlock.

Since my dll depends on `osdk_orig.dll` then it must be loaded after the original one, thus my call of `LoadLibrary` does not violate the condition of modifying the module list! Luckily it worked and `GetProcAddress` was also working. Using `osdk_Allocator_Init` as a pivot, the **offsets** to all the relevant functions inside `osdk_orig.dll` were calculated and relevant functions hooked. It was time to start logging!



### Logging curl_easy_setopt

Performing requests with cURL is quite straightforward, first create a `CURL*` handle with `curl_easy_init` and then set **ALL** connection related settings/parameters `curl_easy_setopt`. The function prototype is as follows:

```c
CURLcode curl_easy_setopt(CURL *handle, CURLoption option, parameter);
```

The first argument was already discussed, the second one is an *enum* which specifies which option will be set and the third argument is the data related to the parameter. E.g with option `CURLOPT_URL` the parameter would be a null-terminated string of the url.

Hooking this function was done with a **trampoline**, which consists of over-writing the function prologue with something like this:

```nasm
mov rax, ADDRESS_OF_MY_DETOUR
push rax
ret
```

One cool thing I learned about cURL's internals is that it uses a self-defined macro to enforce 3 arguments but the function still works with `va_args`?

```c

============ from curl.h ======================
/* This preprocessor magic that replaces a call with the exact same call is
   only done to make sure application authors pass exactly three arguments
   to these functions. */
#define curl_easy_setopt(handle,opt,param) curl_easy_setopt(handle,opt,param)

============ from setopt.c ======================
#undef curl_easy_setopt
CURLcode curl_easy_setopt(struct Curl_easy *data, CURLoption tag, ...)
{
  va_list arg;
  CURLcode result;

  if(!data)
    return CURLE_BAD_FUNCTION_ARGUMENT;

  va_start(arg, tag);

  result = Curl_vsetopt(data, tag, arg);

  va_end(arg);
  return result;
}
```


In order to dump the options I redefined the **CURLoption** `enum` by re-using their `CURLOPT` macro:

```c
#define CURLOPT(na,t,nu) na = t + nu

//CURLOPTTYPE_STRINGPOINT is a define of a constant

typedef enum {

  /* The full URL to get/put */
  CURLOPT(CURLOPT_URL, CURLOPTTYPE_STRINGPOINT, 2)
}
```


This is where the beauty of C comes, for logging it's much more convenient to have the options in text format and not integers like `enums` are. The *stringification preprocessor* was the key for this problem, the `#` sign in macros allows to turn any variable passed to a string, thus I lazily converted the `CURLOPT` macro to `CURLOPT_CASE` as follows:



```c

#define CURLOPT_CASE(name, ignore, ignore1) \
	case name:\
	logger("setopt: %s (%d)\n", #name, name);\
	break;

switch (tag) {
			CURLOPT_CASE(CURLOPT_WRITEDATA, CURLOPT_CASETYPE_OBJECTPOINT, 1)

			CURLOPT_CASE(CURLOPT_URL, CURLOPT_CASETYPE_STRINGPOINT, 2)
		(...)

}
```



The result:

```text
setopt: CURLOPT_PRIVATE (10103)
setopt: CURLOPT_CONNECTTIMEOUT (78)
setopt: CURLOPT_HTTPGET (80)
setopt: CURLOPT_URL (10002)
setopt: CURLOPT_POSTFIELDSIZE (60)
setopt: CURLOPT_READDATA (10009)
setopt: CURLOPT_READFUNCTION (20012)
setopt: CURLOPT_READFUNCTION (20012)
setopt: CURLOPT_LOW_SPEED_TIME (20)
setopt: CURLOPT_LOW_SPEED_LIMIT (19)
setopt: CURLOPT_ERRORBUFFER (10010)
setopt: CURLOPT_WRITEDATA (10001)
```




### Establishing connection to own server

cURL by default performs host and peer verification with HTTPS connections, this means it must be disabled dynamically. To disable `curl_easy_setopt` needs to be called with `CURLOPT_VERIFYPEER` and `CURLOPT_VERIFYHOST` with the parameter set as 0. My solution was to inject these two calls right after `CURLOPT_URL` is set, this guarantees that it's set for all cURL handles.

```c
if (tag == CURLOPT_URL){
		curl_easy_setopt(data, CURLOPT_SSL_VERIFYHOST, 0);
		curl_easy_setopt(data, CURLOPT_SSL_VERIFYPEER, 0);
}

```

For the server I was using flask which has a really useful option `ssl_context='adhoc'` which allows to generate TLS certificates on the fly, which are marked as `Dummy Certificate`. With the verification disabled and traffic redirection(via DNS) done it was time to start logging endpoints and the incoming traffic. Now I had a way of logging what the client sent, it was time to log what the server sent.

The result:

```text
Client sent:
{
  "userSandbox": "steam",
  "user": {
    "uid": "[redacted]",
    "provider": "steam_id"
  },
  "system": "windows",
  "identity": {
    "type": "steam_token",
    "token": "[redacted]"
  }
}
```

### CURLOPT_READFUNCTION/CURLOPT_WRITEFUNCTION Hook

The nomenclature on these functions is a little confusing at first since `READ` is what the server wants to *read* from the client and `WRITE` is what the server *wrote* to the client. These options setup the callback function when any of these actions occur. The buffers that these functions receive are in a decrypted state, thus logging the packets it's just a matter of `printf`.

Since I already controlled what is passed to `curl_easy_setopt` it was quite easy to replace `osdks_orig` callback with mine:

```c

typedef size_t (*write_callback_ptr)(char* ptr, size_t size, size_t nmemb, void* userdata);
write_callback_ptr write_callback_orig = NULL;
size_t write_callback(char* ptr, size_t size, size_t nmemb, void* userdata) {

	logger("got write(%p) here(%d) %.*s\n", userdata, nmemb, nmemb, ptr);
	return write_callback_orig(ptr, size, nmemb, userdata);
}


typedef size_t (*read_callback_ptr)(char* buffer, size_t size, size_t nitems, void* userdata);
read_callback_ptr read_callback_orig = NULL;
size_t read_callback(char* buffer, size_t size, size_t nitems, void* userdata) {

	size_t ret = read_callback_orig(buffer, size, nitems, userdata);
	logger("got read here %.*s\n", ret, buffer);
	return ret;
}

DWORD64 curl_easy_setopt(struct Curl_easy* data, DWORD64 tag, ...)
{
	print_setopt(tag);
	va_list arg;
	DWORD64 result;

	if (!data)
		return 43;

	va_start(arg, tag);


	(...)
	else if (tag == CURLOPT_WRITEFUNCTION) {

		write_callback_ptr tmp = va_arg(arg, write_callback_ptr);
		
		if (tmp != write_callback) {
			logger("Will spoof write callback %p\n", write_callback);
			write_callback_orig = tmp;
			return curl_easy_setopt(data, tag, write_callback);
		}

		va_end(arg);
		va_start(arg, tag);
	}
	else if (tag == CURLOPT_READFUNCTION) {

		read_callback_ptr tmp = va_arg(arg, read_callback_ptr);

		if (tmp != read_callback) {
			logger("Will spoof read callback %p\n", read_callback);
			read_callback_orig = tmp;
			return curl_easy_setopt(data, tag, read_callback);
		}

		va_end(arg);
		va_start(arg, tag);
	}
```

The result:

```text

got read here {
  "userSandbox": "steam",
  "user": {
    "uid": "[redacted]",
    "provider": "steam_id"
  },
  "system": "windows",
  "identity": {
    "type": "steam_token",
    "token": "[redacted]"
  }
}

got write(0000000170938590) here(873) {
  "baseURI": "https://cry-trmv6-beta-prod.os.eidos.com/api",
  "token": "[redacted]",
  "membership": {
    "id": "[redacted]",
    "email": "[redacted]",
    "confirmed": true
  },
  "services": {
    "version": {
      "max_lcm_version": "1.1.0.0",
      "min_lcm_version": "1.0.0.0",
      "max_game_version": "0.0.1",
      "min_game_version": "0.0.1"
    }
  }
}


```

NOTE: There's also `HEADERFUNCTION` which I also hooked, it's not as relevant as the other two so I did not include it in, the ideia of hooking is the exact same.

### Logging and async problem

For small responses like the ones showed above it was all fine and dandy. There are some responses such as for Market, Wartable and level jsons(yes, each level has a json) which are huge, not only they contain a ton of data but also the text displayed in **ALL** available languages. The biggest mission I have recorded is 47KB and it's *Condition: Green*.

The functions hooked above get called as soon there's data so there were times that they're called with as little as 8 characters in the buffer, rebuilding them by hand would be a nightmare since they're all scattered around the logs.


The solution to this problem was also provided by `libcurl` interface. The write and read callbacks contain a 4th parameter which is defined in the documentation as `userdata`. This parameter works like an accumulator, incoming data comes from `ptr` and should be stored in `userdata`. Due to the fact the game uses different buffers for different requests, it was the best way to get complete packet dumps. The implemented solution was not the most performant(it's pretty bad) but it served it's purpose. It consisted in creating and appending to a text file with the name of the buffer's address - a few minutes in the game exploring all the missions was enough to get all the JSON responses.

```c

size_t write_callback(char* ptr, size_t size, size_t nmemb, void* userdata) {

	logger("got write(%p) here(%d) %.*s\n", userdata, nmemb, nmemb, ptr);
	char tmp[255];
	sprintf(tmp, "[REDACTED]\\%p.txt", userdata);

	FILE* fp = fopen(tmp, "a+");
	fwrite(ptr, nmemb, size, fp);
	fclose(fp);
	return write_callback_orig(ptr, size, nmemb, userdata);
}

```

Logging the read callback turned useless, which will be explained in the next section.

The results:

```text

$ stat -c "%n %s" * | sort

000000016FEB6E20.txt 26516
00000001700DBDB0.txt 20120
0000000170938590.txt 1065
0000000184C69780.txt 151453
00000001854A8060.txt 131
0000000188971080.txt 2
0000000188975920.txt 1139
000000018A8EB810.txt 131
000000018B4780D0.txt 1008
000000018EAE6A50.txt 86249
000000018EC02A00.txt 60
000000018EC1B9D0.txt 19522
000000018EC38310.txt 9120
000000018ECD16D0.txt 2420
000000018ECDC7A0.txt 449
000000018ECF2FB0.txt 16318
000000018ED00C10.txt 468939
000000018ED0D4F0.txt 887819
000000018ED20540.txt 1546
000000018F02DDF0.txt 131
000000018F4545E0.txt 20
000000018F4B16E0.txt 20
000000018FA14320.txt 20
000000018FE18FC0.txt 20
000000018FF7A120.txt 20
000000018FFB3340.txt 367056
00000001900737D0.txt 299528
000000019026F0C0.txt 1031
00000001934FA2A0.txt 1008
00000001A2504E60.txt 59
00000001AB601840.txt 236
00000001AB62A270.txt 232
00000001AC2BE620.txt 236
00000001ADEC3150.txt 131
00000001AED47B10.txt 234
00000001BDDE5500.txt 234
00000001BDE06630.txt 232
00000001BF94D330.txt 20
00000001C23D0FD0.txt 20
00000001CCAD22C0.txt 20
```




## Developing the emulator

Anyone that has worked with Flask knows how easy it is to setup a new route, surprisingly all requests were getting `308 Permanent Redirect`. For `GET` requests the client was following it through but with `POST` not only the client got stuck but the server was throwing an error - something along the lines that it only shows in debug mode that the correct endpoint should be used. The cause were the double slashes in the client endpoints such as `/api//health` and `/api//login`.

Googling this problem was quite hard, since the most common problem slash related was the trailing slash which can be solved by setting `strict_slashes=False`. The reason it's not common with flask or other frameworks it's because web servers such as *nginx* and *apache* merge them automatically. There were some [people](https://github.com/pallets/werkzeug/issues/491) having the same problem, but the solution came from this [SO post](https://stackoverflow.com/questions/24000729/flask-route-using-path-with-leading-slash). 

```python
app = Flask(__name__)
app.url_map.merge_slashes = False
```


Serving JSON was done as follows:

```python

def get_json(endpoint):
    with open(f'jsons/{endpoint}.json', encoding='utf8') as f:
        return json.load(f)

@app.route('/api//health')
def health():
    return 'OK'

@app.route('/api//login', methods=['GET', 'POST'])
def login():
    d = jsonify(get_json('login'))
    return d
```


### Login served but not received

Having setup the most important endpoints I quickly got stuck on login, the web server was sending the response correctly but the client was not receiving it. I was sure it was being received but something internally was ditching the response. The contents being exactly the same the problem had to be the HTTP headers, checking the logs I found the following:


```text

HTTP/1.1 200 OK

Server: Medici

Vary: Origin

Vary: Accept-Encoding

Cache-Control: no-transform

Content-Type: application/json

Date: Sun, 22 Aug 2020 19:26:58 GMT

Connection: keep-alive

X-Powered-By: General Sebastiano Di Ravello

X-Location: GCPEW1

Content-Length: 873

Set-Cookie: [REDACTED]; expires=Mon, 23 Aug 2021 06:55:26 GMT; HttpOnly; path=/; Domain=.os.eidos.com

Set-Cookie: [REDACTED]; path=/; Domain=.os.eidos.com

Set-Cookie: [REDACTED]; path=/; Domain=.os.eidos.com

X-CDN: Incapsula

X-Iinfo: [REDACTED]
```

Flask did not send half of these, so by trial and error I managed to conclude the problem was the lack of `Connection-Type: keep-alive` in my response, the fix was quick:


```python
@app.route('/api//login', methods=['GET', 'POST'])
def login():
    d = jsonify(get_json('login'))
    d.headers['Connection'] = 'keep-alive'
    return d
```

It worked perfectly!


### Beta over and emulator stops working

As soon as the beta was over it was time to really test my emulator aaaaaaaaaaaaaaand it got stuck at logging in. The problem was not apparent at all, since lots of requests were creating errors but the game was still playable. Looking at the logs something caught my attention, the `login` endpoint was throwing a `405 Method Not Allowed`, now this is interesting. I had it setup to accept both `GET` and `POST` so what the hell was it trying to do?
Checking the HTTP headers it became clear what was happening, the client was prefixing the HTTP method with a JSON token.

```json
{"userSandbox":"steam","user":{"uid":"[redacted]","provider":"steam_id"},"system":"windows","identity":{"type":"steam_token","token":"[redacted]"}}POST /api//login
```

Square has their reasons for doing this, it probably gets filtered on the listener webserver before even hitting the node responsible before dealing with the request.


The solution was not the best but it did the trick:

```python
@app.errorhandler(405)
def whygod(e):
    if 'trm_warzones' in request.url:
         return warzone_id(request.url.split('/')[-1])

    return login()

```


This handler solved the problem! Sadly I didn't figure why it worked on the previous day, weird stuff.


### Clearing credentials for release

Having it work just fine was time to clear my personal data from it. E.g the `/api` endpoints has answers that contain your IP, your GPS coordinates, country and city, the `/api//login` contains steam tokens(which I'm not sure are really useful, regardless it's better to be safe than sorry).


Everything was cleared but there was still a thing that I didn't know if it was dangerous. The `token` field in the `login` response.

```text
eyJhbGciOiJIUzI1NiJ9.eyJsb2MiOiJCRS1XQUwiLCJzdWIiOiI2OSIsImF0eSI6InByb2QiLCJjbXUiOiJzdGVhbSIsInVieCI6InN0ZWFtIiwic3lzIjoid2luZG93cyIsInJvbCI6WyJwbGF5QGNyeS10cm12Ni1iZXRhLXByb2Q6cmV0YWlsIl0sImlkcCI6InN0ZWFtX2lkIiwic2VnIjoiTmozdHROSExCZl9IZ1A3R1U5VTBZS0tZeUN0cXAyTDAiLCJjYngiOiJkZWZhdWx0Iiwic2VtIjoiMjIzMDI3MDMiLCJ0YWciOiJ0aGljYyBib2kiLCJza3UiOiIxMzU4ODIwIiwiZXhwIjoxNTk4MjQwNDI2LCJpYXQiOjE1OTgxOTcyMjZ9.eyJiYW5hbmEiOjF9
```


It looked like base64 so I decoded it and quickly learned it was a JWT(JSON Web Token). They're composed of three parts, seperated by a period - Header(defines signature algorithm), Payload(data) and Signature. The first two parts are base64 encoded and are JSONS such as these ones:


```json
//Header
{"alg":"HS256"}


//Payload
{
  "loc": "BE-WAL",
  "sub": "69",
  "aty": "prod",
  "cmu": "steam",
  "ubx": "steam",
  "sys": "windows",
  "rol": [
    "play@cry-trmv6-beta-prod:retail"
  ],
  "idp": "steam_id",
  "seg": "Nj3ttNHLBf_HgP7GU9U0YKKYyCtqp2L0",
  "cbx": "default",
  "sem": "22302703",
  "tag": "thicc boi",
  "sku": "1358820",
  "exp": 1598240426,
  "iat": 1598197226
}
```


I read some articles on how to forge one, with methods that include changing the algorithm, `'alg':'none'`, my tests were a success I had succesfully bypassed the signature of JWT. In the meantime I had also isolated the routine responsible for the login procedure which is conveniently named `OnlineSuiteIdentityProvider::PerformLogin`. Something was off, there didn't appear to be any signature checking. In fact changing the `alg` or even the signature worked just fine.


The reason was that the game was trimming the string on the periods, thus only looking at the payload, there was **NO SIGNATURE CHECK**. Why even use JWT if the signature is not enforced?

Here's the excerpt from the responsible function:


```c
__int64 __fastcall sub_180099260(const char *a1, __int64 a2, __int64 a3){
	/*
		removed
	*/
 v14 = memchr(v13, '.', v12 - (_QWORD)v13);    // finds first period
  if ( v14 )
    v12 = (__int64)v14;
  if ( v12 != osdk_String_end(&v121) )
  {
    v15 = v12 - (_QWORD)osdk_String_begin(&v121);
    if ( v15 != -1 )
    {
      v16 = osdk_String_end(&v121);
      v17 = osdk_String_begin(&v121);
      v18 = memchr(&v17[v15 + 1], '.', v16 - (_QWORD)&v17[v15 + 1]);// finds second period
      if ( v18 )
        v16 = (__int64)v18;
      if ( v16 != osdk_String_end(&v121) )
      {
        v19 = v16 - (_QWORD)osdk_String_begin(&v121);
        if ( v19 != -1 )
        {
          osdk_String_begin(&v121);
          v103 = 0i64;
          v104 = 0i64;
          v103 = &osdk_String_begin(&v121)[v15 + 1];
          v104 = (v19 - v15 - 1) & 0x7FFFFFFFFFFFFFFFi64;
          osdk_String_begin(&v121);
          if ( !v121 )
            osdk_String_length(&v121);
          base64decode((__int64)&v109, (__int64)&v103);
          v99 = 0i64;
          jsonparse(&v99, (__int64)&v109);
          v20 = (char *)v99;
          handlepayload((__int64)&v125, (__int64)v99);
	/*
		removed
	*/
}
```


With all of these the first iteration of the server emulator was complete!


### No more HOSTS file

For the release it was clear that asking people to modify the hosts file was way too much. The solution I came up with was an IAT hook `getaddrinfo` on `osdk_orig.dll`. IAT stands for Import Address Table, since at compilation time addresses of the functions in DLL are not known each module has a table reserved for all imported functions. This table gets filled while the module is being loaded.

At the lower level it works like this:

```text
//C
getaddrinfo(); // function of ws2_32.dll

;asm
call [iat_entry_for_getaddrinfo]; [] are a dereference in assembly
```


Thus if I know the offset of a call to `getaddrinfo` I can easily get it's address on table and replace it with function. Checking x-refs in IDA yielded two results inside `Curl_getaddrinfo_ex`.


```nasm
.text:0000000180005540 Curl_getaddrinfo_ex proc near           ; CODE XREF: sub_1800018C0+100?p
.text:0000000180005540                                         ; getaddrinfo_thread+50?p ...
.text:0000000180005540
.text:0000000180005540 arg_0           = qword ptr  8
.text:0000000180005540 arg_8           = qword ptr  10h
.text:0000000180005540 ppResult        = qword ptr  20h
.text:0000000180005540
.text:0000000180005540                 push    rbp
.text:0000000180005542                 push    rsi
.text:0000000180005543                 push    r12
.text:0000000180005545                 push    r14
.text:0000000180005547                 push    r15
.text:0000000180005549                 sub     rsp, 20h
.text:000000018000554D                 xor     r12d, r12d
.text:0000000180005550                 mov     r15, r9
.text:0000000180005553                 mov     [r9], r12
.text:0000000180005556                 mov     esi, r12d
.text:0000000180005559                 lea     r9, [rsp+48h+ppResult] ; ppResult
.text:000000018000555E                 mov     r14d, r12d
.text:0000000180005561                 call    cs:getaddrinfo ;<-----------
.text:0000000180005567                 mov     ebp, eax
.text:0000000180005569                 test    eax, eax
.text:000000018000556B                 jnz     loc_1800056B8
```

```c
INT WSAAPI getaddrinfo_hook(PCSTR pNodeName, PCSTR pServiceName, const ADDRINFOA* pHints,PADDRINFOA* ppResult) {

	logger("getaddrinfo of %s\n", pNodeName);
	INT res = getaddrinfo("localhost", pServiceName, pHints, ppResult);
	return res;
}

HMODULE value = LoadLibraryA("osdk_orig.dll");
DWORD64 allocatorInit = GetProcAddress(value, "osdk_Allocator_Init");
DWORD64 addressGetAddr = &getaddrinfo_hook;
DWORD lmao;
VirtualProtect(allocatorInit + 0x3A210, 8, PAGE_READWRITE, &lmao); //allow writing to the address
memcpy(allocatorInit + 0x3A210, &addressGetAddr, sizeof(addressGetAddr));
```

The best thing about IAT hooks is that they're module specific, this way any other module can call `getaddrinfo` and not be affected by my hook. 
This patch allows for the client to be installed just by simple drag-n-drop, no worries.


## Source code

You can find the source code and binaries at: [MarvelAvengers](https://github.com/krystalgamer/MarvelAvengers)


Video showcasing the emulator:
<iframe width="560" height="315" src="https://www.youtube.com/embed/MLn475soq1M" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


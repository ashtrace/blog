+++
title = 'Black_hat_v_http'
date = 2025-12-10T22:08:34+05:30
draft = true
+++


## HTTP Fundamentals in Go

HTTP, HyperText Transfer Protocol, as per the name suggests was meant to transfer *Hypertext* (text containing links to other texts or files). Over the period it evolved into the form we see it today. Operating on the age old request-response model, it allows the server and client to transfer web pages (that have grown beyond primitive form of hypertexts) and associated scripts, data and media. Server hosting resources over HTTP (reachable through URL - Uniform Resource Locator) exposes endpoints serving *markup*-text, structured data (JSON, XML etc.) over even binary data (images, videos, audios etc.).

HTTP being stateless protocol, does not provide with inbuilt session state handling. This is maintained by leveraing authentication tokens, cookies, HTTP headers among other means. The client and server are both responsible for arriving at and maintaing the state with proper checks and validations in place.

### HTTP Verbs

HTTP defines certain 'verbs' (actions) including `GET`, `POST`, `HEAD`, `PUT`, `OPTIONS`, `DELETE` among others which are commands that define the request type from the client to the server. One can refer [these docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods) to learn more about them.

### Calling HTTP APIs

V provides with the [net/http](https://modules.vlang.io/net.http.html) module to interact with the HTTP protocol suite. Example functions that may be levaraged to craft a `GET`, `POST` or `HEAD` request are:

```go
fn get(url string) !Response
fn post(url string, data string) !Response
fn head(url string) !Response
```

Other more complex options for crafting `POST` requests are available through functions such as

```go
// post_form sends the map `data` as X-WWW-FORM-URLENCODED data to an HTTP POST request to the given `url`.
fn post_form(url string, data map[string]string) !Response
// post_json sends the JSON `data` as an HTTP POST request to the given `url`.
fn post_json(url string, data string) !Response
// post_multipart_form sends multipart form data `conf` as an HTTP POST request to the given `url`.
fn post_multipart_form(url string, conf PostMultipartFormConfig) !Response
```

To use cookies (for authentication purposes)

```go
fn post_form_with_cookies(url string, data map[string]string, cookies map[string]string) !Response
```

The response is represented through the [Response](https://modules.vlang.io/net.http.html#Response) struct while requests are represented through the [Request](https://modules.vlang.io/net.http.html#Request) struct.

V further provides with methods for other HTTP verbs including:

```go
// DELETE
fn delete(url string) !Response
// PUT
fn put(url string, data string) !Response
// PATCH
fn patch(url string, data string) !Response
```

An example usage of the above mentioned functions is as follows:

```go
module main

import net.http

fn main() {
	// make a GET request
	if resp := http.get("https://www.google.com/robots.txt") {
		// print the HTTP status
		println('${resp.status_code} ${resp.status_msg}')

		// print the response body
		println(resp.body)
	}

	// make a POST request
	data := {
		'foo': 'bar'
	}
	if resp := http.post_form("https://www.google.com/robots.txt", data) {
		// Parses the resp.status_code to derive the message from the `Status` enum defined in net.http
		println(resp.status())
	}
}
```

### Generating a Request

Furthermore, there are functions to craft request and responses as per need

```go
fn new_request(method Method, url_ string, data string) Request
fn new_response(conf ResponseConfig) Response
```

While V provides with `delete()` and `put()` functions,  let's try to create them using `new_request()` to demonstrate usage.

```go
module main

import net.http

fn main() {
	// craft a DELETE request
	mut req := http.new_request(http.Method.delete, 'https://www.google.com/robots.txt', '')
	if resp := req.do() {
		println("DELETE request\'s response status code: ${resp.status_code}")
	}

	// craft a PUT request
	data := http.url_encode_form_data({
		'foo': 'bar',
		'ash': 'trace'
	})
	req = http.new_request(.put, 'https://www.google.com/robots.txt', data)
	req.add_header(.content_type, 'application/x-www-form-urlencoded')
	if resp := req.do() {
		println("PUT request\'s response status code: ${resp.status_code}")
	}
}
```

### Parsing structured data

Using V's parsing mechanism we can better handle the API repsonses structured as JSON.

A small python code to create a server to return message `{ "message": "Hello world!", "status": "Success" }` at `http://127.0.0.1:5000/api`, save this script as `server.py` and run through `python server.py`.

```py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/api")
def api():
    return jsonify({
        "message": "Hello world!",
        "status": "Success"
    })

if __name__ == "__main__":
    app.run(debug=True)
```

A sample V code to fetch and parse the response from the API describe above can be of the form:

```go
module main

import net.http
import json
import log

struct Status {
	message string
	status	string
}

fn main() {
	uri := "http://127.0.0.1:5000/api"
	if resp := http.get(uri) {
		if status := json.decode(Status, resp.body) {
			println("${status.status} -> ${status.message}")
		} else {
			log.error("Failed to parse the response from the server.")
		}
	} else {
		log.error("Failed to fetch data from ${uri}.")
	}
}
```

## Building an HTTP Client that interacts with Shodan

[Shodan](https://www.shodan.io/) is (as described on their website) - **Search Engine for the Internet of Everything**. As per [wikipedia](https://en.wikipedia.org/wiki/Shodan_(website)) it lets users search for various types of servers (webcams, routers, servers, etc.) connected to the internet using a variety of filters. Some have also described it as a search engine of service banners, which is metadata that the server sends back to the client. This can be information about the server software, what options the service supports, a welcome message or anything else that the client can find out before interacting with the server.

### Project Structure

Following the philosophy of code-reuse we'd separate source-codes into various modules that may be used across different projects.

```                  
.
├── ./main.v
├── ./shodan
│   ├── ./shodan/api.v
│   ├── ./shodan/host.v
│   └── ./shodan/shodan.v
└── ./v.mod

2 directories, 5 files
```

The `main.v` defines the `module main`, while the files within the *shodan* directory together form the `module shodan`.

### Defining `shodan` client object

The following code within `shodan/shodan.v` is used to define a client structure for interacting with shodan. The `Client.new()` function allows to initialize objects of the type.

```go
module shodan

const base_url := "https://api.shodan.io"

struct Client {
	api_key string
}

pub fn new(api_key string) &Client {
	return &Client{api_key: api_key}
}
```

### Querying Shodan subscription information

On making request to `'https://api.shodan.io/api-info?key=<YOUR_API_KEY_HERE>`, it returns with following JSON:

```json
{
  "scan_credits": 100,
  "usage_limits": {
    "scan_credits": 100,
    "query_credits": 100,
    "monitored_ips": 16
  },
  "plan": "dev",
  "https": false,
  "unlocked": true,
  "query_credits": 100,
  "monitored_ips": 0,
  "unlocked_left": 100,
  "telnet": false
}
```

Within `shodan/api.v` following structure can be defined to parse the message returned while fetching subscription status.

```go
module shodan

struct APIInfo {
	pub:
		plan			string
		scan_credits	int
		query_credits	int
		monitored_ips	int
		unlocked_left	int
		usage_limits	struct {
			scan_credits	int
			query_credits	int
			monitored_ips	int
		}
		https			bool
		unlocked		bool
		telnet			bool
}
```

Next, the following implementation (in `shodan/shodan.v`) helps to query the URI programatically and parse the JSON response.

```go
pub fn (s &Client) api_info() !&APIInfo {
	res := http.get("${base_url}/api-info?key=${s.api_key}") or {
		return err
	}

	ret := json.decode(APIInfo, res.body) or {
		return err
	}

	return &ret
}
```

Finally, within `main.v`, we can create a client and interact with the Shodan API endpoint.

```go
module main

import shodan
import os

fn main() {
	api_key := os.getenv("SHODAN_API_KEY")
	client := shodan.new(api_key)

	if info := client.api_info() {
		println("Query credits: ${info.query_credits}\nScan credits: ${info.scan_credits}")
	}
}
```

```
─$ SHODAN_API_KEY=<YOUR_API_KEY_HERE> v run .
Query credits: 100
Scan credits: 100
```

### Leveraging Shodan search

Within `shodan/host.v` we can implement logic to query hosts data. An example query appears as follows:

```json
{
  "matches": [
    {
      "info": "protocol 3.8",
      "product": "VNC",
      "vnc": {
        "protocol_version": "3.8"
      },
      "os": null,
      "timestamp": "2026-01-27T18:17:11.152398",
      "isp": "Hangzhou Alibaba Advertising Co.,Ltd.",
      "transport": "tcp",
      "_shodan": {
        "region": "na",
        "ptr": true,
        "module": "nodata-tcp",
        "id": "17568ea8-8df9-4d73-bd65-28553202999d",
        "options": {},
        "crawler": "49217c0cdcbcebaf23c2979ae16d4eba64180b1f"
      },
      "hash": -1730858130,
      "asn": "AS37963",
      "hostnames": [],
      "location": {
        "city": "Beijing",
        "region_code": "BJ",
        "area_code": null,
        "longitude": 116.39723,
        "latitude": 39.9075,
        "country_code": "CN",
        "country_name": "China"
      },
      "ip": 661317180,
      "domains": [],
      "org": "Aliyun Computing Co., LTD",
      "data": "RFB 003.008\nVNC:\n  Protocol Version: 3.8\n",
      "port": 4506,
      "ip_str": "39.106.230.60"
    },
    {
      "info": "protocol 3.8",
      "product": "VNC",
      "vnc": {
        "protocol_version": "3.8",
        "security_types": {
          "2": "VNC Authentication",
          "17": "Ultra"
        }
      },
      "os": null,
      "timestamp": "2026-01-27T18:17:03.644175",
      "isp": "Korea Telecom",
      "transport": "tcp",
<SNIP>
```

First, we define the structure of each host object in the `shodan/host.v` file

```go
module shodan

// In the demo we fetch exposed 'vnc' services,
// thus I added a struct for vnc as it was part of matches.
struct Vnc {
pub:
	protocol_version	string	
}

// Struct to parse the relevant fields from response
struct HostSearch {
pub:
	matches [] struct {
		pub:
			ip			int
			ip_str		string
			port		int
			domains		[]string
			org			string
			data		string
			info		string
			product		string
			vnc			Vnc		@[omitempty]
			os			string
			timestamp	string
			isp			string
			asn			string
			hostnames	[]string
			location	struct {
				city			string
				region_code		string
				area_code		string
				country_name	string
				country_code	string
				longitude		f32
				latitude		f32
			}
			shodan struct {
				region	string
				ptr		bool
				module	string
				id		string
				crawler	string
			}	@[json: '_shodan']			// _ is not allowed as start of a field name
			hash	int
		}
}
```

Next, we add function to query the API.

```go
pub fn (s &Client) host_search(q string) !&HostSearch {
	res := http.get("${base_url}/shodan/host/search?key=${s.api_key}&query=${q}") or {
		return err
	}

	ret := json.decode(HostSearch, res.body) or {
		return err
	}

	return &ret
}
```

Finally, we augment the functionality in `main.v` to query hosts.

```go
module main

import shodan
import log
import os

fn main() {	
	if os.args.len != 2 {
		log.fatal("Usage: ${os.args[0]} <search-term-here>")
	}

	api_key := os.getenv("SHODAN_API_KEY")
	client := shodan.new(api_key)

	if info := client.api_info() {
		println("Query credits: ${info.query_credits}\nScan credits: ${info.scan_credits}")
	} else {
		log.fatal("[!] Encountered error while fetching API credits: ${err}")
	}

	if host_search := client.host_search(os.args[1]) {
		for host in host_search.matches {
			println("${host.ip_str:18} ${host.port:8}")
		}
	}
}
```

A sample output is:

```
─$ SHODAN_API_KEY=<YOUR_API_KEY_HERE> v run . vnc
Query credits: 100
Scan credits: 100
       80.13.97.86     9180
   211.194.253.248     5900
     120.77.205.80     1984
       192.3.231.2     6348
      45.131.213.8     5993
     42.200.73.191    12350
    184.67.178.122     5903
<SNIP>
```

## Scraping Bing Search for Metadata

We are going to use [`net/html`](https://modules.vlang.io/net.html.html), an HTML parser written in V and a part of the standard library offerings, to scrap the first page of bing search for Microsoft office documents (addhering to Open XML format: .pptx, .docx, .xlsx, these document formats are just ZIP archives of XMLs and other files). Next, we'd inspect the metadata stored within `docProps/core.xml` and `docProps/apps.xml` files (docProps is a directory within these file archives.)

```
─$ unzip sample_document.docx 
Archive:  sample_document.docx
  inflating: [Content_Types].xml     
  inflating: _rels/.rels             
  inflating: word/document.xml       
  inflating: word/_rels/document.xml.rels  
  inflating: word/theme/theme1.xml   
  inflating: word/settings.xml       
  inflating: word/styles.xml         
  inflating: word/webSettings.xml    
  inflating: word/fontTable.xml      
  inflating: docProps/core.xml       
  inflating: docProps/app.xml        
```

The `core.xml` file contains author information as well as modification details. Among the XML tags of interest are `creator` and `lastModifiedBy`.

```xml
<cp:coreProperties>
<dc:title/>
<dc:subject/>
<dc:creator>ashtrace</dc:creator>
<cp:keywords/>
<dc:description/>
<cp:lastModifiedBy>ashtrace</cp:lastModifiedBy>
<cp:revision>1</cp:revision>
<dcterms:created xsi:type="dcterms:W3CDTF">2026-01-31T18:14:00Z</dcterms:created>
<dcterms:modified xsi:type="dcterms:W3CDTF">2026-01-31T18:14:00Z</dcterms:modified>
</cp:coreProperties>
```

The `app.xml` file contains information about the application type and version used to create the Open XML document. The XML tags of interest include `Application`, `AppVersion`. The *AppVersion* does not directly specify Office 2016, Office 2019 etc but that may be deduced through a mapping.

```xml
<Properties>
<Template>Normal</Template>
<TotalTime>0</TotalTime>
<Pages>1</Pages>
<Words>2</Words>
<Characters>14</Characters>
<Application>Microsoft Office Word</Application>
<DocSecurity>0</DocSecurity>
<Lines>1</Lines>
<Paragraphs>1</Paragraphs>
<ScaleCrop>false</ScaleCrop>
<Company/>
<LinksUpToDate>false</LinksUpToDate>
<CharactersWithSpaces>15</CharactersWithSpaces>
<SharedDoc>false</SharedDoc>
<HyperlinksChanged>false</HyperlinksChanged>
<AppVersion>16.0000</AppVersion>
</Properties>
```

### Defining the metdata

Create a module directory named `metadata` and add a file called `openxml.v` with type definition for each XML file we wish to parse. Finally, we add the logical mapping to retrieve the Office Suite version.

```go
module metadata

struct OfficeCoreProperty {
	xml_name			string
	creator				string
	last_modified_by	string
}

struct OfficeAppProperty {
	xml_name	string
	application	string
	company		string
	version		string
}

OfficeVersions := {
	'16': '2016',
	'15': '2013',
	'14': '2010',
	'12': '2007',
	'11': '2003'
}

pub fn (a &OfficeAppProperty) get_major_version() string {
	tokens := a.version.split('.')

	if tokens.len < 2 {
		return 'Unknown'
	}
	if v := OfficeVersions[tokens[0]] {
		return v
	} else {
		return 'Unknown'
	}
}
```

### Populating the structs

Next, we write the code to parse the document and fill up the metadata fields of interest.
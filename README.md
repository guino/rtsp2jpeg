# RTSP to JPEG and MJPEG using ffmpeg and lighttpd

This repository contains scripts for converting a RTSP source into JPEG and MJPEG. The information here is a derivate of the work provided by https://stevethemoose.blogspot.com/ (specifically [this post](https://stevethemoose.blogspot.com/2021/07/converting-rtsp-to-mjpeg-stream-on.html)) -- which was a great starting point.

The original post used debian linux box while I've been using a Raspberry Pi 4B virtually with no differences (other than slower response time for 1st frame).

### Original information

#### Install lighttpd

`apt-get install lighttpd`

Enable cgi-bin:
```
cd /etc/lighttpd/conf-enabled/
ln -s ../conf-available/10-cgi.conf
```
Configure the cgi-bin folder, and set lighttpd to not buffer the entire file before starting to send:

Alter `/etc/lighttpd/conf-available/10-cgi.conf`:
```
server.modules += ( "mod_cgi" )

$HTTP["url"] =~ "^/cgi-bin/" {
        server.stream-response-body = 2
        cgi.assign = ( "" => "" )
        alias.url += ( "/cgi-bin/" => "/var/www/cgi-bin/" )
}
```
Restart lighttpd to pick up the configuration change:

`systemctl restart lighttpd`

Create the cgi-bin folder:

`mkdir /var/www/cgi-bin`

#### Scripts

Create the scripts that'll generate a single frame and stream, adjust the IP address and RTSP stream URL to suit your particular camera:

Create `/var/www/cgi-bin/webcamframe`:
```
#!/bin/bash

echo "Content-Type: image/jpeg"
echo "Cache-Control: no-cache"
echo ""
ffmpeg -i "rtsp://192.168.60.13:554/user=admin&password=SECRET&channel=1&stream=0.sdp" -vframes 1 -f image2pipe -an -
```
Create `/var/www/cgi-bin/webcamstream`:
```
#!/bin/bash

echo "Content-Type: multipart/x-mixed-replace;boundary=ffmpeg"
echo "Cache-Control: no-cache"
echo ""
ffmpeg -i "rtsp://192.168.60.13:554/user=admin&password=SECRET&channel=1&stream=0.sdp" -c:v mjpeg -q:v 1 -f mpjpeg -an -
```
Make the two scripts executable (otherwise you'll get a 500 Internal Server Error)
```
chmod +x /var/www/cgi-bin/webcamframe
chmod +x /var/www/cgi-bin/webcamstream
```
#### Complete

Now you should be able to access these two URLs on your server:
```
http://192.168.60.10/cgi-bin/webcamstream
http://192.168.60.10/cgi-bin/webcamframe
```
Both of these take 1-2 seconds to start, which I think is a combination of getting the RTSP stream going, and an inherent delay in generating the MJPEG output. Once it is running, there is about a one second delay on the video stream, which I gather is normal for ffmpeg generating MJPEG.

#### Troubleshooting

If you get distorted/smeared/artifacts on the output stream, try adding at the start of the ffmpeg arguments -rtsp_transport tcp  to force RTSP over TCP. Apparently there may be a UDP [buffer issue](https://github.com/ZoneMinder/ZoneMinder/issues/811) in ffmpeg and/or Linux that can cause frames to get truncated. Other options to try out are here.

You can troubleshoot the scripts on the command line like this, which will let you see the output of ffmpeg and the start of what is sent to the web client:
```
cd /var/www/cgi-bin
./webcamframe | hd | head
```

### Modifications

I had decent success with the original scripts and I still have them but I was plagged with browser side issues with the implementation provided (mostly on mobile browsers such as safari).

In the end I renamed the scripts for my device (EZViZ DB1) and created two new scripts:
* ezvizsnap - Same as the webcamframe from original post (takes few seconds for each request as it starts ffmpeg+rtsp on every request)
* ezvizmjpeg  - Same as the webcamstream from original post (takes few seconds to start and **DOES NOT WORK WELL WITH SAFARI** when using automatic refresh)
* ezvizserv - Starts ffmpeg creating a temporary jpg file every second and continues to run while new requests are made using ezvizjpegs
* ezvizjpegs - Ensures ezvizserv is running (starts it if required) and returns the latest JPEG available for the device

Safari seems to load the jpeg (html img src with 1 second reload) multiple times (i.e. 10) which may be caching or retry mechanism and this was starting multiple/concurrent ffmpeg processes which was loading up CPU and causing display issues.

My scripts assume a link was created in /var/www/html to the temporary jpeg file:

`ln -s /tmp/ezviz.jpg /var/www/html`

Basically every time ezvizjpegs is requested it resets the count for shutting down the ffmpeg process (starting it if required), so as long as ezvizjpegs is called once every 20 seconds ffmpeg will continue to writing the temporary jpeg. The ezvizjpegs also returns the most recent jpeg file and when it is not requested within 20 seconds ffmpeg will shutdown automatically.

This approach works well with all browsers I tried including safari despite it making up to 10 concurrent requests for ezvizjpegs at the same time.

Hopefully this will help someone else but mainly I don't want to forget all the time I spent with different approaches till I got this stable working solution.

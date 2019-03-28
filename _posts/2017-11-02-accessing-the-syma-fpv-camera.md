---
layout: post
title: "Accessing the Syma FPV Camera"
date: 2017-11-02
author: yubbit
categories: ['Quadcopter', 'Computer Vision']
---

I was fiddling with a cheap SymaX5HW quadcopter that I got for $40. It advertises itself as having FPV capabilities, and comes with an app you can install on your phone to receive the video feed.

I scoured the internet a little bit and [found that you can access the videostream](http://drone.winnfreenet.com/2016/07/x8c-wifi-camera-hacking/) of a number of Syma models by opening `http://192.168.1.1/videostream.cgi`.

Opening this brings up an authentication dialog box. Using `admin` as the username and leaving the password blank lets you access the feed.

Now I wanted to access this feed using Python, but found that I couldn't, even if it worked fine in Firefox. After checking the headers, it turns out that the camera uses digest authentication, which Firefox could resolve automatically. You can read up about it [here](https://en.wikipedia.org/wiki/Digest_access_authentication).

As it turns out, Python's standard library has tools for dealing with digest authentication, so getting access to the video stream is as simple as running this:

```python
import requests

def main():
    url = 'http://192.168.1.1/videostream.cgi'
    user = 'admin'
    pswd = ''
    auth = requests.auth.HTTPDigestAuth(user, pswd)
    r = requests.get(url, auth=auth, stream=True, timeout=5)
```

This provides access to an MJPEG stream, which is basically sent as a series of JPEG files to your computer. Once again, I got to googling and conveniently found [this post on Stack Overflow](https://stackoverflow.com/questions/21702477/how-to-parse-mjpeg-http-stream-from-ip-camera). The following code should do the trick:

```python
def get_frame_bytes(r):
    buf = 'b'

    for chunk in r.iter_content(chunk_size=1024):
        buf += chunk
        beg = buf.find(b'\xff\xd8')
        end = buf.find(b'\xff\xd9')
        if beg != -1 and end != -1 and beg < end:
            byte_img = buf[beg:end+2]
            buf = buf[end+2:]
            yield byte_img
```

This works by continuously reading data to the buffer, and extracting everything between the beginning and ending byte markers for a JPEG image. You can probably improve on this code significantly by adding destructors and specifying the number of bytes transmitted. In this state, it'll run continuously, and might read two images at once if you're missing an ending byte somewhere.

Now this is just the byte representation of the JPEG image. To display it, we need to use some library to process the JPEG image and return an array of color channels. Either OpenCV or PIL can do this:

```python
import cv2
import numpy as np

def get_frame_arr(byte_img):
    arr = cv2.imdecode(np.fromstring(byte_img, dtype=np.uint8), cv2.IMREAD_COLOR)
    return arr
```

Finally, we want to set up a loop display each frame. You can do this using either OpenCV or matplotlib, but I'll only be doing OpenCV for now:

```python
for byte_img in get_frame_bytes(r):
    arr = get_frame_arr(byte_img)
    cv2.imshow('SymaX5HW Camera', arr)
    if cv2.waitKey(1) == 27:
        break
```

Put together, you have:

```python
import requests
import cv2

def main():
    url = 'http://192.168.1.1/videostream.cgi'
    user = 'admin'
    pswd = ''
    auth = requests.auth.HTTPDigestAuth(user, pswd)
    r = requests.get(url, auth=auth, stream=True, timeout=5)

    for byte_img in get_frame_bytes(r):
        arr = get_frame_arr(byte_img)
        cv2.imshow('SymaX5HW Camera', arr)
        if cv2.waitKey(1) == 27:
            break


def get_frame_bytes(r):
    buf = 'b'

    for chunk in r.iter_content(chunk_size=1024):
        buf += chunk
        beg = buf.find(b'\xff\xd8')
        end = buf.find(b'\xff\xd9')
        if beg != -1 and end != -1 and beg < end:
            byte_img = buf[beg:end+2]
            buf = buf[end+2:]
            yield byte_img


def get_frame_arr(byte_img):
    arr = cv2.imdecode(np.fromstring(byte_img, dtype=np.uint8), cv2.IMREAD_COLOR)
    return arr


if __name__ == '__main__':
    main()
```


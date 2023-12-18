---
title: ImaginaryCTF 2023 - perfect picture
description: Writeup for perfect picture web challenge
date: 2023-12-18
tldr: Someone seems awful particular about where their pixels go...
draft: false
tags: ["ctf", "writeup", "web"]
---

# perfect picture 

We are supposed to upload a picture matching some very specific conditions.

## The app

```python
from flask import Flask, render_template, request
from PIL import Image
import exiftool
import random
import os

app = Flask(__name__)
app.debug = False

os.system("mkdir /dev/shm/uploads/")
app.config['UPLOAD_FOLDER'] = '/dev/shm/uploads/'
app.config['ALLOWED_EXTENSIONS'] = {'png'}

def check(uploaded_image):
    with open('flag.txt', 'r') as f:
        flag = f.read()
    with Image.open(app.config['UPLOAD_FOLDER'] + uploaded_image) as image:
        w, h = image.size
        if w != 690 or h != 420:
            return 0
        if image.getpixel((412, 309)) != (52, 146, 235, 123):
            return 0
        if image.getpixel((12, 209)) != (42, 16, 125, 231):
            return 0
        if image.getpixel((264, 143)) != (122, 136, 25, 213):
            return 0

    with exiftool.ExifToolHelper() as et:
        metadata = et.get_metadata(app.config['UPLOAD_FOLDER'] + uploaded_image)[0]
        try:
            if metadata["PNG:Description"] != "jctf{not_the_flag}":
                return 0
            if metadata["PNG:Title"] != "kool_pic":
                return 0
            if metadata["PNG:Author"] != "anon":
                return 0
        except:
            return 0
    return flag

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload():
    if 'file' not in request.files:
        return 'no file selected'

    file = request.files['file']

    if file.filename == '':
        return 'no file selected'

    if file and allowed_file(file.filename):
        filename = file.filename

        img_name = f'{str(random.randint(10000, 99999))}.png'
        file.save(app.config['UPLOAD_FOLDER'] + img_name)
        res = check(img_name)

        if res == 0:
            os.remove(app.config['UPLOAD_FOLDER'] + img_name)
            return("hmmph. that image didn't seem to be good enough.")
        else:
            os.remove(app.config['UPLOAD_FOLDER'] + img_name)
            return("now that's the perfect picture:<br>" + res)

    return 'invalid file'

if __name__ == '__main__':
    app.run()

```

## edit pixels 

```python
from PIL import Image


image_path = "not_perfect.png"
image = Image.open(image_path).convert("RGBA")
pixel_data = image.load()
x = 412  # Replace with the x-coordinate of the pixel you want to edit
y = 309  # Replace with the y-coordinate of the pixel you want to edit
new_rgba_value = (52, 146, 235, 123)
pixel_data[x, y] = new_rgba_value
x = 12
y = 209
new_rgba_value = (42, 16, 125, 231)
pixel_data[x, y] = new_rgba_value
x = 264
y = 143
new_rgba_value = (122, 136, 25, 213)
pixel_data[x, y] = new_rgba_value


output_path = "colors.png"  # Replace with the desired path to save the modified image
image.save(output_path)
```
## edit exifdata

```
─(kali㉿kali)-[/mnt/ctf/web_perfect_picture]
└─$ exiftool -PNG:Description="jctf{not_the_flag}" not_perfect.png

                                                                                                                                                                                                  
┌──(kali㉿kali)-[/mnt/ctf/web_perfect_picture]
└─$ exiftool -PNG:Description="jctf{not_the_flag}" colors.png     
    1 image files updated
                                                                                                                                                                                                  
┌──(kali㉿kali)-[/mnt/ctf/web_perfect_picture]
└─$ exiftool -PNG:Title="kool_pic" colors.png     
    1 image files updated
```
## upload the picture and get flag

```
now that's the perfect picture:
ictf{7ruly_th3_n3x7_p1c4ss0_753433}
```

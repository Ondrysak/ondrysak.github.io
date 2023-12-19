---
title: Synth presets from Circulate on Novation circuit 
description: Dumping synth presets from an online database currently not supporting the newer Novation Circuit directly.
date: 2023-12-19
tldr: Dumping synth presets from an online database currently not supporting the newer Novation Circuit directly.
draft: false
tags: ["groovebox", "presets", "novation", "circuit", "circulate", "rev"]
---

# Novation Circuit OG
 
The OG circuit device which works just fine with Circulate

> An all-in-one studio with two synth tracks, two MIDI tracks and four drum track

![OG](/images/ogcircuit.png)



# Novation Circuit Tracks 

The newer Circuit Tracks device which in theory should work with circulate too, but does not right now.

> An all-in-one studio with two synth tracks, two MIDI tracks and four drum track

![Tracks](/images/circuittracks.png)


# Circulate

Circulate is a free patch sharing community for the Novation Circuit groovebox. Which uses midi in browser to load the patches into the Novation Circuit OG.
https://circulate.neuma.studio/

![Circulate](/images/circulate.png)


# Circulate with Circuit Tracks

While Novation Circuit Tracks posseses the very same synth engine as OG Circuit, it is in some way different and the circulate website does not work for loading the synth patches into Circuit tracks. 


# Building a dumper

After fiddling with Circulate a bit it seems like it stores the actual patch data as `base64`` encoded string on the source of the page. The idea here is to dump the patches from the page which does not work with Circuit Tracks and load them manually using Novation Components



```python
import base64
from pathlib import Path
import re
import urllib.request    
import os

def dump_patch_from_base64(base64_encoded_patch,folder):
    decoded_patch = base64.b64decode(base64_encoded_patch).decode("utf-8") 

    #print(decoded_patch)
    patch_bytes = []
    for byte_string in str(decoded_patch).split(','):
        patch_bytes += [int(byte_string)]
    #print(patch_bytes)
    name = ''.join([ chr(c) for c in patch_bytes[9:25] ]).strip()
    name = name.replace(' ', '_')
    print(f'  {name}')
    with open(f'{folder}\{name}.syx', 'wb') as f:
        for i in patch_bytes:
            f.write(bytes((i,)))


def find_unique_patches_in_html_dump(html_file_path):
    with open(html_file_path, 'r', encoding='UTF-8') as file:
        html_shit = file.read().replace('\n', '')
    #print(html_shit)
    matches = re.findall(r'atob\(.[0-9a-zA-Z=]+.\)', html_shit)
    unique_patches = set(matches)
    #print(unique_patches)
    patches_list = list(unique_patches)
    #print(patches_list)
    return [p[5:-2] for p in patches_list]

i = 0
urllib.request.urlretrieve("https://circulate.neuma.studio/", "source.html")
dump_folder = "dumps"
try:
    os.mkdir("dumps")
except FileExistsError:
    print(f"{dump_folder} already exists") 
for b64_patch in find_unique_patches_in_html_dump('source.html'):
    i += 1
    dump_patch_from_base64(b64_patch, dump_folder)

print(f'DUMPING FINISHED - {i} patches dumped')

```

bunch of files should be created in a folder `dumps` each representing a single patch 

```
Static_Bass.syx
SpaceRadio.syx
```

now you can just open your https://components.novationmusic.com/ load the patches into a pack and use them. This method still leaves a lot to be desired, but it is a decent workaround until the author of https://circulate.neuma.studio/ manages to fix it properly, unfortunately there is no support from Novation for this so it may never happen.

Up to date code is also available on github  https://github.com/Ondrysak/circulate_dump






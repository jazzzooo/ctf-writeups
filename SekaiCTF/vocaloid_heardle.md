# Vocaloid Heardle

## The Challenge
We are given the file `flag.mp3`, and this source code:
```python
import requests
import random
import subprocess

resources = requests.get("https://sekai-world.github.io/sekai-master-db-diff/musicVocals.json").json()

def get_resource(mid):
    return random.choice([i for i in resources if i["musicId"] == mid])["assetbundleName"]

def download(mid):
    resource = get_resource(mid)
    r = requests.get(f"https://storage.sekai.best/sekai-assets/music/short/{resource}_rip/{resource}_short.mp3")
    filename = f"tracks/{mid}.mp3"
    with open(filename, "wb") as f:
        f.write(r.content)
    return mid

with open("flag.txt") as f:
    flag = f.read().strip()

assert flag.startswith("SEKAI{") and flag.endswith("}")
flag = flag[6:-1]
tracks = [download(ord(i)) for i in flag]

inputs = sum([["-i", f"tracks/{i}.mp3"] for i in tracks], [])
filters = "".join(f"[{i}:a]atrim=end=3,asetpts=PTS-STARTPTS[a{i}];" for i in range(len(tracks))) + \
          "".join(f"[a{i}]" for i in range(len(tracks))) + \
          f"concat=n={len(tracks)}:v=0:a=1[a]"

subprocess.run(["ffmpeg"] + inputs + ["-filter_complex", filters, "-map", "[a]", "flag.mp3"])
```
The code starts with a set of roughly 600 vocaloid songs.
Each song has a `musicId`, however, in this dataset between 1 and 8 songs can have the same id.
For each character in the flag, one song is picked at random, whose `musicId` matches the ascii value of the character.

The song is cut down to its first 3 seconds.
Then each of these segments, one for each flag character, are concatenated.
`flag.mp3` is 33 seconds and contains 11 such segments.


## The Solution

The first idea I had turned out to work perfectly.
Say we have a sound clip, _x_, and a set of other sound clips, _a, b, c,..._, one of which contains the same sound as _x_.
If _x_ and _a_ are different, then the size of compressing _x||a_, x concatenated with _a_, will be roughly the same as the size of compressing _x_ plus the size of compressing _a_.
The compression algorithm can only do so a bit of compression.


On the other hand, say _x_ and _b_ are the same, then `sizeof(compress(x||b)) ≈ sizeof(compress(x||x)) ≈ sizeof(compress(x))`.
The comperssion algorithm is able to tell that the data is the same, and will reduce it to around half the size.

## The Implementation

First, let's narrow down the set of song's to only the relevant ones.
We only care about songs whose `musicId` is a printable character.
Let's make a folder `tracks`, and download all the songs we need.
```python
from string import printable

from requests import get
from tqdm import tqdm

resources = get("https://sekai-world.github.io/sekai-master-db-diff/musicVocals.json").json()
relevant = {
    song["assetbundleName"]: song["musicId"]
    for song in resources
    if song["musicId"] in set(printable.encode())
}


def download(res):
    resp = get(f"https://storage.sekai.best/sekai-assets/music/short/{res}_rip/{res}_short.mp3")
    with open(f"tracks/{res}.mp3", "wb") as f:
        f.write(resp.content)


for r in tqdm(relevant.keys()):
    download(r)
```

I constructed `relevant` as a dictionary, that will help us later to map the song name back to an ascii character.

Next we will split up the flag into 11 parts so that each segment is isolated.

```
ffmpeg -i flag.mp3 -t 3 flag0.mp3
ffmpeg -i flag.mp3 -ss 3 -t 3 flag1.mp3
ffmpeg -i flag.mp3 -ss 6 -t 3 flag2.mp3
...
```

The `-t` flag means to only keep 3 seconds, the `-ss` flag seeks to the specified second or timestamp.

Now to ease the process, let's keep only the first 3 seconds of each song we downloaded.

```
mkdir tracks3s
cd tracks
for f in *; do ffmpeg -i "$f" -t 3 "../tracks3s/$f"; done
```

And now we're ready for the final step, to loop through each segment, then loop thru each song, and compare the compressed sizes.

```python
from lzma import compress


def compressed_size(data):
    return len(compress(data, preset=9))  # maximum compression


songs = {name: open(f"tracks3s/{name}.mp3", "rb").read() for name in relevant.keys()}

for i in range(11):
    flag = open(f"flag{i}.mp3", "rb").read()
    flag_size = compressed_size(flag)
    diffs = []

    for k, song in tqdm(songs.items()):
        size_diff = flag_size + compressed_size(song) - compressed_size(song + flag)
        diffs.append([size_diff, k])

    print(chr(relevant[sorted(diffs)[-1][1]]))
```
Those of you who followed along and ran the code will have noticed that it got a letter wrong: `v0CaloFd<3u`.
This is easily corrected, F should be I, but it shows that there is room for improvement.
Using a different algorithm, comparing by compression ratio instead of total difference, and working with the extracted raw audio data instead of mp3, would all likely yeald better results.



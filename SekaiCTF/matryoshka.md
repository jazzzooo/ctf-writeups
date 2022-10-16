# Matryoshka

## The Challenge
![pain.png](https://2022.ctf.sekai.team/files/fe9c8c0def8beee91bbf292485b48ff2/Matryoshka-Lite.png)

## The Solution

### Part 1

The code splits some data into 4-bit chunks (nibbles), and prints a color for each nibble.
Surely this won't be too painful?

Step one was to get the palette.
The easiest way is to simply install the same editor and use the same theme.
The editor is VS Code, with the "high contrast" theme.
After installing it I rewrote the code and verified the colors looked the same.
Then I wrote some code to print a reference sheet, each color and its associated nibble.
Then it was just some pain to decode the colors by hand.
I knew I could use Pillow to get it easily, but I was too lazy to do anything smart on a guess chall.
Here is the code I ended up with:

```python
from Crypto.Util.number import long_to_bytes

data = b"flag{"
colors = list("0rgybmc21RGYBMC3")
nums = list(range(40, 48)) + list(range(100, 108))

d = "".join(bin(i)[2:].zfill(8) for i in data)
p = ""
for i in range(0, len(d), 8):
    l = d[i : i + 4]
    h = d[i + 4 : i + 8]
    he = int(h[1:], 2) + [40, 100][int(h[0])]
    le = int(l[1:], 2) + [40, 100][int(l[0])]
    p += f"\033[{he}m \033[0m\033[{le}m \033[0m"
# print(p)

nibbles = [bin(e)[2:].zfill(4) for e in range(2**4)]
inv = {}
for nib in nibbles:
    inv[int(nib[1:], 2) + [40, 100][int(nib[0])]] = nib

def inverse(arr):
    res = ""
    for i in range(0, len(arr), 2):
        res += inv[arr[i+1]]
        res += inv[arr[i]]
    return long_to_bytes(int(res, 2))

#       h  t  t  p  s  :  /  /  m  a  t  r  y  o  s  h  k  a  .  s  e  k  a  i  .  t  e  a  m  /  -  q  L  f  -  A  o  a  u  r  8  Z  V  q  K  4  a  F  n  g  Y  g  .  p  n  g \n
ans = "1c b2 b2 02 y2 Gy 3g 3g Mc rc b2 g2 R2 3c y2 1c Yc rc Cg y2 mc Yc rc Rc Cg b2 mc rc Mc 3g Mg r2 Bb cc Mg rb 3c rc m2 g2 1y Gm cm r2 Yb by rc cb Cc 2c Rm 2c Cg 02 Cc 2c G0"
ans = ans.replace(" ", "")
print(inverse([nums[colors.index(e)] for e in ans]))

for c in inv.keys():
   print(f"{c:3} \033[{c}m \033[0m")
```


### Part 2

We get a link to another image:
![morepain.png](https://matryoshka.sekai.team/-qLf-Aoaur8ZVqK4aFngYg.png)

If you have never seen a png like this, it will be very difficult to guess what it is.
Running `pngcheck` and seeing the invalid `iDOT` headers might lead you down the right path if you look into those.
This is a so-called [ambiguous png](https://www.da.vidbuchanan.co.uk/widgets/pngdiff/), which in the past rendered differently on apple products.
Apple recently patched this, so to get the Apple version of the image I could a) do something smart and figure out how to decode these headers or b) do the laziest thing ever and emulate an old macOS version that renders these pngs wrong.

I used [OSX-KVM](https://github.com/kholia/OSX-KVM) and got the oldest macOS version it supported.
Then I could simply open safari and look at the same image, getting a different QR code.

![nomorepain?.png](SekaiCTF/qr2.png)


### Part 3

The QR code decodes to the following code:
```
shc:/56762959532654603460292540772804336028702865676754222809286237253760287028647167452228092863457452621103402467043423376555407105412745693904292640625506400459645404280536627540536459624025250555056338566029120106413333400028742635076939734552056936583171064558751131556353203754372575033328200705643838552934743139500009536061356931346955643709527105115665600005602172234467374542085807222475347132034424395261373056004444002400085237353061222027453167672627082630290769235375711135114127401104212540537525556303742533507136503255653563264154433970205436100050743522116306752331635775741156433654585503107626684254686208403754634470273768056171327607656125712725523234611005361121030308333867583166536725643767425270646323270003005623700860226659203405252357762043663326362209257233442225631073757558424358121058221221247175065067275426364058293454221133236771205077255211441131752363046604226175031256730654443172527522070726232026532434301128375372255668000400627667676055323160225036622041105858255222692922334259596624276377446745261173582545412027102861666538363053246255715622773453607507284404720407630733005623703641432800427011066429357722525365740010257264576557765569557135536228273331723728623059574332602964335058526177070375095735563159552930336664240727603959105433044575393334503567543958542929065332126645230910313334672722391208422438276434441236775655650958267743437110394352455760210354655321596331533463522358444058636442336866670845305568693721662269635473434227715411302507646165766341072469394221072671236868392755064436586159754123754210552170093809524555337700313654703040673106437576344009087611676326535274567421423023706811744311775220407005454032310440346554616620552130066153666738533667226435276755422240350073103639763904405705005555244371301010730641435756764057646755286006396271642377067569577743576669054164110561644535096843257762673432272976686542737404354077010832356003656226634535455971326660756506220359605868077353056052347436404527397258656831553804624139525240420467593362371139026720436433630272626572681040385977300452644174
```

This can be decoded using [shc-extractor](https://github.com/obrassard/shc-extractor).
The result will contain a link, `https://matryoshka.sekai.team/8d789414a7c58b5f587f8a050b8d788e.wav`, to an audio file.
The audio sounds like static with distant mumbling heard.
The final guess, which was my first idea, was to split the audio into a right and left channel, and then subtract them to get just the mumbling in the background.
This turned out to be a TTS reading out the flag.
`SEKAI{KandoRyoko5Five2Two4Four}`

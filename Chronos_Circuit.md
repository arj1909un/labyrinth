# Description:
The door finally opens and you find yourself looking at another one of Daedalus' projects. It looks like a design script that generates a key using the current time as the seed, encrypts the secret and stores it in an EEPROM. But is that really all it does?
Flag format: labyrinth{secret}

# Flag:
`chr0n0s_st4rt3d_t1m3_fr0m_0!`

# Solve:
After going through the challenge description and downloading the attached files we notice that one of the files is a .sal extension, this means we would need a logic analyser tool to view this file the one I used was SALAE Logic 2.0. Now after opening up the .sal file and I saw this on my screen.

<img width="1470" height="380" alt="image" src="https://github.com/user-attachments/assets/ceb99db7-91b3-48a6-afbb-2a7c86f75099" />

now to understand this image we need to open the other attachment.

The other file has a .ino extension which means it is a Arduino sketch written in C++. after going through the code we realise that the we are encrypting the contents of the variable secret (just the placeholder text “secret” in the given file, but the real flag on the captured device) using AES(encryption method to encrypt data) and writing to an external EEPROM(a memory chip that is used to store data), detailed explanation on AES and EEPROM is not necessary for this challenge .

I found out that the communication protocol in this challenge is I2C(inter integrated circuit) we dont need to go too deep into this too solve this challenge so I will just give a brief overview on how I2C works :

I2C has 2 lines
* SDA -> Serial Data, Transfers the actual data bits
* SCL -> Serial Clock, Synchronizes data transmission
  
The Arduino communicates to the EEPROM via I2C.

Now we have a good understanding of what's going on in the challenge.
So the big question is what are we looking for so AES as I said is an encryption algorithm which has 3 important components:
* Plaintext
* Cipher Text
* Key

In this challenge the flag is the plaintext as I discussed above, so we need to find 2 things 1. cipher text 2. key.

For the cipher text we go back to the sal file, what we see is the tapped communication between arduino and eeprom and this was done by clipping a logic analyser to the SDA and SCL now which waveform is SDA and which is SCL is decided by checking the channel waveforms we notice that channel 1 has uniform signals whereas channel 0 has random signals so this means our channel 1 is the clock. we use the add analyzer option to select I2C and respective SDA and SCL.

<img width="1425" height="371" alt="image" src="https://github.com/user-attachments/assets/2e268965-cf5f-43c6-bc3e-dcc38ed26d8a" />

Now it is impossible to understand what data is being sent to the EEPROM in this form so we export the whole file as a csv file to make it easy for us to understand what's going on.
after opening the csv we file we see this:

<img width="473" height="753" alt="image" src="https://github.com/user-attachments/assets/088de01e-725b-4688-acaf-c59806bb59a0" />

To solve this challenge we only need to focus on these columns:
* data: this is the most important as it tells what encrypted bytes were being sent to the address
* address: the address of EEPROM where the data is being sent 

Focusing more on the Data column we notice a pattern :

<img width="672" height="442" alt="image" src="https://github.com/user-attachments/assets/726fb881-bd5a-447e-9fa2-e2762871f76b" />

that for each transaction 3 bytes are being sent , to understand them we need to look up how EEPROM writes data
* 1st byte: always 0 shows upper 8 bits of memory address
* 2nd byte: lower 8 bits of the memory address.
* 3rd byte: the actual data byte

So our final encrypted cipher text that we intercepted is `4b0d83daa4c29be7882623c3beb0ae5feb3ddfdc71bc4bf6ced1c416547a9419` 

You might have noticed that we didn't take the data bytes from the 1st 2 transactions that is because the first two data bytes are a part of the EEPROM’s memory address selection and we figured it out by looking at this block of code:
```bash
void eepromWriteByte(unsigned int addr, byte data) {
    Wire.beginTransmission(EEPROM_I2C_ADDR);

    Wire.write((int)(addr >> 8));    //  First address
    Wire.write((int)(addr & 0xFF));  //  Second address
    Wire.write(data);                //  Actual data

    Wire.endTransmission();
    delay(EEPROM_PAGE_DELAY_MS);
}
```
The expression `addr >> 8` extracts the upper eight bits, while `addr & 0xFF` extracts the lower eight bits.

Next step is to look for the key, for that we will focus on the generateAeskey function:
```
void generateAesKey() {
  srand(time(0));
  for (int i = 0; i < 16; i++) {
    aesKey[i] = (byte)(rand() % 256);
  }
}
```
Firstly in the description it was mentioned that the key is generated using the current time as the seed. After investigating the srand() and time(0) functions I found out that Arduino UNO the model being used in this challenge has no Real Time Clock mind you this is different from a regular clock.
Secondly the srand(seed) function uses the parameter as the number generator's seed.
Thirdly the rand() generates a sequence based on that seed.

All this was just an overview of what all functions were doing but to understand the code we will need to open up the official documentation of these functions i have provided the avr-libc github repo where i found documentation for these functions:
https://github.com/avrdudes/avr-libc/blob/main/libc/stdlib/random.c

After some digging around we found out that time(0) returns 0. this 0 is used as the seed for the srand function lets see how srand() works :

<img width="627" height="541" alt="image" src="https://github.com/user-attachments/assets/c2e7b794-288e-4c50-a0b9-0db1a84bc068" />

In this ss we need to focus on the part where x = 0 , this implements the Park–Miller style algorithm used by certain rand() implementations in avr-libc.

I implemented this code based on the code given by avr-libc in python to get the key:
```bash
def do_rand(x):
    if x == 0:
        x = 123459876
    hi = x // 127773
    lo = x % 127773
    x = 16807 * lo - 2836 * hi
    if x < 0:
        x += 0x7fffffff
    return x, x % 0x8000

state = 0    
key = []
for _ in range(16):
    state, r = do_rand(state)
    key.append(r % 256)                  

key = bytes(key)
print("key:", key.hex())
```
so I did nothing fancy I just changed the random logic of our original code to generate the key.

PHEW. finally we have all the missing pieces of the flag and we will run this final script to get the flag:
```bash
from Crypto.Cipher import AES

ct = bytes.fromhex("4b0d83daa4c29be7882623c3beb0ae5feb3ddfdc71bc4bf6ced1c416547a9419")

def do_rand(x):
    if x == 0:
        x = 123459876
    hi = x // 127773
    lo = x % 127773
    x = 16807 * lo - 2836 * hi
    if x < 0:
        x += 0x7fffffff
    return x, x % 0x8000

state = 0    
key = []
for i in range(16):
    state, r = do_rand(state)
    key.append(r % 256)                 

key = bytes(key)

flag  = AES.new(key, AES.MODE_CBC, bytes(16)).decrypt(ct)
print(flag)
```
I will walkthrough the code explaining all important points.

1. we imported AES from Crypto.Cipher which make's solving AES quite simple.
2. it is the renewed rand() logic that we implemented in python.
3. the last line is important, AES has various modes such as ECB,CBC,CFB and so on, so how do we decide which one to use , to answer that we need to go back to our code:
   ```bash
    byte iv[16];
    memcpy(iv, aesIvSeed, 16);
   ```
After going through this doc on AESLib I found out that a CBC cipher object has a read-only attribute iv, holding the Initialization Vector (bytes).

Refference Link:
https://pycryptodome.readthedocs.io/en/latest/src/cipher/classic.html#cbc-mode

After running the above python script we will get the flag.



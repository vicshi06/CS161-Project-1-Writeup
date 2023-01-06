# Proj1 Writeup 

---
## Off-by-one
### Main Idea:
We used off-by-one to exploit this code. The for loop in the flip function iterates the buffer array 65 times, while the buffer array only has a size of 64. 
 
We first put our shellcode address inside of the buffer array, and because the buffer array is stored right below the SFP of invoke, we are able to change the least significant byte of the SFP of invoke to make it point to the start of the shellcode. When the invoke function ends, the ESP will point to the shellcode, and it will store the address of shellcode inside EIP. Now when invoke returns, it will execute the shellcode. 
 
### Magic Numbers:
We first determined the address of our shellcode (0xffffdfaa), which is stored as an environmental variable, and we found the address for buf (0xffffdbd0) and SFP of invoke (0xffffdc10). 

Because the address of SFP ends in 0xdc10, while the address of buf ends in 0xdbd0, we need to put enough garbage such that when we changed the least significant byte of the address of SFP to 0xdc04, our shellcode is in 0xdc08. Since the first four bytes would be treated as SFP, so we need to add 4 more bytes to make sure our shellcode address is stored inside EIP. (0xffffdc04 - 0xffffdbd0 + 4 = 56)

![image](https://user-images.githubusercontent.com/90171690/210914808-14b9b602-3e78-4b50-bb98-371e833fb546.png)
![image](https://user-images.githubusercontent.com/90171690/210914821-6b396a77-58d1-457d-bb22-ba7d8e0b6a74.png)

### Exploit Structure:
![image](https://user-images.githubusercontent.com/90171690/210914876-c4ef4dd1-454d-41e5-bf49-52326b0956d6.png)

 
The exploit has three parts: 
- Write 56 dummy characters to overwrite buf so that our shellcode is at 0xffffdc08.
- After we add the garage and the shellcode, we need to add 4 more characters to fill the 64 size array. 
- In the end, we need to change the least significant byte of invoke SFP to 0x04, and 0x24 ^ 0x20 = 0x04, so we need to add 0x24 in the end. 
 
### Exploit GDB Output:
When we ran GDB after inputting the malicious exploit string, we got the following output: 
![image](https://user-images.githubusercontent.com/90171690/210914899-3d816e5b-d4cb-414f-b70b-cb2698e6a615.png)


After 56 bytes of garbages, our shellcode is at 0xffffdc08, and the SFP of invoke successfully points to 0xffffdc04. When the function returns, it first pops off the first 4 bytes, then it stores the next 4 bytes into EIP, which is our shellcode address. 


---

## Check Before Read 
### Main Idea:
The main vulnerability for this question is that the file size check happens before the file is actually being read, which means we were able to bypass the size check by changing the input file after the check. 
 
For the attack, we were putting the shellcode inside the buf array and modifying the RIP of the read_file function so that it points to the address where the shellcode is at. 

### Magic Numbers:
We first determined the address of buf (0xffffdc08) and RIP (0xffffdc9c). In order to successfully overwrite the RIP of read_file, we need 148 bytes. (0xffffdc08 - 0xffffdc9c = 148). Our shellcode is 68 bytes, so we need 76 bytes of garbage and 4 bytes of address of shellcode to overwrite the EIP. 
 ![image](https://user-images.githubusercontent.com/90171690/210914070-e5481c38-dc8f-42b9-bfdc-265ec3b08efb.png)
 ![image](https://user-images.githubusercontent.com/90171690/210914179-34dd8262-32c5-48ad-86a0-7ffa7a000706.png)


### Exploit Structure:
![image](https://user-images.githubusercontent.com/90171690/210914202-2e97557b-3226-46a9-8602-2eae3edd171e.png)
 
The exploit has three parts: 
- Write 126 bytes of garbage to bypass the size check and to ensure the read function later will read sufficient bytes from the input file. 
- We then add our shellcode to the buf and add enough garbage so that we can modify the RIP.
- In the end, we need to change the RIP to the address of buf, which is 0xffffdc08. 
 
### Exploit GDB Output:
When we ran GDB after inputting the malicious exploit string, we got the following  output: 
![image](https://user-images.githubusercontent.com/90171690/210914230-ac067a9c-9185-4630-b406-e7531a2b0461.png)
![image](https://user-images.githubusercontent.com/90171690/210914247-9dbc9491-a391-470b-a975-f95e99786a9a.png)

Thre first 68 bytes are our shellcode, and the rest are garbage. In the end, we can see that the RIP is pointing to 0xffffdc08, which is the address of our shellcode. 

--- 

## Bypassing ASLR 
### Main Idea:
We used ret2ret to bypass ASLR. Even though we were not able to overwrite the RIP with an exact address, we changed the least significant bytes of an existing pointer to 0x00 and added a bunch of NOP to increase the chance of running our shellcode. 
 
### Magic Numbers:
Even though the address changes each time when we run the program, the relative distance between RIP of secure and buf array is the same. Here the buf address is 0xffe8af7c, and the RIP of secure address is 0xffe8b00c.(0xffe8afc - 0xffeb00c = 144) Our shellcode is 72 bytes, so we need to put 68 bytes of NOP first to make sure our shellcode has the most chance to be read. In the end, we need to add 4 bytes of garbage so we can modify the RIP into a return address we found from main, and we need to change the least significant byte of a pointer that is right above RIP of secure into 0x00. 
![image](https://user-images.githubusercontent.com/90171690/210915050-e3e73b0e-2703-4696-af13-3ffa328926c8.png)

In order to perform the ret2ret attack, we need to change our RIP of secure into a return address so that when the function returns, it will choose to put the pointer into RIP instead. 
![image](https://user-images.githubusercontent.com/90171690/210915084-e3e1ca0f-768b-433f-a7a0-27fda8b45e5b.png)
![image](https://user-images.githubusercontent.com/90171690/210915090-bff2052f-cf82-474d-b9c4-2189054d274a.png)

### Exploit Structure:
![image](https://user-images.githubusercontent.com/90171690/210915106-251183b6-f8f4-4b65-893a-9c78be9ee970.png)

 
The exploit has three parts: 
- First, we need to add 68 bytes of NOP to make sure that when the function returns, there is a higher chance our SHELLCODE will be read at full 
- We need to add 4 more bytes of garbage after shellcode and change the RIP to a return address we found
- We need to change the least significant bit of the pointer that is right above the RIP into 0x00, so it will point to the start of the buf. 
 
### Exploit GDB Output:
When we ran GDB after inputting the malicious exploit string, we got the following output: 
![image](https://user-images.githubusercontent.com/90171690/210915127-adc83656-4c56-4bcc-9b2c-9f98ba5f78e6.png)
![image](https://user-images.githubusercontent.com/90171690/210915134-c1f08ed9-3f06-4938-873f-50c99e47e21d.png)

After 68 bytes of NOP, we can see our shellcode in the buf. We can also see that our RIP has been changed to the return address we found. The least significant byte of the pointer that is right above the RIP is also 0x00. So our attack will most likely succeed! 


---
layout: post
title: Analyzing forth32's balong-usbdload software
---

#### What is *balong-usbdload*?

> I-quote na lang natin ang **Readme** nya from its [*github
> repository*](https://github.com/forth32/balong-usbdload):
>
> "Balong-usbdload is an emergency USB boot loader utility for Huawei LTE modems
> and routers with Balong V2R7, V7R11 and V7R22 chipsets.
> It loads external boot loader/firmware update tool file (usbloader.bin) via
> emergency serial port available if the firmware is corrupted or boot pin
> (test point) is shorted to the ground."

From the information above, yung trabaho lang nang *balong-usbdload* ay mag-load
nang *usbloader.bin* file. Ang flashing kase nang Huawei LTE modems and routers,
ay two-step process. Una, flash the *usbloader.bin*, which is trabaho nang
*balong-usbdload*, then second yung actual flashing na nang firmware.

*Note: Eto yung ginagamit natin sa pagflash for example nang B315s-936 na
router.*

Since open-source naman sya, let us analyze kung paano nya ba niloload yung
*usbloader.bin*. The question is, bakit naman natin iaanalyze? Ano ang mapapala
natin?

Una, sa mga gustong matuto nang programming, specifically C programming, makakatulong
naman sya kasi makikita natin kung pano yung approach nang programmer
(**forth32**) sa pagcreate nya nang program. Ikalawa, doon sa may idea na kung paano
mag create nang program using C, magkakaroon tayo nang additional idea kung pano ang
program flow nang naturang program. And at the same time, pwede tayong magcreate
nang sarili nating implementation based on that analysis. Yung full source-code
nang implementation can be seen in [my github
repo](https://github.com/zer0325/balong-malalim).

Note: The source code is in **Russian**, specifically yung mga comments lang naman
but yung actual code is in english. Heavily commented sya in **Russian** kaya
malaking tulong sa pagaanalyze natin nang program flow. Meron akong compilation
in English nang mga comments, which can also be found in the above repo.


## Let's start:

Mas maige if we have the source code open habang inaalyze natin sya. You can
use any text editor. In my case I use *vim*. Below is the screen shot of the
partial source of *balong-usbdload.c*.

![Screenshot of the partial source code of balong-usbdload.c](/assets/images/balong/partial_source.png)



Fist, note that the execution of any C program starts from the function
**main**. Note also that a C program can have multiple functions, but yung
pinaka-special  yung **main** function. So ang una nating gagawin is to open
the source and look for the **main** function. Below is the screenshot of the
**main** function.

![Screenshot of the partial source of the  main function](/assets/images/balong/partial_source_main.png)



Since alam na natin na ang trabaho nang *balong-usbdload* is to load the
*usbloader.bin* file, let's look for the code na may kinalaman sa procedure nang
pagloload nang *usbloader.bin* file. Below is the screenshot for the procedure
of loading the *usbloader.bin* file.

![Screenshot of the loading procedure](/assets/images/balong/loading_procedure.png)



Let us check yung lines `574, 595, and 606`, kung mapapansin nyo separate
`sendcmd` sya with different arguments, `cmdhead, cmddata, cmdeod`. So ibig
sabihin merong format yung data na sinisend. We can safely deduce na yung format
nang data packet is:

	 -------------------------------------------
	|  cmdhead  |      cmddata      |   cmdeod  |
	 -------------------------------------------

Yung `cmddata` is sent every `1024+5` bytes, yung `cmdhead` is `14` bytes at
yung `cmdhead` ay `5` bytes. And also, the  sending is done twice as seen in
`line 562`. So yung buong *usbloader.bin* file is divided into two blocks of
data, which means there is some kind of a preparation procedure. True enough,
if we scroll-up the source code makikita natin na there is some kind of
preparation procedure. Below is the image of the partial source of the
preparation procedure.

![Screenshot of the preparation procedure](/assets/images/balong/preparation_procedure.png)

So para mas madali natin maanalyze, let's divide the analysis into two:
*preparing the loader* and *sending the loader*.

### Preparing the loader

The procedure for preparing the loader starts at line 389. Lines 389-394 only
checks for the presence of the *usbloader.bin* file. Kung hindi sya present, mag
eexit lang ang program. Lines 396-400 checks if the file is a valid
*usbloader.bin* file by checking if the the first 4 bytes is equal to 0x20000.
Again if the file is not a valid "loader" file, the program will exit. Line 402
advances the file pointer to 36 bytes starting from the beginning of the file.
Let us focus our attention to the lines 406 - 407. Ang ginagawa nang dalawang
lines na yan is to get the next 32 bytes after the 36-bytes offset from the
beginning of the file. Each 16-byte actually contains information about each
block of data. The first 16-byte contains information about the *raminit* block
and the next 16-byte contains information about the *usbloader* block. Each
16-byte is saved in an array. Each block is a data structure as shown below.

![block structure](/assets/images/balong/block_struct.png)

Looking at the image above, both the *raminit* and *usbloader* block has the
following structure:

* First 4-byte is the boot mode
* Second 4-byte is the size
* Third 4-byte is the address
* Fourth 4-byte is the offset

Note na meron pang limang member yung data structure whose type is a *char
pointer*. This member will contain the address in memory of the actual block of
data. Lines 411 - 422 confirms that as shown in the image below.

![memory allocation](/assets/images/balong/malloc.png)

Using the image above as the reference, let us try to analyze each line
one-by-one. In line 414, it allocates a memory whose size is the defined by the
member **.size** for that block. After allocating the size, the lines 417-418,
it copies the block (*raminit* or *usbloader*), starting from the offset defined
by the **.offset** member to the memory address defined in **.pbuf** member. So,
eto na yung part na kinacopy na yung actual data sa memory. So once it is done,
ready na sya for loading. Note that this data is the original data, hindi pa sya
pinapatch kase the succeeding lines after line 427 is where the data patching is
done. For now, wag na muna natin intindihin ang patching, let us first create a
function that will prepare the loader file.

```
int prepare_loader(char *loader)
{
	int i;

	/* advance the file pointer 36-bytes from the beginning of the file */
	fseek(loader, 36, SEEK_SET);
	fread(&blk[0], 1, 16, loader);	/* read the first 16-bytes */
	fread(&blk[1], 1, 16, loader); 	/* read the next 16-bytes */

	for(i = 0; i < 2; i++){
		blk[0].pbuf = (char *)malloc(blk[i].size); 	/* allocates memory */

		/* advance the file pointer blk[i].offset bytes from the beginning of
		 * the file */
		fseek(loader, blk[i].offset, SEEK_SET);

		/* copy the data to the memory location defined in blk[i].pbuf member */
		fread(blk[i].pbuf, 1, blk[i].size, loader);
	}

	return 1;
}
```
Note na hindi na tayo nag check if it is a valid loader file. Trabaho na kasi
nang caller nang *prepare_loader()* function na to make sure na yung ipapasa
nyang file is a valid loader file.

Now that we have the function that will prepare the loader, analyze naman natin
yung procedure kung paano nya iloload yung loader sa device.


### Sending the loader

![Screenshot of the loading procedure](/assets/images/balong/loading_procedure.png)

As discussed previously the data packet has the following format:

	 -------------------------------------------
	|  cmdhead  |      cmddata      |   cmdeod  |
	 -------------------------------------------

So para mas madali nating mavisualize and maimplement nadin, lets name yung
`cmdhead` as the `HEADER_FRAME`, yung `cmddata` as `DATA_FRAME`, at yung
`cmdeod` as `END_FRAME`. Each `FRAME` has a format as shown in the image below.

![data packet format](/assets/images/balong/packet_info.png)


The `START_FRAME` has a format specified by the line 283 and lines 569 - 570. The
`START_FRAME` format is: `0xfe, 0x00, 0xff, lmode, size, addr, 0x00, 0x00`. Both
the size and the addr is 4 bytes and lmode is 1 byte. In lines 569 - 570 both
the size and addr are converted to host byte order using the *htonl(3)*
function.

The `DATA_FRAME` has a format specified by the line 284 and the lines 590-591.
The `DATA_FRAME` format is: `0xDA, packet_count, ~packet_count, DATA, 0x00, 0x00`. The
`~packet_count` is just the complement of `packet_count`. The `DATA_FRAME` is a
1024+5 byte data whose start address is defined in the **.pbuf** member of the
data structure. Note that the `DATA_FRAME` is sent every 1024+5`th` bytes.

The `END_FRAME` has a format specified by the line 285 and the lines 603 - 604.
The `END_FRAME` format is: `0xED, packet_count, ~packet_count, 0x00, 0x00`.

Now let's create a function that will implement the sending procedure.

``` c
int send_loader(int devfd)
{
	int packet_count, i, offset;
	unsigned char start_frame[14] = {0xfe, 0, 0xff};
	unsigned char data_frame[1040] = {0xda, 0, 0};
	unsigned char end_frame[5] = {0xed, 0, 0, 0, 0};

	for(i = 0; i < 2; i++){
		packet_count = 1;
		start_frame[3] = blk[i].lmode;

		/* convert both size and addr to host byte order */

		*((unsigned int*)&start_frame[4]) = htonl(blk[i].size);
		*((unsgined int*)&start_frame[8]) = htonl(blk[i].addr);

		/* send the start_frame packet */
		send_packet(start_frame, 14, devfd);

		/* format the data_frame packet */
		for(offset = 0; offset+datasize < blk[i].size; offset += 1024){
			data_frame[1] = packet_count;
			data_frame[2] = ~packet_count & 0xff;
			memcpy((void *)(data_frame + 3), (void *)(blk[i].pbuf + offset), datasize);

			/* send the data_frame packet */
			send_packet(data_frame, datasize + 5, devfd);
			packet_count++;
		}

		/* Do the remaining data_frame packet */
		data_frame[1] = packet_count;
		data_frame[2] = ~packet_count & 0xff;
		memcpy((void *)(data_frame + 3), (void *)(blk[i].pbuf + offset), blk[i].size - offset);

		/* send the data_frame packet */
		send_packet(data_frame, blk[i].size - offset + 5, devfd);
		packet_count++;

		/* Format the end_frame packet */
		end_frame[1] = packet_count;
		end_frame[2] = ~packet_count & 0xff;

		/* send the end_frame packet */
		send_packet(end_frame, 5, devfd);
	}

	return 1;
}
```

Kung mapapansin nyo, merong *send_packet()* function that we have included in
the code above. The *send_packet()* function that we use here, also follow the
same principle na ginawa sa original **balong-usbdload** code. Meron lang tayong
idinagdag na additional argument, yun yung *devfd*. So let us analyze the code
from the original **balong-usbdload.c** for the *send_packet()* function. Note
that sa original **balong-usbdload.c** code, ang name nang function is
*sendcmd*.

![Screenshot of the source of sendcmd function](/assets/images/balong/sendcmd.png)

Using the image above, yung lines 86 - 115 is the procedure nya for sending the
packet. Kung makikita natin, ilang lines of code lang sya. Pero ang gist dito is
compute the checksum of the data, send the data via the write(3) function, and
check the reply. The function returns 0 kung ang length nang reply nya is zero
or kung yung first byte nang reply buffer is not `0xAA`. It returns 1 if
successful yung sending of packet. Let's now implement the *send_packet()*
function.

``` c
int send_packet(unsigned char *buf, int len, int devfd)
{
	unsigned char replybuf[1024];	/* reply buffer */
	unsigned int replylen;

	checksum(buf, len);				/* compute the checksum */
	write(devfd, buf, len);			/* write to the device */
	tcdrain(devfd);
	replylen = read(devfd, replybuf, 1024);
	if(replylen = 0 || replybuf[0] != 0xAA)
		return 0;

	return 1;
}
```

Note that there is another function na included sa above code. Yun yung
*checksum()* function. Tingnan natin ulet yung source at ianalyze natin kung
paano nya kinocompute yung checksum. Gamitin natin as a reference yung screenshot
below.

![Screenshot of the source of the checksum function](/assets/images/balong/csum.png)

Yung *checksum()* function is from line 67 - line 84. Sa lines 76 - 80, makikita
natin na nireread nya yung every byte or char from the data up to but not
including the second to the last of the data. So isa-isahin natin. Yung
expression `csum << 4` means shift-left csum four times, which is equivalent to
saying that the `csum` is multiplied by 16. Yung `caret (^)` means an `XOR`
operation. So yung gist nya, is `multiply the current checksum to 16 then X0R
the product to a **CONSTANT**; Do the operation twice`. Now yung **CONSTANT** is
one of the element of the array cconst. The element is computed by performing an
`XOR` between the most significant 4-bits *(nibble)* ng current character na
na-read and yung most significant 4-bits nang current checksum. On the second
operation the element of the array is computed by `XOR`ing the least significant
4-bits nang current character and the most significant 4-bits ng current
checksum. After nya macompute ang checksum, sinave nya yung most significant
byte sa `buf[len-2]` and yung least significant byte sa `buf[len-1]`. Now, based sa
information na nakuha natin, we can implement the checksum.

``` c
int checksum(char *buffer, int len)
{
	unsigned int i, c, csum = 0;
	/* array of constants */
	unsigned int cconst[] = {
		0x0000, 0x1021, 0x2042, 0x3063,
		0x4084, 0x50A5, 0x60C6, 0x70E7,
		0x8108, 0x9129, 0xA14A, 0xB16B,
		0xC18C, 0xD1AD, 0xE1CE, 0xF1EF
	};

	for(i = 0; i < len - 2; i++){
		c = buf[i] & 0xFF;
		csum = ((csum << 4) & 0xFFFF) ^ cconst[(c >> 4) ^ (csum >> 12)];
		csum = ((csum << 4) & 0xFFFF) ^ cconst[(c & 0xF) ^ (csum >> 12)];
	}
	buffer[len - 2] = (csum >> 8) & 0xFF);
	buffer[len -1] = csum & 0xFF;
}
```

Kung mapapansin nyo merong mga `AND` (`&`) operation, that is for bit-masking. Ang
ibig lang sabihin nyan is gusto nya lang makuha yung number of bits na kailangan
mo sa isang value. For example, sa `((csum << 4) & 0xFFFF)`, we only need yung
least-significant 16-bits or 4-nibbles or 2-bytes. Don naman sa `c & 0xF`, we
only need yung least-significant 4-bits or 1-nibble. And don naman sa `csum &
0xFF`, we only need yung least-significant 8-bits or 1-byte.


Now na meron na tayong implementation nang both procedures, kailangan pa natin
nang isa pang procedure. Yun yung procedure nang pag-prepare nang port na
gagamitin para masend yung *usbloader.bin* sa device. Earlier sabi natin na
meron tayong additional argument para sa implementation nang *send_packet()*
function, yung yung **devfd**. Yung **devfd** is yung file descriptor nang port
na gagamitin nang program natin to communicate sa device. So, let us now analyze
yung source kung paano nya prinepare yung port.


### Preparing the port

![Partial source for the procedure in opening the port](/assets/images/balong/openport.png)

Using the image above as reference, the lines 117 - 193 is the procedure in
preparing or opening the port. Yung pinakaimportante lang naman dito is yung
line 140, which opens the device and assigns a file descriptor to the opened
device and the lines 143 - 150 which configures the device. Yung lines before
140 is just manipulation nang device name na eventually maglelead sa device name
na `/dev/ttyUSB0` for default or `/dev/ttyUSBx`, where `x` is a positive number.
So isa-isahin natin ang mga relevant lines para may idea tayo. Yung line 140
opens the device e.g. `/dev/ttyUSB0` and returns the file descriptor for that
device. Yung mga flags na `O_RDWR | O_NOCTTY | O_SYNC` means open the device in
read-write mode, do not make the device a controlling terminal, and synchronous
communication, respectively. Yung line 143 just zero-out the option or the
configuration. The lines 144 -149 just sets the configuration for the device.
Now, lets implement yung pag-prepare nang port and call the function
*open_port()*.

``` c
int open_port(char *devname)
{
	/* lets open the port */
	devfd = open(devname, O_RDWR | O_NOCTTY | O_SYNC);

	/* get the current configuration */
	tcgetattr(devfd, &options);

	/* zero-out the current configuration */
	bzero(&options, sizeof(options));

	/* set the options */
	options.c_cflag |= B115200 | CS8 | C_LOCAL | C_READ;
	options.c_lflag &= 0;
	options.c_iflag &= 0;
	options.c_oflag &= 0;
	options.c_cc[VTIME] = 30;
	options.c_cc[VMIN] = 0;

	/* save the options */
	tcsetattr(devfd, TCSANOW, &options);

	return devfd;

}
```

Remember when I say earlier doon sa *preparing the loader* procedure, na there
is another procedure wherein the *usbloader.bin* is being patched before it is
being sent to the device. Now we are going to analyze kung papano nya pinapatch
ang *usbloader.bin* file.

### Patching the loader

If you look inside the [source's
repository](https://github.com/forth32/balong-usbdload), there is a file named
*patcher.c*. This is the source file where the implementation for patching the
*usbloader.bin* file is located. So, let's open that file and try to analyze the
procedure for patching the *usbloader.bin* file. Below is the screenshot of the
partial source for the patching procedure.

![Partial source for the patching procedure](/assets/images/balong/patcher.png)

Let's concentrate on line 20. Kung mapapansin natin ang ginagawa nya lang is
look for the signature and then if found, perform a patch. Yung patching nya is
dalawang klase, `nop-patch` and `br-patch`. Depende sa value ng *ptype* argument
kung anong patch ang ipeperform nya. Pag `nop-patch` i-papatch nya lang yung
4-bytes starting from the value defined by the `fp.offset`. Pag `br-patch` naman
single-byte patch lang ang gagawin at that same location. Yung *patch()*
function will return the offset of the signature found or 0 if  the signature is
not found. So let's now implement the patching procedure.

``` c
uint32_t patch(struct defpatch fp, uint8_t *buf, uint32_t size, uint32_t ptype)
{
	/* Initialize nop data */
	const char nop[4] = {0, 0, 0xA0, 0xE3};
	uint32_t i;
	int8_t c;

	/* start looking after the 8th element */
	for(i = 8; i < size - 60; i += 4){
		/* Test if the signature is present */
		if(memcmp(buf + i, fp.sig, fp.sigsize) == 0){
			/* If the signature is present, perform patching based on the value
			 * of ptype */
			switch(ptype){
				case 0: /* perform a nop-patch */
					memcpy((buf + i + fp.sigsize + fp.poffset), nop, 4);
					return i;
				case 1:	/* perform br-patch */
					c = *(buf + i + fp.sigsize + fp.poffset);
					c |= 0xE0;
					*(buf + i + fp.sigsize + fp.poffset) = c;
					return i;
				default:
					exit(11);
			}
		}
	}
	return 0;	/* signature not found */
}
```

Yung data structure na defpatch is declared as shown below.

![defpatch data structure](/assets/images/balong/defpatch.png)

Yung signatures naman for the different chipsets is shown below as a screenshot.

![Signatures for different chipsets](/assets/images/balong/signatures.png)


Now the  next question here is kailan ba kino-call ang *patch()* function. If we
go back to the *balong-usbdload.c* source and use the image below as a
reference, the *patch()* function is being called by default as evident sa line
493.

![Screenshot for the bflag and cflag](/assets/images/balong/bflag_cflag.png)

Using the same image above, and looking at the lines 494 - 504, hindi naman sya
direct call sa *patch()* function, instead a function *(pv7r1() for example)* is
being called instead. Kung pupuntahan natin ang source definition nang function
na yun makikita natin na sya ang direct caller ng *patch()* function as shown in
the image below. The next question is paano nya nalalaman kung anong signature
ang gagamitin nya. Again looking at the lines 494 - 504, iniisa-isa nya ang
pag-call for each signature.

![Screenshot for the patch() caller](/assets/images/balong/patch_caller.png)

Now na alam na natin ang procedure, let's add that implementation sa
*prepare_loader()* function natin.

``` c
int prepare_loader(char *loader)
{
	int i;
	int cflag = 0;

	/* advance the file pointer 36-bytes from the beginning of the file */
	fseek(loader, 36, SEEK_SET);
	fread(&blk[0], 1, 16, loader);	/* read the first 16-bytes */
	fread(&blk[1], 1, 16, loader); 	/* read the next 16-bytes */

	for(i = 0; i < 2; i++){
		blk[0].pbuf = (char *)malloc(blk[i].size); 	/* allocates memory */

		/* advance the file pointer blk[i].offset bytes from the beginning of
		 * the file */
		fseek(loader, blk[i].offset, SEEK_SET);

		/* copy the data to the memory location defined in blk[i].pbuf member */
		fread(blk[i].pbuf, 1, blk[i].size, loader);
	}

	/* default behaviour */
	if(!cflag){
		/* try the signature for v7r1 chipset */
		res = pv7r1(blk[1].pbuf, blk[1].size);
		if(res == 0) /* Try the other, if fail */
			res = pv7r2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r11(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_3(blk[1].pbuf, blk[1].size);
		if(res != 0)
			printf("Patch applied on offset 0x%08x.\n", blk[1].offset + res);
		else{
			printf("Patch signature not found. Use -c to boot without patching.\n");
			return 0;
		}
	}
	return 1;
}
```

If you look at line 97 using the image below, there is another function that
calls the *patch()* function, that is *perasebad()* function. Yung *perasebad()*
function is called if the `bflag` is set or the `-b` argument is included sa
pag-run nang **balong-usbdload**. Kung itatranslate natin sa English yung line
484 using [Google translate](https://translate.google.com), magiging malinaw
kung para saan ang procedure if `bflag` is set. As shown in the image below, it
is a patch erase procedure.

![screenshot for the bflag translation](/assets/images/balong/bflag_translate.png)

So update natin yung *prepare_loader()* function natin and add natin yung
procedure if `bflag` is set.

``` c
int prepare_loader(char *loader)
{
	int i;
	int cflag = 0,  bflag = 0;

	/* advance the file pointer 36-bytes from the beginning of the file */
	fseek(loader, 36, SEEK_SET);
	fread(&blk[0], 1, 16, loader);	/* read the first 16-bytes */
	fread(&blk[1], 1, 16, loader); 	/* read the next 16-bytes */

	for(i = 0; i < 2; i++){
		blk[0].pbuf = (char *)malloc(blk[i].size); 	/* allocates memory */

		/* advance the file pointer blk[i].offset bytes from the beginning of
		 * the file */
		fseek(loader, blk[i].offset, SEEK_SET);

		/* copy the data to the memory location defined in blk[i].pbuf member */
		fread(blk[i].pbuf, 1, blk[i].size, loader);
	}
	if(bflag){
		res = perasebad(blk[1].pbuf, blk[1].size);
		if(res == 0){
			printf("Bad block signature not found. Loading not possible.\n");
			return 0;
		}
	}

	/* default behaviour */
	if(!cflag){
		/* try the signature for v7r1 chipset */
		res = pv7r1(blk[1].pbuf, blk[1].size);
		if(res == 0) /* Try the other, if fail */
			res = pv7r2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r11(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_3(blk[1].pbuf, blk[1].size);
		if(res != 0)
			printf("Patch applied on offset 0x%08x.\n", blk[1].offset + res);
		else{
			printf("Patch signature not found. Use -c to boot without patching.\n");
			return 0;
		}
	}
	return 1;
}
```

Bukod sa *patch()* function na dinescribe natin above, meron pang isang patching
procedure na hindi iniemploy yung *patch()* function. Yun yung fastboot patch.
This procedure is done if the `fbflag` is set. So ang ginagawa nya is 1-byte
patch lang sa *usbloader.bin* file, and then it will put the device in
**fastboot mode**. Lets analyze the source kung papaano nya pinapatch ang
*usbloader.bin* file using the image below as a reference.

![screenshot for the fbflag](/assets/images/balong/fbflag.png)


At line 428, kino-call nya ang *locate_kernel()* function. Ang ginagawa lang
nang *locate_kernel()* function is to find the offset or memory location nang
start nang string **ANDROID!**. Then nagpeperform sya nang 1-byte signature
patch by replacing the character **A** with the hex value `0x55`. It also
changes the size of the *usbloader.bin* file to `offset + 8`. Yung definition
nang *locate_kernel()* function is show in the image below.

![Screenshot of the locate kernel function](/assets/images/balong/locate_kernel.png)

Now, let's update our *prepare_loader()* function to include yung procedure if
`fbflag` is set.

``` c
int locate_kernel(char *buf, uint32_t size);

int prepare_loader(char *loader)
{
	int i;
	int cflag = 0,  bflag = 0, int fbflag = 0;
	int koffset;

	/* advance the file pointer 36-bytes from the beginning of the file */
	fseek(loader, 36, SEEK_SET);
	fread(&blk[0], 1, 16, loader);	/* read the first 16-bytes */
	fread(&blk[1], 1, 16, loader); 	/* read the next 16-bytes */

	for(i = 0; i < 2; i++){
		blk[0].pbuf = (char *)malloc(blk[i].size); 	/* allocates memory */

		/* advance the file pointer blk[i].offset bytes from the beginning of
		 * the file */
		fseek(loader, blk[i].offset, SEEK_SET);

		/* copy the data to the memory location defined in blk[i].pbuf member */
		fread(blk[i].pbuf, 1, blk[i].size, loader);
	}

	/* Fastboot patch */
	if(fbflag){
		koffset = locate_kernel(blk[1].pbuf, blk[1].size);
		if(koffset != 0){
			blk[1].pbuf[koffset] = 0xAA;	/* signature patch */
			blk[1].size = koffset + 8;		/* trim the size of the loader */
		} else{
			printf("The bootloader does not have an ANDROID component, ");
			printf("Fastboot loading is not possible.\n");
			return 0;
		}
	}

	/* Note that if bflag is set, fbflag is also set */
	if(bflag){
		res = perasebad(blk[1].pbuf, blk[1].size);
		if(res == 0){
			printf("Bad block signature not found. Loading not possible.\n");
			return 0;
		}
	}

	/* default behaviour */
	if(!cflag){
		/* try the signature for v7r1 chipset */
		res = pv7r1(blk[1].pbuf, blk[1].size);
		if(res == 0) /* Try the other, if fail */
			res = pv7r2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r11(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_2(blk[1].pbuf, blk[1].size);
		if(res == 0)
			res = pv7r22_3(blk[1].pbuf, blk[1].size);
		if(res != 0)
			printf("Patch applied on offset 0x%08x.\n", blk[1].offset + res);
		else{
			printf("Patch signature not found. Use -c to boot without patching.\n");
			return 0;
		}
	}
	return 1;
}

int locate_kernel(char *buf, uint32_t size)
{
	int koff = 0;

	/* start looking from the end of the file */
	for(koff = (size - 8); koff > 0; koff--){
		if(strncmp(buf + koff, "ANDROID!", 8) == 0)
			return koff;
	}
	return 0;
}
```

There are other options na available sa original source nya, yung `-m` and `-t`
options. Yung `-m` option is used to print the partition table na included doon
sa *usbloader.bin* file, specifically doon sa *bootloader block*. Yung `-t`
option naman is used to replace the partition table na nasa *usbloader.bin* file
with an external partition table file. Hindi na natin sya iinclude sa analysis
since wala nanam silang direct effect sa loading nang *usbloader.bin* file. And
also, most of the time hindi naman sya ginagamit sa pag-flash nang device.


If you notice, hindi na natin isinama sa analysis yung part nang source na
specific sa **Windows** like doon sa image below.

![Screenshot of the partial source of the windows part of the
source](/assets/images/balong/windows_part.png)


Originally kase dinivelop ni *forth32* yung software for **linux** then
eventually nagdagdag is *rust3028*
nang code for the **Windows** port. Ginagamit ko kasi is **FreeBSD** and
**Linux**, and yung implementation is **FreeBSD**. But the source compiles both
on **FreeBSD** and **Linux**. Yung logic sa preparation, loading, and yung
patching is the same lang din naman sa **Windows**, nagkakaiba lang naman kung
paano ang port handling sa **Windows**. Since port handling lang naman sila
nagkakaiba, yung modification na gagawin para sa **Windows** port eh dun lang sa
*open_port()* function. This is the reason kaya natin dinivide sa tatlo yung
implementation natin, para yung code modification will be done separately. Yun
nga palang full implementation or source code can be seen or downloaded from
[this](https://github.com/zer0325/balong-malalim) repository. Although yung
source is still a *WIP* it accomplishes the thing that it is supposed to do.
<del>Yung mga *printf* statements sa main function are for debugging purposes which
will be modified later on</del>.

Hanggang dito na lang po, I hope nakapagbigay kahit papaano nang kaunting kaalaman
etong munti kong ambag. Maraming Salamat po.


# REFERENCES

[https://github.com/forth32/balong-usbdload](https://github.com/forth32/balong-usbdload)
[Serial Programming for POSIX Systems](https://www.cmrr.umn.edu/~strupp/serial.html)

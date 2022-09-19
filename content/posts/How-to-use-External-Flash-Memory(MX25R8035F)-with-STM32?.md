---
title: "How to Use External Flash Memory(MX25R8035F) with Stm32?"
date: 2022-09-19T23:58:48+03:00
---

In this article, I will give informations about the 'How to use external flash memory with Nucleo-f303re development board'.
In order to strengthen the narrative, I will proceed through the following topics.

**1) What is Flash Memory?**

**2) What is the internal structure of Flash Memory?** 

**3) How do you communicate with the MX25R8035F Flash Memory?**

**4) How to use MX25R8035F?**

---

**1) What is Flash Memory?**

Flash Memory is a type of memory that does not lose its information even in a power cut and can be written and erased repeatedly. It is included in the development boards as an internal peripheral. If the memory space available here is insufficient, we have to use an external memory. We can solve this requirement with flash memory units.

---

**2) What is the internal structure of Flash Memory?**

The data we send is stored in block structures. There are tens of pages (depending on the amount of memory) in each block structure.

![Internal Structure](/how-to-use-flash-mem/internal-structure.png 'Internal Structure')

Every data is stored in the page structures inside. To send data to page structures, that structure must be cleared beforehand. If the page we want to write data on is not cleaned, we probably cannot write the data or write it in a corrupted way. In order to avoid such problems, we must make sure that these pages are clean before storing data.

There is one point I would like to mention here. Each flash memory has a certain number of write-read lifetimes. If we clean before each write, we will cut the life of the peripheral in half. This poses a serious problem. The way to be followed in this regard, if we want to send data, our priority is always empty pages. Instead of cleaning the full page and writing the data, we should use the clean area.

Things to consider when choosing a Flash Memory are:

* Read/Write Speed
* Read/Write Life
* Communication protocol type

---

**3) How do you communicate with the MX25R8035F Flash Memory?**

MX25R8035F Flash Memory uses SPI as communication protocol. Therefore, you should have a good command of SPI. I'm just going to talk about how to use the SPI protocol.

While answering this question, let's first examine the pins of the peripheral.

![Flash Memory PinOut](/how-to-use-flash-mem/pinOut.png 'Flash Memory PinOut')

We will use pins 1,2,4,5,6 and 8 from the above pins in order to communicate with SPI by running our peripheral.

Pins 4 and 8 will be used for feeds. The remaining pins will be used for the SPI connection.

* CS# : Channel selection
* SO: SPI ->MISO
* SI: SPI -> MOSI
* SCLK: SPI ->SCLK

In order for our peripheral to wake up and become active, the CS# pin is pulled to a low (logic 0).

To use SPI, we perform the configuration settings via STM32CubeIde as follows (You can revise the SPI speed according to the limit in the user guide):

![IDE Configuration](/how-to-use-flash-mem/Stm32CubdeIde-config.png 'IDE Configuration')

The following commands written with the HAL library are used to send and read data:

{{< highlight go "linenos=table,linenostart=1" >}}
HAL_SPI_Transmit(&hspi1, &spi_tx_buf[0], 1, 500);
HAL_SPI_Receive(&hspi1, &spi_rx_buf[0], 1, 500);
{{< / highlight >}}

---
**4) How to use MX25R8035F?**

In order to control the MX25R8035F Flash Memory external peripheral, I will use a library that I have supplied and then revised and brought into working condition, and the user guide published by the manufacturer. I will share example codes in the article. First of all, I will try to examine each of its structures piece by piece.

Please find manufacturer guide !!!

Let's start by reviewing our library.

First we create some struct structures and variables. As we explain the functions in the library, we will understand why we defined the variables in the structures here.

{{< highlight go "linenos=table,linenostart=1" >}}
# define RxBufferSize 260
typedef struct {
	unsigned char Status;
	unsigned short Sector;
	unsigned char Page;
} ExtFlash_t;

typedef struct {
	unsigned char WIP; //write in progress
	unsigned char WEL; //Write Enable Latch
	unsigned char BPbits; //block protect
	unsigned char QE; //quad enable bit
	unsigned char SRWD; //status register write disable bit
} StatusRegister_t;

typedef struct {
	unsigned char SOTP;
	unsigned char LDSO;
	unsigned char PSB;
	unsigned char ESB;
	unsigned char PFAIL;
	unsigned char EFAIL;
} SecurityRegister_t;

typedef struct {

	unsigned int block[16][16];
	unsigned char tx_buf[260];
	uint8_t page_counter;
	uint8_t block_counter;

} Mem_Handle_t;

unsigned char spi_rx_buf[RxBufferSize];
unsigned char spi_tx_buf[RxBufferSize];
{{< / highlight >}}

Now we need to learn the addresses of our writable memory points. For this, we go to the page below in the user guide.

![Memory Map](/how-to-use-flash-mem/memory-map.png 'Memory Map')

Every structure referred to here as 'sector' is the page structure we just learned. There are 256 different writable addresses. In this model, 256bit data can be written to each address.

There is something I want to mention here. We said that each page structure has 256-bit space, but in the previous section, we also said that the page must be clean for every typing operation. Suppose we have piecemeal data of less than 256 bits. For example, in my project I have temperature and humidity data. There will be different small and large data in the projects that you will use. So, how should we write this data into memory with an algorithm? The answer to this question is as simple as you can imagine. First of all, we need to accumulate the data we have until it is 256 bits. Then we have to send it to a page structure we want. Otherwise, if we send each data separately to page structures, we will use our memory very inefficiently and incorrectly.

Let's get back to our topic. We have 256 different addresses that I can use. We will define them in our library in a different way than we know them. If we try to define each address with a different macro, this will take too long and take up too much memory. First we define the base addresses of each block structure.



{{< highlight go "linenos=table,linenostart=1" >}}
#define mem_offset			0x001000
#define block0_add0			0x000000
#define block1_add0			0x010000
#define block2_add0			0x020000
#define block3_add0			0x030000
#define block4_add0			0x040000
#define block5_add0			0x050000
#define block6_add0			0x060000
#define block7_add0			0x070000
#define block8_add0			0x080000
#define block9_add0			0x090000
#define block10_add0		0x0A0000
#define block11_add0		0x0B0000
#define block12_add0		0x0C0000
#define block13_add0		0x0D0000
#define block14_add0		0x0E0000
#define block15_add0		0x0F0000
{{< / highlight >}}


If we pay attention, a new usable address is obtained by adding a certain offset to the basic addresses that I have defined here. We have already defined this offset value above. Now let's write a function using this offset value. Let's create the remaining memory points with the help of this function (Ignore the remaining structures inside the function for now, I will mention them in my next explanations).

{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char FlashInit(Mem_Handle_t* Mem_Control)
{
 	  EXTFLASHCSPIN_HIGH;

 	 	uint8_t k ;
 	 	for(k=0;k<=16;k++)
 	 	{
 	 		Mem_Control->block[0][k]  = ((unsigned int)block0_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[1][k]  = ((unsigned int)block1_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[2][k]  = ((unsigned int)block2_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[3][k]  = ((unsigned int)block3_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[4][k]  = ((unsigned int)block4_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[5][k]  = ((unsigned int)block5_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[6][k]  = ((unsigned int)block6_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[7][k]  = ((unsigned int)block7_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[8][k]  = ((unsigned int)block8_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[9][k]  = ((unsigned int)block9_add0)+((unsigned int)mem_offset*k)  ;
 	 		Mem_Control->block[10][k] = ((unsigned int)block10_add0)+((unsigned int)mem_offset*k) ;
 	 		Mem_Control->block[11][k] = ((unsigned int)block11_add0)+((unsigned int)mem_offset*k) ;
 	 		Mem_Control->block[12][k] = ((unsigned int)block12_add0)+((unsigned int)mem_offset*k) ;
 	 		Mem_Control->block[13][k] = ((unsigned int)block13_add0)+((unsigned int)mem_offset*k) ;
 	 		Mem_Control->block[14][k] = ((unsigned int)block14_add0)+((unsigned int)mem_offset*k) ;
 	 		Mem_Control->block[15][k] = ((unsigned int)block15_add0)+((unsigned int)mem_offset*k) ;

 	 	}
 	 	Mem_Control->page_counter =  0 ;
 	 	Mem_Control->block_counter = 0 ;
 	 	memset(Mem_Control->tx_buf,0,260);   // clear send buffer
    return 1;
}

{{< / highlight >}}

We managed to import all our memory points into our software.

Now let's examine the function that sends a simple command to our peripheral. But before that, let's learn the commands we have by examining the command set in the user guide.


![Cmd Set](/how-to-use-flash-mem/cmd-seet.png 'Cmd Set')

Let's examine our command sending function in the library. The CS# pin is pulled low, then data is sent using SPI. Then the CS# pin is pulled to high level (logic1) and the peripheral is turned off.

{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char WriteCommand(unsigned char command,unsigned short size)
{
    unsigned char transferOK;
    spi_tx_buf[0]=command;


    EXTFLASHCSPIN_LOW;
    HAL_Delay(100);
    //transferOK = SPI_transfer(spi_handle, &spiTransaction);
	  transferOK=HAL_SPI_TransmitReceive(&hspi1,&spi_tx_buf[0],&spi_rx_buf[0],size,500);
	  HAL_Delay(100);
	EXTFLASHCSPIN_HIGH;
    return transferOK;

}
{{< / highlight >}}


Let's examine the status register reading function in the library. Here, using the command sending function above, the relevant command (see the command set) is sent to the peripheral. Then, the received data was extracted by shifting according to the bit rules found in the user guide.

{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char ReadStatusRegister(StatusRegister_t* Status)
{
    unsigned char transferOK;
    memset(spi_rx_buf,0,RxBufferSize);
    transferOK=WriteCommand(0x05,2);
    if(transferOK==HAL_OK)
    {
        Status->BPbits=(spi_rx_buf[1]>>2)&0xFF;
        Status->QE=(spi_rx_buf[1]>>6)&0x01;
        Status->SRWD=(spi_rx_buf[1]>>7)&0x01;
        Status->WEL=(spi_rx_buf[1]>>1)&0x01;
        Status->WIP=spi_rx_buf[1]&0x01;
    }
    return transferOK;

}
{{< / highlight >}}


Let's examine the security register reading function in the library. It has the same working logic as the above function. The only difference is that the relevant commands we send differ. You should review these commands one by one.



{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char ReadSecurityRegister(SecurityRegister_t* Status)
{
    unsigned char transferOK;
    memset(spi_rx_buf,0,RxBufferSize);
    transferOK=WriteCommand(0x2B,2);
    if(transferOK==HAL_OK)
    {
        Status->EFAIL=(spi_rx_buf[1]>>6)&0x01;
        Status->PFAIL=(spi_rx_buf[1]>>5)&0x01;
        Status->ESB=(spi_rx_buf[1]>>3)&0x01;
        Status->PSB=(spi_rx_buf[1]>>2)&0x01;
        Status->LDSO=(spi_rx_buf[1]>>1)&0x01;
        Status->SOTP=spi_rx_buf[1]&0x01;
    }
    return transferOK;
}
{{< / highlight >}}


Let's examine the Sector Erase function in the library. This function is one of the most important functions for us. This is because we clean the relevant field using this function before writing data (I mentioned the importance of this in the first chapters). I did not understand every point of the function found here, but I understood the basic points and I will try to explain these points.

First, the WREN command in the instruction set is sent. To clear a page and send data, the WREN command must be sent before these operations. A special function is defined in the library to send this command (I will add it below). The units that store the sender and receiver data are then cleared. Then the address we are interested in (the address that will be cleaned and made suitable for writing) is loaded into the sending unit (tx buffer). Then the Sector Erase command is loaded into the sending unit (tx buffer). The uploaded address and the command are sent together to the peripheral unit with the command sending function I explained above.

{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char SectorErase(unsigned int Add)
{
    StatusRegister_t FlashStatus;
    SecurityRegister_t SecurityReg;
    unsigned char i=0;
    unsigned char transferOK;
    memset(spi_rx_buf,0,RxBufferSize);	 // alıcı birim temizlenir 
    transferOK=WREN();			 // wren komutu gönderilir 
    FlashStatus.WEL=0;
    while(FlashStatus.WEL==0)
    {
        HAL_Delay(20);
        transferOK=ReadStatusRegister(&FlashStatus);
        i++;
        if(i==200)
        {
            return 0;
        }
    }
    i=0;
    spi_tx_buf[1]=(Add>>16)&0xFF;	// adres , gönderme birimine (buffer) alınır
    spi_tx_buf[2]=(Add>>8)&0xFF;
    spi_tx_buf[3]=Add&0xFF;

    transferOK=WriteCommand(0x20,4);	// sector erase komutu gönderme birimine alınarak adres ile birlikte flash'a gönderilir.
    FlashStatus.WIP=1;
    while(FlashStatus.WIP==1)
    {
        HAL_Delay(20);
        transferOK=ReadStatusRegister(&FlashStatus);
        i++;
        if(i==200)
        {
            return 0;
        }
    }
    i=0;
    transferOK=ReadStatusRegister(&FlashStatus);
    transferOK=ReadSecurityRegister(&SecurityReg);
    if(transferOK==HAL_OK && SecurityReg.EFAIL==0 && SecurityReg.PFAIL==0)
    {
        return 1;
    }
    else
        return 0;
}
{{< / highlight >}}

Let's examine the function used to send data in the library. I will describe the structure here as best as I can, only in outline.

First, the WREN command is sent and the peripheral is ready for writing. Then the send and receive units (tx rx buffer) are cleared. Then the send command is received to the sending unit (see command set). Then, the relevant address is received to the sending unit. Then the information we will send is taken to the sending unit. Then the CS# pin is pulled low. Then the whole sending unit of 260 bits is sent. Finally, the CS# pin is pulled high and the peripheral is turned off.



{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char PageProgram(unsigned Add , unsigned char* buf){
	StatusRegister_t FlashStatus;
	SecurityRegister_t SecurityReg;
	uint32_t i ;
	unsigned char transferOK;
	WREN();		                        // WREN komutu gönderildi
	memset(spi_rx_buf,0,RxBufferSize); // sıfır ile dolduruldu rx buffer
	memset(spi_tx_buf,0,RxBufferSize); // sıfır ile dolduruldu rx buffer

	spi_tx_buf[0]= 0x02;                // yazma komutu gönderme birimine (tx buffer) alındı 
	spi_tx_buf[1]=(Add>>16)&0xFF;       // adres , gönderme birimine alındı
    spi_tx_buf[2]=(Add>>8)&0xFF;
    spi_tx_buf[3]=Add&0xFF;

    for(i=4;i<257;i++)              // bilgi gönderme birimine alındı 
    {
    	spi_tx_buf[i] = *buf;
    	buf++;
    }

    EXTFLASHCSPIN_LOW; 																// cs select
    transferOK = HAL_SPI_Transmit(&hspi1, &spi_tx_buf[0], 260, HAL_MAX_DELAY);		//256 bitlik bütün birim gönderildi 
	EXTFLASHCSPIN_HIGH; 															// cs deselect

	FlashStatus.WIP=1;
    while(FlashStatus.WIP==1)
    {
    	HAL_Delay(20);
    	transferOK=ReadStatusRegister(&FlashStatus);
    	i++;
    	if(i==200)
    	{
    		return 0;
    	}
    }
    i=0;
    transferOK=ReadStatusRegister(&FlashStatus);
    transferOK=ReadSecurityRegister(&SecurityReg);
    if(transferOK==HAL_OK && SecurityReg.EFAIL==0 && SecurityReg.PFAIL==0)
    {

    	return 1;
    }
    else
        return 0;
{{< / highlight >}}

Let's examine the function written to read a page in the library. First, the receiving unit (tx buffer) is cleared. Then the read command and the corresponding address are received to the sending unit. Then the peripheral is activated by pulling the CS# pin to low level. Then the sending unit (tx buffer) is sent. Then, the data is expected and received with the 'receive' command for the next data. Finally, the CS# pin is pulled high and the peripheral is turned off.


{{< highlight go "linenos=table,linenostart=1" >}}
unsigned char CheckPage(unsigned Add){
	unsigned char transferOK ;

	memset(spi_rx_buf,0,RxBufferSize); //rx buferı 0 ile dolduruldu
	spi_tx_buf[0]=0x03;		  // okuma komutu , gönderme birimine alınır 
	spi_tx_buf[1]=(Add>>16)&0xFF;	  // ilgili adres , gönderme birimine alınır 
    spi_tx_buf[2]=(Add>>8)&0xFF;
    spi_tx_buf[3]=Add&0xFF;

    EXTFLASHCSPIN_LOW; 									// cs select

    transferOK = HAL_SPI_Transmit(&hspi1, &spi_tx_buf[0], 4, 500);	//komut ve adres gönderilir 
    transferOK = HAL_SPI_Receive(&hspi1, &spi_rx_buf[0], 260, 500);	//veri okunur
	EXTFLASHCSPIN_HIGH; // cs deselect
	return transferOK;
}
{{< / highlight >}}


Resources I used while writing my article:

* Manufacturer user guideline

* https://flashdba.com/2014/06/20/understanding-flash-blocks-pages-and-program-erases/

* http://donanimara.blogcu.com/flash-memory-flash-bellek-nedir-cesitleri/3007570


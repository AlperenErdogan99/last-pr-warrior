---
title: "How to use Wiznet5500 with Stm32"
date: 2022-09-20T14:03:48+03:00
---

In this article, I will explain how to use wiznet5500 with Stm32. 

**What is Wiznet module?**

WIZnet (Wizard of Internet) is a unique Hardwired Internet Connectivity Solution Provider. WIZnet provides IOcP and HW TCP/IP chips, best fitted for low-end Non-OS devices connecting to the Ethernet for the internet of things. WIZnet's core technology is “Hardwired TCP/IP“.

Let's list the reasons why it is useful.

* Supporting SPI.
* Providing a Useful API to its users. (Socket API)
* Availability of an up-to-date forum page and providing fast support through this page.
* The documents published for users are detailed and understandable.

You can access Wiznet documentation from this link: http://wizwiki.net/wiki/doku.php?id=products:w5500:datasheet

---

In this article, I will run my module in tcp client mode. Let's do this step by step. First, let's learn how to use the pins of the module that we will use from the manufacturer's source that I have added above.

![Pin Out](/how-to-use-wiznet/pinOut.png 'Pin Out')

Let's start programming the module. First, we add the library published by the manufacturer to our project. Then we determine the IP address of our module.

{{< highlight go "linenos=table,linenostart=1" >}}
wizchip_init(bufSize, bufSize);
 wiz_NetInfo netInfo = { .mac 	= {0x00, 0x08, 0xdc, 0xab, 0xcd, 0xef},	// Mac address
                          .ip 	= {192, 168, 2, 192},					// IP address
                          .sn 	= {255, 255, 255, 0},					// Subnet mask
                          .gw 	= {192, 168, 2, 1}};					// Gateway address
 wizchip_setnetinfo(&netInfo);
 wizchip_getnetinfo(&netInfo);
{{< / highlight >}}


The number of sockets I can use in our module varies according to the model. There are 8 sockets in the W5500 model. We define them.

{{< highlight go "linenos=table,linenostart=1" >}}
bufSize[] = {2, 2, 2, 2, 2, 2, 2, 2};
{{< / highlight >}}


We activate the SPI communication on our development board. I am implementing it via STM32CubeIde.
![Ide Config](/how-to-use-wiznet/IdeConfig.png 'Ide Config')

In order to control the manufacturer's library with the SPI interface that I have activated, we define two functions that can read and write 1 byte, working through the SPI communication protocol, to the relevant functions in the library.

{{< highlight go "linenos=table,linenostart=1" >}}
void cs_sel() {
	HAL_GPIO_WritePin(GPIOB, GPIO_PIN_12, GPIO_PIN_RESET); //CS LOW
}
 
void cs_desel() {
	HAL_GPIO_WritePin(GPIOB, GPIO_PIN_12, GPIO_PIN_SET); //CS HIGH
}
 
uint8_t spi_rb(void) {
	uint8_t rbuf;
	HAL_SPI_Receive(&hspi2, &rbuf, 1, 0xFFFFFFFF);
	return rbuf;
}
 
void spi_wb(uint8_t b) {
	HAL_SPI_Transmit(&hspi2, &b, 1, 0xFFFFFFFF);
}
{{< / highlight >}}


The library we use is not only a library published for the w5500 model. Therefore, we go into our library and select the corresponding macro value.

{{< highlight go "linenos=table,linenostart=1" >}}
#define _WIZCHIP_            5500   // 5100, 5200, 5300, 5500
{{< / highlight >}}

We are setting the timeout.

{{< highlight go "linenos=table,linenostart=1" >}}
void timeout_config(void)
{
	wiz_NetTimeout gWIZNETTIME = {.retry_cnt = 3,       		    //RCR = 3
	      	                        .time_100us = 2000};     		//RTR = 2000

	ctlnetwork(CN_SET_TIMEOUT,(void*)&gWIZNETTIME); 				//set timeout w5500
  ctlnetwork(CN_GET_TIMEOUT,(void*)&gWIZNETTIME); 				//set timeout w5500
}
{{< / highlight >}}

We create a struct structure to control the functions we will write.

{{< highlight go "linenos=table,linenostart=1" >}}
typedef struct
{
	uint8_t  connect_cnt ;
	uint8_t  socket_num  ;					/*!<counter for socket()		*/
	uint8_t  close_cnt   ;					/*!<counter for close()			*/
	uint8_t  socket_cnt  ;					/*!<counter for socket port		*/
	uint16_t server_port ;					/*!<counter for server port		*/
	uint16_t socket_port ;					/*!<counter for socket number	*/
	uint8_t  rbuf        ;					/*!<counter for connect()		*/
} W5500_Handle_t;
{{< / highlight >}}

We have made the necessary initial adjustments. Now we are writing a connection initiation function using the library we added from the manufacturer's source to connect to the relevant server by running our module in client mode. In this function, we determine the port number we are interested in, the socket number and the IP address we want to connect to.

{{< highlight go "linenos=table,linenostart=1" >}}
void connect_server(W5500_Handle_t *w5500_ports)
{


	w5500_ports->close_cnt = 10 ;
	w5500_ports->connect_cnt = 0 ;
	w5500_ports->server_port = 5656 ;
	w5500_ports->socket_cnt = 10 ;
	w5500_ports->socket_num = 0 ;
	w5500_ports->socket_port = 5656 ;

	uint8_t server_ip[4] = {192,168,1,2} ; 							// Server's IP Address


	if((w5500_ports->socket_cnt = socket(w5500_ports->socket_num,Sn_MR_TCP,w5500_ports->socket_port,SF_TCP_NODELAY))==w5500_ports->socket_num)
		{
			HAL_Delay(250);
			while(w5500_ports->connect_cnt != SOCK_OK ) 								// When Return == SOCK_OK
				{

					w5500_ports->connect_cnt = connect((uint8_t )w5500_ports->socket_num,(uint8_t *)server_ip,(uint16_t )w5500_ports->server_port); // Connect to server
					HAL_Delay(250);
				}
		}



}
{{< / highlight >}}

There is already a send() function in the producer library to send data to the server we are connected to. You can use it easily by examining it.

Let's write a disconnection function using the generator library to log out from the server we are connected to.


{{< highlight go "linenos=table,linenostart=1" >}}
void close_server(W5500_Handle_t *w5500_ports)
{
	w5500_ports->close_cnt  = 0;
	w5500_ports->socket_num = 0;


	while(SOCK_OK != w5500_ports->close_cnt)
		{
			w5500_ports->close_cnt = close(w5500_ports->socket_num); 							//close to server
		}

	HAL_Delay(2500);
}
{{< / highlight >}}

Resources that I have used :
* https://www.carminenoviello.com/2015/08/28/adding-ethernet-connectivity-stm32-nucleo/
* https://forum.wiznet.io/
* https://www.wiznet.io/product-item/w5500/
* https://wizwiki.net/wiki/doku.php/products:w5500:application:tcp_function


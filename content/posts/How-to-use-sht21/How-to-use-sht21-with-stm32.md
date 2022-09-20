---
title: "How to Use Sht21 With Stm32"
date: 2022-09-20T14:29:44+03:00
author: "Last Pr Warrior"
---

In this article, I will try to explain in an instructive way how I control the SHT21 sensor in the project I have realized. Then I will finish my article with a sample temperature measurement code. I performed this check with the Nucleo-f303re development board I have.

<!--more-->

The SHT21 sensor is designed to measure temperature and humidity. It has two modes, Hold Master and No Hold Master, for measuring. I will use it in Hold Master mode. It uses the I2C communication protocol for data exchange. For more detailed information, you can review the manufacturer's source that I will add below.

* https://www.farnell.com/datasheets/1780639.pdf

First of all, I examine how I should use pins.

![Pin Out](/how-to-use-sht21/pinout.png 'Pin Out')

After these examinations, I examine the command table and image below to learn how to measure using the I2C communication protocol.

![Cmd Table](/how-to-use-sht21/cmdTable.png 'Cmd Table')

![Cmd Send Information](/how-to-use-sht21/cmdSendInfo.png 'Send Cmd')

Let's examine in detail how to read and write operations with the specified commands.

![Read Write Information](/how-to-use-sht21/readWriteInfo.png 'Read Write Information')

To summarize the above review, we have two addresses to choose from according to our writing and reading situations. With these addresses, I will easily measure temperature and humidity by using the commands in the "Command Table" according to our needs.

---

Let's examine the sample project I created using STM32CubeIde. I used the HAL library while creating this project.

![Ide Config](/how-to-use-sht21/ideConfig.png 'Ide Config')

After making the I2C Configuration above, let's create our code.

{{< highlight go "linenos=table,linenostart=1" >}}
#define START_TEMP_HM 					0xE3		        /*!<start hold master temperature measurement 		*/
#define WRITE_ADDRESS                   0X80			                /*!<write address for command						*/

void sht21_start_T_HM(void) {

	uint8_t data[] = { START_TEMP_HM };
	HAL_I2C_Master_Transmit(&hi2c1, (uint8_t) WRITE_ADDRESS, data, 1,
	HAL_MAX_DELAY);
	HAL_Delay(200);
}
{{< / highlight >}}

{{< highlight go "linenos=table,linenostart=1" >}}
typedef struct {
	uint8_t  T_DATA[2] ; 							  	/*!<raw humidity sensor data					      	*/
	uint16_t Temperature_Value[2] ;
	uint16_t Temperature ; 								/*!<processed temperature sensor data				  */
	char data_T[20]; 	  								  /*!<final temperature for LCD  and WIZNET			*/
	double son_sicaklik ; 								/*!<final temperature							          	*/
}SHT21_Handle_t;

void sht21_read_T_HM(SHT21_Handle_t *sht21_1) {

	uint8_t data[1];
	data[1] = READ_ADDRESS;
	HAL_I2C_Master_Transmit(&hi2c1, (uint8_t) WRITE_ADDRESS, (uint8_t*) data, 1,1000);
	HAL_I2C_Master_Receive(&hi2c1, (uint8_t) READ_ADDRESS, sht21_1->T_DATA, 2,1000);
	HAL_Delay(200);

	sht21_1->Temperature_Value[0] = sht21_1->T_DATA[0]; //MSB
	sht21_1->Temperature_Value[0] = sht21_1->Temperature_Value[0] << 8;
	sht21_1->Temperature_Value[1] = sht21_1->T_DATA[1]; //LSB
	sht21_1->Temperature_Value[1] = sht21_1->Temperature_Value[1]	& ((uint16_t) LSB_CONFIG);
	sht21_1->Temperature = sht21_1->Temperature_Value[0]| sht21_1->Temperature_Value[1];

}
{{< / highlight >}}

By examining and revising these functions, you can use the sensor as you wish. I hope it was useful. Waiting for your feedback, good work.


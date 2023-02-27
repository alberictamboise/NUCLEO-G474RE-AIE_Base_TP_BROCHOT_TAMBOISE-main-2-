# TP Commande Numérique Directe

Ce code à pour but de commander une MCC en courant et en vitesse grâce à 2
correcteurs PI.

### Composants utilisés
- Nucléo G474-RE
- Power Module dsPICDEM
- MCC 48V/12A

### Objectifs

- Réaliser un shell pour commander le hacheur, sur la base d'un code fourni.
- Réaliser la commande des 4 transistors du hacheur en commande complémentaire décalée.
- Faire l'acquisition des différents capteurs.
- Réaliser l'asservissement en temps réel.




## Commande MCC Basique

- Générer 4 PWM en complémentaire décalée pour contrôler en boucle ouverte le moteur en respectant le cahier des charges,
- Inclure le temps mort,
- Vérifier les signaux de commande à l'oscilloscope,
- Prendre en main le hacheur,
- Câbler correctement la STM32 au hacheur
- Générer le signal de commande "start" en fonction de la datasheet
- Faire un premier essai de commande moteur

### Generation de 4 PWM

Générer quatre PWM à partir du Timer 1 pour controler le hacheur.

 Cahier des charges :
- Fréquence de la PWM : 16kHz
- Temps mort minimum : 2us
- Résolution minimum : 10bits.

#### Reglage du Timer 1
L'horloge du système est de 170 MHz et on souhaite un timer d'une fréquence de
16 kHz. On va donc avoir un ARR=170x10^6/16x10^3-1=10624. Comme le Timer est réglé
en center align, on divise cette valeur par 2 ce qui fait que ARR=5311.

##### Counter setting
![Counter setting](./Images/TIM1_param.png "Counter setting")

##### Deadtime setting

![Dead time](./Images/TIM_DeadTime.png "deadtime setting")

##### PWM setting

![PWM setting](./Images/TIM1_PWM.png "PWM setting")

#### Observation a l'oscilloscope

##### Observations

![Oscillo PWM](./Images/Oscillo_PWM.png "PWM affichés à l'oscilloscope")

Nos PWM sont bien a 16KHz et complémentaires.

![Oscillo DeadTime](./Images/Oscillo_temps_mort.png "Temps mort")

Il y'as bien un temps mort de 2us entre chaques commutations.

### Prise en main de hacheur

Ce hacheur comporte 3 bras de ponts, nous n'en utiliserons que 2 pour commander notre MCC. Nous avons décidé de prendre les bras Yellow et Red car ils sont tout deux munis de capteur a effet Hall contrairement au troisième bras de pont.

En suivant la datasheet du hacheur et les pins utilisés sur notre Nucléo. Nous avons établi les connections de telle manière:

|Signal|Pin STM32|Pin hacheur|Correspondance hacheur|
|------|---------|-----------|----------------------|
|TIM1_CH1|PA8|Pin 12|CM_Y_TOP|
|TIM1_CH1N|PA11|Pin 30|CM_Y_BOT|
|TIM1_CH2|PA9|Pin 13|CM_R_TOP|
|TIM1_CH2N|PA12|Pin 31|CM_R_BOT|
|ISO_RESET|PC3|Pin 33 |ISO_RESET|
|ADC1_IN1|PA0|Pin 16|Y_HALL|
|TIM3_CH1|PA6|Voie A|CODEUR VOIE A|
|TIM3_CH2|PA4|Voie B|CODEUR VOIE B|

### Commande powerOn

En s'appuyant sur la datasheet, nous avons codé une fonction qui permet d'intialiser le hacheur en envoyant une impulsion d' au moins 2us. Cette fonction se lance si on entre "power on" dans le Shell ou si on appuie sur le bouton bleu de la Nucleo.

#### Detail de la fonction

```c
void motorPowerOn(void){
	HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin); // just for test, you can delete it
	//Phase de démarage//
	HAL_GPIO_WritePin(ISO_RESET_GPIO_Port, ISO_RESET_Pin,GPIO_PIN_SET );
	setAlpha(50);
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_1 );
	HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_2 );
	HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_2);


	int i=0;
	while (i<33)
	{
		i++;
	}
	HAL_GPIO_WritePin(ISO_RESET_GPIO_Port, ISO_RESET_Pin, GPIO_PIN_RESET);

}
```

Cette fonction commence par mettre le port de ISO_RESET a 1, elle va ensuite initialiser le Timer1 et enfin elle va tourner 34 fois dans une boucle ce qui représente environ 2us avant de remmtre le port ISO_RESET à 0.

Afin d'entrer dans cette fonctions grâce à l'appuie sur le bouton bleu, nous traitons les interruptions venant des GPIO:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if (GPIO_Pin== BUTTON_Pin)
	{
		motorPowerOn();
	}

}
```

Pour entrer dans la fonction en utilisant le commandShell, nous avons modifié la fonction shellExec pour que la commande power on nous fasse entrer dans motorPowerOn:

```c
void shellExec(void){
	if(strcmp(argv[0],"set")==0){
		if(strcmp(argv[1],"PA5")==0 && ((strcmp(argv[2],"0")==0)||(strcmp(argv[2],"1")==0)) ){
			HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, atoi(argv[2]));
			stringSize = snprintf((char*)uartTxBuffer,UART_TX_BUFFER_SIZE,"Switch on/off led : %d\r\n",atoi(argv[2]));
			HAL_UART_Transmit(&huart2, uartTxBuffer, stringSize, HAL_MAX_DELAY);
		}
		else if(strcmp(argv[1],"speed")==0){
			if(atoi(argv[2])==0 && strcmp(argv[2],"0")!=0){
				HAL_UART_Transmit(&huart2, motorSpeedInst, sizeof(motorSpeedInst), HAL_MAX_DELAY);
			}
			else{
				motorSetSpeed(atoi(argv[2]));
			}
		}
		else if(strcmp(argv[1],"alpha")==0){
			setAlpha(atoi(argv[2]));
		}
		else if(strcmp(argv[1],"current")==0){
					consignCurrent=(atof(argv[2]));

				}
		else{
			shellCmdNotFound();
		}
	}

	else if(strcmp(argv[0],"help")==0)
	{
		HAL_UART_Transmit(&huart2, help, sizeof(help), HAL_MAX_DELAY);
	}
	else if(strcmp(argv[0],"pinout")==0)
	{
		HAL_UART_Transmit(&huart2, pinout, sizeof(pinout), HAL_MAX_DELAY);
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"on")==0))
	{
		HAL_UART_Transmit(&huart2, powerOn, sizeof(powerOn), HAL_MAX_DELAY);
		motorPowerOn();
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"off")==0))
	{
		HAL_UART_Transmit(&huart2, powerOff, sizeof(powerOff), HAL_MAX_DELAY);
		motorPowerOff();
	}
	else{
		shellCmdNotFound();
	}
}
```

#### Test de la fonction

Pour effectuer le test, nous avons utilisé un oscilloscope en mode single trigger pour mesurer la longeur de l'impulsion:

![Oscillo ISO_RESET](./Images/Oscillo_ISO_RESET.png "Impulsion sur ISO_RESET")

Nous observons ici que notre temps mort est de plus de 2us, nous respectons donc le cahier des charges.

###Commande setAlpha

Cette fonction nous permet de régler directement la valeur de alpha entrant dans le commandShell "set alpha " suivi d'un entier compris entre 0 et 100. A l'initialisation alpha vaut 50, ce qui correspond à une vitesse nulle:

```c
void setAlpha(int alpha1)
{
	TIM1->CCR1=alpha1*(TIM1->ARR)/100;
	TIM1->CCR2=(100-alpha1)*(TIM1->ARR)/100;
}
```

Cette foonction est l'une des plus importante de notre code, elle est utilisée à la fin de notre boucle d'asservissement pour régler le alpha.

## Capteur de courant et position

### Mesure de courant 

Pour mesurer le courant nous allons utiliser un ADC en mode DMA qui sera connecté a un des capteurs à effet Hall présent sur le hacheur. L'ADC sera cadencé par le Timer 2 qui aura une fréquence de 160kHz. Le DMA remplira un Buffer de 10 valeur et nous ferons la moyenne du courant sur ces 10 valeurs pour avoir notre courant moyen. Cette démarche évite de prendre une mesure de courant erronées, car le courant passant dans le moteur comporte de très grandes variations ,et celles-ci peuvent nous géner lors de notre asservissement.

#### Paramètres ADC

![ADC setting1](./Images/ADC_param.png "Paramètres de l'ADC")
![ADC setting2](./Images/ADC_param1.png "Paramètres de l'ADC")

####Fonctions liées à l'ADC

Pour se servir de l'ADC en mode DMA, il faut récuperer la fonction HAL_ADC_ConvCpltCallback. Cette fonction met un flag à 1 quand le buffer de l'ADC est rempli:

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
  if (hadc == &hadc1)
  {
	  adcDMAflag=1;
  }

}

```

Nous avons ensuite codé une fonction qui calcule la moyenne des 10 valeurs présentes dans le buffer pré-rempli par le DMA:

```c
void meanADCValue (void)
{
	int i;
	int sum=0;
	if (adcDMAflag==1)
	{
		for (i=0; i<ADC_HALL_BUFFER; i=i+1)
		{
			sum=sum+ adcBuffer[i];
		}

		hallVoltageValue= ((sum/ADC_HALL_BUFFER)*3.3/4096.0)+OFFSET_DEFAULT_ADC;
		hallCurrentValue= (hallVoltageValue-VOLTAGE_HALL_OC)*HALL_GAIN;

	}
}
```

Nous avons aussi codé une fonction nous permettons d'afficher la valeur du courant sur le commandShell  grâce à la commande "measure current":

```c
void uartPrintADCValue(void)
{
	meanADCValue();
	sprintf(uartTxBuffer,"Current: %.2f A\r\n",hallCurrentValue);
	HAL_UART_Transmit(&huart2, uartTxBuffer, sizeof(uartTxBuffer), HAL_MAX_DELAY);

}
```
Le commandShell a été modifié pour interpréter la commande "measure Current" et lancer la fonction uartPrintADCValue.

### Mesure de Vitesse

Pour mesurer la vitesse, nous allons utiliser les roues codeuses présentent sur les MCC, Il y'as 2 roues de 1024 segments, nous allons compter les fronts montants et déscendants, nous avons donc 4096 points sur un tour.

Nous allons utiliser le Timer3 en mode counter pour compter les incréments sur les roues codeuses. Nous utiliserons le Timer5 pour venir récupérer la valeur du codeur à une fréquence de 10Hz

Nous devons penser à initialiser le compteur de TIM3 à son point milieu pour éviter que la vitesse fasse un bond si le moteur passe en marche arrière ce qui ferai passer le compteur de 0 a 65535.

#### Paramètres des Timers
##### Timer 3: Mode Counter

![Counter setting](./Images/TIM3_param.png "TIM3 setting")

##### Timer 5: Mode Interruption

![Counter setting](./Images/TIM5_param.png "TIM5 setting")

#### Fonctions de calcul de la vitesse

Nous devons d'abord traiter l'interruption envoyée par TIM5, nous utlisons pour cela la fonction HAL_TIM_PeriodElapsedCallBack:
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	/* USER CODE BEGIN Callback 0 */


	/* USER CODE END Callback 0 */
	if (htim->Instance == TIM6) {
		HAL_IncTick();
	}
	/* USER CODE BEGIN Callback 1 */
	else if (htim->Instance == TIM5){
		codeurValue= TIM3->CNT;
		TIM3->CNT = TIM3->ARR/2;
	}
	/* USER CODE END Callback 1 */
}
```
Dans cette fonction nous recuperons la valeur du compteur de TIM3 et nous le remettons à son point milieu pour la prochaine mesure.

Nous devons maintenant calculer la vitesse a partir des valeurs mesurées, pour se faire nous avons créé la fonction calcSpeed:

```c
void calcSpeed (void){
	speed=(codeurValue-((TIM3->ARR)/2.0))*FREQ_ECH_SPEED*60.0/NUMBER_OF_POINT;
}
```

Nous avons aussi une fonction qui nous permet d'afficher la vitesse sur le commandSheel quand on entre "measure speed":

```c
void uartPrintSpeed(void)
{
	calcSpeed();
	sprintf(uartTxBuffer,"Speed: %.2f tr/min\r\n",speed);
	HAL_UART_Transmit(&huart2, uartTxBuffer, sizeof(uartTxBuffer), HAL_MAX_DELAY);

}
```
##Asservissement

Maintenant que nous reussissons  a mesure le courant et la vitesse, nous allons effectuer un asservissement en courant puis en vitesse.

### Asservissement en courant

Nous allons utiliser une régulation PI parrallèle pour mettre en place notre asservissement en courant. Pour se faire, nous devons calculer notre commande selon le gain proportionnel et notre commande selon le gain intégral. Nous devons veiller a ce que notre commande ne dépasse pas 1 et que l'intégrale prenne en compte l'anti-windup.

```c
void asserCurrent (void)
{
	meanADCValue();

	float eps= consignCurrent - hallCurrentValue;

	// Proportional part
	if (Kp*eps < 0){
		alphaKp=0.0;
	}
	else if (Kp*eps > 1) {
		alphaKp=1.0;
	}
	else {
		alphaKp=eps*(float)Kp;
	}

	// Integral part

	alphaKi=alphaKiOld+((Ki*Te)/2)*(eps+epsOld);
	if (alphaKi < 0){
		alphaKi=0.0;
	}
	else if (alphaKi > 1) {
		alphaKi=1.0;
	}

	alphaKiOld=alphaKi;
	epsOld=eps;

	// Summ of the two coeff

	alpha=(int)((alphaKi+alphaKp)*100);

	if (alpha < 0){
		alpha=0;
	}
	else if (alpha > 100) {
		alpha=100;
	}
	setAlpha(alpha);

}
```
Nous devons aussi modifier certaines autres fonctions pour que notre asservissement ne démarre pas avant que la fonction motorPowerOn soit enclenché, on ajoute donc un startFlag qui permet de démarrer notre asservissement.

```c
void motorPowerOn(void){
	HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin); // just for test, you can delete it
	//Phase de démarage//
	HAL_GPIO_WritePin(ISO_RESET_GPIO_Port, ISO_RESET_Pin,GPIO_PIN_SET );
	setAlpha(50);
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_1 );
	HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_2 );
	HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_2);


	int i=0;
	while (i<33)
	{
		i++;
	}
	HAL_GPIO_WritePin(ISO_RESET_GPIO_Port, ISO_RESET_Pin, GPIO_PIN_RESET);

	consignCurrent=0;
	startFlag=1;
	alphaKiOld=0.5;
	epsOld=0;


}
```
Quand le startflag est à 1, nous passons par la fonction asserCurrent dans notre Superloop.

Il a fallu aussi ajouter un commande au Shell pour pouvoir entre la valeur de courant souhaité, voici notre nouvelle fonction shellExec:

```c
void shellExec(void){
	if(strcmp(argv[0],"set")==0){
		if(strcmp(argv[1],"PA5")==0 && ((strcmp(argv[2],"0")==0)||(strcmp(argv[2],"1")==0)) ){
			HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, atoi(argv[2]));
			stringSize = snprintf((char*)uartTxBuffer,UART_TX_BUFFER_SIZE,"Switch on/off led : %d\r\n",atoi(argv[2]));
			HAL_UART_Transmit(&huart2, uartTxBuffer, stringSize, HAL_MAX_DELAY);
		}
		else if(strcmp(argv[1],"speed")==0){
			if(atoi(argv[2])==0 && strcmp(argv[2],"0")!=0){
				HAL_UART_Transmit(&huart2, motorSpeedInst, sizeof(motorSpeedInst), HAL_MAX_DELAY);
			}
			else{
				motorSetSpeed(atoi(argv[2]));
			}
		}
		else if(strcmp(argv[1],"alpha")==0){
			setAlpha(atoi(argv[2]));
		}
		else if(strcmp(argv[1],"current")==0){
					consignCurrent=(atof(argv[2]));

				}
		else{
			shellCmdNotFound();
		}
	}
	else if (strcmp(argv[0],"measure")==0)
	{
		if(strcmp(argv[1],"current")==0){
			uartPrintADCValue();
		}
		else if (strcmp(argv[1],"speed")==0){
			uartPrintSpeed();
		}

	}
	else if(strcmp(argv[0],"help")==0)
	{
		HAL_UART_Transmit(&huart2, help, sizeof(help), HAL_MAX_DELAY);
	}
	else if(strcmp(argv[0],"pinout")==0)
	{
		HAL_UART_Transmit(&huart2, pinout, sizeof(pinout), HAL_MAX_DELAY);
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"on")==0))
	{
		HAL_UART_Transmit(&huart2, powerOn, sizeof(powerOn), HAL_MAX_DELAY);
		motorPowerOn();
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"off")==0))
	{
		HAL_UART_Transmit(&huart2, powerOff, sizeof(powerOff), HAL_MAX_DELAY);
		motorPowerOff();
	}
	else{
		shellCmdNotFound();
	}
}
```

Avec cette modification, nous pouvons régler le courant souhaité via la commande set current suivit de la valeur de type float souhaitée.

### Asservissement de Vitesse

Nous allons utiliser une régulation PI parrallèle pour mettre en place notre asservissement en vitesse. Pour se faire, nous devons calculer notre courant de référence selon le gain proportionnel et notre courant de référence selon le gain intégral. Nous devons veiller a ce que notre courant de référence ne dépasse pas 7A et que l'intégrale prenne en compte l'anti-windup.

```c
void asserSpeed (void)
{
	calcSpeed();
	float eps= consignSpeed - speed;

	// Proportional part
	if (KpSpeed*eps < -IMAX){
		currentKp=-IMAX;
	}
	else if (KpSpeed*eps > IMAX) {
		currentKp=IMAX;
	}
	else {
		currentKp=eps*(float)KpSpeed;
	}

	// Integral part

	currentKi=currentKiOld+((KiSpeed*TeSpeed)/2)*(eps+speedEpsOld);
	if (currentKi < -IMAX){
		currentKi=-IMAX;
	}
	else if (currentKi > IMAX) {
		currentKi=IMAX;
	}

	currentKiOld=currentKi;
	speedEpsOld=eps;

	// Summ of the two coeff

	consignCurrent=(float)(currentKi+currentKp);

	if (consignCurrent < -IMAX){
		consignCurrent=-IMAX;
	}
	else if (consignCurrent > IMAX) {
		consignCurrent=IMAX;
	}
}
```
Nous avons ensuite modifié le Shell pour pouvoir choisir entre l'asservissement en courant ou en vitesse grâce à la commande "mode [int]" avec 1 pour un asservissement en courant et 2 pour un asservissement en tension.

```c
void shellExec(void){
	if(strcmp(argv[0],"set")==0){
		if(strcmp(argv[1],"PA5")==0 && ((strcmp(argv[2],"0")==0)||(strcmp(argv[2],"1")==0)) ){
			HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, atoi(argv[2]));
			stringSize = snprintf((char*)uartTxBuffer,UART_TX_BUFFER_SIZE,"Switch on/off led : %d\r\n",atoi(argv[2]));
			HAL_UART_Transmit(&huart2, uartTxBuffer, stringSize, HAL_MAX_DELAY);
		}
		else if(strcmp(argv[1],"speed")==0){
			consignSpeed= (atof(argv[2]));
		}
		else if(strcmp(argv[1],"alpha")==0){
			setAlpha(atoi(argv[2]));
		}
		else if(strcmp(argv[1],"current")==0){
			consignCurrent=(atof(argv[2]));

		}
		else{
			shellCmdNotFound();
		}
	}
	else if (strcmp(argv[0],"measure")==0)
	{
		if(strcmp(argv[1],"current")==0){
			uartPrintADCValue();
		}
		else if (strcmp(argv[1],"speed")==0){
			uartPrintSpeed();
		}

	}
	else if (strcmp(argv[0],"mode")==0)
	{
		chooseModeFlag= atoi(argv[1]);

	}
	else if(strcmp(argv[0],"help")==0)
	{
		HAL_UART_Transmit(&huart2, help, sizeof(help), HAL_MAX_DELAY);
	}
	else if(strcmp(argv[0],"pinout")==0)
	{
		HAL_UART_Transmit(&huart2, pinout, sizeof(pinout), HAL_MAX_DELAY);
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"on")==0))
	{
		HAL_UART_Transmit(&huart2, powerOn, sizeof(powerOn), HAL_MAX_DELAY);
		motorPowerOn();
	}
	else if((strcmp(argv[0],"power")==0)&&(strcmp(argv[1],"off")==0))
	{
		HAL_UART_Transmit(&huart2, powerOff, sizeof(powerOff), HAL_MAX_DELAY);
		motorPowerOff();
	}
	else{
		shellCmdNotFound();
	}
}
```
Nous avons ensuite ajouté un flage qui se positionne à 1 quand on entre dans l'interruption du TIM5 afin de cadencer notre asservissement de vitesse à 10Hz.

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	/* USER CODE BEGIN Callback 0 */


	/* USER CODE END Callback 0 */
	if (htim->Instance == TIM6) {
		HAL_IncTick();
	}
	/* USER CODE BEGIN Callback 1 */
	else if (htim->Instance == TIM5){
		codeurValue= TIM3->CNT;
		TIM3->CNT = TIM3->ARR/2;
		speedFlag=1;
	}
	/* USER CODE END Callback 1 */
}
```

Notre super loop finale qui nous permet de lancer nos asservissement est la suivante:
```c
	while (1)
	{
		// SuperLoop inside the while(1), only flag changed from interrupt could launch functions
		if(uartRxReceived){
			if(shellGetChar()){
				shellExec();
				shellPrompt();
			}
			uartRxReceived = 0;
		}
		switch (chooseModeFlag){
		case 1:
			if (adcDMAflag)
			{
				if (startFlag){
					asserCurrent();
				adcDMAflag=0;
			}
			break;
		case 2:
			if (adcDMAflag)
			{
				if (startFlag){
					asserCurrent();
				}
				adcDMAflag=0;
			}
			if (speedFlag)
			{
				if (startFlag){
					asserSpeed();
				}
				speedFlag=0;
			}
			break;
		}
```

Nous avons tracé la réponse en courant et en vitesse à un échelon de 1500tr/min:

![Step_Response_Speed](./Images/Réponse_echelon_vitesse.png "Réponse à un échelon de vitesse")

Observations: On peut voir ici que au démarage notre courant est bien saturé a IMAX pce qui nous permet d' avoir une accélération constante et une fois proche de notre vitesse de consigne, le courant baisse afin de stabiliser la vitesse a 1500 tr/min.

## Commande de Test

### Test Asservissemment Courant:
Commandes à lancer:

- power on
- mode 1
- set current [float]

### Test Asservissement Vitesse :
Commandes à lancer:

- power on
- mode 2
- set speed [float]

### Test set Alpha:
Commande à lancer:

- power on
- set alpha [int]


## Author

- [@Erwan Brochot](https://github.com/ErwanBrochot/)


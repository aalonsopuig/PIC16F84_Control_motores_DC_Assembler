; Programa para control de motores cc con encoder directo 
; Por: Alejandro Alonso
; Fecha: 26/12/2002
; Función: 
;
; Mide revoluciones y las muestra en una barrera de leds
; Selecciona cuatro posibles velocidades según la combinación
; de dos switches, cargando el patrón PWM adecuado en caso de 
; presionar el pulsador. 
; Asegura mediante trenes PWM el funcionamiento del motor a la
; velocidad seleccionada.


	list 		p=16f84
	include	"P16F84a.INC"


	Contador	EQU	0x0C	;Contador multiuso
	Contador2	EQU	0x0E	;Contador multiuso
	Velocidad	EQU	0x0D	;Cálculos de velocidad real
	PWM		EQU	0x0F	;Cadena PWM a enviar al motor

	Backup	EQU	0xA0	;Bits de backup para casos varios... 
;...Definidos como sigue:
	Acarreo	EQU	0


	; Definiciones bits del registro RA

	Switch1	EQU	0	;RA0 - Switches que definen patrón...
	Switch2	EQU	1	;RA1 - de PWM (Velocidad)
	Pulsador	EQU	2	;RA2 - Permiso de carga de patrón.
	Motor		EQU	3	;RA3 - Salida PWM control velocidad
	Encoder	EQU	4	;RA4 - Entrada señal encoder motor
	;Atención, RA4 en modo salida trabaja en colector abierto


	org	0
	goto	INICIO
	org	4			; Vector de interrupción
	goto	INTERRUPT

INTERRUPT
	movf	Velocidad,W
	movwf	PORTB			;Muestra leds velocidad
	movlw	b'11111111'		;Inicializamos contador
	movwf	Velocidad		;de velocidad
	movlw	b'11110000'		;Inicializamos contador pulsos
	movwf	TMR0	
	bcf	INTCON,T0IF		; quita flag
	RETFIE

	



; ----------------------------------------------------------------------------
INICIO		;Inicio del cuerpo del programa

	bsf	STATUS,RP0		;Apunta a banco 1
	movlw	b'11110111'		;Establece puerta A como ENTRADA...
	movwf	TRISA			;...excepto RA3 (Motor)
	movlw	b'00000000'		;Establece puerta B como SALIDA (Leds)
	movwf	TRISB			;
	movlw	b'00100000'		;Configuración OPTION para TMR0
	movwf	OPTION_REG
	bcf	STATUS,RP0		;Apunta a banco 0

	movlw	b'10100000'		;Establece interrupciones
	movwf	INTCON		;para overflow TMR0

	clrf	PORTA
	clrf	PORTB

	movlw	b'11111111'		;Inicializamos contador
	movwf	Velocidad		;de velocidad

	movlw	b'11110000'		;reiniciamos contador pulsos
	movwf	TMR0	

	clrf	Backup		;Limpiamos variable de backups

BUCLE	;Bucle principal del programa

	btfss	PORTA,Pulsador	;Vemos si hay que cargar patrón PWM
	Goto	Ciclo			;No, continuamos ciclo
	;Si, Chequeamos switches para asignar velocidad (cadena PWM)
		
	btfss	PORTA,Switch1	;Vemos valor Switch1
	goto	Sw_0x			;Sw1=0
	btfss	PORTA,Switch2	;Sw1=1. Vemos valor Switch2
	goto	Sw_10			;Sw1=1, Sw2=0
	goto	Sw_11			;Sw1=1, Sw2=1
Sw_0x	btfss	PORTA,Switch2	;Sw1=0. Vemos valor Switch2
	goto	Sw_00			;Sw1=0, Sw2=0
	goto	Sw_01			;Sw1=0, Sw2=1

Sw_00	movlw	b'10001000'
	goto	Sw_Ok
Sw_01	movlw	b'10010010'
	goto	Sw_Ok
Sw_10	movlw	b'11011011'
	goto	Sw_Ok
Sw_11	movlw	b'11111111'
	goto	Sw_Ok

Sw_Ok	movwf	PWM			;Cargamos el valor seleccionado en PWM		
	bcf	STATUS,C		;Reiniciamos valores
	bcf	Backup,Acarreo	;Reiniciamos valores


Ciclo	Movlw	d'230'		;
	movwf	Contador		;...se carga en "contador"
	call	Retardo	
	bcf	STATUS,C		;Ponemos a 0 acarreo
	rrf	Velocidad,F


	GOTO BUCLE




; ----------------------------------------------------------------------------
	

; Subrutinas


	
Retardo	;Provoca un retardo según el valor de "Contador"
	btfss	Backup,Acarreo	;Vemos cual era el valor anterior de acarreo
	goto	C_Cero		;..y lo ponemos de nuevo como acarreo
	bsf	STATUS,C
	goto	C_Ok
C_Cero	bcf	STATUS,C
C_Ok	NOP
Bucle1	movlw	255		;Inicialización bucle interno
	movwf	Contador2
Bucle2	btfss	PWM,0		; Chequea bit 0 de cadena PWM
	goto	EsCero		; Si es cero pone cero en Salida control motor
	bsf	PORTA,Motor		; Si es uno pone cero en Salida control motor
 	goto	Sigue
EsCero	bcf	PORTA,Motor
Sigue	rrf	PWM,F			; Rota PWM
	btfss	STATUS,C		;Vemos cual es el valor de acarreo y hacemos backup
	goto	Cc_Cero		;..y lo ponemos de nuevo como acarreo
	bsf	Backup,Acarreo
	goto	Cc_Ok
Cc_Cero	bcf	Backup,Acarreo
Cc_Ok	NOP
	decfsz	Contador2,F	;Decrementar contador bucle externo
	goto	Bucle2		;y repetir bucle interno hasta fin
	decfsz	Contador,F	;Decrementar contador bucle externo
	goto	Bucle1		;y repetir bucle externo hasta fin

	RETURN



Fin
	END


// ProjetoChocadeira

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#define FOSC 16000000U // Clock Speed
#define BAUD 9600
#define MYUBRR ((FOSC / (16 * BAUD)) - 1)

//Prototipos das funcoes
void UART_Init(unsigned int ubrr);
void UART_Transmit(char *dados);

char msg_tx[20];
char msg_rx[1];
int pos_msg_rx = 0;
int tamanho_msg_rx = 1;

int temp = 0;
bool sistema_state = false;

// Valor de THRESHOLD
#define THRES 500
void ADC_init(void)
{
    // Configurando Vref para VCC = 5V
    ADMUX = (1 << REFS0);
    /*
        ADC ativado e preescaler de 128
        16MHz / 128 = 125kHz
        ADEN = ADC Enable, ativa o ADC
        ADPSx = ADC Prescaler Select Bits
        1 1 1 = clock / 128
    */
    ADCSRA = (1 << ADEN) | (1 << ADPS2) | (1 << ADPS1) | (1 << ADPS0);
}
int ADC_read(u8 ch)
{
    char i;
    int ADC_temp = 0; // ADC temporário, para manipular leitura
    int ADC_read = 0; // ADC_read
    ch &= 0x07;
    // Zerar os 3 primeiros bits e manter o resto
    ADMUX = (ADMUX & 0xF8) | ch;
    // ADSC (ADC Start Conversion)
    ADCSRA |= (1 << ADSC); // Faça uma conversão
    // ADIF (ADC Interrupt Flag) é setada quando o ADC pede interrupção
    // e resetada quando o vetor de interrupção
    // é tratado.
    while (!(ADCSRA & (1 << ADIF)));                   // Aguarde a conversão do sinal
    for (i = 0; i < 8; i++) // Fazendo a conversão 8 vezes para maior precisão
    {
        ADCSRA |= (1 << ADSC); // Faça uma conversão
        while (!(ADCSRA & (1 << ADIF)));                    // Aguarde a conversão do sinal
        ADC_temp = ADCL;         // lê o registro ADCL
        ADC_temp += (ADCH << 8); // lê o registro ADCH
        ADC_read += ADC_temp;    // Acumula o resultado (8 amostras) para média
    }
    ADC_read = ADC_read >> 3; // média das 8 amostras
    return ADC_read;
}

ISR(INT0_vect) {
  if(OCR0A == 280 || sistema_state == false){
    OCR0A = 0;
    }
  else{
  OCR0A += 28;
   _delay_ms(100);
    }
}

void setup()
{
    UART_Init(MYUBRR);
    ADC_init(); // Inicializa ADC

    DDRB = (1 << PB5); // PB5 como saída
  	DDRD = (1 << PD3); // PD3 como saída

    //Configurando o PWM
    TCCR0A |= (1 << WGM01) | (1 << WGM00); // Modo fast PWM

    
    TCCR0A |= (1 << COM0A1); //TCNT0 = 0, PD6(OC0A) = 1
    TCCR0B |= (1 << CS00); // preescaler 1

    //Configuração da interrupção externa
    EICRA |= (1 << ISC01);
    EIMSK |= (1 << INT0); // Habilita o INT0
    sei();

    //Configurando pd6 saida
    DDRD |= (1 << PD6);

    //Pull UP em PD2
    PORTD |= (1 << PD2);

    //Inicia Resistencia em 0%
    OCR0A = 0;
}

void loop() {
    u16 adc_result0, adc_result1;
    unsigned long int aux;
    unsigned int tensao;

    if(msg_rx[0] == 'L') {
      	PORTB |= (1<< PB6);
        temp = 0;
        UART_Transmit("LIGADO");
        UART_Transmit("\n");
        sistema_state = true;
        msg_rx[0] = ' ';
      	PORTD |= (1<< PD3);
    }
    else if(msg_rx[0] == 'T') {
      if(sistema_state){
        aux = ((long)adc_result0*20)/1023;
        aux += 20;
        tensao = (unsigned int)aux;
        UART_Transmit("Temperatura: ");
        itoa(tensao, msg_tx, 10);
        UART_Transmit(msg_tx);
        UART_Transmit(" ºC\n");
        msg_rx[0] = ' ';
      }
      else{
      	UART_Transmit("Sensor desligado\n");
        msg_rx[0] = ' ';
      }
    }
    else if(msg_rx[0] == 'D') {
        temp = 0;
        UART_Transmit("DESLIGADO");
        UART_Transmit("\n");
        sistema_state = false;
        msg_rx[0] = ' ';
      	PORTD &= ~(1 << PB3);
    }
    if(sistema_state) {
        adc_result0 = ADC_read(ADC0D); // lê o valor do ADC0 = PC0
        _delay_ms(50);                 // Tempo para troca de canal
        adc_result1 = ADC_read(ADC1D); // lê o valor do ADC1 = PC1
        // condição do led
        if (adc_result0 < THRES && adc_result1 < THRES)
            PORTB |= (1 << PB5);
        else
            PORTB &= ~(1 << PB5);
      
        temp = adc_result0;
        _delay_ms(1000);
    }
    else {
        OCR0A = 0;
        temp = 0;
    }
}

ISR(USART_RX_vect)
{
    // Escreve o valor recebido pela UART na posição pos_msg_rx do buffer msg_rx
    msg_rx[pos_msg_rx++] = UDR0;
    if (pos_msg_rx == tamanho_msg_rx)
        pos_msg_rx = 0;
}
void UART_Transmit(char *dados)
{
    // Envia todos os caracteres do buffer dados ate chegar um final de linha
    while (*dados != 0)
    {
        while (!(UCSR0A & (1 << UDRE0)))
            ; // Aguarda a transmissão acabar
        // Escreve o caractere no registro de tranmissão
        UDR0 = *dados;
        // Passa para o próximo caractere do buffer dados
        dados++;
    }
}
void UART_Init(unsigned int ubrr)
{
    // Configura a baud rate */6
    UBRR0H = (unsigned char)(ubrr >> 8);
    UBRR0L = (unsigned char)ubrr;
    // Habilita a recepcao, tranmissao e interrupcao na recepcao */
    UCSR0B = (1 << RXEN0) | (1 << TXEN0) | (1 << RXCIE0);
    // Configura o formato da mensagem: 8 bits de dados e 1 bits de stop */
    UCSR0C = (1 << UCSZ01) | (1 << UCSZ00);
}

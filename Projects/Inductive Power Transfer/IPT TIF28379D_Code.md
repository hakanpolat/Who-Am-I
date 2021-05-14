#include <F2837xD_Device.h>
#include <F2837xD_Examples.h>
#include <math.h>
#include<stdlib.h>
/*Bu kodda esnek seçim yapýlabilecek.Phase_shift ve frequency controlleri arasýnda geçiþ yapýlabilecek.Primer taraftan
 * akým kontrolü yapabilmek için gerekli adcler aktifleþtirilecek. */

/////////////////////////////////////////////////////////////////////////////////////////////////////
// Fonksiyonlarýn temelleri
void ConfigureADC(void); // ADC'nin özelliðini belirtiyorum. Single mode 12bit seçiyorum
void ConfigureEPWM(void);// ADC'nin hangi pwm ile aktifleþtirileceðini seçiyorum.
void SetupADCEpwmFast(void); // Hangi SOC modülünün kullanýlacaðý ve EOC'da ADCA1_int flagini aktifleþtiriyorum.
void SetupADCEpwmSlow(void); // Hangi SOC modülünün kullanýlacaðý ve EOC'da ADCA1_int flagini aktifleþtiriyorum.
void EPwm1(void); // Complementary 2 tane pwm üretiliyor. Bu üretilen iki pwm her zaman sabit. EPWM2A ise bunlara göre deðiþiyor. 1. leg pwmleri
void EPwm2(void); // Complementary 2 tane pwm üretiliyor. Bu 2PWM frequency veya phase e göre EWPM1 e göre deðiþiyor.
void EPwm6(void); // Complementary 2 tane pwm üretiliyor. Bu 2PWM frequency veya phase e göre EWPM1 e göre deðiþiyor.
void EPwm5(void); // Complementary 2 tane pwm üretiliyor. Bu 2PWM frequency veya phase e göre EWPM1 e göre deðiþiyor.
void ConfigureGPIO(void);// GPIO pinlerinin ayarlarýný yapýyorum.
//BT Function temelleri
void ScibBackgroundTaskReceive(void);
void InitializeScibRegisters(float fSciBaudRate);

/////////////////////////////////////////////////////////////////////////////////////////////////////
//INTERRUPTLAR
__interrupt void adca1_isr(void); // ADCA1 interrupt fonksiyonu temeli (sine current measurement) HIZLI
__interrupt void adca2_isr(void); // ADCA1 interrupt fonksiyonu temeli YAVAS
__interrupt void epwm1_tzint_isr(void);//Devreyi kapama one-shot Trip zone
__interrupt void epwm2_tzint_isr(void);//Devreyi kapama one-shot Trip zone
/////////////////////////////////////////////////////////////////////////////////////////////////////
//DEFINITIONS

//Led definitions
#define OVP_LED GPIO31 // Over Voltage Protection girdiðinde GPIO31 yanýcak. Trip_zone'da aktifleþicek.
#define OCP_LED GPIO34 // Over Current Protection girdiðinde GPIO34 yanýcak. Trip_zoneda aktifleþicek.
#define NOP_LED GPIO16 // GPIO 16 yandýðýnda sorun yok demek. NOP= NO PROBLEM
#define ON_LED GPIO24  // Devre çalýþtýðýnda GPIO24 yansýn.
#define TZ_EWPM GPIO14 // GPIO14 devreyi kapatmak için kullanýcak. Trip-Zone
//System Definitions
#define sysclk_frequency    200000000   // Hz
#define sysclk_period       5               // ns
#define pwmclk_frequency    200000000   // Hz
#define pwmclk_period       5               // ns
#define PI                  3.141592654 //Burayý defineladým ama henüz kullanmayacaðým çünkü bilmiyorum.
#define sample_window       65          // clock
#define adc_frequency       100000    //1MHz
#define adc_period          50         //ns
#define Cur_Buf_Size 48
#define V_Buf_Size 40
//Bluetooth Definitions
#define MFDS_LIB_CPU_FREQ   200e6
#define MFDS_THETAG              "hsrc"
#define MFDS_THETAGBYTESIZE      4
#define MFDS_SCILIBBUFFERLENGTH      2
#define MFDS_ScibRXBUFFERSIZE        2
#define MFDS_MULTIPLEFLOATARRAYSIZE (24+MFDS_THETAGBYTESIZE)
//PI Controller
#define integrator_size 1000
//Control mode selection
// Birini 1 yaparken öbürünü 0 yapmayý unutma
#define PHASECONT 0 // 1 olunca PHASE  control
#define FREQCONT 1 // 1 olunca Frequency Control
#define SOFTSTART 0// Soft Start kodu aktif freq veya faz kontrolü bile olsa önce fazý 0 dan baþlatýp belli bir yere kadar yavaþça getirmek lazým.
#define PHSMEASURE 0//Phase measurement aktifleþtirildi.
#define PHSMEASURE_COMP 2268 //Phase measurement sýrasýnda karþýlaþtýrýlýcak deðer
#define CODESTART 1 // Kodu bir button ile baþlatma
#define CURRENTMEASURE 1
#define TZ_TURNOFF 0
#define OVP_PROT 1
#define OPENLOOP 1
//Switching Definitions
#if FREQCONT
#define switching_frequency 200000 // Switching frequency 150khz olarak seçildi.
#endif
#if PHASECONT
#define switching_frequency 156000 // Switching frequency 150khz olarak seçildi.
#define CurUNBalance 1
#define CurBalance 0
#endif
#define switching_period 6666 //ns
#define dead_time 300 //Dead time 100ns seçildi.
/////////////////////////////////////////////////////////////////////////////////////////////////////
//VARIABLES
int AdcaResults_freq;
Uint16 AdcaResults_phase;
Uint16 Current_data=0;
Uint16 Cb1[Cur_Buf_Size];
Uint16 Cb2[Cur_Buf_Size];
Uint16 Vm[V_Buf_Size];
float cur_int=0;
float Phase=0;
float Phase1=0;
float Phase2=0;
float Phase3=0;
float Phase4=0;
float Cerror=0;
float Cerror_sum=0;
float cur_p=1;
float cur_i=0;
float current_peak=0;
unsigned int cnt=0; //soft start yapýlýrken kullanýlan bir counter
unsigned int aa=0;
unsigned int starter=0;
unsigned int cur_cnt=0;
unsigned int cur_offset=900;
unsigned int cnt_deneme=0;
unsigned int resultsIndex=0;
unsigned int resultsIndex2=0;
unsigned int bufferFull=0;
unsigned int bufferFull2=0;
float Vsum=0;
float Vavg=0;
float Vset=30;
float Verror=0;
float Verror_sum=0;
float Verror_window[integrator_size];
int   Verror_index=0;
float p_cont=-0.1;
float p_cont_1=-3;
float phase_p=0;
float phase_i=0;
float i_cont =0;
float Verror_sum_max=2500;
float Verror_sum_min=-2500;
long int tot_cur1=0;
long int tot_cur2=0;
float cur1_avg=0;
float cur2_avg=0;
float V_deneme=14;
//////////BT Variables
unsigned int uiScibRxBufferIndex = 0;
unsigned int ucScibRxBuffer[MFDS_ScibRXBUFFERSIZE];
char cTransmitScibLibDataBuffer[MFDS_SCILIBBUFFERLENGTH];
unsigned int uiTransmitScibLibReadFromBufferIndex = 0;
unsigned int uiTransmitScibLibWriteToBufferIndex=0;
unsigned int uiTransmitScibLibBufferNumberOfMessages=0;
unsigned int uiReceiveScibLibBufferNumberOfMessages=0;
 float freq_value=0;
int main(void)
 {    InitSysCtrl();
      InitPieCtrl();  /*initialize the PIE table (interrupt related)*/

    EALLOW;
    ClkCfgRegs.PERCLKDIVSEL.bit.EPWMCLKDIV = 0; // EPWM Clock Divide Select: /1 of PLLSYSCLK
    EDIS;

////////////////////////////////////////////////////////////////////////////////////////////////
 //CODE STARTER
    /*Burada GPIO5'e button baðlanacak default olarak 1kohm resistor ile GNDye baðlý olacak
     * sonra high verildiði zaman starter zamanla artýcak ve 100 olduðunda devre baþlayacak. */
    GpioCtrlRegs.GPAPUD.bit.GPIO5=1; // pull up enable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO5=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO5=0; //GPIO5 GPIO seçildi
    GpioCtrlRegs.GPADIR.bit.GPIO5=0; //GPIO5 input seçildi
#if CODESTART
    while(starter<100)
    {DELAY_US(5000);
     if(GpioDataRegs.GPADAT.bit.GPIO5==1)
    {starter++;}}
#endif
///////////////////////////////////////////////////////////////////////////////////////////////
for(resultsIndex = 0; resultsIndex < Cur_Buf_Size; resultsIndex++)
{
        Cb1[resultsIndex] = 0;
        Cb2[resultsIndex] = 0;
}
resultsIndex = 0;
bufferFull = 0;
for(resultsIndex2= 0; resultsIndex2 < V_Buf_Size; resultsIndex2++)
{
        Vm[resultsIndex2] = 0;
}
resultsIndex2 = 0;
bufferFull2 = 0;

for(Verror_index= 0; Verror_index < integrator_size; Verror_index++)
{
        Verror_window[Verror_index] = 0;

}
Verror_index=0;


    IER = 0x0000;   /*clear the Interrupt Enable Register   (IER)*/
    IFR = 0x0000;   /*clear the Interrupt Flag Register     (IFR)*/
    InitPieVectTable();
    EALLOW;
    CpuSysRegs.PCLKCR0.bit.TBCLKSYNC = 0;   /*stop the TimeBase clock for later synchronization*/
    CpuSysRegs.PCLKCR0.bit.GTBCLKSYNC = 0;  /*stop the global TimeBase clock for later synchronization*/
    EDIS;
   InitCpuTimers(); // CPU timerlarý aktifleþtiriliyor.
   ConfigCpuTimer(&CpuTimer0, 200, 200); //1 seconds
   CpuTimer0Regs.TCR.all = 0x4000; // enable cpu timer interrupt

   EALLOW;//
   PieVectTable.ADCA1_INT = &adca1_isr;  /*ADC1_INT flag aktif olduðunda hangi fonksiyona gitmesi gerektiðini söylüyorum. adca1_isr fonksiyonuna pointerladim.*/
   PieVectTable.ADCA2_INT = &adca2_isr;  /*ADC2_INT flag aktif olduðunda hangi fonksiyona gitmesi gerektiðini söylüyorum. adca2_isr fonksiyonuna pointerladim.*/
 #if TZ_TURNOFF
   PieVectTable.EPWM1_TZ_INT = &epwm1_tzint_isr;
   PieVectTable.EPWM2_TZ_INT = &epwm2_tzint_isr;
#endif
   EDIS;
   EALLOW;
   ClkCfgRegs.PERCLKDIVSEL.bit.EPWMCLKDIV = 0; // EPWM Clock Divide Select: /1 of PLLSYSCLK
   EDIS;

    IER |= M_INT1;  /*PIE Vectorlerinden ilk row'u aktif ediyorum. Referans Manual 96. sayfaya bakabilirsiniz. ADCA1 ve Timer0 Interruptlarý burada*/
    IER |= M_INT2;  /*PIE Vectorlerinden 2. row'u aktif ediyorum. Referans Manual 96. sayfaya bakabilirsiniz. Trip_zone interruptý burada*/
    IER |= M_INT3; /*PIE Vectorlerinden 3. row'u aktif ediyorum. EPWM3 ile okuma yapýyoruz. Onun interruptý burada ama aslýnda gerekte yok*/
    IER |= M_INT10; /*PIE Vectorlerinden 10. row'u aktif ediyorum. ADCA2 falan burada. Sayfa 96 ya bakabilirsin*/
    PieCtrlRegs.PIECTRL.bit.ENPIE = 1;  // Enable the PIE block
    PieCtrlRegs.PIEIER1.bit.INTx1 = 1;  //M_INT1 ile PIE 1. rowunu aktifleþtirdik. Burada PIEIER1(yani 1. row)_INTx1(birinci interruptu) ADCA1 e denk geliyor.
    PieCtrlRegs.PIEIER2.bit.INTx1 = 1;  //M_INT2 ile PIE 2. rowunu aktifleþtirdik. Burada PIEIER2(yani 2. row)_INTx1(birinci interruptu) Trip Zone EPWM1 e denk geliyor.
    PieCtrlRegs.PIEIER10.bit.INTx2 = 1;  //10. row 2.column aktifleþtirildi. burada adc2 var.

    EALLOW;
    CpuSysRegs.PCLKCR2.bit.EPWM1 = 1; // Furkan K. ekle dedi ama bence gerek yok.
    CpuSysRegs.PCLKCR2.bit.EPWM2 = 1; // Furkan K. ekle dedi ama bence gerek yok.
    CpuSysRegs.PCLKCR2.bit.EPWM3 = 1; // Furkan K. ekle dedi ama bence gerek yok.
    CpuSysRegs.PCLKCR0.bit.TBCLKSYNC = 1;   /*start the TimeBase clock */
    CpuSysRegs.PCLKCR0.bit.GTBCLKSYNC = 1;  /*start the global TimeBase clock */
    EDIS;
    EINT;  // Enable Global interrupt INTM
    ERTM;  // Enable Global realtime interrupt DBGM

    ConfigureGPIO();
    ConfigureADC();
    ConfigureEPWM();
    SetupADCEpwmFast();
    SetupADCEpwmSlow();
    EPwm1();
    EPwm2();
    EPwm6();
    EPwm5();
    EPwm3Regs.ETSEL.bit.SOCAEN = 1;  //enable SOCA
    EPwm3Regs.TBCTL.bit.CTRMODE = 0; //unfreeze, and enter up count mode

    //BT initilize
      InitializeScibRegisters(115200.0);
  while(1)
  {
 cur1_avg=tot_cur1/Cur_Buf_Size;//Average akım değerleri hesaplanıyor.
 cur2_avg=tot_cur2/Cur_Buf_Size;//Average akım değerleri hesaplanıyor.

 }

 }

__interrupt void adca1_isr(void)
{/* Bu ISR içinde moving average olarak yaklaşık 4 cycle iki primer akımını okuyoruz. Bu okumalar daha sonrasında
primer modül akımlarının eşitlenmesinde kullanılacaktır.*/
GpioDataRegs.GPATOGGLE.bit.ON_LED=1;
tot_cur1= tot_cur1-Cb1[cur_cnt];
tot_cur2= tot_cur2-Cb2[cur_cnt];
Cb1[cur_cnt]=abs(AdcaResultRegs.ADCRESULT1-cur_offset);
Cb2[cur_cnt]=abs(AdcaResultRegs.ADCRESULT4-cur_offset);
tot_cur1= tot_cur1+Cb1[cur_cnt];
tot_cur2= tot_cur2+Cb2[cur_cnt];
cur_cnt++;
if(cur_cnt>=Cur_Buf_Size)
{cur_cnt=0;}
    AdcaRegs.ADCINTFLGCLR.bit.ADCINT1 = 1; //clear INT1 flag
    PieCtrlRegs.PIEACK.all = PIEACK_GROUP1;

}



__interrupt void adca2_isr(void)
{/*Bu ISR daha yavaş bir ISR'dır. İlk olarak definition kısmından uygulanmak istenen kontrol metodu seçilmelidir.*/

//Softstart kodu
#if(SOFTSTART)
while(cnt<120)
{
#if FREQCONT
EPwm2Regs.TBPHS.bit.TBPHS= EPwm2Regs.TBPRD*(cnt/120);
#endif
 cnt=cnt+1;
DELAY_US(1000); //þimdilik 100us delay yani 12ms 'de 180 derece phase oluyor. Simulasyonlara göre yeterli bir süre
}
#endif
////////////////////////
#if FREQCONT
/*Frekans kontrolü sırasında bluetoothtan gelen Vout değeri okunur. Daha sonrasında ADC okuması gerçek voltaj değerine
* dönüştürülüp Verror hesaplanır. Verror daha sonrasında toplanıp PI controllera sokulur. Kp Verrorün değerine göre değişiklik göstermektedir. */
//Vsum=4.5*ucScibRxBuffer[0];
Vsum=4.5*ScibRegs.SCIRXBUF.all;
Vavg= 3300*(Vsum)/(4095.0*7.35);
Verror=Vset-Vavg;
if(Verror>15&&Verror<(-15))
{p_cont=p_cont_1*6;}
else {p_cont=p_cont_1;}
Verror_sum=Verror_sum+Verror;
AdcaResults_freq=AdcaResults_freq+p_cont*Verror+i_cont*Verror_sum;
if(AdcaResults_freq>=0)
{AdcaResults_freq=0;}
if(AdcaResults_freq<=(-34000))
{AdcaResults_freq=-34000;}
if(freq_value>=0)
{freq_value=0;}
if(freq_value<=(-100000))
{freq_value=-100000;}

//EPwm1Regs.TBPRD=0.5*((float)pwmclk_frequency)/((float)switching_frequency+AdcaResults_freq);
EPwm1Regs.TBPRD=0.5*((float)pwmclk_frequency)/((float)switching_frequency+freq_value);
EPwm1Regs.CMPA.bit.CMPA=EPwm1Regs.TBPRD/2;
EPwm1Regs.CMPB.bit.CMPB=EPwm1Regs.TBPRD/2;
EPwm2Regs.TBPRD=EPwm1Regs.TBPRD;
EPwm2Regs.CMPA.bit.CMPA=EPwm2Regs.TBPRD/2;
EPwm2Regs.CMPB.bit.CMPB=EPwm2Regs.TBPRD/2;
EPwm2Regs.TBPHS.bit.TBPHS=EPwm2Regs.TBPRD;
EPwm6Regs.TBPRD=EPwm1Regs.TBPRD;
EPwm6Regs.CMPA.bit.CMPA=EPwm2Regs.TBPRD/2;
EPwm6Regs.CMPB.bit.CMPB=EPwm2Regs.TBPRD/2;
EPwm6Regs.TBPHS.bit.TBPHS=0;
EPwm5Regs.TBPRD=EPwm1Regs.TBPRD;
EPwm5Regs.CMPA.bit.CMPA=EPwm2Regs.TBPRD/2;
EPwm5Regs.CMPB.bit.CMPB=EPwm2Regs.TBPRD/2;
EPwm5Regs.TBPHS.bit.TBPHS=EPwm2Regs.TBPRD;
#endif
/*Phase control metotunda akım eşitleme ve eşitlememe aktif edilebilir.
* Akım eşitleme sırasında primer akımların fazları ayarlanarak akım değerlerinin eşit olması amaçlanıyor.
* İki adet Kp ve Ki değeri vardır. Vout ve akım aynı anda eşitlenmeye çalışılmaktadır.*/
#if PHASECONT
#if CurUNBalance
Vsum=4.5*ScibRegs.SCIRXBUF.all;
Vavg= 3300*(Vsum)/(4095.0*7.35);
Verror=Vset-Vavg;
Verror_sum=Verror_sum+Verror;
//Phase=Phase+phase_p*Verror+phase_i*Verror_sum;

if(Phase>=180)
{Phase=180;}
if(Phase<=20)
{Phase=20;}
EPwm2Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase/180.0;
EPwm5Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase/180.0;
EPwm6Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase/180.0;
#endif
#if CurBalance
Vsum=V_deneme;
Vavg= 3300*(Vsum)/(4095.0*7.35);
Verror=Vset-Vavg;
Verror_sum=Verror_sum+Verror;
Cerror=cur1_avg-cur2_avg;
Cerror_sum=Cerror_sum+Cerror;
Phase1=Phase1+phase_p*Verror+phase_i*Verror_sum-cur_p*Cerror-cur_i*Cerror;
Phase2=Phase2+phase_p*Verror+phase_i*Verror_sum+cur_p*Cerror+cur_i*Cerror;
if(Phase1>=180)
{Phase1=180;}
if(Phase1<=0)
{Phase1=0;}
if(Phase2>=180)
{Phase2=180;}
if(Phase2<=0)
{Phase2=0;}
EPwm2Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase1/180.0;
Phase3=Phase1/2-Phase2/2;
Phase4=Phase1/2+Phase2/2;
if(Phase3<0)
{Phase3=Phase3+180;
EPwm5Regs.TBCTL.bit.PHSDIR = TB_UP;
}
else
{EPwm5Regs.TBCTL.bit.PHSDIR = TB_DOWN;}
EPwm5Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase3/180.0;
EPwm6Regs.TBPHS.bit.TBPHS=EPwm1Regs.TBPRD*Phase4/180.0;
#endif
#endif
AdcaRegs.ADCINTFLGCLR.bit.ADCINT1 = 1; //clear INT1 flag
AdcaRegs.ADCINTFLGCLR.bit.ADCINT2 = 1; //clear INT1 flag
PieCtrlRegs.PIEACK.all = PIEACK_GROUP10;
}
__interrupt void Cur_Eq_Loop(void)
{GpioDataRegs.GPATOGGLE.bit.NOP_LED=1;
PieCtrlRegs.PIEACK.all = PIEACK_GROUP1;
}



__interrupt void epwm1_tzint_isr(void) // Bu ve alttaki interruptlar OVP durumu için yazýldý.
{GpioDataRegs.GPASET.bit.OVP_LED = 1;
  PieCtrlRegs.PIEACK.all = PIEACK_GROUP2;

}

__interrupt void epwm2_tzint_isr(void) // bu interrupt OVP için yazýldý ve high voltage durumunda tüm pwmler low'a çekiliyor.
{
  PieCtrlRegs.PIEACK.all = PIEACK_GROUP2;
}
void InitializeScibRegisters(float fSciBaudRate){
    float lspclkdivider = 0;
    float lspclkfreq = 0;
#define SCI_FREQ        fSciBaudRate
#define SCI_PRD         (((float)lspclkdivider)/(SCI_FREQ*8))-1

    switch(ClkCfgRegs.LOSPCP.bit.LSPCLKDIV)
    {
        case 0:
            lspclkdivider = 1;
        case 1:
            lspclkdivider = 2;
        case 2:
            lspclkdivider = 4;
        case 3:
            lspclkdivider = 6;
        case 4:
            lspclkdivider = 8;
        case 5:
            lspclkdivider = 10;
        case 6:
            lspclkdivider = 12;
        case 7:
            lspclkdivider = 14;
        default:
            lspclkdivider = 4;
    }
    lspclkfreq = ((float)MFDS_LIB_CPU_FREQ)/lspclkdivider;

   ScibRegs.SCICCR.all = 0x0007;      // 1 stop bit,  No loopback
                                      // No parity,8 char bits,
                                      // async mode, idle-line protocol
   ScibRegs.SCICTL1.all = 0x0003;     // enable TX, RX, internal SCICLK,
                                      // Disable RX ERR, SLEEP, TXWAKE
   ScibRegs.SCICTL2.bit.TXINTENA = 0;
   ScibRegs.SCICTL2.bit.RXBKINTENA = 0;
   ScibRegs.SCIHBAUD.all = 0x0000;
   ScibRegs.SCILBAUD.all = round((((float)lspclkfreq)/(fSciBaudRate*8.0))-1);
   //ScibRegs.SCIHBAUD.all = 0x0002;
   //ScibRegs.SCILBAUD.all = 0x008B;
   ScibRegs.SCICCR.bit.LOOPBKENA = 0; // Enable loop back
   ScibRegs.SCIFFTX.all = 0xC022;
   ScibRegs.SCIFFRX.all = 0x0022;
   ScibRegs.SCIFFCT.all = 0x00;
   ScibRegs.SCICTL1.all = 0x0023;     // Relinquish SCI from Reset
   ScibRegs.SCIFFTX.bit.TXFIFORESET = 1;
   ScibRegs.SCIFFRX.bit.RXFIFORESET = 1;
}
void ConfigureGPIO(void) //GPIO pinleri aktileþtiriliyor.
{
EALLOW;
///////PWM GPIO Definitions
    GpioCtrlRegs.GPAPUD.bit.GPIO0=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO0=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO0=1; //Epwm1A gpio0 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO1=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO1=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO1=1; //Epwm1B gpio1 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO2=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO2=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO2=1; //Epwm2A gpio0 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO3=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO3=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO3=1; //Epwm2B gpio3 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO6=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO6=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO6=1; //Epwm4A gpio6 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO7=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO7=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO7=1; //Epwm4B gpio7 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO8=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO8=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO8=1; //Epwm5A gpio8 secildi


    GpioCtrlRegs.GPAPUD.bit.GPIO9=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO9=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO9=1; //Epwm5B gpio9 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO10=1; // pull-up disable
    GpioCtrlRegs.GPAGMUX1.bit.GPIO10=0;
    GpioCtrlRegs.GPAMUX1.bit.GPIO10=1; //Epwm6A gpio10 secildi

    GpioCtrlRegs.GPAPUD.bit.GPIO11=1; // pull-up disable
      GpioCtrlRegs.GPAGMUX1.bit.GPIO11=0;
      GpioCtrlRegs.GPAMUX1.bit.GPIO11=1; //Epwm6B gpio10 secildi


////////Çalýþma sýrasýnda bakýlacak GPIOLAR
    //GpioCtrlRegs.GPAPUD.bit.OVP_LED=1;
    GpioCtrlRegs.GPAGMUX2.bit.OVP_LED=0;
    GpioCtrlRegs.GPAMUX2.bit.OVP_LED=0; //OVP ledi aktifleþtirildi
    GpioCtrlRegs.GPADIR.bit.OVP_LED=1; // OVP GPIOsu output olarak seçildi.

    GpioCtrlRegs.GPBPUD.bit.OCP_LED=1;
    GpioCtrlRegs.GPBGMUX1.bit.OCP_LED=0;
    GpioCtrlRegs.GPBMUX1.bit.OCP_LED=0; //OCP ledi aktifleþtirildi
    GpioCtrlRegs.GPBDIR.bit.OCP_LED=1; // OCP GPIOsu output olarak seçildi.

    GpioCtrlRegs.GPAPUD.bit.NOP_LED=1;
    GpioCtrlRegs.GPAGMUX2.bit.NOP_LED=0;
    GpioCtrlRegs.GPAMUX2.bit.NOP_LED=0; //No Problem ledi aktifleþtirildi
    GpioCtrlRegs.GPADIR.bit.NOP_LED=1; // No Problem GPIOsu output olarak seçildi.

    GpioCtrlRegs.GPAPUD.bit.ON_LED=1;
    GpioCtrlRegs.GPAGMUX2.bit.ON_LED=0;
    GpioCtrlRegs.GPAMUX2.bit.ON_LED=0; //System ON ledi aktifleþtirildi
    GpioCtrlRegs.GPADIR.bit.ON_LED=1; // System ON GPIOsu output olarak seçildi.
#if TZ_TURNOFF
    GpioCtrlRegs.GPAPUD.bit.TZ_EWPM=1; // pull up disable
    GpioCtrlRegs.GPAINV.bit.TZ_EWPM=1; // high gelince çalýþsýn diye inputu invertledim.
    GpioCtrlRegs.GPAGMUX1.bit.TZ_EWPM=0;
    GpioCtrlRegs.GPAGMUX1.bit.TZ_EWPM=3; // GPIO14 Trip Zone seçildi.
    InputXbarRegs.INPUT1SELECT = 14;
#endif
#if OVP_PROT
    //Primary 1
    GpioCtrlRegs.GPAPUD.bit.GPIO15=1; // pull up disable
    GpioCtrlRegs.GPAINV.bit.GPIO15=1; // high gelince çalýþsýn diye inputu invertledim.
    GpioCtrlRegs.GPAGMUX1.bit.GPIO15=0;
    GpioCtrlRegs.GPAGMUX1.bit.GPIO15=3; // GPIO15 Trip Zone seçildi.
    InputXbarRegs.INPUT2SELECT = 15;
    //Primary 2
    GpioCtrlRegs.GPAPUD.bit.GPIO25=1; // pull up disable
    GpioCtrlRegs.GPAINV.bit.GPIO25=1; // high gelince çalýþsýn diye inputu invertledim.
    GpioCtrlRegs.GPAGMUX2.bit.GPIO25=0;
    GpioCtrlRegs.GPAGMUX2.bit.GPIO25=3; // GPIO15 Trip Zone seçildi.
    InputXbarRegs.INPUT3SELECT = 25;
#endif
//BT GPIOLARI
 GpioCtrlRegs.GPAGMUX2.bit.GPIO18 = 0;
 GpioCtrlRegs.GPAGMUX2.bit.GPIO19 = 0;
 GpioCtrlRegs.GPAMUX2.bit.GPIO18 = 2;
 GpioCtrlRegs.GPAMUX2.bit.GPIO19 = 2;


EDIS;
}


void EPwm1(void)
{
#if TZ_TURNOFF
    EALLOW;
        EPwm1Regs.TZSEL.bit.OSHT1 = 1;//1. event benim butona basýp kapatmam
        EPwm1Regs.TZSEL.bit.OSHT2 = 1;
        EPwm1Regs.TZSEL.bit.OSHT3 = 1;
        EPwm1Regs.TZCTL.bit.TZA = TZ_FORCE_LO;
        EPwm1Regs.TZCTL.bit.TZB= TZ_FORCE_LO;
        EPwm1Regs.TZEINT.bit.OST = 1;
     EDIS;
#endif
    EALLOW;
    EPwm1Regs.TBCTL.all = 0x00;
    EPwm1Regs.TBCTL.bit.CTRMODE = TB_COUNT_UPDOWN;  // Set counter to be up-down
    EPwm1Regs.TBCTL.bit.CLKDIV = 0;
    EPwm1Regs.TBCTL.bit.HSPCLKDIV =0;
    EPwm1Regs.TBPRD = pwmclk_frequency/switching_frequency/2;  /*period is set to be 10khz (for up down count)*/

    EPwm1Regs.CMPCTL.all = 0x00;
    EPwm1Regs.CMPCTL.bit.SHDWAMODE = CC_SHADOW;         //only active registers are used
    EPwm1Regs.CMPCTL.bit.LOADAMODE = CC_CTR_ZERO;
    EPwm1Regs.CMPCTL.bit.SHDWBMODE = 1;
    EPwm1Regs.CMPCTL.bit.LOADBMODE = CC_CTR_ZERO;

    EPwm1Regs.CMPA.bit.CMPA = EPwm1Regs.TBPRD / 2; //EPWM1A için seçildi
    EPwm1Regs.CMPB.bit.CMPB = EPwm1Regs.TBPRD / 2; //EPWM1B için seçildi

    EPwm1Regs.AQCTLA.all = 0x00;
    EPwm1Regs.AQCTLA.bit.CAU = AQ_SET;      //set high
    EPwm1Regs.AQCTLA.bit.CAD = AQ_CLEAR;    //Set low

    EPwm1Regs.AQCTLB.all = 0x00;
    EPwm1Regs.AQCTLB.bit.CAU = AQ_CLEAR;      //set low
    EPwm1Regs.AQCTLB.bit.CAD = AQ_SET;    //Set high

    EPwm1Regs.DBCTL.bit.OUT_MODE = 3;
    EPwm1Regs.DBCTL.bit.POLSEL = 2;
    EPwm1Regs.DBFED.bit.DBFED = dead_time / pwmclk_period;
    EPwm1Regs.DBRED.bit.DBRED = dead_time / pwmclk_period;
    EPwm1Regs.DBCTL2.all = 0x00;


    EPwm1Regs.TBCTL.bit.PHSEN = 0;
    EPwm1Regs.TBCTL.bit.SYNCOSEL = TB_CTR_ZERO; //SYNCOUT
    EDIS;
}

void EPwm2(void)
{
#if TZ_TURNOFF
    EALLOW;

            EPwm2Regs.TZSEL.bit.OSHT1 = 1; //1. event benim butona basýp kapatmam
            EPwm2Regs.TZSEL.bit.OSHT2 = 1;
            EPwm2Regs.TZSEL.bit.OSHT3 = 1;
            EPwm2Regs.TZCTL.bit.TZA = TZ_FORCE_LO;
            EPwm2Regs.TZCTL.bit.TZB= TZ_FORCE_LO;
            EPwm2Regs.TZEINT.bit.OST = 1;
#endif
    EDIS;
    EALLOW;
    EPwm2Regs.TBCTL.all = 0;
    EPwm2Regs.TBCTL.bit.CLKDIV =0;  // CLKDIV = 1
    EPwm2Regs.TBCTL.bit.HSPCLKDIV =0;   // HSPCLKDIV = 1
    EPwm2Regs.TBCTL.bit.CTRMODE = TB_COUNT_UPDOWN;  // up - down mode
    EPwm2Regs.TBCTL.bit.PHSEN = 1;      // enable phase shift for ePWM2
    EPwm2Regs.CMPCTL.all = 0x00;
    EPwm2Regs.CMPCTL.bit.SHDWAMODE = CC_SHADOW;         //only active registers are used
    EPwm2Regs.CMPCTL.bit.LOADAMODE = CC_CTR_ZERO;
    EPwm2Regs.CMPCTL.bit.SHDWBMODE = 1;
    EPwm2Regs.CMPCTL.bit.LOADBMODE = CC_CTR_ZERO;
    EPwm2Regs.AQCTLA.bit.CAU = AQ_SET;      //set high
    EPwm2Regs.AQCTLA.bit.CAD = AQ_CLEAR;    // Set low
    EPwm2Regs.AQCTLB.bit.CAU = AQ_CLEAR;    //set low
    EPwm2Regs.AQCTLB.bit.CAD = AQ_SET;      //set high
    EPwm2Regs.DBCTL.bit.OUT_MODE = 3;
    EPwm2Regs.DBCTL.bit.POLSEL = 2;
    EPwm2Regs.DBFED.bit.DBFED = dead_time / pwmclk_period;
    EPwm2Regs.DBRED.bit.DBRED = dead_time / pwmclk_period;
    EPwm2Regs.DBCTL2.all = 0x00;
    EPwm2Regs.TBCTL.bit.SYNCOSEL =  TB_SYNC_IN;
    EPwm2Regs.TBCTL.bit.PHSDIR = TB_DOWN;
    EPwm2Regs.TBPRD =(pwmclk_frequency)/(switching_frequency)/2;            // 10KHz - PWM signal
    EPwm2Regs.CMPA.bit.CMPA = EPwm2Regs.TBPRD/2; // EPWM2A için %50duty
    EPwm2Regs.CMPB.bit.CMPB = EPwm2Regs.TBPRD/2; // EPWM2B için %50duty
    #if FREQCONT
    EPwm2Regs.TBPHS.bit.TBPHS= 0;   // 1/3 phase shift
  #endif
#if PHASECONT
    EPwm2Regs.TBPHS.bit.TBPHS= 0;
#endif
    EDIS;
}

void EPwm6(void)
{
#if TZ_TURNOFF
    EALLOW;
            EPwm6Regs.TZSEL.bit.OSHT1 = 1; //1. event benim butona basýp kapatmam
            EPwm6Regs.TZSEL.bit.OSHT2 = 1;
            EPwm6Regs.TZSEL.bit.OSHT3 = 1;
            EPwm6Regs.TZCTL.bit.TZA = TZ_FORCE_LO;
            EPwm6Regs.TZCTL.bit.TZB= TZ_FORCE_LO;
            EPwm6Regs.TZEINT.bit.OST = 1;
#endif

    EDIS;
    EALLOW;
    EPwm6Regs.TBCTL.all = 0;
    EPwm6Regs.TBCTL.bit.CLKDIV =0;  // CLKDIV = 1
    EPwm6Regs.TBCTL.bit.HSPCLKDIV =0;   // HSPCLKDIV = 1
    EPwm6Regs.TBCTL.bit.CTRMODE = TB_COUNT_UPDOWN;  // up - down mode
    EPwm6Regs.TBCTL.bit.PHSEN = 1;      // enable phase shift for ePWM2
    EPwm6Regs.CMPCTL.all = 0x00;
    EPwm6Regs.CMPCTL.bit.SHDWAMODE = CC_SHADOW;         //only active registers are used
    EPwm6Regs.CMPCTL.bit.LOADAMODE = CC_CTR_ZERO;
    EPwm6Regs.CMPCTL.bit.SHDWBMODE = 1;
    EPwm6Regs.CMPCTL.bit.LOADBMODE = CC_CTR_ZERO;
    EPwm6Regs.AQCTLA.bit.CAU = AQ_SET;      //set high
    EPwm6Regs.AQCTLA.bit.CAD = AQ_CLEAR;    // Set low
    EPwm6Regs.AQCTLB.bit.CAU = AQ_CLEAR;    //set low
    EPwm6Regs.AQCTLB.bit.CAD = AQ_SET;      //set high
    EPwm6Regs.DBCTL.bit.OUT_MODE = 3;
    EPwm6Regs.DBCTL.bit.POLSEL = 2;
    EPwm6Regs.DBFED.bit.DBFED = dead_time / pwmclk_period;
    EPwm6Regs.DBRED.bit.DBRED = dead_time / pwmclk_period;
    EPwm6Regs.DBCTL2.all = 0x00;
    EPwm6Regs.TBCTL.bit.SYNCOSEL =  TB_SYNC_IN;
    EPwm6Regs.TBCTL.bit.PHSDIR = TB_DOWN;
    EPwm6Regs.TBPRD =(pwmclk_frequency)/(switching_frequency)/2;            // 10KHz - PWM signal
    EPwm6Regs.CMPA.bit.CMPA = EPwm6Regs.TBPRD/2; // EPWM2A için %50duty
    EPwm6Regs.CMPB.bit.CMPB = EPwm6Regs.TBPRD/2; // EPWM2B için %50duty
#if FREQCONT
    EPwm6Regs.TBPHS.bit.TBPHS= 0;   // 1/3 phase shift
#endif
#if PHASECONT
    EPwm6Regs.TBPHS.bit.TBPHS= 0;
#endif
    EDIS;
}

void EPwm5(void)
{
#if TZ_TURNOFF
    EALLOW;
            EPwm5Regs.TZSEL.bit.OSHT1 = 1; //1. event benim butona basýp kapatmam
            EPwm5Regs.TZSEL.bit.OSHT2 = 1;
            EPwm5Regs.TZSEL.bit.OSHT3 = 1;
            EPwm5Regs.TZCTL.bit.TZA = TZ_FORCE_LO;
            EPwm5Regs.TZCTL.bit.TZB= TZ_FORCE_LO;
            EPwm5Regs.TZEINT.bit.OST = 1;
#endif
    EDIS;
    EALLOW;
    EPwm5Regs.TBCTL.all = 0;
    EPwm5Regs.TBCTL.bit.CLKDIV =0;  // CLKDIV = 1
    EPwm5Regs.TBCTL.bit.HSPCLKDIV =0;   // HSPCLKDIV = 1
    EPwm5Regs.TBCTL.bit.CTRMODE = TB_COUNT_UPDOWN;  // up - down mode
    EPwm5Regs.TBCTL.bit.PHSEN = 1;      // enable phase shift for ePWM2
    EPwm5Regs.CMPCTL.all = 0x00;
    EPwm5Regs.CMPCTL.bit.SHDWAMODE = CC_SHADOW;         //only active registers are used
    EPwm5Regs.CMPCTL.bit.LOADAMODE = CC_CTR_ZERO;
    EPwm5Regs.CMPCTL.bit.SHDWBMODE = 1;
    EPwm5Regs.CMPCTL.bit.LOADBMODE = CC_CTR_ZERO;
    EPwm5Regs.AQCTLA.bit.CAU = AQ_SET;      //set high
    EPwm5Regs.AQCTLA.bit.CAD = AQ_CLEAR;    // Set low
    EPwm5Regs.AQCTLB.bit.CAU = AQ_CLEAR;    //set low
    EPwm5Regs.AQCTLB.bit.CAD = AQ_SET;      //set high
    EPwm5Regs.DBCTL.bit.OUT_MODE = 3;
    EPwm5Regs.DBCTL.bit.POLSEL = 2;
    EPwm5Regs.DBFED.bit.DBFED = dead_time / pwmclk_period;
    EPwm5Regs.DBRED.bit.DBRED = dead_time / pwmclk_period;
    EPwm5Regs.DBCTL2.all = 0x00;
    EPwm5Regs.TBCTL.bit.SYNCOSEL =  TB_SYNC_IN;
    EPwm5Regs.TBCTL.bit.PHSDIR = TB_DOWN;
    EPwm5Regs.TBPRD =(pwmclk_frequency)/(switching_frequency)/2;            // 10KHz - PWM signal
    EPwm5Regs.CMPA.bit.CMPA = EPwm5Regs.TBPRD/2; // EPWM2A için %50duty
    EPwm5Regs.CMPB.bit.CMPB = EPwm5Regs.TBPRD/2; // EPWM2B için %50duty
#if FREQCONT
    EPwm5Regs.TBPHS.bit.TBPHS= 0;   // 1/3 phase shift
#endif
    EDIS;
#if PHASECONT
    EPwm5Regs.TBPHS.bit.TBPHS= 0;
#endif
}
void ConfigureEPWM(void) //ADC de kullanýyorum
{
    EALLOW;
    EPwm3Regs.ETSEL.bit.SOCAEN    = 0;    // Disable SOC on A group
    EPwm3Regs.ETSEL.bit.SOCASEL    = 4;   // Select SOC on up-count
    EPwm3Regs.ETPS.bit.SOCAPRD = 1;       // Generate pulse on 1st event
    EPwm3Regs.TBPRD =20; // 1.250MHz sample
    EPwm3Regs.CMPA.bit.CMPA = 10 ;  // %50 duty cycle
    EPwm3Regs.TBCTL.bit.CTRMODE = 3;      // freeze counter
    EDIS;
}
void SetupADCEpwmFast(void)
{    EALLOW;
    //Primer akım okumaları bu iki ADC1 ve ADC4 aracılığı ile yapılmaktadır.
    AdcaRegs.ADCSOC1CTL.bit.CHSEL = 1; //SOC0 will convert ADCINA1
    AdcaRegs.ADCSOC1CTL.bit.ACQPS = 14; //SOC0 will use sample duration of 20 SYSCLK cycles
    AdcaRegs.ADCSOC1CTL.bit.TRIGSEL = 9; //SOC0 will begin conversion on ePWM3 SOCB

    AdcaRegs.ADCSOC4CTL.bit.CHSEL = 4; //SOC4 will convert ADCINA4
    AdcaRegs.ADCSOC4CTL.bit.ACQPS = 14; //SOC4 will use sample duration of 20 SYSCLK cycles
    AdcaRegs.ADCSOC4CTL.bit.TRIGSEL = 9; //SOC4 will begin conversion on ePWM3 SOCB

    AdcaRegs.ADCINTSEL1N2.bit.INT1SEL = 1; //EOC1 will set INT1 flag
    AdcaRegs.ADCINTSEL1N2.bit.INT1E = 1;   //enable INT1 flag
    AdcaRegs.ADCINTFLGCLR.bit.ADCINT1 = 1; //make sure INT1 flag is cleared


    EDIS;
}
void SetupADCEpwmSlow(void)//Bu adc setupı sayesinde ADCINA2 okunuyor.
{
    EALLOW;
    #if OPENLOOP
    AdcaRegs.ADCSOC2CTL.bit.CHSEL = 2; //SOC2 will convert ADCINA2
    AdcaRegs.ADCSOC2CTL.bit.ACQPS = 14; //SOC2 will use sample duration of 20 SYSCLK cycles
    AdcaRegs.ADCSOC2CTL.bit.TRIGSEL = 1; //SOC2 will begin conversion on TIMER0
   #endif
     AdcaRegs.ADCINTSEL1N2.bit.INT2SEL = 2; //EOC2 will set INT2 flag
     AdcaRegs.ADCINTSEL1N2.bit.INT2E = 1;   //enable INT2 flag
     AdcaRegs.ADCINTFLGCLR.bit.ADCINT2 = 1; //make sure INT1 flag is cleared
     EDIS;
}

void ConfigureADC(void)
{
    EALLOW;
    AdcaRegs.ADCCTL2.bit.PRESCALE = 6; //set ADCCLK divider to /4
    AdcSetMode(ADC_ADCA, ADC_RESOLUTION_12BIT, ADC_SIGNALMODE_SINGLE);
    AdcaRegs.ADCCTL1.bit.INTPULSEPOS = 1;
    AdcaRegs.ADCCTL1.bit.ADCPWDNZ = 1;
    DELAY_US(1000);//ADClerin power-up olmasý için örneklerde 1ms delay atýyorlar. Deðiþtirmedim. Furkan K. da öyle yapmýþ
    EDIS;
}

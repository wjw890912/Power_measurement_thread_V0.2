
  日志

		
		1、20160103完成ADE7878程序移植
		2、重写SPI驱动。
			

3、校准电表
		
校准方法
使用两种方法校准三相电表：一种使用基准电表，另一种
使用精确源。
基准电表方法要求信号源能够提供各种电流和电压，但精
度和稳定性无需太高。校准电表的读数与基准电表的读数
相比较，然后相应地调整校准量。
精确源方法要求信号源能够提供精确的电流和电压。校准
电表的读数与期望值相比较，然后相应地调整校准量。
两种方法均要求根据电表性能规格计算CFxDEN(x = 1、2或3)
寄存器、满量程电流(IFS)和满量程电压(VFS)。



 ADE7878的手册和校准手册写的有些蛋疼。前前后后连同PCB带校准程序花费月余时间。
 主要是校准这块。而校准这块花费时间最多的是
 对xWTHR的值计算。前期因为PGA增益开启了忘记了造成了门槛值总是PMAX小的多引入了一定的错误。
 后对Ifs 进行重新计算。

 还有一个电表的量程和精度的问题。

 电流量程条件是IRMS=Iin*10^6 。且PGA=16来计算 输入的最大电流。也即是Ifs
 电压量程条件是VRMS=Vin*10^4 .且PGA=0来计算	 输入的最大电压。也即是Vfs

功率这个可以通过Pfs=Vfs*Ifs*1  条件是coS=1来计算出来纯阻性负载的最大允许测量功率
				Pfs=5.5A*387v
				   =2131.5 (瓦特)

也就是说这个配置只能测量最大2131KW的功率。

 电流量程太小了，如果要提高电流量程怎么办？

 如果要提高量程那么需要牺牲的是电流通道的精度来解决，比如现在是uA级的你可以提高到mA级这样就可以提高量程了。
 不过我认为我们的目标是按照电表那种方式，所以应该做成标准表，也就是说测量范围在0-5A之间。然后配合In/5的电流互感器来使用
 这里主要是牵扯到一个电流增益寄存器的设定问题。手册也说了必须在范围（+100%~-100%）内 。同时PGA的增益也是一个配合参数哦！

 电压量程。这个不会超过220V如果需要增加量程可以适当的调整一下互感器的比率。

 
		
一、计算CFxDEN、IFS和VFS
		
		电表通常用标称电流(IN)、标称电压(VN)和电表常数(MC)
		来表征。电表常数指的是对于每千瓦时(kWh)的电能消
		耗，CF1、CF2或CF3脉冲输出引脚产生的脉冲数�


1、IFS的计算
现在使用兵子的50A:1mA的电流互感器CT。RL=200R的负载电阻。求Ifs.已知ADE7878的输入电压最高0.5V（峰值）
首先求出来在达到峰值的时的上的压降是0.5V时对应的有效值。
			 Vrms=0.5/√2=0.3536V
			 有如下关系
			 @Ifs/CT*RL=Vrms
			 数值代入上式可得
			 @Ifs/50000*200=0.3536V
			 解方程
			 @Ifs=88.4A


			 可以这样理解，按上式输入电流最大为88A计算
			因为变比CT=50000这个已知了负载电阻200R已知，所以可以计算出来二次侧感生电压为I1/CT=I2=88/50000*200R=0.352V
			注意这个已经是运放输入的极限了因为Urms=Um/√2=0.352V所以可以求出Um=500mV.
			但是我开启了PGA增益，而且为16倍增益。这样运放的输入应该乘以16变成了
			ADCinput= 0.352*16=	5.648V
			这个值显然是个天文数字。也就是说他的量程没有这么大，
			怎么办呢？降低量程就是了！于是可以求出来PGA增益为16时的运放输入，然后根据变比反推Ifs如下：

			ADCinput =0.353V这个是已知条件，手册中规定的，然后PGA=16所以输入到运放差分电压应该是
			 v2=0.353/16=0.022V
			 又知道负载电阻200R如此就可以计算出来二次侧电流
			 I2= V2/200R
			   = 0.022/200
			   = 0.0001A
			  已经知道CT=50000
			  代入得
			  I1=50000*0.0001103125= 5.5A
			  Ifs=I1=5.5A

			 



2、VFS的计算
现在使用兵子的电压互感器，说白了也是电流互感器，其原理是输入/输出=2MA/2MA,RL=200R的负载电阻.已知ADE7878的输入电压最高0.5V（峰值）
类似的先求出来在达到峰值的时的上的压降是0.5V时对应的有效值。	
			   Vrms=0.5/√2=0.3536V
			   (Vfs/150k)*200R= Vrms=0.3536V
				Vfs= (0.3536V/200)*150k
				=265.2v
			   显然我没有计算进线圈的阻抗，也就是上式是线圈的阻抗=0的情况下，为了更加逼近实际，线圈也要算上，
			   可以采取这个办法测量一次侧的电压V1在测量二次侧在200R的压降V2就可以得到一个比例VT
			   用这个VT可以推算出来VFS的实际值了
			   实际测量VT=227/0.207=1096
							
			   
			   VfS  = VT* Vrms
			        = 1096*0.3536
					= 387v

			     





3、计算CFxDEN
 CFxDEN是一个正整数，用于ADE7878的电能频率转换器
模块。CF1DEN、CF2DEN和CF3DEN寄存器利用此值进行
初始化。

			  CFxDEN = 10^3/(MC*10^α)

  MC是电表常数，用“脉冲数/kWh”表示。
10 ( < 0)决定各电能寄存器的1 LSB代表多少电能，如WATT-
小时和VAR-小时等。
例如，要校准的电表的MC = 6400脉冲/kWh，为了获得正整
数CxFDEN，选择 α= 3。

			 CFxDEN = 10^3/(6400*10^-3)	=156

 这里我们取MC=10000imp/kw.h

			 CFxDEN = 10^3/(10000*10^-3) = 100



  标称电流(IN)=0.45A (校准标称电流)
  标称电压(VN)=230v	  （校准标称电压）
  电表常数(MC) =10000impkw.h
  满量程电流(IFS)=5.5A
  满量程电压(VFS)=387V

 




 二、计算VLEVEL寄存器

			   通道计算数据参考
		   VLEVEL =Vfs/Vn*491520
		          =387v/220V*491520
				  =0xD3174

三、计算VxGAIN、IxGAIN、xVRMSOS、AIRMSOS值

	 /*
	 				  xVGAIN寄存器校准流程
	  配置A相电压通道的增益（用来校准VRMS值的偏差）
	   1、给A相通入一个标准电压Vn（实际是输入市电230V左右）
	   2、配置 AVGAIN值为0。配置AVRMSOS为0。
	   3、输出VRMS值（递推平均或者排序后中值滤波）
	   5、将输入的电压互感器采样电压进行转换为Vref=	230*10^4
	   6、然后读取当前的VRMS值=2398439
	   7、GAIN = (Vref/VRMS -1 )= {(230*10^4/2398439)-1}
	   		   =(-0.041)
	   	       =((+0.041)*2^23)结果转换为十六进制然后NOT，最后+1
			   =0xFFFAC084
	   8、再次测量VRMS寄存器值和预期值接近。OK完成校准。
	   9、真实的电压等于xVRMS寄存器的值除以10^4
	 */
 		SPIWrite4Bytes(AVGAIN,0xFFFAC084);//
	 /*			   xVRMSOS 寄存器校准流程
	   配置A相电压通道的增益之后再来校准这个寄存器。
	   1、给A相通入一个标准电压Vn=230V 需要测量的最小电压Vmin=0V
	   2、A相输入这个标准电压会产生一个VRMSn=230*10^4,当A相输入0V时也会对应一个噪声值实测VRMSmin=0x2f8
	   3、按理说输入是0V那么VRMS一定是0，但是并不是。所以需要校准到0.
	   4、根据公式： xVRMSOS =[((VRMSmin^2*Vn^2)-(VRMSn^2*Vmin^2)) /(128*(Vmin^2-Vn^2))]
	   						 =(0x2f8^2*230^2) /128*(-230^2)
	   						 =结果转换为十六进制然后NOT，最后+1
	   						 =0xFFFFEE60
	 */
	SPIWrite4Bytes(AVRMSOS ,0xFFFFEE60);//


	 /*
	 				  xIGAIN寄存器校准流程
	  配置A相电流通道的增益（用来校准IRMS值的偏差）
	   1、给A相通入一个标准电流In（实际是0.45A）
	   2、配置 AIGAIN值为0。配置AIRMSOS为0。PGA=0.
	   3、输出IRMS值（递推平均或者排序后中值滤波）
	   5、将输入的电压互感器采样电压进行转换为Vref=	0.45*10^6 (450000)
	   6、然后读取当前的IRMS值0x4556(17750)
	   7、Vref/VRMS=450000/17750= 25.3远远大于+-1之间的范围，显然需要配置运放增益
	   8、设置运放Gain增益= 16 也就是说GAIN增益寄存器的值为PGA1[2:0]=100b
	   9、再次读取IRMS值 0x47544(292164)
	   10、GAIN = (Iref/VRMS -1 )= {(0.45*10^6/292164)-1}
	   		   =(0.5402)
	   	       =((+0.5402)*2^23)结果转换为十六进制
			   =0x452648
	   8、再次测量IRMS寄存器值和预期值接近。OK完成校准。
	   9、真实的电压等于xIRMS寄存器的值除以10^6
	 */
						 
	 	SPIWrite2Bytes(Gain,0x0004);  //PGA=16倍增益
		SPIWrite4Bytes(AIGAIN ,0x452648);//


	   /*			   xIRMSOS 寄存器校准流程
	   配置A相电压通道的增益之后再来校准这个寄存器。
	   1、给A相通入一个标准电流In=0.45A 需要测量的最小电流Imin=0A
	   2、A相输入这个标准电流会产生一个IRMSn=0.45*10^6,当A相输入0A时也会对应一个噪声值实测IRMSmin=0xA20
	   3、按理说输入是0V那么IRMS一定是0，但是并不是。所以需要校准到0.
	   4、根据公式： xIRMSOS =[((IRMSmin^2*In^2)-(IRMSn^2*Imin^2)) /(128*(Imin^2-In^2))]
	   						 =(0xA20^2*0.45^2) /128*(-0.45^2)
	   						 =结果转换为十六进制然后NOT，最后+1
	   						 =0xFFFF32D0
	 */
	SPIWrite4Bytes(AIRMSOS  ,0XFFFF32D0);//

	  /*
			使用25W实测为23.1W

		 AWGAIN= [（AWATTref/AWATT）-1]
		 	   =23.1/29.4 -1
			   =0.2142
			   =结果乘以2^23后转换为16进制，取反后+1.
			   =0x0xFFE4927A
		  	使用100W实测为98.9W

		 AWGAIN= [（AWATTref/AWATT）-1]
		 	   =98.9/93.3 -1
			   =0.6
			   =结果乘以2^23后转换为16进制，取反后+1.
			   =0x0xFFF85138

			   将2者求和平均
			   =0xffee71d9
	  */
	SPIWrite4Bytes(AWGAIN  ,0xffee71d9);//


 没有必要执行基波有功电能校准，因为它们与针对总有功
电能计算的增益和失调应相同。然而，为了实现出色的精
度，仍然可以使用下面针对总有功电能校准所述的步骤校
准基波有功电能。


 四、总/基波有功电能校准

 为了补偿电流互感器、电阻分压器和ADC所
决定的幅度误差，需要校准增益寄存器，这可以全部通过
xIGAIN和xVGAIN寄存器来完成。此外，xWGAIN寄存器
还能补偿为ADE7878提供CLK信号的晶振所引起的时间测
量误差。该误差通常较小，可忽略。这种情况下，如果已
经得出xIGAIN和xVGAIN寄存器，则可以根据下式计算

1、WGAIN寄存器的校准

   
  功增益就 WATTHRref就是应该的功率。 AWATTHR是读取的实际功率 。

	AWGAIN = （WATTHRref / AWATTHR）-1


2、WATTOS寄存器的校准


3、	 xWTHR 门槛设定	（对于精准的电能计量非常重要，要反复的调整其在一分钟内的误差。然后再冠以长时间测试对比校准之）


	WTHR寄存器

	WTHR = (PMAX*fs*3600*10^n)/(Vfs*Ifs)


 其中，fS = 8 kHz，即DSP用于计算瞬时功率的频率。
 THR必须始终大于或等于PMAX——相电压和相电流具有
满量程幅度时ADE7878计算的有功功率：

	PMAX = 33,516,139

 如果THR小于PMAX，应调整VRMSREF或IRMSREF。


PMAX = 33516139 = 0x1FF6A6B

WTHR=(PMAX*fs*3600*10^n)/(Vfs*Ifs)
	=33516139*8000*3600*10^-3/ 5.5A*387.5V
	= 965264803200 /2131.5
	= 0x1AFE0CDA (5.5A)
THR0=0x00FE0CDA
THR1=0x0000001A	



4、计算LINECYC寄存器以确定累计时间	 【α=-6】

	 LINECYC >=	   10000 *2*(10^α*3600/VN*IN) *(256*10^3/period)
			  =    10000 *2*10^-3*3600/230*0.45*256*10^3/5120
			  =	   347









modbus

+移植modbus到NUC400xx上。使用modbus读取holdreg 寄存器读取电能结果。
20160303



+互感器跨入非线性区
+由于互感器使用的负载电阻为200R测试现象为小功率负载正常<5A，大功率负载=10A  时穿心导线影响到互感器的电流值，导致功率
+波动。继而重新把200R的电阻换为20R，确保互感器在线性区间。之后重新校准使用USE_0_55A_RANG	校准参数
20160316


+移植eefsl from github
+基于EEPROM的文件系统
+include <fcntl.h>并修改只保留必要的宏定义

+修改eefsl的struct void *全部改为uint8*
+time(NULL)重新修改为	Get_current_time()已完成编译
/* This macro defines the time interface function.  Defaults to time(NULL)  */
/*reused it to  Get_current_time() that reched complited*/
#define  Get_current_time()	  88888
#define EEFS_LIB_TIME                          Get_current_time()
+底层接口  尚未做修改，现在编译完成，准备测试。
/* These macros define the lower level EEPROM interface functions.  Defaults to memcpy(Dest, Src, Length) */
#define EEFS_LIB_EEPROM_WRITE(Dest, Src, Length) memcpy(Dest, Src, Length)
#define EEFS_LIB_EEPROM_READ(Dest, Src, Length)  memcpy(Dest, Src, Length)
#define EEFS_LIB_EEPROM_FLUSH 


 +修改了read中求长度字节 原因是如果是MIN将无法有效读取文件。
 EEFS_LibRead(myfile, Eepromdata, len);
 BytesToRead = EEFS_MAX((EEFS_FileDescriptorTable[FileDescriptor].FileSize - EEFS_FileDescriptorTable[FileDescriptor].ByteOffset), Length);
			   （MIN）
+修改了myfile=EEFS_LibOpen(&myeefsInodeTable, "eeprom", O_CREAT, 1);
中的mode属性，
原来为：EEFS_FileDescriptorTable[FileDescriptor].Mode = (EEFS_FCREAT | EEFS_FWRITE );
增加读使能
EEFS_FileDescriptorTable[FileDescriptor].Mode = (EEFS_FCREAT | EEFS_FWRITE | EEFS_FREAD);
20160416





+增加一个自定义的协议解析方法。如果要开启那么首先应该在头文件
#include "port.h"	 中定义一个宏：
/*use bt protocol ???yes define it or not .*/
#define _USE_BITTLE_PROTOCOL
+临时定义的协议如下格式如下
/*
						MYSELF PROTOCOL FARME

		         0	1	  2		  3		4-5	 6-7   8-9	   10	11
主机发送：		FF AA  + ADDR + FUNC + STA + STO + CRC +   0D  0A
		        1B 1B   1B     1B	  2B	2B	  2B	  1B 1B	
		  	   	
从机应答：		FF AA  + ADDR + FUNC + data + CRC +   0D  0A  
		  			
				CRC 校验与等于除CRC和0d0a以外的所有数据的校验
*/

+在btfunc.c中解析 接收到的数据即可。注意
eBTFuncReadHoldingRegister( UCHAR * pucFrame, USHORT * usLen )
这个pucFrame是从功能码开始的。并且他指向原始数据输入输出的BUF中，所以他还把数据从他带出去，usLen的用法同样。
仅仅实现了一个功能码为0x30的读保持寄存器。保持寄存器数据为虚构
			pucFrame[1] =0x01;
			pucFrame[2] =0x02;
			pucFrame[3] =0x03;
			pucFrame[4] =0x04;
			pucFrame[5] =0x05;
			pucFrame[6] =0x06;	 //data area
			pucFrame[7] =0x07;
			pucFrame[8] =0x08;

+主要是为了进入比特系统的房态中。同时也为了兼容了MODBUS。
20160408



$$$重大改动$$$
++++++增加rtos操作系统并分配modbus poll和 电能测量为2个独立线程。目前运行稳定。20160411


+通过抓包分析姚工协议，修改BT协议为兼容姚工样机查询的485命令并发送假的测试数据已经OK。20160412

+为了兼容姚工主机查询的ID从8号改为2号以兼容之。从机地址变为2！ 20160412

@重新整理目录，删除新塘BSP包内不必要的文件。20160413






















							直接采样式三相贴片分压式





			1、最大输入电流5A
			2、最大输入电压630V（正常工作电压230V）
			3、如果要测量超过5A的电流，可以根据系统要求灵活改变外置电流互感器的变比来实现
			比如要测量目标系统电流是10A最大，则可以选用精度1%的10/5的通用电力电流互感器。依次类推。测量结果乘以变流比2即可。



		

		一、电流采样电路
		1、阻值确定
		   已知电流采样根据ADE7878输入的最大电压幅值500MV。
		   可知
		   UinRMS=500/√2=353mV	=0.353V
		   所以采样电阻RI最大为
		   RI=V/I= 0.353V/5A= 0.0706ohm
		   取满量程的80%
		   =   0.0706ohm*0.8= 0.056 
		   根据市面可实现采样阻值接近为
		   0.05ohm,
		2、功率确定
			最大电流5A，采样电阻 0.05ohm,
			功率=I2R=25*0.05= 1.25w
			留出来热余量取2W电阻

	    3、可以选取2512封装 2W 1% 0.05ohm的精密电阻作为电流采样

		二、电压采样电路

	    1、采样最大电流的确定
		  已知互感器允许输入电压最大为630V
		  根据互感器型电压互感器二次电流输出最大电流为2mA，则电阻分压试电阻亦参照
		2、分压电阻总和的确定
		  R=U/I
		  R=630/2mA
		   = 315kohm
	    3、 电压采样电阻阻值确定
		  已知电压采样根据ADE7878输入的最大电压幅值500MV。
		   可知
		   UinRMS=500/√2=353mV	=0.353V
		   所以采样电阻Rv最大为
		   Rv=V/I= 0.353V/0.002A= 176.5ohm
		   考虑到正常使用的电压不会超过400V
		   并且实际并没有这个大的电阻所以取近似值200R
		 4、实际电阻分压值的确定
			由于当电流一定时电阻越大其功耗也越大，
			所以这样前面计算的总的电阻值需要分解为几段电阻串联。
			以减小器件的物理尺寸
			已知
			R总=315Kohm
			采样电阻200ohm
		为了便于实际选取电阻并且采样电阻阻值较大所以近似R总=3000200ohm，实际满量程电流略大于2MA。
		则有
		   R总=	300200R-(75K*4)-200=0

			所以选取四只75K的电阻，上端2只下端两只中间放200R采样电阻即可，
			他们的功率是
			 I2R=75K*0.002*0.002A=0.3w
			 I2R=200R*0.002*0.002A=0.0008w
			应该选用1206（1/2W）的精度1%的75K电阻四只+1只200R精度1%的0603（1/8W贴片电阻即可。


		热地--不安全





								  













  20160430开始写称重传感器代码



  中断优先级
   	NVIC_SetPriority(SysTick_IRQn,1);  SYSTICK
	NVIC_SetPriority(GPB_IRQn, 5 );	ir and ADE7878
	NVIC_SetPriority(TMR1_IRQn,4); ir
	NVIC_SetPriority(UART3_IRQn, 2 ); 485
	NVIC_SetPriority(TMR0_IRQn, 3 );  485



/*
这是MeArm机械臂的控制程序2.0版本
控制电路版在20230505完成设计
其中，D5控制B底座，D6控制C爪子，D9控制F前臂，D10控制R后臂
蓝牙模块程序单独烧录
*/

#include<Servo.h>
#include <SoftwareSerial.h> // 引入SoftwareSerial库

SoftwareSerial bt(0, 1); // 定义蓝牙模块的软串口对象，接收引脚为2，发送引脚为3


Servo base,fArm,rArm,claw;

//电机极限值，const不能改变
const int baseMin = 0;
const int baseMax = 180;
const int rArmMin = 70;
const int rArmMax = 150;
const int fArmMin = 56;
const int fArmMax = 123;
const int clawMin = 60;
const int clawMax = 125;
//不要改动
int DSD = 15;//电机运动延迟15ms，控制运动速度
bool mode;  //mode = 1:指令，mode = 0：手柄
int moveStep = 3; //每一次按下手柄，舵机移动量

void setup() {
  base.attach(5);  //base舵机接引脚5
  delay(200);
  rArm.attach(10);
  delay(200);
  fArm.attach(9);
  delay(200);
  claw.attach(6);
  delay(200);

  base.write(90);
  delay(10);
  rArm.write(90);
  delay(10);
  fArm.write(90);
  delay(10);
  claw.write(90);
  delay(10);

Serial.begin(9600);
bt.begin(9600); // 初始化蓝牙模块

Serial.println("欢迎来到我的机械臂~");
}

void loop() {
 if(Serial.available()>0){
   char serialCmd = Serial.read();
   if(mode == 1){
     armDataCmd(serialCmd);//指令模式
   }else{
     armJoyCmd(serialCmd);//手柄模式
   }
 }

 // 将机械臂当前状态发送回蓝牙模块

}

void armDataCmd(char serialCmd){ //指令
  if( serialCmd == 'w' || serialCmd == 's' || serialCmd == 'a' || 
  serialCmd == 'd' || serialCmd == '5' || serialCmd == '4'|| 
  serialCmd == '6' || serialCmd == '8'){
    Serial.println("注意：处于指令模式！");
    delay(100);
    while(Serial.available()>0)
    char wrongCommand = Serial.read();//清除串口缓存的错误指令
    return;
  }

if(serialCmd == 'b' || serialCmd == 'c' || serialCmd == 'f' || serialCmd == 'r'){
  int servoData = Serial.parseInt();
  servoCmd(serialCmd,servoData,DSD);//运行参数。舵机名，目标角度，延迟（速度）
}else{
  switch(serialCmd){
    case 'm': //手柄模式
    mode = 0;
    Serial.println("指令：切换至手柄模式！");
    break;
    case 'o': //输出舵机状态信息
    reportStatus();
    break;
    case '1':
    armIniPos();
    break;
    default: //未知指令反馈
    Serial.println("未知指令");
                  }
      }
}

void armJoyCmd(char serialCmd){ //手柄按键
if(serialCmd == 'b' || serialCmd == 'c' || serialCmd == 'f' || serialCmd == 'r'){
  Serial.println("注意：处于手柄模式！");
  delay(100);
  while(Serial.available()>0)
  char wrongCommand = Serial.read();
  return;
  }

  int baseJoyPos;
  int rArmJoyPos;
  int fArmJoyPos;
  int clawJoyPos;

if (bt.available()) { // 如果蓝牙模块有数据可读取
    char servoCmd = bt.read(); // 读取指令


  switch(serialCmd){

    case 'a': //Base向左
    Serial.println("收到命令：底座向左");
    baseJoyPos = base.read() - moveStep;
    servoCmd('b',baseJoyPos,DSD);
    break;

    case 'd': //右
    Serial.println("收到命令：底座向右");
    baseJoyPos = base.read() + moveStep;
    servoCmd('b',baseJoyPos,DSD);
    break;

    case 's': //后臂下
    Serial.println("收到命令：后臂向下");
    baseJoyPos = rArm.read() + moveStep;
    servoCmd('r',rArmJoyPos,DSD);
    break;

    case 'w': //上
    Serial.println("收到命令：后臂上");
    baseJoyPos = rArm.read() - moveStep;
    servoCmd('r',rArmJoyPos,DSD);
    break;

    case '8': //前臂上
    Serial.println("收到命令：前臂向上");
    baseJoyPos = fArm.read() + moveStep;
    servoCmd('f',fArmJoyPos,DSD);
    break;

    case '5': //右
    Serial.println("收到命令：前臂下");
    baseJoyPos = fArm.read() - moveStep;
    servoCmd('f',fArmJoyPos,DSD);
    break;

    case '4': //爪子关
    Serial.println("收到命令：关爪子");
    baseJoyPos = claw.read() + moveStep;
    servoCmd('c',clawJoyPos,DSD);
    break;

    case '6': //爪子开
    Serial.println("收到命令：开爪子");
    baseJoyPos = claw.read() - moveStep;
    servoCmd('c',clawJoyPos,DSD);
    break;

    case 'm': //指令模式
    mode = 1;
    Serial.println("命令：处于指令模式！");
    break;

    case 'o':
    reportStatus();
    break;

    case 'i':
    armIniPos();
    break;
    default:
    Serial.println("未知命令！");
    return;
  }
}

void servoCmd(char servoName,int toPos,int servoDelay)
{
  Servo servo2go;
  Serial.println("");
  Serial.print("+Command: Servo");
  Serial.print(servoName);
  Serial.print("to");
  Serial.print(toPos);
  Serial.print(" at servoDelay value ");
  Serial.print(servoDelay);
  Serial.println(".");
  Serial.println("");

  int fromPos;//建立变量，储存电机起始运动角度值

  switch(servoName){
    case 'b':
    if(toPos >= baseMin && toPos <= baseMax){
      servo2go = base;
      fromPos = base.read(); //获取当前电机角度值，用于电机运动起始角度值
      break;
    }else{
      Serial.println("注意： 舵机角度值超出限制！");
      return;
    }
    case 'c':
    if(toPos >= clawMin && toPos <= clawMax){
      servo2go = claw;
      fromPos = claw.read(); //获取当前电机角度值，用于电机运动起始角度值
      break;
    }else{
      Serial.println("注意： 舵机角度值超出限制！");
      return;
    }
    case 'f':
    if(toPos >= fArmMin && toPos <= fArmMax){
      servo2go = fArm;
      fromPos = fArm.read(); //获取当前电机角度值，用于电机运动起始角度值
      break;
    }else{
      Serial.println("注意： 舵机角度值超出限制！");
      return;
    }
    case 'r':
    if(toPos >= rArmMin && toPos <= rArmMax){
      servo2go = rArm;
      fromPos = rArm.read(); //获取当前电机角度值，用于电机运动起始角度值
      break;
    }else{
      Serial.println("注意： 舵机角度值超出限制！");
      return;
    }
  }

//指示电机运行
if(fromPos <= toPos){
  for(int i=fromPos; i<=toPos;i++){
    servo2go.write(i);
    delay(servoDelay);
  }
}else{
  for(int i=fromPos;i>=toPos;i--){
    servo2go.write(i);
    delay(servoDelay);
  }
}
}

void reportStatus(){  //舵机状态信息
  Serial.println("");
  Serial.println("");
  Serial.println("+ Robot-Arm Status Report +");
  Serial.print("Claw Position: "); Serial.println(claw.read());
  Serial.print("Base Position: "); Serial.println(base.read());
  Serial.print("Rear  Arm Position:"); Serial.println(rArm.read());
  Serial.print("Front Arm Position:"); Serial.println(fArm.read());
  Serial.println("++++++++++++++++++++++++++");
  Serial.println("");
}
 
void armIniPos(){
  Serial.println("+Command: Restore Initial Position.");
  int robotIniPosArray[4][3] = {
    {'b', 90, DSD},
    {'r', 90, DSD},
    {'f', 90, DSD},
    {'c', 90, DSD} 
  };
 
  for (int i = 0; i < 4; i++){
    servoCmd(robotIniPosArray[i][0], robotIniPosArray[i][1], robotIniPosArray[i][2]);
  }
}

# Mobile-screen-clicker
手机屏幕点击器
// 引入步进电机控制库（必须安装）
#include <AccelStepper.h>

/*==== 硬件连接定义 ====*/
// 电机1控制引脚（接ULN2003驱动板）
#define IN1 8    // 接驱动板IN1（通常对应蓝线）
#define IN2 9    // 接驱动板IN2（粉线）
#define IN3 10   // 接驱动板IN3（黄线）
#define IN4 11   // 接驱动板IN4（橙线）

// 电机2控制引脚（另一个驱动板）
#define IN5 4    // 第二个驱动板IN1
#define IN6 5    // IN2
#define IN7 6    // IN3
#define IN8 7    // IN4

// 按钮引脚定义
#define BTN_CW  2   // 电机1正转按钮（按下=接地）
#define BTN_CCW 3   // 电机1反转按钮
#define BTN_STOP 0  // 系统复位按钮
#define BTN_CW2 12  // 电机2正转按钮
#define BTN_CCW2 13 // 电机2反转按钮

/*==== 全局状态变量 ====*/
// 创建两个步进电机对象（四线全步进模式）
AccelStepper stepper1(AccelStepper::FULL4WIRE, IN1, IN3, IN2, IN4); // 特殊引脚顺序
AccelStepper stepper2(AccelStepper::FULL4WIRE, IN5, IN7, IN6, IN8);

bool isRunning1 = false;    // 电机1运行状态
bool isRunning2 = false;    // 电机2运行状态
int motorDirection1 = 0;    // 1=正转 -1=反转 0=停
int motorDirection2 = 0;    // 同上

// 按钮状态缓存（用于检测变化）
bool lastBtnCW = HIGH;      // 电机1正转前次状态
bool lastBtnCCW = HIGH;     // 电机1反转前次状态
bool lastBtnSTOP = HIGH;    // 复位按钮前次状态
bool lastBtnCW2 = HIGH;     // 电机2正转前次状态
bool lastBtnCCW2 = HIGH;    // 电机2反转前次状态

/*==== 初始化设置 ====*/
void setup() {
    Serial.begin(115200); // 启动串口通信
    
    // 配置按钮引脚为输入模式（启用内部上拉电阻）
    pinMode(BTN_CW, INPUT_PULLUP);   // 按钮未按下时引脚为HIGH
    pinMode(BTN_CCW, INPUT_PULLUP);  // 按下时变为LOW
    pinMode(BTN_STOP, INPUT_PULLUP);
    pinMode(BTN_CW2, INPUT_PULLUP);
    pinMode(BTN_CCW2, INPUT_PULLUP);
    
    pinMode(LED_BUILTIN, OUTPUT); // 板载LED指示灯
    
    // 配置电机1参数
    stepper1.setMaxSpeed(500);      // 最大转速（步/秒）
    stepper1.setAcceleration(300);  // 加速度（步/秒²）
    stepper1.enableOutputs();       // 使能电机驱动
    
    // 配置电机2参数（同上）
    stepper2.setMaxSpeed(500);
    stepper2.setAcceleration(300);
    stepper2.enableOutputs();
    
    Serial.println("System Ready"); // 初始化完成提示
}

/*==== 主循环 ====*/
void loop() {
    checkButtons(); // 检测按钮状态
    
    // 持续驱动运行中的电机
    if (isRunning1) stepper1.runSpeed(); // 以设定速度持续运行
    if (isRunning2) stepper2.runSpeed();
}

/*==== 按钮检测与处理 ====*/
void checkButtons() {
    // 读取当前按钮状态（LOW表示按下）
    bool currentBtnCW = digitalRead(BTN_CW);
    bool currentBtnCCW = digitalRead(BTN_CCW);
    bool currentBtnSTOP = digitalRead(BTN_STOP);
    bool currentBtnCW2 = digitalRead(BTN_CW2);
    bool currentBtnCCW2 = digitalRead(BTN_CCW2);
    
    // 检测复位按钮按下（下降沿触发）
    if (currentBtnSTOP == LOW && lastBtnSTOP == HIGH) {
        resetSystem(); // 执行系统复位
    }
    
    /*-- 电机1控制逻辑 --*/
    if (currentBtnCW == LOW) {        // 正转按钮按下
        startMotor1(true);            // 启动正转
    } else if (currentBtnCCW == LOW) { // 反转按钮按下
        startMotor1(false);           // 启动反转
    } else {                          // 按钮未按下
        stopMotor1();                 // 停止电机
    }
    
    /*-- 电机2控制逻辑（同上）--*/
    if (currentBtnCW2 == LOW) {
        startMotor2(true);
    } else if (currentBtnCCW2 == LOW) {
        startMotor2(false);
    } else {
        stopMotor2();
    }
    
    // 更新按钮状态缓存
    lastBtnCW = currentBtnCW;
    lastBtnCCW = currentBtnCCW;
    lastBtnSTOP = currentBtnSTOP;
    lastBtnCW2 = currentBtnCW2;
    lastBtnCCW2 = currentBtnCCW2;
}

/*==== 电机1控制函数 ====*/
void startMotor1(bool forward) {
    if (!isRunning1) { // 仅在未运行时启动
        isRunning1 = true;
        motorDirection1 = forward ? 1 : -1; // 方向标记
        stepper1.setSpeed(forward ? 400 : -400); // 设置转速
        stepper1.enableOutputs();   // 使能驱动
        digitalWrite(LED_BUILTIN, HIGH); // LED亮起
        Serial.println(forward ? "Motor1 Forward" : "Motor1 Reverse");
    }
}

void stopMotor1() {
    if (isRunning1) {
        isRunning1 = false;
        motorDirection1 = 0;
        stepper1.setSpeed(0);     // 速度归零
        stepper1.disableOutputs(); // 关闭驱动（省电）
        Serial.println("Motor1 Stopped");
    }
}

/*==== 电机2控制函数（逻辑同电机1）====*/
void startMotor2(bool forward) {
    if (!isRunning2) {
        isRunning2 = true;
        motorDirection2 = forward ? 1 : -1;
        stepper2.setSpeed(forward ? 400 : -400);
        stepper2.enableOutputs();
        Serial.println(forward ? "Motor2 Forward" : "Motor2 Reverse");
    }
}

void stopMotor2() {
    if (isRunning2) {
        isRunning2 = false;
        motorDirection2 = 0;
        stepper2.setSpeed(0);
        stepper2.disableOutputs();
        Serial.println("Motor2 Stopped");
    }
}

/*==== 系统复位函数 ====*/
void resetSystem() {
    Serial.println("System Reset");
    delay(500); // 等待0.5秒防止误触
    
    // 软复位指令（跳转到程序起始地址）
    // 注意：这是非标准方法，可能导致不稳定
    asm volatile ("jmp 0");  
}

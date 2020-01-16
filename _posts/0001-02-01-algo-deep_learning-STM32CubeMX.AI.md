---
layout: post
algo: true
comments: true
title:  "deep learning - STM32CubeMX"
excerpt: "..."
tag:
- algo
- deeplearning
- vae

---



# reference

- STM32CubeMX：https://www.st.com/en/embedded-software/x-cube-ai.html
- CubeAI相关文档：file:///D:/STMicroelectronics/STM32CubeMX/Repository/Packs/STMicroelectronics/X-CUBE-AI/4.0.0/Documentation/index.html
- 示例-FP-AI： https://www.st.com/content/st_com/en/products/embedded-software/mcu-mpu-embedded-software/stm32-embedded-software/stm32-ode-function-pack-sw/fp-ai-sensing1.html
- 示例-keras-anomaly-detection：     https://github.com/chen0040/keras-anomaly-detection

# project

## src-CubeAI

- Home  >>  Manage software installations  >>  INSTALL/REMOVE  >>  STMicroelectronics >> X-CUBE-AI >> 最新版本 >> Install Now
- File -> New Project -> MCU Select
- System Core >> RCC >> High Speed Clock(HSE)**=BYPASS Clcok Source**
- Clock Configuration**=最大**
- Connectivity >> USART1 >> Mode**=Asynchronous**，选择引脚
- Additional Software(上)
  - X-CUBE-AI/Application  >>  Selection**=SystemPerformance**
  - X-CUBE-AI/X-CUBE-AI  >>  Selection**打勾**
- Additional Softwares(左下) >> STMicroelectronics.X-CUBE-AI
  - Artificial Intelligence X-CUBE-AI**打勾**
  - Artificial Intelligence Application**打勾**
  - Add network=**network_name** >> **Keras** >> **Saved model** >> Model Bowse（.h5文件) >> Compression=None >> Analyze >> Validate on desktop
  - 生成代码时提示未配置Platform Settings，yes忽略



## files

|   |   |
| ---- | ---- |
| CubeMX-Project\Middlewares | software\proj\Middlewares |
| CubeMX-Project\Src\network_name.c |software\proj\Middlewares\ST\User\network_name.c |
| CubeMX-Project\Src\network_name_data.c | software\proj\Middlewares\ST\User\network_name_data.c |
| CubeMX-Project\Src\app_x-cube-ai.c     | software\proj\Middlewares\ST\User\app_x-cube-ai.c     |
| CubeMX-Project\Inc\network_name.h      | software\proj\Middlewares\ST\User\network_name.h      |
| CubeMX-Project\Inc\network_name_data.h | software\proj\Middlewares\ST\User\network_name_data.h |
| CubeMX-Project\Inc\bsp_ai.h            | software\proj\Middlewares\ST\User\bsp_ai.h            |
| CubeMX-Project\Inc\constants_ai.h      | software\proj\Middlewares\ST\User\constants_ai.h      |
| CubeMX-Project\Inc\RTE_Components.h | software\proj\Middlewares\ST\User\RTE_Components.h |
| ---- | software\proj\Middlewares\ST\User\network_name_app.c  |
|           | software\proj\Middlewares\ST\User\network_name_app.h    |



## modify


- 修改串口打印冲突

  ```
  @aiSystemPerformance.c
  -int fputc(int ch, FILE *f)
  -{
  -      HAL_UART_Transmit(&UartHandle, (uint8_t *)&ch, 1,
  -            HAL_MAX_DELAY);
  -    return ch;
  -}
  
           printf("Profiling mode (%d)...\r\n", profiling_factor);
  -        fflush(stdout); 
  
               printf(".");
  -            fflush(stdout);  
  
  -extern UART_HandleTypeDef UartHandle;
  +extern UART_HandleTypeDef Uart1_Handle; //使用自己定义的串口
  
  @bsp_ai.h
  -#define UartHandle huart1
  ```



- 模型运行函数

  CubeAI生成的代码中含有一个运行模型的函数**aiTestPerformance**，用于测试，以该函数作为模板编写实际使用的运行函数：

  ```
  @aiSystemPerformance.c
  #include "algo_anomaly_detection_app.h"
  int AI_Process(ai_float *user_input,float *result)
  {
      int iter;
      uint32_t irqs;
      ai_i32 batch;
      int niter;
  
      struct dwtTime t;
      uint64_t tcumul;
      uint32_t tstart, tend;
      uint32_t tmin;
      uint32_t tmax;
  
      ai_buffer ai_input[1];
      ai_buffer ai_output[1];
  	
      ai_float user_output[3]={0,0,0};
  
  
      if (net_ctx[0].handle == AI_HANDLE_NULL) 
  		{
          printf("E: network handle is NULL\r\n");
          return -1;
      }
  
  
      if (profiling_mode)
          niter = _APP_ITER_ * profiling_factor;
      else
          niter = _APP_ITER_;
  
      printf("\r\nRunning PerfTest on \"%s\" with random inputs (%d iterations)...\r\n", net_ctx[0].report.model_name, niter);
      irqs = disableInts();
  
      /* reset/init cpu clock counters */
      tcumul = 0ULL;
      tmin = UINT32_MAX;
      tmax = 0UL;
  
      memset(&ia_malloc,0,sizeof(struct ia_malloc));
  
      if ((net_ctx[0].report.n_inputs != 1) ||(net_ctx[0].report.n_outputs != 1))
      {
          printf("E: multiple I/O network support not yet supported...\r\n");
          HAL_Delay(100);
          return -1;
      }
  
      ai_input[0] = net_ctx[0].report.inputs[0];
      ai_output[0] = net_ctx[0].report.outputs[0];
      ai_input[0].n_batches  = 1;
      ai_output[0].n_batches = 1;
      ai_input[0].data = AI_HANDLE_PTR(user_input); 
      ai_output[0].data = AI_HANDLE_PTR(out_data);
  		
      for (iter = 0; iter < niter; iter++) 
  		{
        /* Fill input vector */
          const ai_buffer_format fmt = AI_BUFFER_FORMAT(&ai_input[0]);
  
          dwtReset();
          tstart = dwtGetCycles();
  
          batch = ai_mnetwork_run(net_ctx[0].handle, &ai_input[0], &ai_output[0]);
  				
  				user_output[0] = ((float*)(ai_output[0].data))[0];  
  				user_output[1] = ((float*)(ai_output[0].data))[1];  
  				user_output[2] = ((float*)(ai_output[0].data))[2];		  					
  				
          if (batch != 1) 
  				{
              aiLogErr(ai_mnetwork_get_error(net_ctx[0].handle), "ai_mnetwork_run");
              break;
          }
  
          tend = dwtGetCycles() - tstart;
  
  
          if (tend < tmin)
              tmin = tend;
  
          if (tend > tmax)
              tmax = tend;
  
          tcumul += (uint64_t)tend;
      }
      restoreInts(irqs);
  
      tcumul /= (uint64_t)iter;
  
      dwtCyclesToTime(tcumul, &t);
  
  		*result = (float)(sqrt(pow((double)user_output[0],2)+pow((double)user_output[1],2)+pow((double)user_output[2],2))); 
  		printf("AI Result : %f  \r\n",*result); 
  		
  		return 0;	
  }
  
  ```






## keil MDK

- source

  Drivers\CMSIS\DSP_Lib\Source\BasicMathFunctions\arm_dot_prod_f32.c
  Drivers\CMSIS\DSP_Lib\Source\BasicMathFunctions\arm_dot_shift_q15.c
  Drivers\CMSIS\DSP_Lib\Source\BasicMathFunctions\arm_dot_shift_q7.c

  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_float_to_q7.c
  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_float_to_q15.c
  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_q7_to_float.c
  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_q15_to_float.c
  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_q7_to_q15.c
  Drivers\CMSIS\DSP_Lib\Source\SupportFunctions\arm_q15_to_q7.c

  Drivers\CMSIS\DSP_Lib\Source\MatrixFunctions\arm_mat_init_f32.c

  algo\algo_name\src\ * .c
  algo\algo_name\src\AI\Lib\NetworkRuntime400_CM4_Keil.lib
  algo\algo_name\src\Application\SystemPerformance\Src\aiSystemPerformance.c
  algo\algo_name\user\ * .c
  algo\algo_name\user\stm32l433 \  * .c

- include

  ..\Middlewares\ST\AI\Inc
  ..\Middlewares\ST\Application\SystemPerformance\Inc
  ..\Middlewares\ST\User

- define

  ARM_MATH_CM4,__FPU_PRESENT=1



# 获取数据



- 使用EXCEL打开.csv文件，选择数据列，数据》分列》固定宽度，分列得到多维数据，注意数据格式使用**文本**格式
- 十六进制转换为十进制数：=IF(HEX2DEC(LEFT(B1,1))<8,HEX2DEC(B1),-HEX2DEC(DEC2HEX(2^16-HEX2DEC(B1),4)))
- 复制粘贴，粘贴时右键选择值，得到数据，保留有效数据，删除其他，保存文件



# 嵌入式端调试



- aiSystemPerformanceProcess

    - 

    - do-while循环

      执行各个network
      idx表示network的序号，idx = (idx+1) % AI_MNETWORK_NUMBER

      - if (!r) 程序段用于检查network执行结果

- aiTestPerformance

	- ai_input 输入




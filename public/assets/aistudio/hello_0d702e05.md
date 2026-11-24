┌──(bg9211014🎃kali)-[~]
└─$ bash << 'EOF'
set -euo pipefail
export GIT_PAGER=cat
cd ~/lpc1788_workspace/nuttx

echo "=== 1. diff (open1788.h, lpc17_40_lcd.c, lpc17_40_bringup.c) ==="
git --no-pager diff boards/arm/lpc17xx_40xx/open1788/src/open1788.h \
                    boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c \
                    boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c

echo "=== 2. arch lpc17_40_lcd.c VD16..23 & lcdclear ==="
nl -ba arch/arm/src/lpc17xx_40xx/lpc17_40_lcd.c | sed -n '495,530p'
nl -ba arch/arm/src/lpc17xx_40xx/lpc17_40_lcd.c | sed -n '795,830p'

echo "=== 3. board.h lines 140-460 ==="
nl -ba boards/arm/lpc17xx_40xx/open1788/include/board.h | sed -n '140,460p'

echo "=== 4. lpc17_40_bringup.c lines 250-450 ==="
nl -ba boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c | sed -n '250,450p'

echo "=== 5. lpc17_40_allocateheap.c lines 200-360 ==="
nl -ba arch/arm/src/lpc17xx_40xx/lpc17_40_allocateheap.c | sed -n '200,360p'

echo "=== 6. ea_gui defconfig ==="
cat boards/arm/lpc17xx_40xx/open1788/configs/ea_gui/defconfig

echo "=== 7. RAMLOG Kconfig & .config check ==="
grep -n -C 3 "config RAMLOG" drivers/syslog/Kconfig || true
grep -E "^CONFIG_(ARM_TOOLCHAIN|MM_REGIONS|LPC17_40_EMAC|LPC17_40_SDCARD|MMCSD|FS_PROCFS|NSH_PROC_MOUNTPOINT|RAMLOG|SYSLOG|INPUT_DJOYSTICK|LPC17_40_LCD)" .config || true
EOF
=== 1. diff (open1788.h, lpc17_40_lcd.c, lpc17_40_bringup.c) ===
diff --git a/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c b/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c
index 5769162933..81dd81c11c 100644
--- a/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c
+++ b/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c
@@ -400,6 +400,9 @@ int open1788_bringup(void)
     {
       syslog(LOG_ERR, "ERROR: fb_register() failed: %d\n", ret);
     }
+
+  extern void ea_lcd_backlight_init(void);
+  ea_lcd_backlight_init();
 #endif
 
 #ifdef CONFIG_INPUT_ADS7843E
diff --git a/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c b/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c
index 75ca85ff4e..0fce0d0c95 100644
--- a/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c
+++ b/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c
@@ -1,73 +1,42 @@
-/****************************************************************************
- * boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_lcd.c
- *
- * SPDX-License-Identifier: Apache-2.0
- *
- * Licensed to the Apache Software Foundation (ASF) under one or more
- * contributor license agreements.  See the NOTICE file distributed with
- * this work for additional information regarding copyright ownership.  The
- * ASF licenses this file to you under the Apache License, Version 2.0 (the
- * "License"); you may not use this file except in compliance with the
- * License.  You may obtain a copy of the License at
- *
- *   http://www.apache.org/licenses/LICENSE-2.0
- *
- * Unless required by applicable law or agreed to in writing, software
- * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
- * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the
- * License for the specific language governing permissions and limitations
- * under the License.
- *
- ****************************************************************************/
-
-/****************************************************************************
- * Included Files
- ****************************************************************************/
-
 #include <nuttx/config.h>
-
+#include <stdint.h>
 #include <stdbool.h>
-#include <nuttx/debug.h>
-
+#include <stdio.h>
+#include <nuttx/i2c/i2c_master.h>
 #include "lpc17_40_lcd.h"
 #include "lpc17_40_gpio.h"
-
+#include "lpc17_40_i2c.h"
 #include "open1788.h"
 
 #ifdef CONFIG_LPC17_40_LCD
+#define EA_PCA9532_ADDR      0x64
+#define EA_PCA9532_PSC0_INC  (0x02 | 0x10)
 
-/****************************************************************************
- * Public Functions
- ****************************************************************************/
-
-/****************************************************************************
- * Name: open1788_lcd_initialize
- *
- * Description:
- *   Initialize the LCD.  Setup backlight (initially off)
- *
- ****************************************************************************/
+void open1788_lcd_initialize(void) {}
 
-void open1788_lcd_initialize(void)
+void ea_lcd_backlight_init(void)
 {
-  /* Configure the LCD backlight (and turn the backlight off) */
-
-  lpc17_40_configgpio(GPIO_LCD_BL);
+#ifdef CONFIG_LPC17_40_I2C0
+  struct i2c_master_s *i2c;
+  struct i2c_msg_s msg;
+  uint8_t cmd[] = { EA_PCA9532_PSC0_INC, 0x00, 0x00, 0x00, 0x00, 0x05, 0x40, 0x00, 0x00 };
+  int ret;
+
+  i2c = lpc17_40_i2cbus_initialize(0);
+  if (i2c != NULL)
+    {
+      msg.frequency = 100000;
+      msg.addr      = EA_PCA9532_ADDR;
+      msg.flags     = 0;
+      msg.buffer    = cmd;
+      msg.length    = sizeof(cmd);
+      ret = I2C_TRANSFER(i2c, &msg, 1);
+      printf("EA PCA9532 backlight init: ret=%d\n", ret);
+    }
+#endif
 }
 
-/****************************************************************************
- * Name: lpc17_40_backlight
- *
- * Description:
- *   If CONFIG_LPC17_40_LCD_BACKLIGHT is defined, then the board-specific
- *   logic must provide this interface to turn the backlight on and off.
- *
- ****************************************************************************/
-
 #ifdef CONFIG_LPC17_40_LCD_BACKLIGHT
-void lpc17_40_backlight(bool blon)
-{
-  lpc17_40_gpiowrite(GPIO_LCD_BL, blon);
-}
+void lpc17_40_backlight(bool blon) {}
+#endif
 #endif
-#endif /* CONFIG_LPC17_40_LCD */
diff --git a/boards/arm/lpc17xx_40xx/open1788/src/open1788.h b/boards/arm/lpc17xx_40xx/open1788/src/open1788.h
index 85906ca8b3..ba130ae013 100644
--- a/boards/arm/lpc17xx_40xx/open1788/src/open1788.h
+++ b/boards/arm/lpc17xx_40xx/open1788/src/open1788.h
@@ -119,7 +119,7 @@
 
 /* Backlight enable, P2[1].  Initial state is OFF (zero) */
 
-#define GPIO_LCD_BL      (GPIO_OUTPUT | GPIO_VALUE_ZERO | GPIO_PORT2 | GPIO_PIN1)
+#define GPIO_LCD_BL      0
 
 /* XPT2046 Touchscreen ******************************************************/
 
@@ -143,9 +143,9 @@
  * the LCD schematic, the are pulled-up on board the LCD module.
  */
 
-#define GPIO_TC_PENIRQ   (GPIO_INTBOTH | GPIO_FLOAT | GPIO_PORT2 | GPIO_PIN15)
+#define GPIO_TC_PENIRQ   (GPIO_INTFE | GPIO_FLOAT | GPIO_PORT2 | GPIO_PIN15)
 #define GPIO_TC_BUSY     (GPIO_INPUT | GPIO_FLOAT | GPIO_PORT2 | GPIO_PIN14)
-#define GPIO_TC_CS       (GPIO_OUTPUT | GPIO_VALUE_ONE | GPIO_PORT1 | GPIO_PIN8)
+#define GPIO_TC_CS       (GPIO_OUTPUT | GPIO_VALUE_ONE | GPIO_PORT0 | GPIO_PIN20)
 
 #define LPC17_40_IRQ_PENIRQ LPC17_40_IRQ_P2p15
 
=== 2. arch lpc17_40_lcd.c VD16..23 & lcdclear ===
   495    lpc17_40_configgpio(GPIO_LCD_VD7);
   496
   497    lpc17_40_configgpio(GPIO_LCD_VD8);
   498    lpc17_40_configgpio(GPIO_LCD_VD9);
   499    lpc17_40_configgpio(GPIO_LCD_VD10);
   500    lpc17_40_configgpio(GPIO_LCD_VD11);
   501    lpc17_40_configgpio(GPIO_LCD_VD12);
   502    lpc17_40_configgpio(GPIO_LCD_VD13);
   503    lpc17_40_configgpio(GPIO_LCD_VD14);
   504    lpc17_40_configgpio(GPIO_LCD_VD15);
   505
   506  #if LPC17_40_BPP > 16
   507    lpc17_40_configgpio(GPIO_LCD_VD16);
   508    lpc17_40_configgpio(GPIO_LCD_VD17);
   509    lpc17_40_configgpio(GPIO_LCD_VD18);
   510    lpc17_40_configgpio(GPIO_LCD_VD19);
   511    lpc17_40_configgpio(GPIO_LCD_VD20);
   512    lpc17_40_configgpio(GPIO_LCD_VD21);
   513    lpc17_40_configgpio(GPIO_LCD_VD22);
   514    lpc17_40_configgpio(GPIO_LCD_VD23);
   515  #endif
   516
   517    /* Other pins */
   518
   519    lpc17_40_configgpio(GPIO_LCD_DCLK);
   520    lpc17_40_configgpio(GPIO_LCD_LP);
   521    lpc17_40_configgpio(GPIO_LCD_FP);
   522    lpc17_40_configgpio(GPIO_LCD_ENABM);
   523    lpc17_40_configgpio(GPIO_LCD_PWR);
   524
   525    /* Turn on LCD clock */
   526
   527    modifyreg32(LPC17_40_SYSCON_PCONP, 0, SYSCON_PCONP_PCLCD);
   528
   529    lcdinfo("Configuring the LCD controller\n");
   530
   795   *
   796   ****************************************************************************/
   797
   798  void lpc17_40_lcdclear(nxgl_mxpixel_t color)
   799  {
   800    int i;
   801  #if LPC17_40_BPP > 16
   802    uint32_t *dest = (uint32_t *)CONFIG_LPC17_40_LCD_VRAMBASE;
   803
   804    lcdinfo("Clearing display: color=%08jx VRAM=%08x size=%d\n",
   805            (uintmax_t)color, CONFIG_LPC17_40_LCD_VRAMBASE,
   806            CONFIG_LPC17_40_LCD_HWIDTH * CONFIG_LPC17_40_LCD_VHEIGHT *
   807            sizeof(uint32_t));
   808
   809  #else
   810    uint16_t *dest = (uint16_t *)CONFIG_LPC17_40_LCD_VRAMBASE;
   811
   812    lcdinfo("Clearing display: color=%08x VRAM=%08x size=%d\n",
   813            color, CONFIG_LPC17_40_LCD_VRAMBASE,
   814            CONFIG_LPC17_40_LCD_HWIDTH * CONFIG_LPC17_40_LCD_VHEIGHT *
   815            sizeof(uint16_t));
   816  #endif
   817
   818    for (i = 0; i < (CONFIG_LPC17_40_LCD_HWIDTH * CONFIG_LPC17_40_LCD_VHEIGHT);
   819         i++)
   820      {
   821        *dest++ = color;
   822      }
   823  }
=== 3. board.h lines 140-460 ===
   140
   141  #undef  CONFIG_LPC17_40_FLASH
   142  #define CONFIG_LPC17_40_FLASH         1
   143
   144  /* Flash access use 6 CPU clocks - Safe for any allowed conditions */
   145
   146  #define BOARD_FLASHCFG_VALUE       (SYSCON_FLASHCFG_TIM_5 | 0x03a)
   147
   148  /* Ethernet configuration */
   149
   150  #define ETH_MCFG_CLKSEL_DIV        ETH_MCFG_CLKSEL_DIV20
   151
   152  #ifdef CONFIG_LPC17_40_SDCARD
   153  /* SDIO dividers.  Note that slower clocking is required when DMA is disabled
   154   * in order to avoid RX overrun/TX underrun errors due to delayed responses
   155   * to service FIFOs in interrupt driven mode.
   156   * SDCARD_CLOCK=PCLK/(2*(SDCARD_CLKDIV+1))
   157   */
   158
   159  #  define SDCARD_CLKDIV_INIT       74   /* 400Khz  */
   160  #  define SDCARD_INIT_CLKDIV       (SDCARD_CLKDIV_INIT)
   161
   162  #  define SDCARD_NORMAL_CLKDIV     1    /* DMA ON:  SDCARD_CLOCK=15MHz */
   163  #define SDCARD_SLOW_CLKDIV         14   /* DMA OFF: SDCARD_CLOCK=2MHz */
   164
   165  #  ifdef CONFIG_SDIO_DMA
   166  #    define SDCARD_MMCXFR_CLKDIV   (SDCARD_NORMAL_CLKDIV)
   167  #  else
   168  #    define SDCARD_MMCXFR_CLKDIV   (SDCARD_SLOW_CLKDIV)
   169  #  endif
   170
   171  #  ifdef CONFIG_SDIO_DMA
   172  #    define SDCARD_SDXFR_CLKDIV    (SDCARD_NORMAL_CLKDIV)
   173  #  else
   174  #    define SDCARD_SDXFR_CLKDIV    (SDCARD_SLOW_CLKDIV)
   175  #  endif
   176  #endif
   177
   178  /* Set EMC delay values:
   179   *
   180   * CMDDLY: Programmable delay value for EMC outputs in command delayed
   181   *   mode.  The delay amount is roughly CMDDLY * 250 picoseconds.
   182   * FBCLKDLY: Programmable delay value for the feedback clock that controls
   183   *   input data sampling.  The delay amount is roughly (FBCLKDLY+1) * 250
   184   *   picoseconds.
   185   * CLKOUT0DLY: Programmable delay value for the CLKOUT0 output. This would
   186   *   typically be used in clock delayed mode.  The delay amount is roughly
   187   *  (CLKOUT0DLY+1) * 250 picoseconds.
   188   * CLKOUT1DLY: Programmable delay value for the CLKOUT1 output. This would
   189   *  typically be used in clock delayed mode.  The delay amount is roughly
   190   *  (CLKOUT1DLY+1) * 250 picoseconds.
   191   *
   192   * Optimal for NOR: {1,1,1,1}
   193   * Needed for NAND and SDRAM: {17,1,2,1}
   194   */
   195
   196  #ifdef CONFIG_LPC17_40_EMC
   197  #if defined(CONFIG_LPC17_40_EXTNAND) || defined(CONFIG_LPC17_40_EXTDRAM)
   198  #  define BOARD_CMDDLY             17
   199  #  define BOARD_FBCLKDLY           17
   200  #  define BOARD_CLKOUT0DLY         1
   201  #  define BOARD_CLKOUT1DLY         1
   202  #else
   203  #  define BOARD_CMDDLY             1
   204  #  define BOARD_FBCLKDLY           1
   205  #  define BOARD_CLKOUT0DLY         1
   206  #  define BOARD_CLKOUT1DLY         1
   207  #endif
   208  #endif
   209
   210  /* LED definitions **********************************************************/
   211
   212  /* If CONFIG_ARCH_LEDS is not defined, then the user can control the LEDs in
   213   * any way.  The following definitions are used to access individual LEDs.
   214   *
   215   * LED1 -- Connected to P1[14]
   216   * LED2 -- Connected to P0[16] (shared with UART1 RXD)
   217   * LED3 -- Connected to P1[13]
   218   * LED4 -- Connected to P4[27]
   219   *
   220   * These LEDs are connected to ground so a high output value will illuminate
   221   * them.
   222   */
   223
   224  /* LED index values for use with board_userled() */
   225
   226  #define BOARD_LED1                 0
   227  #define BOARD_LED2                 1
   228  #define BOARD_LED3                 2
   229  #define BOARD_LED4                 3
   230  #define BOARD_NLEDS                4
   231
   232  /* LED bits for use with board_userled_all() */
   233
   234  #define BOARD_LED1_BIT             (1 << BOARD_LED1)
   235  #define BOARD_LED2_BIT             (1 << BOARD_LED2)
   236  #define BOARD_LED3_BIT             (1 << BOARD_LED3)
   237  #define BOARD_LED4_BIT             (1 << BOARD_LED4)
   238
   239  /* If CONFIG_ARCH_LEDs is defined, then NuttX will control the four LEDs
   240   * on the WaveShare Open1788K.
   241   * The following definitions describe how NuttX controls the LEDs:
   242   */
   243
   244                                 /* LED1 LED2 LED3 LED4                      */
   245  #define LED_STARTED         0  /*  OFF  OFF  OFF  OFF                      */
   246  #define LED_HEAPALLOCATE    1  /*  ON   OFF  OFF  OFF                      */
   247  #define LED_IRQSENABLED     2  /*  OFF   ON  OFF  OFF                      */
   248  #define LED_STACKCREATED    3  /*  ON    ON  OFF  OFF                      */
   249  #define LED_INIRQ           4  /*  LED3 glows, on while in interrupt       */
   250  #define LED_SIGNAL          4  /*  LED3 glows, on while in signal handler  */
   251  #define LED_ASSERTION       4  /*  LED3 glows, on while in assertion       */
   252  #define LED_PANIC           4  /*  LED3 Flashes at 2Hz                     */
   253  #define LED_IDLE            5  /*  LED4 glows: ON while active
   254                                  *              OFF while sleeping
   255                                  */
   256
   257  /* Button definitions *******************************************************/
   258
   259  /* The Open1788 supports several buttons.
   260   * All must be pulled up by the Open1788.
   261   * When closed, the pins will be pulled to ground.
   262   * So the buttons will read "1" when open and "0" when closed.
   263   * All except USER1 are capable of generating interrupts.
   264   *
   265   * USER1           -- Connected to P4[26]
   266   * USER2           -- Connected to P2[22]
   267   * USER3           -- Connected to P0[10]
   268   *
   269   * And a Joystick
   270   *
   271   * JOY_A           -- Connected to P2[25]
   272   * JOY_B           -- Connected to P2[26]
   273   * JOY_C           -- Connected to P2[23]
   274   * JOY_D           -- Connected to P2[19]
   275   * JOY_CTR         -- Connected to P0[14] (shared with SSP1 SSEL)
   276   *
   277   * For the interrupting buttons, interrupts are generated on both edges
   278   * (press and release).
   279   */
   280
   281  #define BOARD_BUTTON_USER1         0
   282  #define BOARD_BUTTON_USER2         1
   283  #define BOARD_BUTTON_USER3         2
   284
   285  #define BOARD_JOYSTICK_A           3
   286  #define BOARD_JOYSTICK_B           4
   287  #define BOARD_JOYSTICK_C           5
   288  #define BOARD_JOYSTICK_D           6
   289  #define BOARD_JOYSTICK_CTR         7
   290
   291  #define NUM_BUTTONS                8
   292
   293  #define BOARD_BUTTON_USER1_BIT     (1 << BOARD_BUTTON_USER1)
   294  #define BOARD_BUTTON_USER2_BIT     (1 << BOARD_BUTTON_USER2)
   295  #define BOARD_BUTTON_USER3_BIT     (1 << BOARD_BUTTON_USER3)
   296
   297  #define BOARD_JOYSTICK_A_BIT       (1 << BOARD_JOYSTICK_A)
   298  #define BOARD_JOYSTICK_B_BIT       (1 << BOARD_JOYSTICK_B)
   299  #define BOARD_JOYSTICK_C_BIT       (1 << BOARD_JOYSTICK_C)
   300  #define BOARD_JOYSTICK_D_BIT       (1 << BOARD_JOYSTICK_D)
   301  #define BOARD_JOYSTICK_CTR_BIT     (1 << BOARD_JOYSTICK_CTR)
   302
   303  /* Alternate pin selections *************************************************/
   304
   305  /* UART0:
   306   *
   307   * TX    --- Connected to P0[2]
   308   * RX    --- Connected to P0[3]
   309   */
   310
   311  #define GPIO_UART0_TXD             GPIO_UART0_TXD_2
   312  #define GPIO_UART0_RXD             GPIO_UART0_RXD_2
   313
   314  /* UART1:
   315   *
   316   * All pin options are controlled by older bridges on the bottom of the
   317   * board.
   318   * There are the default settings on my board as it came out of the box:
   319   *
   320   * RTS   --- Connected to P0[22]
   321   * RI    --- Connected to P0[21]
   322   * DSR   --- Connected to P0[19]
   323   * DCD   --- Connected to P0[18]
   324   * CTS   --- Connected to P0[17]
   325   * DTR   --- Connected to P0[20]
   326   * TXD   --- Connected to P0[15]
   327   * RXD   --- Connected to P0[16] (Shared with LED2)
   328   */
   329
   330  #define GPIO_UART1_RTS             GPIO_UART1_RTS_2
   331  #define GPIO_UART1_RI              GPIO_UART1_RI_1
   332  #define GPIO_UART1_DSR             GPIO_UART1_DSR_1
   333  #define GPIO_UART1_DCD             GPIO_UART1_DCD_1
   334  #define GPIO_UART1_CTS             GPIO_UART1_CTS_1
   335  #define GPIO_UART1_DTR             GPIO_UART1_DTR_1
   336  #define GPIO_UART1_TXD             GPIO_UART1_TXD_1
   337  #define GPIO_UART1_RXD             GPIO_UART1_RXD_1
   338
   339  /* MCI-SDIO:
   340   *
   341   * D0    --- Connected to P1[6]
   342   * D1    --- Connected to P2[11]
   343   * D2    --- Connected to P2[12]
   344   * D3    --- Connected to P2[13]
   345   * CLK   --- Connected to P1[2]
   346   * CMD   --- Connected to P1[3]
   347   */
   348
   349  #define GPIO_SD_DAT0               GPIO_SD_DAT0_2
   350  #define GPIO_SD_DAT1               GPIO_SD_DAT1_2
   351  #define GPIO_SD_DAT2               GPIO_SD_DAT2_2
   352  #define GPIO_SD_DAT3               GPIO_SD_DAT3_2
   353  #define GPIO_SD_CLK                GPIO_SD_CLK_2
   354  #define GPIO_SD_CMD                GPIO_SD_CMD_2
   355
   356  /* LCD R:
   357   *
   358   * VD0   --- Connected to P0[4]
   359   * VD1   --- Connected to P0[5]
   360   * VD2   --- Connected to P4[28]
   361   * VD3   --- Connected to P4[29]
   362   * VD4   --- Connected to P2[6]
   363   * VD5   --- Connected to P2[7]
   364   * VD6   --- Connected to P2[8]
   365   * VD7   --- Connected to P2[9]
   366   */
   367
   368  #define GPIO_LCD_VD0                GPIO_LCD_VD0_1
   369  #define GPIO_LCD_VD1                GPIO_LCD_VD1_1
   370  #define GPIO_LCD_VD2                GPIO_LCD_VD2_2
   371  #define GPIO_LCD_VD3                GPIO_LCD_VD3_2
   372  #define GPIO_LCD_VD4                GPIO_LCD_VD4_1
   373  #define GPIO_LCD_VD5                GPIO_LCD_VD5_1
   374  #define GPIO_LCD_VD6                GPIO_LCD_VD6_2
   375  #define GPIO_LCD_VD7                GPIO_LCD_VD7_2
   376
   377  /* LCD G:
   378   *
   379   * VD8    --- Connected to P0[6]
   380   * VD9    --- Connected to P0[7]
   381   * VD10   --- Connected to P1[20]
   382   * VD11   --- Connected to P1[21]
   383   * VD12   --- Connected to P1[22]
   384   * VD13   --- Connected to P1[23]
   385   * VD14   --- Connected to P1[24]
   386   * VD15   --- Connected to P1[25]
   387   */
   388
   389  #define GPIO_LCD_VD8                GPIO_LCD_VD8_1
   390  #define GPIO_LCD_VD9                GPIO_LCD_VD9_1
   391  #define GPIO_LCD_VD10               GPIO_LCD_VD10_1
   392  #define GPIO_LCD_VD11               GPIO_LCD_VD11_1
   393  #define GPIO_LCD_VD12               GPIO_LCD_VD12_1
   394  #define GPIO_LCD_VD13               GPIO_LCD_VD13_1
   395  #define GPIO_LCD_VD14               GPIO_LCD_VD14_1
   396  #define GPIO_LCD_VD15               GPIO_LCD_VD15_1
   397
   398  /* LCD B:
   399   *
   400   * VD16   --- Connected to P0[8]
   401   * VD17   --- Connected to P0[9]
   402   * VD18   --- Connected to P2[12]
   403   * VD19   --- Connected to P2[13]
   404   * VD20   --- Connected to P1[26]
   405   * VD21   --- Connected to P1[27]
   406   * VD22   --- Connected to P1[28]
   407   * VD23   --- Connected to P1[29]
   408   *
   409   * DCLK   --- Connected to P2[2]
   410   * LP     --- Connected to P2[5]
   411   * FP     --- Connected to P2[3]
   412   * ENAB_M --- Connected to P2[4]
   413   * PWR    --- Connected to P2[0]
   414   */
   415
   416  /* XPT2046 Touchscreen:
   417   *
   418   * -------------- -------------------- ------------ -------------------------
   419   * XTPT2046       Module               Module       Open1788 LED
   420   *                Signal               Connector    Connector
   421   * -------------- -------------------- ------------ -------------------------
   422   * Pin 11 PENIRQ\ PENIRQ (pulled high) PORT3 Pin 1  P2.15 PENIRQ
   423   * Pin 12 DOUT    MISO                 PORT3 Pin 4  P1.18 MISO1
   424   *                                                  (Also USB HOST UP LED)
   425   * Pin 13 BUSY    BUSY (pulled high)   PORT3 Pin 9  P2.14 BUSY
   426   * Pin 14 DIN     MOSI                 PORT3 Pin 3  P0.13 MOSI1
   427   *                                                  (Also USB Device up LED
   428   *                                                  and SD CD pin)
   429   * Pin 15 CS\     SSEL (pulled high)   PORT3 Pin 6  P1.8  GPIO
   430   *                                                  (Also RMII_CRS_DV)
   431   * Pin 16 DCLK    SCK                  PORT3 Pin 5  P1.19 SCK1
   432   * -------------- -------------------- ------------ -------------------------
   433   */
   434
   435  #define GPIO_SSP1_MISO   GPIO_SSP1_MISO_3
   436  #define GPIO_SSP1_MOSI   GPIO_SSP1_MOSI_2
   437  #define GPIO_SSP1_SCK    GPIO_SSP1_SCK_2
   438  #define GPIO_SSP0_MISO   GPIO_SSP0_MISO_1
   439  #define GPIO_SSP0_MOSI   GPIO_SSP0_MOSI_1
   440  #define GPIO_SSP0_SCK    GPIO_SSP0_SCK_1
   441  #define GPIO_I2C0_SCL    GPIO_I2C0_SCL_1
   442  #define GPIO_I2C0_SDA    GPIO_I2C0_SDA_1
   443
   444
   445  /* EA LPC1788 UART2, UART3, CAN2 Pin Definitions */
   446  #define GPIO_UART2_TXD   GPIO_UART2_TXD_1
   447  #define GPIO_UART2_RXD   GPIO_UART2_RXD_1
   448  #define GPIO_UART3_TXD   GPIO_UART3_TXD_3
   449  #define GPIO_UART3_RXD   GPIO_UART3_RXD_3
   450  #define GPIO_CAN2_RD     GPIO_CAN2_RD_1
   451  #define GPIO_CAN2_TD     GPIO_CAN2_TD_1
   452
   453
   454  /* EA LPC1788 LAN8720 PHY Management Pins */
   455  #define GPIO_ENET_MDC    GPIO_ENET_MDC_1
   456  #define GPIO_ENET_MDIO   GPIO_ENET_MDIO_1
   457
   458  #endif /* __BOARDS_ARM_LPC17XX_40XX_OPEN1788_INCLUDE_BOARD_H */
=== 4. lpc17_40_bringup.c lines 250-450 ===
   250
   251  #endif
   252  #endif
   253
   254    /* First, get an instance of the SDIO interface */
   255
   256    g_sdiodev = sdio_initialize(CONFIG_NSH_MMCSDSLOTNO);
   257    if (!g_sdiodev)
   258      {
   259        syslog(LOG_ERR, "ERROR: Failed to initialize SDIO slot %d\n",
   260               CONFIG_NSH_MMCSDSLOTNO);
   261        return -ENODEV;
   262      }
   263
   264    /* Now bind the SDIO interface to the MMC/SD driver */
   265
   266    ret = mmcsd_slotinitialize(CONFIG_NSH_MMCSDMINOR, g_sdiodev);
   267    if (ret != OK)
   268      {
   269        syslog(LOG_ERR,
   270               "ERROR: Failed to bind SDIO to the MMC/SD driver: %d\n",
   271               ret);
   272        return ret;
   273      }
   274
   275    /* Check if there is a card in the slot and inform the SDCARD driver.  If
   276     * we do not support the card  detect, then let's assume that there is
   277     * one.
   278     */
   279
   280  #ifdef NSH_HAVE_MMCSD_CD
   281    g_sd_inserted = !lpc17_40_gpioread(GPIO_SD_CD);
   282    sdio_mediachange(g_sdiodev, g_sd_inserted);
   283  #else
   284    sdio_mediachange(g_sdiodev, true);
   285  #endif
   286    return OK;
   287  }
   288  #else
   289  #  define nsh_sdinitialize() (OK)
   290  #endif
   291
   292  /****************************************************************************
   293   * Name: nsh_usbhostinitialize
   294   *
   295   * Description:
   296   *   Initialize SPI-based microSD.
   297   *
   298   ****************************************************************************/
   299
   300  #ifdef NSH_HAVE_USBHOST
   301  static int nsh_usbhostinitialize(void)
   302  {
   303    int ret;
   304
   305    /* First, register all of the class drivers needed to support the drivers
   306     * that we care about:
   307     */
   308
   309    syslog(LOG_INFO, "Register class drivers\n");
   310
   311  #ifdef CONFIG_USBHOST_MSC
   312    /* Register the USB host Mass Storage Class */
   313
   314    ret = usbhost_msc_initialize();
   315    if (ret != OK)
   316      {
   317        syslog(LOG_ERR,
   318               "ERROR: Failed to register the mass storage class: %d\n", ret);
   319      }
   320  #endif
   321
   322  #ifdef CONFIG_USBHOST_CDCACM
   323    /* Register the CDC/ACM serial class */
   324
   325    ret = usbhost_cdcacm_initialize();
   326    if (ret != OK)
   327      {
   328        syslog(LOG_ERR,
   329               "ERROR: Failed to register the CDC/ACM serial class: %d\n",
   330               ret);
   331      }
   332  #endif
   333
   334    /* Then get an instance of the USB host interface */
   335
   336    syslog(LOG_INFO, "Initialize USB host\n");
   337    g_usbconn = lpc17_40_usbhost_initialize(0);
   338    if (g_usbconn)
   339      {
   340        /* Start a thread to handle device connection. */
   341
   342        syslog(LOG_INFO, "Start nsh_waiter\n");
   343
   344        ret = kthread_create("usbhost", CONFIG_USBHOST_DEFPRIO,
   345                             CONFIG_USBHOST_STACKSIZE,
   346                             nsh_waiter, NULL);
   347        return ret < 0 ? -ENOEXEC : OK;
   348      }
   349
   350    return -ENODEV;
   351  }
   352  #else
   353  #  define nsh_usbhostinitialize() (OK)
   354  #endif
   355
   356  /****************************************************************************
   357   * Public Functions
   358   ****************************************************************************/
   359
   360  /****************************************************************************
   361   * Name: open1788_bringup
   362   *
   363   * Description:
   364   *   Perform architecture-specific initialization
   365   *
   366   *   CONFIG_BOARD_LATE_INITIALIZE=y :
   367   *     Called from board_late_initialize().
   368   *
   369   ****************************************************************************/
   370
   371  int open1788_bringup(void)
   372  {
   373    int ret;
   374
   375  #ifdef CONFIG_FS_PROCFS
   376    /* Mount the procfs file system at the default location, /proc */
   377
   378    ret = nx_mount(NULL, "/proc", "procfs", 0, NULL);
   379    if (ret < 0)
   380      {
   381        syslog(LOG_ERR, "ERROR: Failed to mount procfs: %d\n", ret);
   382      }
   383  #endif
   384
   385    /* Initialize SPI-based microSD */
   386
   387    ret = nsh_sdinitialize();
   388    if (ret == OK)
   389      {
   390        /* Initialize USB host */
   391
   392        ret = nsh_usbhostinitialize();
   393      }
   394
   395  #ifdef CONFIG_VIDEO_FB
   396    /* Initialize and register the framebuffer driver */
   397
   398    ret = fb_register(0, 0);
   399    if (ret < 0)
   400      {
   401        syslog(LOG_ERR, "ERROR: fb_register() failed: %d\n", ret);
   402      }
   403
   404    extern void ea_lcd_backlight_init(void);
   405    ea_lcd_backlight_init();
   406  #endif
   407
   408  #ifdef CONFIG_INPUT_ADS7843E
   409    /* Initialize the touchscreen */
   410
   411    ret = open1788_tsc_setup(0);
   412    if (ret < 0)
   413      {
   414        syslog(LOG_ERR, "ERROR: open1788_tsc_setup failed: %d\n", ret);
   415      }
   416  #endif
   417
   418  #ifdef CONFIG_OPEN1788_DJOYSTICK
   419    /* Initialize and register the joystick driver */
   420
   421    ret = lpc17_40_djoy_initialization();
   422    if (ret != OK)
   423      {
   424        syslog(LOG_ERR,
   425               "ERROR: Failed to register the joystick driver: %d\n", ret);
   426        return ret;
   427      }
   428  #endif
   429
   430    return ret;
   431  }
=== 5. lpc17_40_allocateheap.c lines 200-360 ===
   200   *     User .bss region  Size determined at link time.
   201   *     Kernel heap.  Size determined by CONFIG_MM_KERNEL_HEAPSIZE.
   202   *     User heap.  Extends to the end of SRAM.
   203   *
   204   ****************************************************************************/
   205
   206  void up_allocate_heap(void **heap_start, size_t *heap_size)
   207  {
   208  #if defined(CONFIG_BUILD_PROTECTED) && defined(CONFIG_MM_KERNEL_HEAP)
   209    /* Get the unaligned size and position of the user-space heap.
   210     * This heap begins after the user-space .bss section at an offset
   211     * of CONFIG_MM_KERNEL_HEAPSIZE (subject to alignment).
   212     */
   213
   214    uintptr_t ubase = (uintptr_t)USERSPACE->us_bssend +
   215                       CONFIG_MM_KERNEL_HEAPSIZE;
   216    size_t    usize = CONFIG_RAM_END - ubase;
   217    int       log2;
   218
   219    DEBUGASSERT(ubase < (uintptr_t)CONFIG_RAM_END);
   220
   221    /* Adjust that size to account for MPU alignment requirements.
   222     * NOTE that there is an implicit assumption that the CONFIG_RAM_END
   223     * is aligned to the MPU requirement.
   224     */
   225
   226    log2  = (int)mpu_log2regionfloor(usize);
   227    DEBUGASSERT((CONFIG_RAM_END & ((1 << log2) - 1)) == 0);
   228
   229    usize = (1 << log2);
   230    ubase = CONFIG_RAM_END - usize;
   231
   232    /* Return the user-space heap settings */
   233
   234    board_autoled_on(LED_HEAPALLOCATE);
   235    *heap_start = (void *)ubase;
   236    *heap_size  = usize;
   237
   238    /* Allow user-mode access to the user heap memory */
   239
   240    lpc17_40_mpu_uheap((uintptr_t)ubase, usize);
   241
   242  #elif CONFIG_MM_REGIONS >= 3 && defined(CONFIG_LPC17_40_EXTDRAM) && \
   243      defined(CONFIG_LPC17_40_EXTDRAMHEAP)
   244    /* We are going to allocate a DRAM heap.  In the case where a bootloader
   245     * is used (and has initialized SDRAM), it is possible that .data, .bss,
   246     * and the IDLE stack reside in SDRAM.
   247     */
   248
   249    uintptr_t sram_start = CONFIG_RAM_START;
   250
   251    /* Is SRAM the primary RAM? I.e., do .data, .bss, and the IDLE thread
   252     * stack lie in SRAM?
   253     */
   254
   255    if (g_idle_topstack >= CONFIG_RAM_START &&
   256        g_idle_topstack < CONFIG_RAM_END)
   257      {
   258        /* Yes, then the SRAM heap starts in SRAM after the IDLE stack */
   259
   260        sram_start = g_idle_topstack;
   261      }
   262    else
   263      {
   264        /* Use the entire SRAM for heap */
   265
   266        sram_start = CONFIG_RAM_START;
   267      }
   268
   269    board_autoled_on(LED_HEAPALLOCATE);
   270    *heap_start = (void *)sram_start;
   271    *heap_size  = CONFIG_RAM_END - sram_start;
   272
   273  #else
   274
   275    /* Return the heap settings (assuming that .data, .bss, and the IDLE
   276     * stack lie in SRAM).
   277     */
   278
   279    board_autoled_on(LED_HEAPALLOCATE);
   280    *heap_start = (void *)g_idle_topstack;
   281    *heap_size  = CONFIG_RAM_END - g_idle_topstack;
   282  #endif
   283  }
   284
   285  /****************************************************************************
   286   * Name: up_allocate_kheap
   287   *
   288   * Description:
   289   *   For the kernel build (CONFIG_BUILD_PROTECTED=y) with both kernel- and
   290   *   user-space heaps (CONFIG_MM_KERNEL_HEAP=y), this function allocates
   291   *   (and protects) the kernel-space heap.
   292   *
   293   *   REVISIT:  This does not account for the possibility that the kernel
   294   *   heap might lie in SDRAM.
   295   *
   296   ****************************************************************************/
   297
   298  #if defined(CONFIG_BUILD_PROTECTED) && defined(CONFIG_MM_KERNEL_HEAP)
   299  void up_allocate_kheap(void **heap_start, size_t *heap_size)
   300  {
   301    /* Get the unaligned size and position of the user-space heap.
   302     * This heap begins after the user-space .bss section at an offset
   303     * of CONFIG_MM_KERNEL_HEAPSIZE (subject to alignment).
   304     */
   305
   306    uintptr_t ubase = (uintptr_t)USERSPACE->us_bssend +
   307                      CONFIG_MM_KERNEL_HEAPSIZE;
   308    size_t    usize = CONFIG_RAM_END - ubase;
   309    int       log2;
   310
   311    DEBUGASSERT(ubase < (uintptr_t)CONFIG_RAM_END);
   312
   313    /* Adjust that size to account for MPU alignment requirements.
   314     * NOTE that there is an implicit assumption that the CONFIG_RAM_END
   315     * is aligned to the MPU requirement.
   316     */
   317
   318    log2  = (int)mpu_log2regionfloor(usize);
   319    DEBUGASSERT((CONFIG_RAM_END & ((1 << log2) - 1)) == 0);
   320
   321    usize = (1 << log2);
   322    ubase = CONFIG_RAM_END - usize;
   323
   324    /* Return the kernel heap settings (i.e., the part of the heap region
   325     * that was not dedicated to the user heap).
   326     */
   327
   328    *heap_start = (void *)USERSPACE->us_bssend;
   329    *heap_size  = ubase - (uintptr_t)USERSPACE->us_bssend;
   330  }
   331  #endif
   332
   333  /****************************************************************************
   334   * Name: arm_addregion
   335   *
   336   * Description:
   337   *   Memory may be added in non-contiguous chunks.  Additional chunks are
   338   *   added by calling this function.
   339   *
   340   ****************************************************************************/
   341
   342  #if CONFIG_MM_REGIONS > 1
   343  void arm_addregion(void)
   344  {
   345    /* Banks 0 and 1 are each 16Kb.  If both are present, they occupy a
   346     * contiguous 32Kb memory region.
   347     *
   348     * If Ethernet is enabled, it will take some or all of bank 0 for packet
   349     * buffering and descriptor tables; If USB host is enabled, it will take
   350     * some or all of bank 1 for descriptor memory.  The complex conditional
   351     * compilation above should boil this all down to a very simple check
   352     * here:
   353     *
   354     * Is any memory available in AHB SRAM for the heap?
   355     */
   356
   357  #ifdef LPC17_40_AHB_HEAPBASE
   358  #if defined(CONFIG_BUILD_PROTECTED) && defined(CONFIG_MM_KERNEL_HEAP)
   359
   360    /* Yes.. allow user-mode access to the AHB SRAM user heap memory */
=== 6. ea_gui defconfig ===
#
# This file is autogenerated: PLEASE DO NOT EDIT IT.
#
# You can use "make menuconfig" to make any modifications to the installed .config file.
# You can then do "make savedefconfig" to generate a new defconfig file that includes your
# modifications.
#
# CONFIG_NSH_DISABLE_IFCONFIG is not set
# CONFIG_NSH_DISABLE_PS is not set
# CONFIG_NXFONTS_DISABLE_16BPP is not set
# CONFIG_NX_DISABLE_16BPP is not set
# CONFIG_SPI_EXCHANGE is not set
CONFIG_ARCH="arm"
CONFIG_ARCH_BOARD="open1788"
CONFIG_ARCH_BOARD_OPEN1788=y
CONFIG_ARCH_CHIP="lpc17xx_40xx"
CONFIG_ARCH_CHIP_LPC1788=y
CONFIG_ARCH_CHIP_LPC17XX_40XX=y
CONFIG_ARCH_STACKDUMP=y
CONFIG_ARM_TOOLCHAIN_BUILDROOT_OABI=y
CONFIG_BOARD_LOOPSPERMSEC=11934
CONFIG_BUILTIN=y
CONFIG_CAN=y
CONFIG_CXX_STANDARD="gnu++17"
CONFIG_DEBUG_ASSERTIONS=y
CONFIG_DEBUG_FEATURES=y
CONFIG_DEBUG_HARDFAULT_ALERT=y
CONFIG_DEV_LOOP=y
CONFIG_DRIVERS_VIDEO=y
CONFIG_ETH0_PHY_LAN8720=y
CONFIG_EXAMPLES_FB=y
CONFIG_EXAMPLES_NXLINES=y
CONFIG_EXAMPLES_NXLINES_BPP=16
CONFIG_EXAMPLES_PWFB=y
CONFIG_EXAMPLES_PWFB_BPP=16
CONFIG_FAT_LCNAMES=y
CONFIG_FAT_LFN=y
CONFIG_FS_FAT=y
CONFIG_FS_ROMFS=y
CONFIG_HAVE_CXX=y
CONFIG_I2C=y
CONFIG_INIT_ENTRYPOINT="nsh_main"
CONFIG_INPUT=y
CONFIG_INPUT_ADS7843E=y
CONFIG_INPUT_DJOYSTICK=y
CONFIG_INTELHEX_BINARY=y
CONFIG_LIBCXX=y
CONFIG_LIBC_MAX_EXITFUNS=8
CONFIG_LIBMINIABI=y
CONFIG_LPC17_40_ETHERNET=y
CONFIG_LPC17_40_EXTDRAM=y
CONFIG_LPC17_40_EXTDRAMSIZE=33554432
CONFIG_LPC17_40_GPDMA=y
CONFIG_LPC17_40_GPIOIRQ=y
CONFIG_LPC17_40_I2C0=y
CONFIG_LPC17_40_LCD=y
CONFIG_LPC17_40_LCD_BPP16_565=y
CONFIG_LPC17_40_RTC=y
CONFIG_LPC17_40_SDCARD=y
CONFIG_LPC17_40_SDRAM_32BIT=y
CONFIG_LPC17_40_SSP0=y
CONFIG_LPC17_40_UART0=y
CONFIG_LPC17_40_UART2=y
CONFIG_LPC17_40_UART3=y
CONFIG_MMCSD=y
CONFIG_MMCSD_SDIO=y
CONFIG_MM_REGIONS=3
CONFIG_MQ_MAXMSGSIZE=64
CONFIG_NDEBUG=y
CONFIG_NET=y
CONFIG_NETINIT_THREAD=y
CONFIG_NETUTILS_DHCPD=y
CONFIG_NETUTILS_FTPD=y
CONFIG_NETUTILS_PING=y
CONFIG_NETUTILS_TELNETD=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y
CONFIG_NSH_BUILTIN_APPS=y
CONFIG_NSH_FILEIOSIZE=512
CONFIG_NSH_READLINE=y
CONFIG_NX=y
CONFIG_NXFONT_MONO5X8=y
CONFIG_NXFONT_SANS23X27=y
CONFIG_NXGLYPHS_BACKGROUNDCOLOR=0x0
CONFIG_NXTERM=y
CONFIG_NXTERM_NXKBDIN=y
CONFIG_NXWIDGETS=y
CONFIG_NXWIDGETS_BPP=16
CONFIG_NXWIDGETS_SIZEOFCHAR=1
CONFIG_NXWM=y
CONFIG_NX_BLOCKING=y
CONFIG_NX_KBD=y
CONFIG_NX_XYINPUT_TOUCHSCREEN=y
CONFIG_PIPES=y
CONFIG_PREALLOC_TIMERS=4
CONFIG_RAM_SIZE=65536
CONFIG_RAM_START=0x10000000
CONFIG_RR_INTERVAL=200
CONFIG_SCHED_HPWORK=y
CONFIG_SCHED_HPWORKPRIORITY=192
CONFIG_SCHED_WAITPID=y
CONFIG_START_DAY=4
CONFIG_START_MONTH=3
CONFIG_START_YEAR=2013
CONFIG_SYMTAB_ORDEREDBYNAME=y
CONFIG_SYSTEM_NSH=y
CONFIG_TASK_NAME_SIZE=0
CONFIG_TLS_NELEM=4
CONFIG_UART0_SERIAL_CONSOLE=y
CONFIG_VIDEO_FB=y
=== 7. RAMLOG Kconfig & .config check ===
113-            only the urgent messages while the RAMLOG keeps all of them.  See
114-            SYSLOG_CONSOLE_LEVEL.
115-
116:config RAMLOG
117-    bool "RAM log device support"
118-    default n
119-    ---help---
--
128-            details as needed to support logging.
129-
130-if RAMLOG
131:config RAMLOG_NONBLOCKING
132-    bool "RAMLOG non-block reads"
133-    default y
134-    ---help---
135-            Reading from the RAMLOG will never block if the RAMLOG is empty.  If the RAMLOG
136-            is empty, then zero is returned (usually interpreted as end-of-file).
137-
138:config RAMLOG_POLLTHRESHOLD
139-    int "The threshold value of circular buffer to notify poll waiters"
140-    default 1
141-    ---help---
--
337-            SYSLOG device is provided by SYSLOG_DEVPATH. A valid character device (or
338-            file) must exist at this path.  It will by opened by syslog_initialize.
339-
340:config RAMLOG_SYSLOG
341-    bool "Use RAMLOG for SYSLOG"
342-    depends on RAMLOG
343-    default n
--
430-
431-if RAMLOG_SYSLOG
432-
433:config RAMLOG_BUFFER_SECTION
434-    string "The section where ramlog buffer is located"
435-    ---help---
436-            The section where ramlog buffer is located.
437-            The section shall not be initialized on system boot.
438-
439:config RAMLOG_BUFSIZE
440-    int "RAMLOG buffer size"
441-    default 1024
442-    ---help---
CONFIG_ARM_TOOLCHAIN_BUILDROOT_OABI=y
CONFIG_LPC17_40_LCD=y
CONFIG_LPC17_40_SDCARD=y
CONFIG_LPC17_40_SDCARD_DMA=y
CONFIG_LPC17_40_EMACRAM_SIZE=16384
CONFIG_LPC17_40_LCD_VRAMBASE=0xa0010000
CONFIG_LPC17_40_LCD_REFRESH_FREQ=50
CONFIG_LPC17_40_LCD_BACKLIGHT=y
CONFIG_LPC17_40_LCD_TFTPANEL=y
CONFIG_LPC17_40_LCD_BPP16_565=y
CONFIG_LPC17_40_LCD_BACKCOLOR=0x0
CONFIG_LPC17_40_LCD_HWIDTH=480
CONFIG_LPC17_40_LCD_HPULSE=2
CONFIG_LPC17_40_LCD_HFRONTPORCH=5
CONFIG_LPC17_40_LCD_HBACKPORCH=40
CONFIG_LPC17_40_LCD_VHEIGHT=272
CONFIG_LPC17_40_LCD_VPULSE=2
CONFIG_LPC17_40_LCD_VFRONTPORCH=8
CONFIG_LPC17_40_LCD_VBACKPORCH=8
CONFIG_MMCSD_HAVE_CARDDETECT=y
CONFIG_INPUT_DJOYSTICK=y
CONFIG_INPUT_DJOYSTICK_NPOLLWAITERS=2
CONFIG_MMCSD=y
CONFIG_MMCSD_IOCSUPPORT=y
CONFIG_MMCSD_NSLOTS=1
CONFIG_MMCSD_MULTIBLOCK_LIMIT=0
CONFIG_MMCSD_MMCSUPPORT=y
CONFIG_MMCSD_HAVE_WRITEPROTECT=y
CONFIG_MMCSD_SPI=y
CONFIG_MMCSD_SPICLOCK=20000000
CONFIG_MMCSD_SPIMODE=0
CONFIG_MMCSD_IDMODE_CLOCK=400000
CONFIG_MMCSD_SPIRETRY_COUNT=0
CONFIG_MMCSD_SDIO=y
CONFIG_MMCSD_BLOCK_WDATADELAY=260
CONFIG_SYSLOG=y
CONFIG_SYSLOG_DEFAULT_MASK=0xff
CONFIG_SYSLOG_OPENLOG=y
CONFIG_SYSLOG_CRLF=y
CONFIG_SYSLOG_MAX_CHANNELS=1
CONFIG_SYSLOG_DEVPATH="/dev/ttyS1"
CONFIG_SYSLOG_RPMSG_BUFFER_SECTION=""
CONFIG_SYSLOG_DEFAULT=y
CONFIG_MM_REGIONS=3
                             

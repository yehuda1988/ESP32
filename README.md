# ESP32
Here is the complete project documentation for your CYD (Cheap Yellow Display) Control Center. You can copy and save this entire block as a Markdown file (e.g., README.md) to keep it organized.

🚀 Mobileye IT Control Center (ESP32 CYD)
This project turns an ESP32-2432S028 (CYD) touchscreen into a 3-page macro pad and system monitor for macOS using Hammerspoon.

📑 Interface Overview
Page 1: Work Mode (IT Tools)

ITHD: Opens Jira ITHD Queue.

ITLAB: Opens Jira ITLABS Board.

OUTLOOK: Launches/Focuses Microsoft Outlook.

TEAMS: Launches/Focuses Microsoft Teams.

TERM: Opens Terminal.

Page 2: Media Mode (Entertainment)

SPOTIFY: Launches/Focuses Spotify.

PLAY: Play/Pause toggle.

VOL UP / DOWN: Adjusts macOS system volume by 5% increments.

Page 3: System Mode (Security & Maintenance)

LOCK: Instantly locks the macOS screen.

SNIP: Triggers Screen Snipping tool (Cmd+Shift+4).

MUTE: Toggles system mute.

RELOAD: Refreshes Hammerspoon configuration.

🛠️ 1. Hammerspoon Configuration (init.lua)
Path: ~/.hammerspoon/init.lua

Lua
local portName = "usbserial-1330"
if globalDevice then globalDevice:close() end
globalDevice = hs.serial.newFromName(portName)

if globalDevice then
    globalDevice:baudRate(115200)
    globalDevice:callback(function(obj, event, data)
        if event == "received" and data then
            local cmd = data:match("[A-Z_]+")
            if not cmd then return end
            
            -- Work
            if cmd == "ITHD" then hs.urlevent.openURL("https://jira.mobileye.com/projects/ITHD/queues/custom/374")
            elseif cmd == "ITLAB" then hs.urlevent.openURL("https://jira.mobileye.com/secure/RapidBoard.jspa?projectKey=ITLABS&rapidView=2212")
            elseif cmd == "OUTLOOK" then hs.application.launchOrFocus("Microsoft Outlook")
            elseif cmd == "TEAMS" then hs.application.launchOrFocus("Microsoft Teams")
            elseif cmd == "TERM" then hs.application.launchOrFocus("Terminal")
            -- Media
            elseif cmd == "SPOTIFY" then hs.application.launchOrFocus("Spotify")
            elseif cmd == "PLAY" then hs.spotify.playpause()
            elseif cmd == "NEXT" then hs.spotify.next()
            elseif cmd == "VOL_UP" then hs.audiodevice.defaultOutputDevice():setVolume(hs.audiodevice.defaultOutputDevice():volume() + 5)
            elseif cmd == "VOL_DOWN" then hs.audiodevice.defaultOutputDevice():setVolume(hs.audiodevice.defaultOutputDevice():volume() - 5)
            elseif cmd == "MUTE" then 
                local dev = hs.audiodevice.defaultOutputDevice()
                dev:setMuted(not dev:muted())
            -- System
            elseif cmd == "LOCK" then hs.caffeinate.lockScreen()
            elseif cmd == "SNIP" then hs.eventtap.keyStroke({"cmd", "shift"}, "4")
            elseif cmd == "RELOAD" then hs.reload() end
        end
    end)
    globalDevice:open()
end

-- Status Update Timer (Every 5 seconds)
hs.timer.doEvery(5, function()
    if globalDevice and globalDevice:isOpen() then
        local batt = hs.battery.percentage() or 0
        local cpu = math.floor((hs.host.cpuUsage() or {overall=0}).overall)
        local memTotal = hs.host.vmStat().memSize / (1024^3)
        local memFree = hs.host.vmStat().pagesFree * 4096 / (1024^3)
        local ram = math.floor(((memTotal - memFree) / memTotal) * 100)
        local track = "Idle"
        if hs.spotify.isRunning() then
            local current = hs.spotify.getCurrentTrack()
            if current then track = current:sub(1, 10) end
        end
        globalDevice:write(string.format("STAT:%d%% CPU:%d%% R:%d%% | %s\n", batt, cpu, ram, track))
    end
end)
📟 2. ESP32 Arduino Code
Requirements: TFT_eSPI (configured for CYD) and XPT2046_Touchscreen libraries.

C++
#include <SPI.h>
#include <TFT_eSPI.h>
#include <XPT2046_Touchscreen.h>

#define XPT2046_IRQ 36
#define XPT2046_MOSI 32
#define XPT2046_MISO 39
#define XPT2046_CLK 25
#define XPT2046_CS 33

SPIClass touchSPI = SPIClass(VSPI);
XPT2046_Touchscreen touch(XPT2046_CS, XPT2046_IRQ);
TFT_eSPI tft = TFT_eSPI();

int currentPage = 1;

void setup() {
  Serial.begin(115200);
  tft.init();
  tft.setRotation(1);
  touchSPI.begin(XPT2046_CLK, XPT2046_MISO, XPT2046_MOSI, XPT2046_CS);
  touch.begin(touchSPI);
  touch.setRotation(1);
  drawUI();
}

void drawBtn(int x, int y, int w, int h, String txt, uint16_t col) {
  if(txt == "") return; 
  tft.fillRect(x, y, w, h, col);
  tft.drawRect(x, y, w, h, TFT_WHITE);
  tft.setTextColor(TFT_WHITE);
  tft.setTextSize(2);
  tft.setCursor(x + 10, y + (h/2) - 8);
  tft.print(txt);
}

void drawUI() {
  tft.fillScreen(TFT_BLACK);
  if (currentPage == 1) { 
    drawBtn(5, 5, 150, 45, "ITHD", TFT_BLUE);
    drawBtn(165, 5, 150, 45, "ITLAB", TFT_BLUE);
    drawBtn(5, 55, 150, 45, "OUTLOOK", 0x01E0); 
    drawBtn(165, 55, 150, 45, "TEAMS", 0x411A); 
    drawBtn(5, 105, 150, 45, "TERM", TFT_DARKGREY);
    drawBtn(165, 105, 150, 45, "...", TFT_BLACK); 
  } 
  else if (currentPage == 2) { 
    drawBtn(5, 5, 150, 60, "SPOTIFY", TFT_DARKGREEN);
    drawBtn(165, 5, 150, 60, "PLAY", TFT_MAROON);
    drawBtn(5, 70, 150, 60, "VOL UP", 0x03E0); 
    drawBtn(165, 70, 150, 60, "VOL DOWN", 0x03E0);
  }
  else if (currentPage == 3) { 
    drawBtn(5, 5, 150, 60, "LOCK", TFT_RED);
    drawBtn(165, 5, 150, 60, "SNIP", 0xF81F); 
    drawBtn(5, 70, 150, 60, "MUTE", TFT_ORANGE);
    drawBtn(165, 70, 150, 60, "RELOAD", TFT_BLUE);
  }
  drawBtn(5, 160, 310, 40, "NEXT PAGE >", 0x3186);
  tft.drawFastHLine(0, 205, 320, TFT_WHITE);
}

void loop() {
  if (Serial.available() > 0) {
    String input = Serial.readStringUntil('\n');
    if (input.startsWith("STAT:")) {
      tft.fillRect(0, 210, 320, 30, TFT_BLACK);
      tft.setCursor(5, 220);
      tft.setTextColor(TFT_YELLOW); tft.setTextSize(1);
      tft.print(input.substring(5));
    }
  }

  if (touch.touched()) {
    TS_Point p = touch.getPoint();
    int x = map(p.x, 200, 3800, 0, 320);
    int y = map(p.y, 200, 3800, 0, 240);

    if (y > 160) {
      currentPage = (currentPage >= 3) ? 1 : currentPage + 1;
      drawUI(); delay(300);
    } else if (y < 155) {
      bool left = (x < 160);
      if (currentPage == 1) {
        if (y < 50) Serial.println(left ? "ITHD" : "ITLAB");
        else if (y < 100) Serial.println(left ? "OUTLOOK" : "TEAMS");
        else if (left) Serial.println("TERM");
      } else if (currentPage == 2) {
        if (y < 65) Serial.println(left ? "SPOTIFY" : "PLAY");
        else Serial.println(left ? "VOL_UP" : "VOL_DOWN");
      } else if (currentPage == 3) {
        if (y < 65) Serial.println(left ? "LOCK" : "SNIP");
        else Serial.println(left ? "MUTE" : "RELOAD");
      }
      delay(250);
    }
  }
}
📈 Dashboard Data (Bottom Bar)
The ESP32 screen continuously displays:

BAT: Battery Percentage.

CPU: CPU Usage Load.

R: RAM Usage.

Track: Current Spotify track name.

## Frame concerns and others

### The _overrun_ (超載) concern

domain 是否有能力在下一個事件發生前給予回應。

範例:

- John 和 Lucy 輸入指令的速度可能比派對計畫編輯器(party plan editor)處理的速度還要快。
- 轉速脈衝(WheelPulses)的發生頻率可能比里程表微晶片(odometer microchip)重新計算速度和距離的速度還要快。
- 紅綠燈控制器(lights controller)可能會在相同的燈光單元上產生一組 (RPulse, GPulse) 的脈衝對，但兩者之間的間隔過短，導致該單元無法偵測並回應這組脈衝中的第二個脈衝。
- 在本地交通監控(local traffic monitoring problem)問題中，監控電腦(monitor computer)產生資訊輸出事件的速度可能比紙帶式印表機(strip printer)的列印速度還要快。

#### 如果 machine 過快

machine 過快通常好解，加上 delay 讓 machine 慢下來就好。例如紅綠燈控制器需要在每一個脈衝間隔 10 ms，那麼 `RPulse(2); GPulse(2);` 就調整成 `RPulse(2); wait(10ms); GPulse(2);` 即可。

在交通監控的問題，machine 可以在紙帶式印表機準備接受訊息時，才對其發送訊號。

#### overruns 的教戰守

三種可能的解法:

1. simple inhibition: machine 抑制 domain 的 event 速度。
2. ignoring: machine 忽略過快的 domain event。
3. buffering: machine 不及處理的 event 先存入 buffer，等待稍後處理。

### initialisation

### reliability

### identities

### completeness

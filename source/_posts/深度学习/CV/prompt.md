提升画质提示词：
- `masterpiece` 大師級作品
- `best quality` 最佳画质
- `high resolution`高解像度
- `8K` 比4K還要高一級的解像度
- `HDR`高動態範圍(**亮的地方贼亮，暗的地方贼暗，富有层次，细节还贼多**)

光源提示词：
- `bloom` 令原本的光源更亮了，看頭頂及肩膊有變亮的效果
- `soft lighting` 比較柔和的光源，面部有光，背部也有一點
- `hard lighting` 直接打在人物上的光，見到面部輪廓會比較突出
- `backlight` 就是背光，樣子明顯變暗了，肩膊及頭髮有背後來的光源
- `god rays` 另一種背光，由較高的位置射燈式射下來，見到頭頂部份特別亮
- `volumetic lighting` 就像柔光版的背光，整體較暗，有點生化危機的感覺
- `sun light` 比較自然的陽光，連背景的樹都會見到陽光照射
- `studio light` 左右都有光源打在面上，立體感很強，就像廣告照
- `bioluminescent light` 本體在發光，就像螢火蟲一樣的夜光

影子，光追提示词：
- `detailed shadows` 就是加多點精細的影子，鼻子及衣服也有些光影出現
- `intricate tree shadow` 因為是樹林所以就加點錯綜複雜的樹影，看起來更真實
- `raytracing` 見到下巴位置有些衣服反射的光線

照片效果提示词
- `bokeh` `depth of field` 散景及景深，看起來像大光圈的鏡頭拍出來的照片
- `film photography` `film grain` 相片的顆粒感，不過圖太少看不出來
- `glare` 鏡頭炫光，有一點點但不明顯

做用權重控制風力
- `wind` 直接使用 prompt 權重 = 1 預設值
- `(wind:0.5)` 你可以括起來設定 0 - 2 的小數，數字越小效果越弱
- `(wind:1.5)` 數字越大效果越強